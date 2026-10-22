# Linguaggio dei messaggi del collaudo SAM ERP

Lo schema formale è [test_language_schema.json](test_language_schema.json), versione **1.0** (JSON Schema Draft 2020-12). Definisce i messaggi tra **Console Collaudo Client**, **Console Collaudo Server** e **Worker esecuzione collaudo**; la semantica dei flussi è descritta qui. Le firme REST sono in [contratti-api.md](contratti-api.md). Il server inoltra richieste dell’utente, ma le azioni su SAM sono decise dall’agente Azure interno al worker.

## Regole comuni

- **Ogni messaggio del collaudo** è un oggetto JSON con `schemaVersion`, `messageId`, `sessionId`, `userId`, `vmId`, `origin`, `type`, `occurredAt` e `payload`. `sessionId` è l’ID univoco del **collaudo**, non della connessione del browser; è assegnato prima di `test.start` e non cambia quando la pagina viene ricaricata o il test riprende dopo un chiarimento.
- `userId` è l’identificativo dell’utente della console **già autenticato**; `vmId` identifica la VM prenotata. Il server verifica associazione di utente, VM e collaudo invece di fidarsi dei soli valori nel JSON ricevuto. Il worker riceve gli identificativi associati alla sessione assegnata.
- `messageId` è un UUID generato dal mittente. Un ritentativo per mancata risposta di rete riutilizza **lo stesso `messageId`**; una nuova azione genera un altro ID. `occurredAt` è una data/ora UTC ISO 8601 con `Z`. `inReplyTo`, se presente, identifica un messaggio precedente correlato.
- Il server registra il messaggio e aggiunge `serverSequence`, progressivo **per sessione di collaudo**, alla copia inoltrata e memorizzata. Client e worker non lo inviano: il server rifiuta `serverSequence` nelle richieste in ingresso. La sequenza consente di recuperare gli eventi mancanti dopo una riconnessione.
- `origin` rimane `client` o `worker` anche quando il server inoltra il messaggio all’altro componente. Gli avvisi mostrati dal server al browser rappresentano l’evento originale, non un secondo evento generato dal server. I tipi ammessi e il relativo mittente sono vincolati nello schema.
- REST/HTTP trasporta comandi ed eventi senza conservare lo stato autorevole nella connessione. SQL Server conserva collaudi, stati e ordine dei messaggi. SignalR **non è stateless come trasporto**: viene usato soltanto per notificare tempestivamente il browser; dopo una disconnessione la pagina recupera lo stato persistito tramite REST.

Esempio di involucro comune, con un `payload` specifico del tipo:

```json
{
  "schemaVersion": "1.0",
  "messageId": "f94e6c0a-a969-46a7-8589-825ee423d237",
  "sessionId": "7fb64870-4059-4f0d-94b9-729e5b8469ec",
  "userId": "utente-12",
  "vmId": "VM-SAM-01",
  "origin": "worker",
  "type": "step.event",
  "occurredAt": "2026-10-22T10:15:00Z",
  "payload": {
    "stepId": "apertura-opportunita",
    "status": "completed",
    "description": "Aperta la scheda dell'opportunità",
    "observations": ["Stato: Proposta", "Totale: 200,00"],
    "logs": [{ "level": "info", "text": "Dati riletti dalla form" }],
    "artifactIds": ["fae18c37-a61c-4735-b0a9-09641a537638"]
  }
}
```

## Tipi di messaggio

| `type` | Mittente → destinatario attraverso il server | Scopo |
|---|---|---|
| `test.start` | Client → server → worker | Avvio del test con piano e dati di accesso a SAM; lo storico conserva il nome utente, non la password |
| `command.ack` | Worker → server → client | Presa in carico o rifiuto di un comando, **non** prova che sia stato già eseguito |
| `step.event` | Worker → server → client | Stato di un passaggio, osservazioni, log e riferimenti a file |
| `worker.notice` | Worker → server → client | Informazione, errore, blocco o domanda che richiede intervento |
| `user.reply` | Client → server → worker | Risposta a una domanda, nuova istruzione o estensione del test |
| `test.activities_completed` | Worker → server → client | Fine delle attività richieste; il collaudo rimane aperto |
| `test.close_requested` | Client → server → worker | Conferma di conclusione da parte dell’utente e richiesta di logout da SAM |
| `test.closed` | Worker → server → client | Chiusura confermata con esito effettivo di logout e consegna, anche con errori residui |
| `test.interrupt_requested` | Client → server → worker | Ordine prioritario di fermare un test non concluso |
| `test.interrupted` | Worker → server → client | Resoconto parziale e situazione lasciata in SAM dopo l’interruzione |

### Avvio: `test.start`

L’utente e l’agente della console redigono insieme il piano. **Il testo in linguaggio naturale** viene riportato in `payload.plan.description`; non occorre predefinire ogni click. Il client non può caricare file né inserire `artifactIds` nel piano o in `user.reply`. Il worker interpreta il piano osservando SAM e verifica i risultati durante l’esecuzione.

```json
{
  "plan": {
    "description": "Apri Scheda opportunità; crea una trattativa per il contatto 000219, aggiungi una riga con quantità 2 e prezzo 100, salva, riapri e verifica i totali."
  },
  "samLogin": {
    "userName": "utente-sam",
    "password": "<password fornita durante la predisposizione>"
  }
}
```

Questo è il **solo payload** esemplificativo; viene inserito nell’involucro con `type: "test.start"` e `origin: "client"`. Il modello LLM **non è un campo del messaggio**: inizialmente è configurato nel worker; una futura scelta automatica del modello non richiederà di modificare il piano comunicato dall’utente. La forma dettagliata del piano oltre a `description` sarà definita nel suo contratto successivo.

`samLogin` nel comando diretto al worker contiene nome utente e password. Il `PayloadJson` storico di `test.start` ha esattamente la forma `{ "plan": { "description": string }, "samLogin": { "userName": string } }`: non contiene la password. Il server conserva quest’ultima **solo in memoria volatile**, eliminandola dopo l’ack (`accepted` o `rejected`) di consegna; non la scrive su SQL Server, file o log. Se il server riparte prima della consegna, il browser la reinserisce tramite l’API dedicata senza modificare il test storico. Se un chiarimento richiede nuove credenziali, il worker le preleva attraverso l’API a lui riservata prima di confermare la presa in carico della risposta. `step.event`, notifiche e screenshot non contengono la password. Lo schema del messaggio completo di trasporto non si applica alla proiezione storica sanificata: il server valida questa forma separatamente e ne rifiuta proprietà aggiuntive o segreti.

Il server registra e consegna il comando alla VM prenotata. Il worker verifica di essere pronto nella sessione Windows dove gira SAM, invia `command.ack` con `acknowledgedMessageId` uguale al `messageId` dell’avvio, poi procede. Un `status: "accepted"` significa **presa in carico**, non riuscita del login o dell’intero test. Prima della conferma di chiusura, un ack `rejected` include `reason` ed è seguito da `worker.notice` con `requiresResponse: true` e `requestId`: l’utente modifica la richiesta o interviene nella VM, quindi risponde con `clarification`. Il worker rilegge lo stato e riprova nello stesso test senza rieseguire automaticamente il vecchio comando; se il problema persiste, l’utente interrompe. Gli ack valgono anche per `user.reply`, `test.close_requested` e `test.interrupt_requested`, ma la chiusura confermata non viene rifiutata per errori operativi: questi vengono riportati in `test.closed`.

### Passaggi ed evidenze: `step.event`

Il worker può produrre più eventi per lo stesso `stepId`: `started`, poi `completed` o `failed`; `skipped` indica un passaggio non eseguito. Ogni evento può contenere `description`, `observations`, `logs` con livello `info`/`warning`/`error` e `artifactIds` dei file già caricati. L’ID del passaggio è assegnato dal worker per collegare osservazioni e file alla stessa attività, anche se il piano è scritto in linguaggio naturale.

Un evento `failed` registra il risultato del passaggio: **non implica automaticamente la fine del collaudo**. Il worker decide se può proseguire oppure invia `worker.notice` per richiedere aiuto.

### Avvisi e richieste: `worker.notice` e `user.reply`

`worker.notice` contiene `severity` (`info`, `warning`, `error`, `blocked`), testo e `requiresResponse`. Può includere uno `stepId` e riferimenti agli screenshot. Se `requiresResponse` è `true`, deve includere **`requestId` e `question`**: il worker sospende le azioni che dipendono dalla risposta.

```json
{
  "severity": "blocked",
  "text": "È richiesta una scelta non prevista nel piano.",
  "requiresResponse": true,
  "requestId": "b89c23a7-e7c4-470b-b67a-026814ac741e",
  "question": "Quale delle due modalità di vendita devo utilizzare?"
}
```

Il client mostra la notifica tramite SignalR. L’utente può rispondere via chat oppure intervenire nella VM e spiegare ciò che ha fatto. Il payload di `user.reply` contiene **`kind` e `text`**:

- `clarification`: risposta a una domanda precisa, anche dopo il rifiuto di prendere in carico una richiesta prima della chiusura; `requestId` è obbligatorio e identifica il `worker.notice` ancora in attesa. `manualActions` e `resultingSituation` descrivono un eventuale intervento nella VM. Dopo la risposta il worker rilegge la UI prima di agire. Eventuali modifiche al piano si esprimono nel testo della risposta, senza alterare la fotografia iniziale.
- `instruction`: nuova informazione o indicazione spontanea **durante la fase `Esecuzione`**, senza necessità di una domanda del worker. Il worker la integra a un punto controllabile, evitando di interrompere alla cieca un’operazione in corso. Se il worker è in attesa di risposta a un `requestId`, occorre invece una `clarification`: una semplice `instruction` non chiude quella domanda.
- `extension`: richiesta di ulteriori prove **dopo `test.activities_completed`**, prima della richiesta di chiusura. Riattiva la stessa sessione di collaudo e conserva tutta la cronologia precedente.

Esempio di istruzione spontanea, senza `requestId`:

```json
{
  "kind": "instruction",
  "text": "Controlla anche che il totale sia aggiornato dopo aver riaperto il documento."
}
```

Per una risposta a una domanda, il payload può invece contenere `requestId`, `manualActions` e `resultingSituation`. Ogni `user.reply` mantiene `sessionId` e `userId` nell’involucro ed è registrato nella conversazione ordinata dal server.

### Attività terminate, approfondimenti e chiusura ordinaria

Quando il worker ha eseguito le attività richieste, invia `test.activities_completed` con `outcome` (`passed`, `partial`, `failed`), `summary`, eventuali `artifactIds` e `completionNumber`, inizialmente **1**. Il messaggio **non** fa logout da SAM e **non** libera la VM.

Da qui l’utente può:

1. Inviare `user.reply` con `kind: "extension"`. Il server conserva il nuovo messaggio; il worker riapre l’esecuzione **nello stesso `sessionId`**, verifica nuovamente la UI, svolge gli approfondimenti ed emette un altro `test.activities_completed` con `completionNumber` successivo.
2. Dichiarare concluso il collaudo con `test.close_requested` e `payload.confirmedByUser: true`. Il worker riceve la richiesta, completa la consegna dei risultati, effettua il logout da SAM e risponde con `test.closed`.

La conferma dell’utente è definitiva: da `test.close_requested` il worker **non pone più domande**. Tenta logout e upload; se qualcosa fallisce, invia `worker.notice` di gravità `error` con `requiresResponse: false` e poi `test.closed` con i valori reali di `samLogoutCompleted`, `uploadsCompleted`, `manualInterventionRequired`, `summary` ed eventuali `missingEvidence`. Il test diventa `Completata` anche in presenza di errori; il resoconto segnala ciò che manca. Se il server è irraggiungibile, il test resta in chiusura fino alla conferma ricevuta dal server o al recupero manuale. Prima di disconnettersi da RDP e dichiarare disponibile la VM, l’utente verifica e sistema manualmente SAM se richiesto; il rilascio del pool non è un messaggio del test.

### Interruzione prima della chiusura

L’utente può chiedere di fermare il collaudo con `test.interrupt_requested`, anche durante l’esecuzione o una pausa. Il server registra e consegna questo comando **con precedenza sulle altre istruzioni in attesa**.

Il worker non avvia ulteriori operazioni su SAM, interrompe ciò che può in modo controllato, verifica gli effetti parziali e invia i risultati disponibili. Se il logout da SAM è possibile senza perdere il controllo dello stato, lo effettua; altrimenti lascia la VM all’utente con una descrizione della schermata. Risponde con `test.interrupted`:

```json
{
  "summary": "Interruzione durante la modifica di una trattativa.",
  "samLogoutCompleted": false,
  "uploadsCompleted": true,
  "manualInterventionRequired": true,
  "samSituation": "La scheda della trattativa è aperta con modifiche non confermate."
}
```

Questa conferma indica **interruzione**, non successo del test. Se il worker è bloccato o irraggiungibile, il server non può garantirne l’arresto dalla sola API: registra l’interruzione come **non confermata**, mantiene la VM prenotata e informa l’utente che deve verificare la situazione via RDP. In quel caso non esiste un `test.interrupted` inventato dal server.

Dopo `test.close_requested` o `test.interrupt_requested` il server non accetta nuove istruzioni, chiarimenti o approfondimenti per lo stesso collaudo. Richieste contemporanee vengono ordinate e valutate in base allo stato già registrato; una sessione `test.closed` o confermata `test.interrupted` non viene riavviata con `user.reply`.

## File: JSON dei metadati e upload binario separato

Lo schema principale valida gli **eventi JSON**. Per i file, [test_language_schema.json](test_language_schema.json) contiene anche due definizioni separate:

- `#/$defs/artifactUploadMetadata`: oggetto JSON che accompagna l’upload binario; contiene `sessionId`, `userId`, `vmId`, **`eventMessageId` dell’evento futuro**, nome, tipo, categoria (`screenshot`, `document`, `export`, `other`), dimensione, hash SHA-256 ed eventuale `stepId`.
- `#/$defs/artifactUploadReceipt`: risposta del server contenente `artifactId` e `sessionId`.

**Solo il worker** trasferisce i byte con una richiesta di upload separata, multipart con parte `metadata` JSON e parte `file` binaria; il browser non dispone di upload. Il server salva il file nella **root organizzata per `sessionId`**, registra i metadati in SQL Server e restituisce `artifactId`. Il worker cita quell’ID in `step.event`, `worker.notice`, `test.activities_completed` o `test.interrupted` **solo dopo la conferma dell’upload**. Gli endpoint sono definiti in [contratti-api.md](contratti-api.md).

## Ricevute, ritenti e ripresa

La definizione `#/$defs/transportReceipt` rappresenta la **risposta HTTP JSON del server** quando accetta un messaggio: `stored` o `duplicate`, riferita all’ID della sessione e al `messageId` originario e con lo stesso `serverSequence` attribuito all’evento. Non è un passaggio del test. Una richiesta non valida o non ammessa viene respinta con un codice HTTP e un corpo **Problem Details**, non con un evento né con una ricevuta `rejected`.

Se la risposta HTTP si perde, il mittente reinvia il messaggio **con lo stesso `messageId`**. Il server non duplica l’evento e non riesegue il comando. `command.ack` è diverso: è la conferma **del worker** di aver ricevuto o rifiutato un comando già registrato dal server; non prova che il comando abbia raggiunto il suo risultato finale.

Dopo una disconnessione del browser o del worker, il server fornisce lo stato del collaudo e gli eventi successivi all’ultimo `serverSequence` ricevuto. La connessione SignalR può essere ristabilita senza perdere la conversazione: la copia autorevole è persistita dal server, non dalla connessione in tempo reale.

## Stati da rappresentare sul server

| Stato di massima | Entrata e prossima transizione |
|---|---|
| In attesa | `test.start` registrato; attesa della presa in carico del worker |
| In esecuzione | Worker operativo; produce `step.event` e può ricevere `user.reply` di tipo `instruction` |
| In attesa dell’utente | `worker.notice` con `requiresResponse: true`; ripresa dopo `user.reply` di tipo `clarification` |
| Attività terminate | `test.activities_completed`; possibile estensione con `user.reply` oppure richiesta di chiusura |
| Chiusura richiesta | `test.close_requested`; attesa dei tentativi di logout e upload e di `test.closed`, anche se uno fallisce |
| Chiuso | `test.closed`, anche con errori finali segnalati; eventuale sistemazione di SAM e rilascio manuale della VM sono operazioni distinte |
| Interruzione richiesta | `test.interrupt_requested` registrato; priorità rispetto alle altre istruzioni |
| Interrotto | `test.interrupted` ricevuto e risultati parziali disponibili |
| Interruzione non confermata | Worker senza risposta o irraggiungibile; VM ancora prenotata, necessario controllo dell’utente |

Lo schema JSON controlla **forma e tipi dei messaggi**; è il server a verificare autorizzazione, appartenenza di VM e sessione, ordine delle transizioni, unicità dei messaggi e applicabilità delle istruzioni allo stato corrente. Le firme delle API e il contratto dettagliato del piano potranno essere definiti sulla base di questo linguaggio.
