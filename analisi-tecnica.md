# Analisi tecnica — Prerequisiti del sistema di collaudo SAM ERP

Questo documento raccoglie i prerequisiti tecnici di **Console Collaudo Server**, della sua **UI amministrativa locale**, di **Console Collaudo Client** e del **Worker esecuzione collaudo**. Il flusso e le regole di disponibilità delle VM sono descritti in [analisi-funzionale.md](analisi-funzionale.md); gli approfondimenti sono in [test-life-cycle.md](test-life-cycle.md), [contratti-api.md](contratti-api.md), [tracciato-db.md](tracciato-db.md) e, come riferimento ufficiale per il worker, [analisi-tecnica-worker.md](analisi-tecnica-worker.md).

## 1. Scelte tecnologiche condivise

| Elemento | Prerequisito concordato |
|---|---|
| Linguaggio e runtime server | C# / **.NET 10**, ASP.NET Core MVC |
| Interfaccia client | Pagine web **Razor, HTML, CSS e JavaScript** dove necessario, con grafica e componenti **esclusivamente dal tema aziendale immutabile `ui-theme/`**; nessuna applicazione da installare sui PC degli utenti |
| Interfaccia amministrativa | Pagine web **Razor, HTML, CSS e JavaScript** sullo stesso server, con **lo stesso tema aziendale** della console client, accessibili **esclusivamente in localhost dalla macchina server** |
| Comunicazione in tempo reale verso il browser | **SignalR** per eventi di avanzamento e notifiche; API HTTP/REST per le altre funzionalità |
| Linguaggio e runtime worker | C# / **.NET 10**, applicazione desktop Windows; **non** applicazione MVC e **non** servizio Windows per l’interazione con SAM |
| Sistemi operativi VM | **Windows 11**, con SAM ERP e worker installati |
| Database | **SQL Server** sulla macchina dedicata della Console Collaudo Server |
| Accesso alle interfacce | Utenti client: nome utente/password assegnati dall’amministratore; UI amministrativa locale: nessuna autenticazione applicativa nella prima versione |
| Modelli iniziali | Deployment Azure **`gpt-6-sol`** e **`gpt-6-luna`**, configurabili ed estendibili |
| Client RDP | `mstsc` sui PC degli utenti; file `.rdp` generato dalla console e aperto dall’utente |

La Console Collaudo Server risiede su una macchina dedicata non presidiata. Serve le pagine Razor, ospita la logica applicativa e coordina tutte le sessioni; può essere eseguita come processo server gestito dal sistema operativo, senza necessità di una sessione desktop. Le VM sono censite per nome, **già accese** e gestite manualmente quanto ad accensione e spegnimento: la console non richiede per ora integrazione con l’hypervisor.

### Tema UI web aziendale — vincolo per tutte le interfacce

La cartella [ui-theme/](ui-theme/) è la **fonte immutabile e vincolante** della grafica web per **entrambe** le interfacce: Console Collaudo Client e UI amministrativa locale del server. Pagine di accesso, chat, pool VM, cruscotti, storico e consultazione delle evidenze devono utilizzare **esclusivamente i temi, gli stili, le icone e i componenti grafici predisposti in tale cartella**. Non adottare un design system alternativo, un altro framework CSS di presentazione o componenti grafici con uno stile indipendente dal tema.

Inventario rilevante degli asset forniti:

| Gruppo | Contenuto e impiego previsto |
|---|---|
| `cs-themes/` | Varianti CSS aziendali che comprendono **Bootstrap 4.6.2**: light/dark, grigio, viola e varianti shape v1/v2. Lo stile predefinito per entrambe le UI è **chiaro**, tramite `cs-light-bootstrap.min.css`. |
| `cs-portals.js` | Comportamenti predisposti per sidebar (`cs-main-sidebar`), topbar, toolbar e schede di navigazione responsive. Se incluso, il markup Razor deve contenere gli elementi richiesti dalla sua inizializzazione, in particolare la sidebar. |
| `cs-web-components/` | Elementi già disponibili `cs-avatar`, `cs-avatar-group`, `cs-calendar`, `cs-context-menu` e `cs-file-upload`, da usare quando la funzione richiesta corrisponde al componente. |
| `font-awesome/` | Icone e relativi font già compresi nel tema. |
| `jquery-popper-bootstrap.min.js`, `select2/`, `momentjs/`, `chartjs/` | Script e stili di supporto già forniti, da includere **solo dove servono** ai componenti effettivamente utilizzati. |
| `devexpress/` | Risorse JS e CSS per eventuali controlli DevExpress; `dx.cs.css` usa le variabili del tema aziendale e va associato alla variante DevExpress appropriata se tali controlli vengono adottati. |

I file `.css`, `.js`, font e relative versioni precompresse `.gz`/`.br` **non devono essere modificati, rigenerati o sovrascritti**. In distribuzione possono essere pubblicati come asset statici preservandone contenuto e struttura dei percorsi. Le pagine Razor compongono i componenti e collegano dati/azioni con JavaScript, senza ridefinirne colori, tipografia, icone, forme o comportamento visivo tramite CSS concorrente. Se un requisito UI non trova una rappresentazione adeguata negli asset forniti, la scelta del componente va chiarita prima di introdurre una nuova soluzione grafica.

Il componente `cs-file-upload` è presente negli asset ma **non viene utilizzato**: i client web non possono caricare file al server. L’upload degli artefatti, con file binario e metadati JSON, è riservato al worker autenticato via API.

## 2. Topologia e connettività

```text
PC utente                                 Macchina server dedicata
Browser ── HTTPS / REST + SignalR ──────► UI Console Collaudo Client
                                          │ Console Collaudo Server
Browser sulla macchina server ─────────► │ UI amministrativa (solo localhost)
                                          ├─ SQL Server (utenti, dati strutturati)
                                          ├─ root file per ID collaudo
                                          └─ HTTPS ──► deployment LLM Azure

PC utente ── RDP tramite mstsc ─────────► VM Windows 11 (sessione utente)
                                          ├─ SAM ERP
                                          └─ Worker desktop
                                               ├─ HTTPS / API ──► Console Server
                                               └─ HTTPS ────────► deployment LLM Azure
```

Prerequisiti di rete e instradamento:

1. **Browser degli utenti → server:** i PC degli utenti devono poter raggiungere l’indirizzo HTTPS della Console Collaudo Client, autenticarsi con nome utente/password e usare le funzionalità loro destinate. La connessione SignalR usa il server e deve poter rimanere aperta per aggiornamenti e notifiche; se indisponibile, il browser recupera lo stato persistito alla riconnessione.
2. **PC utente → VM:** il nome o indirizzo inserito nel file `.rdp` deve essere risolvibile e raggiungibile dal PC con il protocollo RDP. Il browser scarica il file, ma non avvia direttamente `mstsc`; l’utente apre il file e inserisce le credenziali Windows nella connessione RDP.
3. **Worker VM → server:** il worker avvia una connessione **in uscita** all’indirizzo HTTPS della console server per registrarsi, ricevere lavoro, inviare risultati e caricare file. Non è richiesto esporre un’API del worker ai PC degli utenti. Il contratto iniziale prevede long polling dei comandi; percorsi e parametri sono in [contratti-api.md](contratti-api.md).
4. **Server e worker → Azure:** entrambi devono poter raggiungere via HTTPS i deployment Azure configurati. Il browser non deve comunicare direttamente con il modello: la console server gestisce la chat di definizione, il worker gestisce l’interpretazione durante il test.
5. **Server → SQL Server e root file:** il processo server deve disporre dell’accesso al database e dei permessi di scrittura/lettura nella root degli artefatti. Il browser accede alle evidenze attraverso il server, non tramite percorsi condivisi delle VM; la proposta di schema fisico è in [tracciato-db.md](tracciato-db.md).
6. **Browser sulla macchina server → UI amministrativa:** l’interfaccia amministrativa deve rispondere solo su un endpoint associato all’interfaccia loopback (`127.0.0.1`/`::1`). Non deve essere esposta sull’indirizzo usato dai PC degli utenti, tramite proxy o inoltro verso la rete. L’amministratore la apre da un browser eseguito **sulla macchina server**; la protezione dell’accesso a tale macchina sostituisce, per ora, un login nell’interfaccia amministrativa.

**RDP è un canale separato dalle API:** collega l’utente alla VM per il login Windows e gli eventuali interventi manuali. Tutti i comandi, messaggi e file scambiati tra Console Collaudo Client e worker attraversano invece la Console Collaudo Server. La finestra `mstsc` può stare piccola e dietro le altre applicazioni dell’utente: le schermate vengono acquisite **all’interno della VM**, non dalla finestra RDP sul PC.

Gli indirizzi effettivi, i certificati HTTPS e le regole di rete devono essere configurati nell’ambiente di distribuzione. Gli endpoint e i formati dei messaggi sono in [contratti-api.md](contratti-api.md) e [test-language.md](test-language.md); i numeri di porta sono configurati nell’ambiente. RDP usa la configurazione della VM e del client `mstsc`.

## 3. Console Collaudo Server — prerequisiti

- Ospitare **ASP.NET Core MVC .NET 10** sulla macchina dedicata. La stessa applicazione serve la UI Razor dei client e la UI Razor amministrativa locale **utilizzando gli asset immutabili `ui-theme/`**, espone le API necessarie a browser e worker e fornisce gli aggiornamenti SignalR. Non è necessario un programma distinto installato su ciascun PC per la console.
- Predisporre **SQL Server locale** con database applicativo e permessi per il processo console. Nel database saranno mantenuti almeno: utenti abilitati e hash delle password, VM censite e stato dichiarato, prenotazioni, sessioni web attribuite agli utenti, ID di sessione di collaudo, piani, stati, eventi ordinati, richieste/risposte di chiarimento, risultati e metadati dei file. Il tracciato proposto è in [tracciato-db.md](tracciato-db.md); restano da realizzare e verificare le migrazioni eseguibili.
- Predisporre l’**accesso utenti della console client** con verifica della password sul server e sessione web associata all’identificativo dell’utente. Operazioni e SignalR della console client devono riferirsi alla sessione autenticata e ai suoi collaudi; la UI amministrativa resta separata e non viene resa raggiungibile attraverso lo stesso ingresso di rete.
- Eseguire la **prenotazione esclusiva** sul server come operazione atomica nel database: due richieste concorrenti non devono riservare la stessa VM. La disponibilità è quella marcata manualmente dagli utenti; **non è prevista una verifica automatica** di accensione, raggiungibilità o sessioni Windows ancora presenti.
- Associare ogni collaudo a un **ID univoco di sessione di collaudo**, indipendente dal nome della VM. Conservare l’ordine degli eventi e confermare la ricezione dei risultati, così che una riconnessione non produca comandi o salvataggi duplicati.
- Il modello usato nella chat deve poter creare un piano interpretabile dal worker, senza ridurlo a coordinate o clic prestabiliti.
- Configurare l’accesso ai deployment Azure, separatamente dalla UI del browser. La scelta concreta delle credenziali Azure e il formato del piano saranno precisati nella fase successiva.
- Esporre dal server download di file `.rdp` per VM prenotate, consultazione dei risultati e accesso ai file di evidenza salvati nella root. Nessuna credenziale Windows deve essere incorporata nel file `.rdp`. Le API browser non prevedono upload: solo il worker può inviare file al server.

### 3.1 UI amministrativa locale del server

- Usare **Razor, HTML, CSS e JavaScript con lo stesso tema `ui-theme/` adottato dalla Console Collaudo Client**; SignalR può aggiornare il cruscotto dei collaudi in corso. L’interfaccia risiede sulla macchina server e si raggiunge solo dal suo browser tramite un indirizzo `localhost`.
- Predisporre un **endpoint di ascolto separato, vincolato a loopback**, per la UI amministrativa e le sue operazioni: non è sufficiente nascondere i collegamenti nelle pagine dei client o controllare soltanto l’indirizzo dichiarato dal browser. Anche controller, API, SignalR e file destinati all’amministrazione devono rimanere esclusivi dell’endpoint locale. Nessuna autenticazione applicativa per tale UI nella prima versione; per le operazioni che modificano dati prevedere comunque le normali protezioni delle richieste web contro invii da pagine estranee.
- Prevedere le operazioni minime per gli utenti della console client: **creazione**, assegnazione e **reimpostazione della password**. Il nome utente deve identificare in modo univoco l’utente registrato nel database. L’amministratore comunica la password assegnata; non esporre funzioni con cui l’utente la modifica autonomamente e non mostrarne il valore dopo la registrazione.
- La password richiesta come “criptata” va conservata tramite **hash con sale e derivazione resistente ai tentativi**, non cifratura reversibile. In .NET si può adottare un componente standard come `PasswordHasher` di ASP.NET Core Identity, con parametri configurati e verificati; al login si confronta la password fornita con l’hash. Una reimpostazione sostituisce l’hash, senza recuperare la password precedente.
- Predisporre un **cruscotto minimo** con collaudi attivi, stato, VM e utente; uno **storico consultabile** per ID/sessione di collaudo, con resoconto ed elenco dei file storicizzati. Il download dei file avviene dal server tramite la root e i riferimenti SQL, senza copiarli nel database.
- È richiesto l’accesso Windows alla macchina server per utilizzare questa UI. Esposizione amministrativa remota, ulteriori ruoli e funzioni di gestione saranno valutati in seguito.

## 4. Console Collaudo Client — prerequisiti

- Browser moderno sui PC degli utenti, in grado di usare le pagine Razor con gli stili e componenti `ui-theme/`, JavaScript, download `.rdp` e la connessione SignalR al server.
- All’ingresso nella console, **richiedere nome utente e password assegnati dall’amministratore**. Il server verifica le credenziali registrate in SQL Server e associa le sessioni web all’utente. Nella prima versione l’utente non può cambiare la password; deve rivolgersi all’amministratore per una nuova assegnazione.
- Mantenere un **identificativo tecnico di sessione web** oltre all’ID dell’utente, per distinguere schede e connessioni anche in caso di ricaricamento della pagina. Lo stato dei collaudi rimane sul server, associato all’utente e al rispettivo ID di sessione di collaudo; i dettagli di persistenza e ripresa della sessione web saranno definiti con il modello dei dati.
- Presentare il pool e permettere la prenotazione, il download del file `.rdp`, la chat di definizione, l’avvio del test, le schede dei collaudi paralleli e il resoconto. SignalR consegna le notifiche e l’avanzamento; lo stato persistito sul server permette di ricostruire la pagina dopo una riconnessione.
- Consentire la **chiusura esplicita**: l’utente dichiara concluso il collaudo, attende l’esito del worker inoltrato dal server, sistema manualmente eventuali problemi residui in SAM, si disconnette dalla VM e marca manualmente la VM come di nuovo disponibile. Un errore di logout o upload viene segnalato ma non impedisce la chiusura confermata dal worker. Non basta chiudere il browser o `mstsc`.
- Su ogni PC dove si usa RDP devono essere disponibili `mstsc` e la raggiungibilità della VM. Il client web non installa né controlla software locale per comandare SAM.

## 5. Worker esecuzione collaudo — prerequisiti

- Installare **SAM ERP e worker .NET 10** in ciascuna VM Windows 11. Il worker deve essere configurato per **avviarsi automaticamente dopo il login Windows effettuato via RDP**, nella sessione grafica dell’utente. Una modalità concreta è un’attività pianificata *all’accesso dell’utente*, configurata per operare soltanto nella sessione interattiva. Un normale servizio Windows nella Session 0 non sostituisce il worker desktop.
- Evitare più worker concorrenti nella stessa sessione desktop; associare il worker attivo a nome VM, sessione Windows e collaudo assegnato. Dopo l’avvio, verificare desktop utilizzabile, presenza della sessione, possibilità di leggere controlli e acquisire un’immagine prima di dichiararsi pronto al server.
- Integrare gli strumenti necessari per **SAM Delphi**: Windows UI Automation, enumerazione e gestione delle finestre con API Win32, input di mouse e tastiera nella sessione, lettura di popup e messaggi, acquisizione screenshot e OCR. I controlli personalizzati e le griglie possono non esporre il contenuto delle celle a UI Automation; il worker deve combinare queste fonti e verificare l’effetto di ogni azione.
- Il **riferimento tecnico ufficiale del worker**, con strumenti, librerie .NET 10, pattern osservati, runtime, dialogo con il server e prove minime, è [analisi-tecnica-worker.md](analisi-tecnica-worker.md).
- Il worker riceve il piano e le credenziali SAM **indicate dall’utente all’avvio**, avvia SAM ed esegue il login nell’applicazione. Nello storico del piano e dell’evento di avvio rimane il nome utente SAM, mai la password: il server la usa solo in memoria volatile per consegnarla al worker, senza persisterla; dopo un riavvio prima della consegna, l’utente la reinserisce. Non includere la password nella chat preliminare, nei log, negli screenshot o nei file di evidenza.
- Il worker usa i deployment Azure per interpretare la UI e scegliere il passo successivo. Le capacità effettive di input immagini dei modelli **non sono ancora verificate**: il funzionamento deve prevedere testo dei controlli e OCR, conservando comunque gli screenshot come evidenze.
- Inviare eventi e file al server riferendoli all’ID del collaudo. In caso di interruzione della rete conservare **temporaneamente** i messaggi e i file non consegnati sulla VM e ritentare il trasferimento; dopo conferma del server, la copia autorevole risiede sulla macchina server, non sulla VM.
- Sospendere le azioni quando serve l’intervento dell’utente o non si può prendere in carico una richiesta prima della chiusura, e notificare il server. Dopo la comunicazione di sblocco, rileggere finestre e schermata prima di riprendere. Dopo la conferma di chiusura dell’utente, non formulare altre richieste: tentare invio e logout da SAM, segnalare gli errori e confermare la chiusura anche se restano interventi manuali. Alla riconnessione verificare tramite il server che il test non sia già terminale prima di eseguire nuove azioni.

## 6. File, database e trasporto delle evidenze

**I file non vengono salvati come contenuto binario in SQL Server.** Il worker li invia al server via HTTPS/API; il server li scrive nella propria root e salva in SQL Server identificativo di collaudo, tipo, nome, dimensione, percorso relativo ed eventuale associazione al passo/evento.

Esempio di organizzazione, con ID generati dal server:

```text
<root-artefatti>/
  <id-sessione-collaudo>/
    screenshot/
    documenti/
    esportazioni/
    altri/
```

- Il server attribuisce i file alla sessione di collaudo verificando identificativo del worker, prenotazione e stato del test; non usa direttamente un percorso assoluto inviato dalla VM come destinazione di scrittura.
- Dimensioni massime, eventuale suddivisione in blocchi, conferma dell’upload e ritrasmissione saranno definite con le API. Un file è disponibile nel resoconto solo dopo conferma della sua ricezione e registrazione da parte del server.
- La root deve disporre di spazio adeguato a screenshot e report contemporanei ed essere compresa, assieme a SQL Server, nelle procedure di conservazione e ripristino. I riferimenti nel database devono continuare a identificare i file dopo il ripristino.

## 7. Stati operativi e verifiche tecniche preliminari

- **VM:** disponibili, prenotate e nuovamente disponibili sono **stati manuali nel database**, con l’utente che ha effettuato la prenotazione. La prenotazione esclusiva è applicata dal server; una VM non si libera automaticamente alla disconnessione RDP o per perdita della connessione del worker.
- **Worker:** può dichiararsi pronto soltanto dopo il login Windows e le verifiche della UI; prima del login non è presente un worker desktop. Il server non scambia l’assenza del worker per una VM automaticamente libera.
- **Collaudo:** la chiusura richiede la conferma dell’utente e l’esito finale del worker, anche quando logout SAM o upload risultano incompleti e richiedono un intervento manuale sulla VM. Lo stato del collaudo è distinto dallo stato manuale disponibile/prenotata della VM.
- **Connettività RDP/schermate:** la [prova-rdp.md](prova-rdp.md) su una macchina fisica collegata via `mstsc` ha confermato lettura e interazione con la finestra RDP piccola e coperta, **non minimizzata**. Ripetere la verifica sulla VM Windows 11 di destinazione prima di farvi affidamento; se l’immagine non è attendibile, il worker mette in pausa il passo e avvisa l’utente tramite il server.
- **Modelli Azure:** verificare accessibilità dei deployment, modalità di input e disponibilità del modello configurato sia dalla macchina server sia dalle VM. Non assumere che il modello possa interpretare direttamente screenshot solo perché il worker è in grado di acquisirli.

## 8. Definizioni rinviate alle fasi successive

La proposta di schema SQL e i contratti REST sono già documentati, mentre **implementazione e verifica** rimangono da svolgere. Restano da precisare parametri operativi di heartbeat/riconnessione, eventuali ampliamenti del piano, durata delle sessioni web, distribuzione delle credenziali dei deployment Azure e dell’identità tecnica dei worker, politica di conservazione dei file e supporto all’input visivo dei modelli. L’accesso utenti della console client e l’isolamento localhost della UI amministrativa sono prerequisiti della prima versione.
