# SAM ERP — Guida tecnica di interazione per l’agente

## 1. Scopo e manutenzione della memoria operativa

Questo file è destinato all’agente autonomo. Conserva tecniche riutilizzabili per leggere e azionare la UI, ridurre i tentativi inefficaci, gestire gli errori e produrre evidenze attendibili.

- Leggerlo prima di ogni sessione SAM. A ogni iterazione, indipendentemente dallo scopo, aggiungere, correggere o rimuovere le indicazioni in base all’esperienza acquisita.
- Non inserirvi percorsi, campi, pulsanti o casi propri di una singola gestione. La descrizione funzionale va nel relativo manuale; le riproduzioni dei problemi nel registro di collaudo e nelle issue.
- Inserire regole funzionali comuni solo quando lo stesso comportamento è stato verificato in **più gestioni**. Un’osservazione isolata non è una regola generale SAM.
- Per una nuova tecnica precisare precondizioni, risultato verificabile e limiti. Distinguere una tecnica sperimentale da una modalità affidabile.
- Conservare principi e procedure, non PID, handle, coordinate di una sessione, nomi di record o screenshot di casi specifici.
- Questo documento consiste nell'apprendimento che l'agente autonomo acqusisce durante le diverse sessioni di iterazione con SAM. Ogni esperienza utile che può essere considerata come suo apprendimento nell'utilizzo ed iterazione con il software deve essere tracciata in modo da essere mantenuta per le sessioni a future.

## 2. Ciclo rapido di una singola interazione

Usare questo ciclo come procedura predefinita:

1. **Identificare** processo, finestra attiva, eventuale modale, controllo e stato abilitato.
2. **Verificare il contesto**: titolo, modalità della form, focus, selezione e operazione prevista.
3. **Agire una volta** sul controllo individuato.
4. **Attendere una postcondizione**: nuova finestra, cambio titolo/testo, messaggio, controllo abilitato o dati aggiornati.
5. **Leggere l’esito**, registrare eventuali errori e scegliere l’azione successiva.

Non avanzare nella sequenza sulla sola base dell’invio riuscito di un click o di un tasto. Un comando inviato non dimostra che SAM lo abbia applicato.

## 3. Preparazione e contesto della sessione

- Verificare eseguibile, processo già in esecuzione, utente e ambiente/database indicati nella UI prima di lavorare.
- Riutilizzare una sessione idonea quando possibile; evitare istanze aggiuntive involontarie.
- Inventariare finestre e schede già aperte prima di creare o modificare dati. Possono influenzare focus, posizioni e disponibilità dei record.
- Verificare sessione grafica disponibile, risoluzione e area di lavoro dei monitor. Tastiera, focus e cattura schermo richiedono un contesto interattivo utilizzabile.
- Usare dati di prova riconoscibili e registrare in `dati_generati.md` ciò che è stato realmente salvato, distinguendolo da input temporanei e operazioni annullate.
- Non memorizzare password nei documenti o negli script persistenti; evitare di includerle nelle schermate.

## 4. Lettura dell’interfaccia: ordine di preferenza

SAM utilizza componenti Delphi personalizzati. Un controllo esposto come pannello o privo di nome non è necessariamente vuoto o inutilizzabile.

### 4.1 UI Automation

- Preferire proprietà e pattern strutturati disponibili: nome, classe, bounds, abilitazione, focus e pattern di valore, selezione, invocazione o scorrimento.
- Non assumere che una griglia esponga celle e conteggi: componenti come `TSMDBGrid` possono esporre solo il contenitore e lo scorrimento.
- Verificare che il controllo appartenga al processo e alla finestra desiderati. Escludere dati di schede sottostanti, anche se Windows le considera visibili.

### 4.2 Enumerazione nativa Windows

- Integrare UI Automation con `EnumWindows`, filtrando PID e visibilità, quindi `AutomationElement.FromHandle`.
- Non usare soltanto `RootElement.FindAll(Children, ProcessId)`: può omettere finestre modali con proprietario.
- Usare `EnumChildWindows`, classe, testo, bounds e stato per ricostruire la gerarchia. Il testo delle etichette può essere disponibile nei controlli nativi anche quando UI Automation non lo espone.
- Gli handle sono transitori e possono essere riutilizzati: risolverli nuovamente dopo chiusure, riaperture e cambi di form.

### 4.3 Tooltip e OCR

- Per pulsanti a icona leggere i tooltip con passaggio del mouse; le finestre di suggerimento possono essere `THintWindow`.
- Un tooltip generico non identifica la funzione: osservare il risultato prima di assegnare un significato al pulsante.
- Usare OCR quando le informazioni non sono esposte dai controlli. Ritagliare e ingrandire il testo piccolo invece di ripetere l’OCR dell’intero desktop.
- Distinguere sempre valori strutturati da testo OCR, soprattutto per codici, date, importi e caratteri simili.
- Non confondere righe visibili, totale estratto, record selezionati, pagine di report e altri contatori.

## 5. Focus, coordinate e geometria

- Preferire azioni sui controlli identificati; usare coordinate come ripiego, ricavate dalle bounds correnti.
- Rileggere la geometria dopo massimizzazione/ripristino, cambio risoluzione, apertura di schede o modifica del layout. Non riutilizzare coordinate fisse indefinitamente.
- Prima di input da tastiera controllare `FocusedElement` e il suo parent. Una modale può intercettare i tasti destinati alla form sottostante.
- Per verificare il bersaglio del mouse usare `GetCursorPos` e `WindowFromPoint`.
- Una modale può rendere disabilitata la finestra principale: il fallimento di un click sullo sfondo non implica che il controllo previsto sia guasto.
- Verificare il risultato di `SetForegroundWindow`: un’applicazione esterna può aver preso il focus.
- Se necessario, usare `BringWindowToTop` e un aggancio temporaneo tramite `AttachThreadInput`; sganciarlo sempre, anche in caso di eccezione. Verificare nuovamente finestra e focus dopo il recupero.
- Per una finestra fuori area: acquisire bounds, area dei monitor e stato prima del recupero; se recuperabile, riposizionarla con `SetWindowPos` su un’area disponibile e rileggere la UI.

## 6. Input affidabile nei controlli Delphi

### 6.1 Campi di testo

- Preferire un pattern UI Automation effettivamente supportato oppure click, selezione del testo, digitazione e uscita dal campo con Tab.
- Nei controlli legati ai dati, `WM_SETTEXT` può modificare solo la visualizzazione senza aggiornare il dataset. Non usarlo come prova di compilazione o salvataggio.
- `WM_CHAR` è una tecnica alternativa sperimentale: verificare eventi, validazione e persistenza prima di considerarla affidabile sul tipo di controllo interessato.
- Dopo il salvataggio riaprire o rinfrescare il record e confrontare i valori. Non dedurre la persistenza dal testo ancora presente nella form.
- Se si usa la clipboard, verificare il focus prima di incollare. Non assumere che una copia applicativa equivalga alla clipboard Windows.

### 6.2 Combo e selezioni

- Preferire il pattern di selezione supportato o la scelta reale dal dropdown con mouse/tastiera.
- `CB_SETCURSEL` può aggiornare la selezione visiva senza lo stato applicativo. Anche notifiche `CBN_SELCHANGE` costruite artificialmente richiedono verifica e possono attivare sequenze non equivalenti all’uso reale.
- Dopo la scelta verificare valore effettivo e conseguenze attese sui controlli dipendenti. Non basarsi soltanto sull’indice programmato.
- Una scelta può sostituire il controllo di input con uno nuovo nella stessa posizione: rileggere classe, handle, visibilità e focus dopo la selezione; non inviare testo all’edit precedente nascosto.
- Distinguere controllo corrente, elemento selezionato e selezione multipla; controllare il criterio effettivo prima di azioni su più record.

### 6.3 Pulsanti e scorciatoie

- Usare InvokePattern se supportato e verificato. In alternativa, click reale o messaggi Windows compatibili con il controllo.
- `SendMessage(BM_CLICK)` può restare in attesa quando il pulsante apre una modale. Preferire `PostMessage` con attesa e verifica delle finestre quando questo rischio è presente.
- `SendKeys ENTER` non chiude necessariamente un messaggio Delphi: se persiste, individuare e azionare il pulsante esplicito.
- Le combinazioni con Ctrl possono attivare scorciatoie della form invece dell’edit se il focus è errato. Controllarlo prima di Ctrl+A, Ctrl+C o Ctrl+V.
- Non attribuire a SAM un difetto basato solo su input artificiale, notifica errata o focus sbagliato: confrontare con un percorso reale.

## 7. Sincronizzazione e ottimizzazione

- Preferire attese basate sullo stato a pause fisse lunghe: controllare titolo, finestre, abilitazione e indicatori di caricamento.
- Un titolo transitorio, `TViewLoading` o un monitor di elaborazione non rappresentano ancora la form finale e non sono automaticamente errori.
- I messaggi possono apparire dopo il primo controllo: quando l’azione è asincrona, rileggere lo stato prima di procedere.
- Un timeout del comando di automazione non dimostra un blocco di SAM. Verificare processo, modali e postcondizione prima di ripetere l’azione.
- Evitare doppi click o reinvii automatici per operazioni potenzialmente già eseguite, in particolare salvataggi e generazioni di dati.
- Riutilizzare gli strumenti tecnici di enumerazione, cattura e OCR; aggiornarli quando si individua un limite generale.
- Leggere solo la zona o i controlli necessari all’azione corrente; eseguire un inventario completo dopo cambi importanti di contesto.
- Eseguire batch solo con contesto stabile e postcondizioni controllabili. Se si apre una nuova form o un messaggio, interrompere il batch e rivalutare.
- Individuare gli accessi equivalenti e provare la stessa funzione una volta; riaprirla solo per una variante significativa, una verifica o una riproduzione.
- Non ripetere tentativi identici che producono lo stesso esito: cambiare una precondizione alla volta e registrare la differenza.
- Distinguere form di griglia e form di documento: toolbar con icone simili possono avere funzioni e coordinate diverse. Dopo ogni cambio risolvere di nuovo tooltip e controllo prima dell’azione.
- Nei processi che producono più anteprime, chiudere la finestra corrente può soltanto avanzare al documento successivo. Cercare un comando esplicito di interruzione dell’intero processo e verificarne la conferma, invece di ripetere chiusure indefinitamente.
- Nei popup il punto d’apertura cambia la posizione delle voci; nei menu a cascata verificare il sottomenu visibile prima del click. Non assumere che coordinate di una precedente apertura siano ancora valide.

## 8. Gestione degli errori e decisione operativa

### 8.1 Tracciare sempre prima di intervenire

Per ogni messaggio inatteso registrare:

- data/ora, ambiente, eseguibile/build se acquisita, utente;
- finestra e percorso corrente, operazioni precedenti e dati necessari alla riproduzione;
- testo esatto, titolo, pulsanti disponibili e stato della UI;
- screenshot del messaggio e del contesto utile;
- effetto sui dati e sul flusso, azione di recupero tentata e suo risultato.

Non chiudere automaticamente messaggi non ancora letti. Esaminare anche **Dettagli** quando disponibile e utile. Tenere una traccia anche quando il caso non merita una issue.

### 8.2 Classificare e agire

| Tipo | Come riconoscerlo | Azione dell’agente |
|---|---|---|
| **Operativo / validazione** | Campi obbligatori, incongruenze, formato o prerequisiti non rispettati, richieste di conferma | Comprendere il messaggio e il campo/contesto interessato; correggere o completare dati coerenti e proseguire sul percorso più adatto |
| **Non bloccante** | Messaggio chiudibile e possibilità di riprendere o continuare altre prove | Registrare, acquisire evidenze, chiudere/recuperare e tentare di proseguire; verificare lo stato e gli eventuali effetti parziali |
| **Bloccante** | Nessun recupero praticabile, processo/UI non utilizzabili o impossibilità di continuare in modo controllato | Sospendere le attività, preservare il punto del problema e documentare l’ultima sequenza; non riavviare o forzare la chiusura perdendo le evidenze |
| **Automazione / infrastruttura** | Focus errato, target non valido, sessione grafica indisponibile, timeout dello strumento | Verificare SAM separatamente dallo strumento, correggere l’interazione e confrontare con uso reale prima di attribuire il problema all’applicazione |

- Una validazione normale non è automaticamente un bug. Segnalare se il messaggio è incoerente, impedisce una correzione valida o rivela un difetto riproducibile.
- Per errori operativi usare dati di prova appropriati; non ignorare il significato delle conferme per avanzare a ogni costo.
- Se una funzione è impedita ma SAM resta utilizzabile, documentare il ramo incompleto e continuare con gli altri percorsi disponibili.
- Prima di ripetere un salvataggio controllare la persistenza: un avviso può accompagnare effetti parziali e non garantisce l’annullamento dell’intera operazione.
- Verificare il numero di record anche dopo una copia o un’uscita con annullamento: operazioni intermedie possono avere già prodotto effetti persistenti. Non assumere che Annulla equivalga a rollback di tutte le azioni precedenti.
- Prima di considerare un record bloccato, controllare se è già aperto in un’altra scheda; se possibile liberarlo chiudendo normalmente la scheda. Non intervenire direttamente nel database come scorciatoia di recupero.
- Se il recupero peggiora la situazione o lo stato non è più controllabile, riclassificare il caso come bloccante e sospendere.

### 8.3 Valutare la segnalazione

- Dopo aver tracciato il caso, decidere se aprire/aggiornare una issue: errore applicativo riprodotto, comportamento da chiarire rispetto ai requisiti, problema di usabilità o infrastruttura rilevante.
- Esplicitare ciò che è osservato e ciò che resta da verificare. Non inventare cause, query, stack o versione del software.
- Conservare nel registro i casi esclusi dalla segnalazione e la motivazione; conservare qui solo la tecnica generale appresa.

## 9. Screenshot ed evidenze attendibili

- `CopyFromScreen` acquisisce il desktop effettivamente visibile: verificare finestra in primo piano, sovrapposizioni e titolo prima di attribuire la cattura a SAM.
- Salvare schermate prima del recupero, dopo un passaggio significativo e dopo la verifica dell’esito. Per problemi fuori schermo integrare bounds e topologia dei monitor; una cattura dopo lo spostamento non dimostra la posizione iniziale.
- Usare nomi descrittivi e una sequenza riconoscibile. Verificare il contenuto effettivo, non soltanto il nome file.
- Conservare originali utili e ritagli leggibili del messaggio; evitare ritagli che eliminano il contesto necessario.
- Collegare le immagini nei documenti con percorsi relativi e verificare che i file esistano.
- Se il modello in uso non interpreta direttamente immagini o PDF, usare controlli e OCR e dichiarare il metodo di lettura; non affermare di aver validato visivamente contenuti non letti.
- Un output presente sul filesystem non dimostra successo: verificare formato, contenuto e completezza, soprattutto dopo messaggi o interruzioni.
- Per valutare precisione dei dati costruire un esempio con risultato atteso calcolabile e confrontare griglia, form e riapertura. Valori di prova, campi vuoti o descrizioni insolite preesistenti non sono difetti senza una regola o un risultato atteso verificabile.

## 10. Issue: deduplicazione, contenuto e print-screen

### Alla rilevazione di un caso sospetto, prima di segnalarlo

1. Tracciare l’osservazione e **consultare prima le issue GitHub già chiuse**, usando messaggio, sintomo e parole chiave del comportamento. Consultare anche `issue_list.md` e `issue_skipped.md` come indici locali, senza considerarli sostitutivi dello stato remoto.
2. Per ogni corrispondenza leggere **descrizione, commenti, motivazione di chiusura e spiegazione conclusiva**. Il solo titolo o lo stato chiuso non bastano: la chiusura può indicare comportamento previsto, correzione, duplicazione o altra decisione.
3. Confrontare precondizioni, contesto, dati, risultato osservato e ambito della spiegazione. Se il comportamento è già stato valutato come **non erroneo** e il nuovo caso coincide, **non segnalarlo nuovamente**. Applicare la spiegazione acquisita per proseguire i test.
4. Se emergono casistiche diverse, variazioni del fenomeno o una possibile regressione, documentare esplicitamente le differenze rispetto alla issue chiusa. La precedente spiegazione non esclude automaticamente una nuova segnalazione fuori dal suo ambito. Una issue chiusa come corretta non equivale a un comportamento dichiarato previsto.
   - Se **l’errore già segnalato si presenta ancora dopo che un utente ha chiuso la issue**, verificare che sia lo stesso difetto e **riaprire la issue esistente**, senza crearne una duplicata. Aggiungere un commento che dichiari esplicitamente la persistenza dell’errore, con ambiente/versione, passaggi, dati rilevanti, messaggio, print-screen necessari ed eventuali differenze rispetto alla prova precedente o alla soluzione indicata.
   - Prima della riapertura leggere la motivazione di chiusura: se il comportamento coincidente era stato spiegato come corretto, applicare il punto 3 e non riaprire sulla sola ricorrenza. Documentare eventuali nuove evidenze che contraddicono la spiegazione o dimostrano una casistica diversa.
   - Dopo la riapertura verificare commento e stato su GitHub e aggiornare `issue_list.md`, il registro di collaudo e l’eventuale voce in `issue_skipped.md`, conservando la cronologia.
5. Consultare quindi le issue **aperte**, leggendo anche le risposte, e applicare il flusso di verifica seguente. Accorpare le occorrenze dello stesso fenomeno nella segnalazione pertinente. Separare solo con differenze documentate; un testo identico non dimostra da solo una stessa causa tecnica.

### Issue aperta già esistente: risposta e verifica

Una risposta dell’utente o del manutentore su una issue ancora aperta è **un’indicazione da verificare**, non una prova che il problema sia risolto. Non creare una nuova issue per la stessa problematica.

1. Leggere descrizione e conversazione completa, identificando la risposta pertinente più recente e gli eventuali test già svolti dall’agente. Non confondere un commento automatico o una semplice conferma di ricezione con una spiegazione operativa.
2. Decidere in base allo stato:

| Situazione | Azione |
|---|---|
| **Nessuna spiegazione o indicazione verificabile ricevuta** | Non segnalare nuovamente: la issue è ancora in attesa di elaborazione. Censire l’occorrenza in `issue_skipped.md` con URL, stato aperto, data e motivo «già segnalata, in attesa di risposta». Proseguire i test secondo le regole di gestione degli errori. |
| **Risposta con spiegazione, correzione o istruzioni** | Leggere e seguire le indicazioni, verificandole direttamente in SAM con le precondizioni richieste. Confrontare risultato atteso e osservato sul percorso originario e sulle varianti pertinenti. |
| **Indicazioni verificate e risolutive** | Aggiungere un commento con istruzioni seguite, ambiente/versione, passi, esito ed evidenze; quindi **chiudere la issue**. Se la spiegazione conferma un comportamento previsto, esplicitarlo; se una correzione elimina il difetto, documentare la verifica della correzione. |
| **Indicazioni verificate ma non risolutive** | Lasciare la issue aperta e **aggiungere un commento** con prove effettuate, risultato osservato, messaggi, screenshot necessari e differenze rispetto alla risposta ricevuta. Centralizzare qui le nuove osservazioni. |
| **Verifica non completabile o risposta ambigua** | Non chiudere e non dichiarare la risoluzione. Commentare il prerequisito mancante, il punto impedito o il chiarimento necessario; registrare la verifica incompleta e mantenere il caso aperto. |

3. Prima di commentare o chiudere rileggere lo stato remoto e gli ultimi commenti per evitare decisioni basate su informazioni superate. Non ripetere una verifica già documentata senza una nuova risposta, una variazione dell’ambiente o un’altra ragione concreta.
4. Dopo l’intervento verificare che commento e stato siano stati aggiornati su GitHub; aggiornare `issue_list.md`, il registro di collaudo e l’eventuale voce di `issue_skipped.md`.
5. La chiusura richiede un esito realmente verificato: assenza di nuovi messaggi, mancata riproduzione occasionale o semplice presenza di un commento non bastano. Registrare eventuali limiti della prova.

### Censimento dei casi non segnalati: `issue_skipped.md`

- Censire **ogni problematica per cui non viene creata una nuova issue perché già trattata**, includendo comportamenti chiariti nelle issue chiuse e occorrenze di issue aperte ancora senza risposta. Creare il file al primo caso se non esiste.
- Per ogni caso registrare: data/sessione, ambiente e versione se nota, comportamento sospetto, percorso/precondizioni e dati rilevanti, URL/numero e stato della issue consultata, riferimento alla risposta pertinente quando disponibile, spiegazione della chiusura oppure stato di attesa/verifica e motivo per cui non viene creata una nuova segnalazione.
- Indicare la decisione presa (nessuna nuova issue), le evidenze disponibili e come sono proseguiti i test. Distinguere la conclusione del manutentore dalle deduzioni dell’agente.
- Per ricorrenze equivalenti aggiornare il caso censito con nuove date e punti di accesso; conservare la tracciabilità delle osservazioni senza creare voci identiche inutili.
- Se successivamente emergono differenze che giustificano una segnalazione, aggiornare il caso con le differenze e il collegamento alla issue pertinente. Non eliminare la motivazione della decisione precedente.
- Se arriva una risposta su una issue aperta precedentemente censita come in attesa, seguire il flusso di verifica sopra e aggiornare la voce con esito, commento pubblicato ed eventuale chiusura. Conservare la cronologia della decisione.
- Se lo storico remoto non è consultabile o la spiegazione è ambigua, registrare la verifica come incompleta: non classificare il comportamento come già chiarito sulla sola base di una somiglianza.

### Contenuto minimo

- Limitare la descrizione alla **casistica specifica dell’errore** e ai dettagli utili per identificarlo e riprodurlo in SAM. Usare contenuto preciso, sintetico e tecnico; evitare cronache, inventari di prove e informazioni estranee.
- Struttura preferita: **problema**, **ambiente/precondizioni rilevanti**, **passaggi numerati**, **risultato atteso e osservato**, **screenshot**. Aggiungere messaggio esatto, ripetibilità, impatto o recupero soltanto quando aiutano a comprendere il caso.
- Includere i dati minimi necessari alla replica. Distinguere fatti osservati, input automatizzato e ipotesi da verificare senza dilungarsi sui dettagli dello strumento non pertinenti.
- **Non citare il giro di collaudo**, la sessione numerata o la storia della redazione: presentare il problema come una nuova segnalazione autonoma. Per commenti di verifica/riapertura descrivere la prova e l’esito pertinenti, conservando la cronologia estesa nel registro locale.
- Citare altre issue **solo se strettamente collegate** e se il collegamento aggiunge valore alla diagnosi, alla replica o al tracciamento. Non elencare issue consultate soltanto per dimostrare la deduplicazione: tale verifica resta nel censimento locale.

### Print-screen caricati nella issue

- **Caricare sempre nella issue stessa le immagini necessarie**, in modo che chi la consulta le veda senza accedere alla copia di lavoro. I PNG non verranno committati nel repository: percorsi locali e link a file del repository non sono modalità di consegna degli screenshot.
- GitHub consente il caricamento diretto delle immagini nell’editor della descrizione o di un commento tramite selezione file, trascinamento o incolla. Utilizzare il browser autenticato e, se disponibile, l’automazione della UI per effettuare il caricamento.
- La login di GitHub CLI non garantisce una sessione browser autenticata. `gh issue create/edit/comment --body-file` pubblica il testo, **non carica i PNG** citati nel Markdown. Non considerare questa operazione un upload di allegati.
- Acquisire messaggio leggibile, contesto della form e passaggi decisivi; includere Dettagli o stato successivo quando rilevanti. Evitare immagini ridondanti o estranee.
- Attendere il completamento dell’upload e inserire nel testo l’immagine con l’URL dell’allegato restituito da GitHub. Non inventare URL e non usare immagini inline codificate come sostituto del caricamento.
- Pubblicare e riaprire la issue: verificare che le immagini siano effettivamente visibili e che i collegamenti puntino agli allegati caricati. Censire URL della issue, allegati e stato in `issue_list.md`.
- Se il browser autenticato o il caricamento non sono utilizzabili, tentare di individuare il prerequisito mancante e comunicarlo esplicitamente. Registrare il caricamento come **incompleto**; una citazione del file locale non soddisfa la richiesta e non deve essere presentata come evidenza consegnata.

**Applicazione della regola screenshot:** vale per le prossime segnalazioni. Non modificare retroattivamente le issue già inserite per adeguarle a questa disposizione.

## 11. Separazione dei risultati

| Documento | Cosa conservare |
|---|---|
| `linee_guida_agente.md` | Tecniche generali, limiti degli strumenti, procedure di recupero e ottimizzazione; pattern comuni verificati in più gestioni |
| Manuale operativo della gestione | Funzioni, campi, percorsi e istruzioni per l’utente; niente issue, stack o cronaca del collaudo |
| `linee_guida_manuale.md` | Regole di scrittura da leggere e applicare prima di creare o aggiornare un manuale utente |
| Registro di collaudo | Sequenze eseguite, esiti, copertura, errori e osservazioni specifiche |
| `issue_list.md` e issue GitHub | Censimento e segnalazioni tecniche centralizzate |
| `issue_skipped.md` | Occorrenze senza nuova segnalazione perché già trattate: spiegazioni di issue chiuse, issue aperte in attesa, verifiche delle risposte e relativi esiti |
| `dati_generati.md` | Dati effettivamente salvati, modifiche, output, prove annullate e residui |

Prima di concludere una sessione verificare stato dell’applicazione, dati residui, copertura reale, file prodotti e collegamenti. Riportare i limiti senza trasformare un percorso soltanto aperto in un test concluso.
