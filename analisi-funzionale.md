# Macro-analisi — Sistema di collaudo SAM ERP

## 1. Console Collaudo Server

Componente applicativo centrale, in esecuzione su una macchina dedicata non presidiata da utenti. Sulla stessa macchina è presente **SQL Server** per memorizzare le informazioni necessarie al sistema di collaudo.

Il server:

1. Espone le funzionalità e il canale di comunicazione necessari alle Console Collaudo Client e ai worker.
2. Mantiene il censimento del pool di VM, identificate per nome macchina, e ne pubblica lo stato di disponibilità **dichiarato dagli utenti e registrato nel database**.
3. Assegna **in esclusiva** una VM a un utente quando viene scelta, impedendo assegnazioni contemporanee della stessa macchina.
4. Coordina l’identificativo e lo stato di ciascun collaudo; riceve piano, eventi, risultati, file e richieste di intervento dei worker, rendendoli disponibili ai client. Associa le informazioni a un **ID di sessione di collaudo**.
5. Conserva nel database le informazioni strutturate; riceve i file prodotti dai worker e li salva **fuori dal database**, in una root sulla macchina server organizzata per sessione di collaudo. Nel database conserva gli identificativi e i riferimenti ai file.
6. Espone una **UI amministrativa web accessibile soltanto tramite localhost dalla macchina server**. Questa UI è distinta dalla Console Collaudo Client utilizzata sui PC degli utenti e non è accessibile da remoto.

### UI amministrativa locale del server

Per questa prima versione la UI amministrativa **non richiede un proprio login**: l’amministratore vi accede direttamente dalla macchina server, protetta dall’accesso Windows alla macchina stessa. La UI consente di:

- **Gestire gli utenti della Console Collaudo Client:** creare gli utenti, assegnare una password e reimpostarla quando necessario. L’amministratore comunica personalmente la password all’utente; l’utente può usarla per accedere alla console client, ma **non può cambiarla autonomamente**. Le password non sono conservate in forma leggibile nel database.
- **Consultare un cruscotto minimo:** visualizzare i collaudi in corso, il loro stato, la VM utilizzata e l’utente assegnato.
- **Consultare lo storico dei collaudi:** aprire il resoconto di una sessione conclusa e recuperare i file e le schermate conservati sulla macchina server.

La UI amministrativa ha gli stessi prerequisiti di presentazione della UI client: pagine Razor, HTML, CSS e JavaScript dove necessario; gli aggiornamenti in tempo reale possono usare SignalR. Le funzioni ulteriori di amministrazione saranno valutate successivamente.

**Ogni comunicazione tra Console Collaudo Client e worker passa attraverso il Console Collaudo Server.** Non è prevista una connessione operativa diretta tra client e worker; la connessione RDP dell’utente alla VM è distinta da questo canale.

## 2. Console Collaudo Client

Interfaccia web servita dalla Console Collaudo Server e utilizzabile dal browser sui PC degli utenti. Più utenti possono utilizzarla contemporaneamente, ciascuno con le proprie sessioni web e schede di collaudo. **Per accedere alla console client l’utente inserisce il nome utente e la password assegnati dall’amministratore** tramite la UI locale del server. In questa fase l’utente non può modificare la propria password. Il client permette di:

1. **Definire il collaudo tramite chat.** Dopo l’accesso, un agente LLM su Azure aiuta l’utente a formulare obiettivi, flussi e risultati attesi.
2. **Consultare e scegliere le VM.** L’utente può censire macchine predisposte nel pool, scegliere una VM dichiarata disponibile e **prenotarla** tramite la console. La prenotazione è registrata dal server prima della connessione RDP.
3. **Preparare l’accesso alla VM.** Il client consente di scaricare un file `.rdp` per la VM prenotata; l’utente lo apre per avviare `mstsc` ed effettua il login Windows con le proprie credenziali. La console **non accende né spegne le VM**: per il collaudo si presume che siano già accese. L’eventuale spegnimento è effettuato manualmente al di fuori della console.
4. **Avviare e seguire il test.** Il client invia il piano al server, che lo consegna al worker della VM; mostra in una scheda dedicata a quel collaudo eventi, osservazioni, schermate e risultati ricevuti tramite il server.
5. **Gestire richieste di intervento.** Notifica all’utente le situazioni in cui il worker è in pausa e consente di fornire chiarimenti via chat o di intervenire direttamente nella VM.
6. **Consultare il resoconto finale.** Presenta passaggi svolti, verifiche, dati generati, evidenze, problemi e parti non completate.
7. **Chiudere il collaudo e liberare la VM.** Quando decide di terminare, l’utente dichiara concluso il collaudo dalla console, che inoltra al worker la richiesta di chiusura. Dopo la conferma del worker, l’utente verifica gli eventuali errori e sistema manualmente SAM nella VM se necessario. Poi si disconnette e **indica esplicitamente nella console che la VM è nuovamente disponibile**.

I messaggi ordinari del worker vengono registrati senza interrompere l’utente; richieste di intervento e blocchi producono notifiche visibili.

## 3. Worker esecuzione collaudo

Applicazione installata nella VM Windows 11. **Si avvia automaticamente al login Windows dell’utente via RDP**, nella stessa sessione grafica in cui opererà SAM. Non è un servizio Windows incaricato di interagire con il desktop.

Il worker:

1. Si connette al Console Collaudo Server e segnala di essere pronto **solo dopo aver verificato che la sessione grafica sia utilizzabile**.
2. Riceve il piano di collaudo e le credenziali SAM fornite dall’utente all’avvio, quindi avvia SAM ERP ed effettua il login. Il nome utente SAM rimane nello storico; la password non è persistita dal server, che la tiene solo temporaneamente in memoria per consegnarla al worker. Se il server riparte prima della consegna, l’utente la reinserisce senza creare un nuovo test.
3. Usa il proprio agente LLM su Azure per interpretare finestre, controlli, testi e schermate e decidere le azioni successive. Il piano indica obiettivi e verifiche; non è una sequenza rigida di clic.
4. Dopo ogni azione controlla l’esito effettivo e registra passaggi, messaggi, screenshot, dati generati ed eventuali errori, associandoli all’ID di sessione di collaudo.
5. Invia osservazioni, risultati e file al server, che li conserva per la sessione di collaudo e li rende disponibili al relativo client. I file di collaudo vengono raccolti sulla macchina server, non conservati stabilmente sulle singole VM.
6. Gestisce autonomamente le normali scelte operative previste dal piano. Quando incontra una situazione imprevista che richiede una decisione dell’utente, **sospende le azioni** e invia una richiesta al server, che la notifica al client dell’utente.
7. Concluse le attività previste, trasmette risultati e stato al server. **Quando riceve dalla console la conferma con cui l’utente dichiara terminato il collaudo**, tenta di completare l’invio dei feedback e il logout da SAM e conferma al server la chiusura anche se una di queste operazioni fallisce, segnalando l’errore e ciò che resta da sistemare manualmente. Dopo tale conferma non chiede più chiarimenti all’utente per quel test. Il worker rimane disponibile per eventuali nuovi test finché la sessione Windows in cui è avviato resta aperta.

## 4. Intervento manuale e ripresa

Quando il worker richiede aiuto:

1. Memorizza il punto raggiunto e si mette in pausa, senza continuare a usare mouse e tastiera della VM.
2. L’utente interviene via chat o direttamente nella VM.
3. Nel proprio client l’utente indica **che cosa ha fatto** e quale situazione lascia al worker.
4. Il client trasmette queste informazioni al server, che le inoltra al worker.
5. Prima di riprendere, il worker **osserva nuovamente la schermata e lo stato di SAM**. Confronta ciò che trova con la spiegazione dell’utente e prosegue dal nuovo stato, senza ripetere automaticamente l’azione interrotta.

Se lo stato non è ancora comprensibile, il worker mantiene la pausa e chiede un ulteriore chiarimento.

## 5. Ciclo di vita e disponibilità delle VM

Le VM vengono censite per nome macchina; la console non ne gestisce l’accensione o lo spegnimento. In questa fase **lo stato disponibile/riservata è dichiarato dall’utente e memorizzato nel database**. Il server non verifica automaticamente se la VM è accesa, raggiungibile o se vi siano sessioni Windows aperte o disconnesse. Per usare una VM si presume che sia già accesa e raggiungibile.

Quando un utente prenota una VM marcata disponibile, il server **registra la prenotazione in modo esclusivo prima di rendere disponibile il file `.rdp`**. Non può essere prenotata contemporaneamente da un altro utente, neppure nell’intervallo tra la scelta e il login Windows. Il collaudo inizia solo dopo l’avvio del worker nella sessione RDP e la sua conferma di disponibilità del desktop.

Al termine delle attività si distinguono due possibilità:

- **L’utente vuole continuare sulla stessa VM:** può chiudere il primo collaudo e iniziarne un altro nella sessione Windows già aperta, dopo aver sistemato manualmente eventuali errori residui di SAM. Il worker deve essere di nuovo pronto; la VM **resta riservata e non torna nel pool disponibile**.
- **L’utente rilascia la VM:** dichiara il collaudo concluso dal client; il server inoltra la richiesta al worker e riceve l’esito della chiusura, inclusi eventuali errori nell’invio dei risultati o nel logout da SAM. Se necessario, l’utente sistema manualmente SAM nella VM. Poi si disconnette e **marca manualmente la macchina come disponibile** nella console. Solo questa operazione aggiorna lo stato del pool nel database.

**Disconnettere RDP o chiudere `mstsc` non equivale a fare logout da Windows.** In questa fase il rilascio si basa sulla dichiarazione manuale dell’utente, non su una verifica automatica delle sessioni Windows: chi dichiara disponibile la VM si assume la responsabilità di lasciarla pronta per il successivo utilizzatore. Se dimentica di dichiararla disponibile, la VM resta riservata nel pool. Se la dichiara disponibile pur lasciando condizioni che ne impediscono l’uso, lo stato mostrato potrebbe non corrispondere alla situazione reale.

Un’interruzione imprevista della VM o della comunicazione non deve essere interpretata come «collaudo terminato» né liberare automaticamente la macchina: la console mantiene il test e la prenotazione in uno stato da recuperare, invece di assegnare subito la VM a qualcun altro.

## 6. Collegamento tra Console Collaudo Client, Server e worker

I client e i worker comunicano **esclusivamente tramite il server**. La Console Collaudo Client richiede l’accesso con le credenziali assegnate dall’amministratore; la UI amministrativa, disponibile **soltanto in localhost sulla macchina server**, non ha un login applicativo nella prima versione. Canali e API sono specificati in [contratti-api.md](contratti-api.md). Ogni comunicazione relativa al collaudo identifica almeno:

- **VM** e sessione Windows;
- **utente autenticato della console client**, ID della sessione di collaudo, sessione web interessata e relativo stato;
- messaggio o azione, con ordine degli eventi;
- eventuali screenshot e altri artefatti associati.

Il collegamento deve prevedere conferma di ricezione e riconnessione: se la rete cade temporaneamente, non si devono perdere risultati né eseguire due volte un’azione. Il worker può conservare temporaneamente gli eventi non ancora consegnati e inviarli al server quando la comunicazione torna disponibile. Il server memorizza in **SQL Server** utenti e verificatori delle password non reversibili, VM, prenotazioni, piani, sessioni di collaudo, eventi, stati e riferimenti ai file. Screenshot e altri file ricevuti dai worker vengono salvati **fuori dal database** nella root del server organizzata per ID di sessione; la struttura dei dati sarà definita successivamente.

Il server mantiene distinti i flussi di comunicazione delle diverse VM e delle diverse istanze client. **Una VM esegue un collaudo alla volta; VM diverse possono eseguire collaudi in parallelo.** Ciascun client presenta una scheda dedicata per ogni proprio collaudo attivo.

## 7. Desktop RDP durante il collaudo

L’utente può lasciare `mstsc` **in una finestra piccola e dietro agli altri programmi** sul proprio PC. Il worker legge SAM e acquisisce le immagini direttamente nella VM.

Prima e durante il test il worker verifica che desktop, controlli e screenshot siano disponibili. Se non riesce più a ottenere osservazioni attendibili, sospende il passaggio e invia la notifica al server, che la mostra nel client dell’utente. Le finestre del PC locale non sono usate come fonte per leggere SAM.

## 8. Scelte tecnologiche di massima

- **Console Collaudo Server:** C# e .NET 10; logica applicativa e integrazione con i modelli su Azure; SQL Server installato sulla stessa macchina dedicata. Espone le funzionalità necessarie ai client web e ai worker; i contratti delle API sono in [contratti-api.md](contratti-api.md).
- **Console Collaudo Client e UI amministrativa locale:** interfacce web servite dal server, realizzate con **Razor, HTML, CSS e JavaScript** dove necessario. La UI client richiede le credenziali assegnate dall’amministratore; la UI amministrativa non richiede login ma è accessibile solo in localhost sulla macchina server. **SignalR** è previsto per aggiornamenti e notifiche in tempo reale.
- **Worker:** applicazione desktop **C# e .NET 10** installata in ciascuna VM Windows 11, avviata al login nella sessione RDP. Contiene gli strumenti di interazione con la UI di SAM e l’integrazione con il modello Azure; comunica con il server via HTTP/API.
- **Modelli linguistici iniziali:** `gpt-6-sol` e `gpt-6-luna`, estendibili successivamente. Le capacità effettive dei deployment, inclusa l’eventuale lettura diretta delle immagini, saranno verificate nell’implementazione; il worker dispone comunque di lettura dei controlli e OCR.
- **Persistenza:** utenti e verificatori delle password non reversibili in SQL Server, insieme alle informazioni strutturate collegate all’ID della sessione di collaudo; file conservati nella root della macchina server organizzata per sessione, mai nel database come contenuto binario.
