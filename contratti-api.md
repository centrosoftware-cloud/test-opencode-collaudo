# Contratti API REST — Console Collaudo SAM ERP

Versione iniziale **v1**. Questo contratto copre Console Collaudo Client, UI amministrativa locale e Worker esecuzione collaudo. È coerente con [test-life-cycle.md](test-life-cycle.md) e con i messaggi definiti in [test_language_schema.json](test_language_schema.json) e [test-language.md](test-language.md). I nomi delle risorse API non vincolano i nomi delle future tabelle SQL.

## 1. Convenzioni comuni

### Indirizzi, rappresentazioni e trasporti

- **API utenti:** `/api/v1/...`, raggiungibili via HTTPS dai browser. La UI Razor risiede sulla stessa Console Collaudo Server.
- **API worker:** `/api/v1/worker-instances/...`, raggiungibili via HTTPS dalle VM; sono risorse separate da quelle dei browser. Il worker avvia sempre la comunicazione verso il server: non espone API in ingresso sulla VM.
- **API amministrative:** `/admin/api/v1/...`, raggiungibili **solo** tramite un listener separato vincolato a `127.0.0.1`/`::1` sulla macchina server. Controller, file e hub amministrativi non devono essere accessibili dal listener di rete degli utenti. L’amministrazione non ha login applicativo nella v1.
- **Corpi e risposte applicative:** UTF-8, `application/json`, nomi proprietà `camelCase`. L’upload dei file è riservato al worker: `multipart/form-data` con una parte `metadata` JSON e una parte `file` binaria. Nessuna API client o amministrativa accetta upload. Download `.rdp` e contenuti dei file sono risposte binarie.
- **Identificativi:** `userId`, `vmId`, `sessionId`, `reservationId`, `artifactId`, `messageId`, `requestId` e `workerInstanceId` sono UUID rappresentati come stringhe; `machineName` è il nome leggibile e univoco della VM, **non** il suo `vmId`. Tutte le date/ore sono ISO 8601 UTC con suffisso `Z`.
- **Stati pubblici:** prenotazione `active`/`released`; VM `available`/`reserved` (disponibilità dichiarata); collaudo `created`/`in_progress`/`completed`/`interrupted`; fasi di `in_progress`: `awaiting_worker`, `executing`, `awaiting_user`, `activities_completed`, `closing`, `interrupt_requested`, `interrupt_unconfirmed`. Sono le rappresentazioni API degli stati descritti nel lifecycle.
- **Versionamento:** la versione è nel percorso (`/v1`) e i messaggi mantengono `schemaVersion: "1.0"`. Un cambiamento incompatibile richiede una nuova versione del contratto.

### Accesso e ambito

- Le API utenti richiedono il cookie di accesso emesso dal server dopo login con le credenziali della console. Il cookie è `HttpOnly` e `Secure`; il server ricava `userId` da esso, non dal `userId` dichiarato in una richiesta. Operazioni e download sono limitati ai collaudi e alle prenotazioni di quell’utente. Le richieste browser che modificano dati usano la protezione antiforgery di ASP.NET Core.
- Le API worker richiedono un’identità tecnica associata alla VM, provisionata sulla VM e riconosciuta dal server tramite una credenziale **Bearer per VM**. Le modalità di distribuzione e rotazione della credenziale restano di configurazione; non è la password dell’utente web né quella di SAM. Il server verifica `vmId`, `workerInstanceId`, prenotazione e sessione a ogni chiamata.
- Le API amministrative non richiedono un cookie applicativo: il loro accesso è limitato al browser sulla macchina server dal binding locale. Anche le modifiche amministrative usano la protezione antiforgery.
- Password Windows non compaiono nei file `.rdp`. La password SAM fornita per l’avvio è consegnata solo al worker: **non viene persistita**, né inserita nella chat preliminare, nello storico `test.start`, nei log, nel resoconto o nelle risposte browser. **Il nome utente SAM resta nel piano e nello storico.** Il server conserva la password solo in memoria volatile fino alla presa in carico o al rifiuto del comando, poi la elimina; il worker la usa soltanto per il login. Se il server riparte prima della consegna, l’utente la reinserisce. Messaggio completo e copia storica sanificata sono rappresentazioni distinte, validate separatamente.

### Esiti, conflitti e ritenti

- `200 OK`: lettura o operazione con risultato; `201 Created`: nuova risorsa, con `Location`; `202 Accepted`: comando registrato per esecuzione asincrona; `204 No Content`: eliminazione della sessione di login; `400 Bad Request`: JSON/parametri invalidi; `401 Unauthorized`: login o credenziale worker assenti/invalidi; `403 Forbidden`: risorsa non dell’utente o worker non associato; `404 Not Found`: risorsa inesistente; `409 Conflict`: conflitto con stato o idempotenza; `412 Precondition Failed`: versione `If-Match` non più corrente; `413 Payload Too Large`: file oltre il limite configurato; `428 Precondition Required`: manca un `If-Match` richiesto. Errori con corpo `application/problem+json` e proprietà `status`, `title`, `detail`, `code` e `traceId`; `sessionId` se pertinente.
- Ogni `POST` **persistente** che crea VM, prenotazioni, test, comandi, audit o upload richiede `Idempotency-Key` UUID. Login, chat preliminare e reinserimento non persistente della password SAM ne sono esclusi. Ritentare la stessa chiave e lo stesso contenuto restituisce lo stesso risultato senza ripetere l’operazione; riutilizzarla con contenuto diverso restituisce `409`. **Eccezione per la sola creazione del test:** la password SAM non entra nell’impronta persistita; un ritentativo con chiave, piano e nome utente SAM identici può rifornire la password volatile se `test.start` non è ancora stato preso in carico. Per le richieste con un messaggio di [test_language_schema.json](test_language_schema.json), la chiave coincide con `messageId`. Il server conserva le chiavi per il tempo necessario a riconoscere i ritenti dei collaudi ancora attivi.
- Per un `PATCH` alla VM usare `If-Match` con l’`ETag` ricevuto nella precedente `GET` (per esempio `If-Match: "5"`); una versione non più corrente restituisce `412`. La prenotazione usa invece una transazione atomica sul server: una sola prenotazione aperta per VM. Il server non deduce automaticamente da RDP o dal worker se una VM sia disponibile.
- Liste paginabili con `limit` (predefinito 50, massimo 200) e `cursor` restituito dal server; ordinamento stabile. Le pagine hanno forma `{ "items": [...], "nextCursor": string | null }`. Il diario di un collaudo usa `afterSequence` (predefinito 0) e `limit`, ordinato per `serverSequence` crescente. La risposta include `nextSequence` e `hasMore`. I filtri non cambiano le autorizzazioni.
- Il server registra prima eventi e cambi di stato nel database e **poi** notifica via SignalR. SignalR non è fonte autorevole: il browser recupera sempre eventi e stato via REST dopo una riconnessione.

### Rappresentazioni condivise

Le risposte delle API fanno riferimento alle seguenti forme minime; eventuali proprietà additive nella stessa versione non cambiano il significato di quelle indicate.

| Rappresentazione | Proprietà |
|---|---|
| `UserView` | `userId`, `userName`, `isEnabled`, `createdAt`; **mai** password o hash |
| `VirtualMachineView` | `vmId`, `machineName`, `rdpHost`, `displayName`, `description`, `isEnabled`, `availability`, `activeReservationId`, `version` |
| `ReservationView` | `reservationId`, `vmId`, `userId`, `status`, `reservedAt`, `releasedAt`, `version` |
| `WorkerInstanceView` | `workerInstanceId`, `vmId`, `windowsSessionId`, `status`, `desktopCaptureOk`, `lastSeenAt` |
| `TestView` | `sessionId`, `reservationId`, `vmId`, `userId`, `status`, `phase` (nullo se terminale), `createdAt`, `startedAt`, `endedAt`, `lastServerSequence`, `outcome`, `terminationOrigin`, `workerConfirmed`, `samLoginRequired` (calcolato dal server per un avvio ancora da consegnare); include `plan.description` e `samLogin.userName`, mai `samLogin.password` |
| `TestEventView` | `eventId`, `sessionId`, `serverSequence`, `messageId`, `type`, `origin`, `occurredAt`, `displayText`, `stepId`, `requestId`, `severity`, `artifactIds`, payload **sanificato**; per `test.start` il piano è visibile ma non la password SAM |
| `ArtifactView` | `artifactId`, `sessionId`, `eventId`, `stepId`, `category`, `fileName`, `contentType`, `sizeBytes`, `sha256`, `uploadedAt`, `downloadUrl`; non il percorso assoluto della root server |
| `ReportView` | `sessionId`, `isFinal`, `outcome`, `summary`, riepilogo passaggi, `samLogoutCompleted`, `uploadsCompleted`, `manualInterventionRequired`, `missingEvidence`, `artifactIds`, `generatedAt` |

## 2. Console Collaudo Client: login e preparazione

Tutti gli endpoint di questa sezione sono sull’ingresso HTTPS utenti. Salvo la creazione del login, richiedono l’utente autenticato.

| Metodo e risorsa | Richiesta | Successo | Scopo e regole |
|---|---|---|---|
| `POST /api/v1/auth/session` | JSON `{ "userName": string, "password": string }` | `200` `UserView` e cookie di accesso | Verifica le credenziali attribuite dall’amministratore. Account disabilitato o credenziali errate: `401`. Nessuna funzione di cambio password dell’utente. |
| `GET /api/v1/auth/session` | — | `200` `UserView` | Ristabilisce la vista dopo il ricaricamento della pagina; `401` se non autenticato. |
| `DELETE /api/v1/auth/session` | Antiforgery | `204` | Termina la sessione web; non libera prenotazioni né interrompe collaudi. |
| `POST /api/v1/draft-chat/replies` | JSON `{ "messages": [{ "role": "user" \| "assistant", "text": string }, ...] }`; ultimo messaggio dell’utente | `200` `{ "reply": string }` | Dialogo pre-collaudo con l’agente della console. Il browser invia la cronologia necessaria a ogni chiamata: nessun `sessionId`, nessuna bozza o cronologia persistita sul server, **nessun upload di file**. Il modello è configurato sul server, non scelto nella richiesta. |

La UI sostituisce la chat preliminare con quella persistita del collaudo **solo dopo** una risposta positiva alla creazione del test.

## 3. Pool VM, prenotazioni e accesso RDP

| Metodo e risorsa | Richiesta | Successo | Scopo e regole |
|---|---|---|---|
| `GET /api/v1/virtual-machines` | Filtri `availability`, `isEnabled`, `limit`, `cursor` | `200` pagina di `VirtualMachineView` | Elenco del pool. `availability` riflette lo stato dichiarato e la prenotazione nel DB, non un controllo dell’alimentazione o delle sessioni RDP. |
| `POST /api/v1/virtual-machines` | `Idempotency-Key`; JSON `{ "machineName": string, "rdpHost": string, "displayName"?: string, "description"?: string }` | `201` `VirtualMachineView` | Censisce una VM predisposta. Nome macchina univoco; il server attribuisce `vmId`. La VM nasce abilitata e dichiarata disponibile. |
| `GET /api/v1/virtual-machines/{vmId}` | — | `200` `VirtualMachineView`, header `ETag: "<version>"` | Consulta dettagli e versione della VM, necessari per l’aggiornamento condizionale. |
| `GET /api/v1/virtual-machines/{vmId}/worker-status` | — | `200` `{ "ready": boolean, "worker": WorkerInstanceView \| null }` | Ricostruisce nella UI la prontezza del worker dopo un login RDP o una riconnessione del browser. Non influisce sulla disponibilità manuale della VM. |
| `PATCH /api/v1/virtual-machines/{vmId}` | `If-Match: "<version>"`; JSON contenente solo `rdpHost`, `displayName`, `description` o `isEnabled` da modificare | `200` `VirtualMachineView`, nuovo `ETag` | Aggiorna il censimento quando la VM non è prenotata. Disabilitarla la rende non selezionabile; non libera una prenotazione. Nessuna accensione o spegnimento. |
| `GET /api/v1/reservations` | Filtri `status`, `limit`, `cursor` | `200` pagina di `ReservationView` | Mostra **solo** le prenotazioni dell’utente corrente, incluse quelle senza test avviato. |
| `POST /api/v1/virtual-machines/{vmId}/reservations` | `Idempotency-Key`; corpo JSON `{}` | `201` `ReservationView` | Riserva atomicamente una VM dichiarata disponibile; `userId` deriva dal login. Se già riservata, disabilitata o non disponibile: `409`. **Non crea un test.** |
| `GET /api/v1/reservations/{reservationId}` | — | `200` `ReservationView` | Consulta la prenotazione dell’utente corrente. |
| `GET /api/v1/reservations/{reservationId}/rdp-file` | — | `200`, `application/x-rdp`, `Content-Disposition: attachment` | Genera il file con `rdpHost` della VM riservata, senza password Windows. `409` se la prenotazione non è più aperta. L’utente apre il file con `mstsc`. |
| `POST /api/v1/reservations/{reservationId}/releases` | `Idempotency-Key`; JSON `{ "confirmedDisconnected": true, "note"?: string }` | `201` `{ "releaseId", "reservationId", "releasedAt", "availability": "available" }` | Dichiarazione **manuale** di rilascio con audit. `409` se esiste un collaudo non terminale sulla prenotazione. Il server non verifica automaticamente RDP o alimentazione. Non altera i test conclusi. |

Una prenotazione può contenere più collaudi successivi dello stesso utente, **uno attivo alla volta**. Il rilascio e la successiva prenotazione da altri utenti sono operazioni separate dalla chiusura del test.

## 4. Sessioni di collaudo e messaggi dell’utente

### Creazione e avvio atomici

`test_language_schema.json` richiede `sessionId` in ogni messaggio, compreso `test.start`, ma prima dell’invio del piano il browser **non conosce ancora** tale ID. Per questo la creazione del test è un’operazione di risorsa, non un POST di un envelope `test.start` già completo:

| Metodo e risorsa | Richiesta | Successo | Scopo e regole |
|---|---|---|---|
| `POST /api/v1/reservations/{reservationId}/tests` | `Idempotency-Key` UUID; JSON `{ "plan": { "description": string }, "samLogin": { "userName": string, "password": string } }` | `201` `TestView`; `Location: /api/v1/tests/{sessionId}` | Richiede prenotazione dell’utente e worker `ready`, altrimenti `409`. In una transazione il server assegna `sessionId`, salva piano e **nome utente SAM**, registra `test.start` sanificato con `messageId = Idempotency-Key` e `serverSequence = 1` e mette il comando in consegna. Solo dopo il commit tiene la password in memoria volatile per consegnarla al worker, senza scriverla nei dati persistenti. Stato `in_progress / awaiting_worker`; un ritentativo con stessa chiave e dati persistibili restituisce lo stesso test e può rifornire la password non ancora consegnata. |

Nel comando consegnato **solo al worker**, `test.start.payload.samLogin` include nome utente e password forniti dall’utente. Il piano e l’evento storici conservano il **nome utente SAM**, mai la password; `TestView`, `TestEventView`, log e notifiche non ne espongono il valore. Il server forma il comando usando la password volatile solo se disponibile; altrimenti lo lascia in attesa e `samLoginRequired: true` avverte il client di reinserirla. L’envelope completo segue lo schema dei messaggi; la proiezione storica sanificata è validata separatamente e non pretende di rispettare il requisito `samLogin.password` dello schema di trasporto.

### Lettura, dialogo e comandi

| Metodo e risorsa | Richiesta | Successo | Scopo e regole |
|---|---|---|---|
| `GET /api/v1/tests` | Filtri `status`, `reservationId`, `limit`, `cursor` | `200` pagina di `TestView` | Elenca solo i collaudi dell’utente autenticato, anche dopo il rilascio della VM. |
| `GET /api/v1/tests/{sessionId}` | — | `200` `TestView` | Stato e fase correnti, piano, ultimo `serverSequence` e riepilogo; il contenuto terminale è di sola lettura. |
| `POST /api/v1/tests/{sessionId}/sam-login` | Antiforgery; JSON `{ "password": string }` | `202` `{ "status": "delivery_pending" }` | Reinserimento della password SAM per il **nome utente già storicizzato**. Consentito solo al proprietario, prima della chiusura, quando `test.start` attende ancora consegna o il worker ha chiesto aiuto per il login. Il server la mantiene solo in memoria volatile per la consegna al worker; non crea eventi, non modifica il piano, non salva password né impronte della password. Ripetere la chiamata sostituisce il valore volatile. `409` se non pertinente o il test è terminale. |
| `GET /api/v1/tests/{sessionId}/events` | `afterSequence` ≥ 0, `limit` ≤ 200 | `200` `{ "items": [TestEventView], "nextSequence": integer, "hasMore": boolean }` | Diario ordinato. `nextSequence` è l’ultimo numero consegnato, o `afterSequence` se non ci sono eventi. Nessuna credenziale SAM nella vista. |
| `GET /api/v1/tests/{sessionId}/interventions` | Filtro `status=pending\|answered`, `limit`, `cursor` | `200` pagina `{ "requestId", "question", "status", "eventId", "createdAt", "answeredAt" }` | Ricostruisce le domande anche dopo una disconnessione di SignalR. |
| `POST /api/v1/tests/{sessionId}/messages` | `Idempotency-Key = messageId`; envelope JSON **completo** di tipo `user.reply` secondo lo schema, senza allegati | `202` `transportReceipt` e fase aggiornata | Accetta `clarification` solo per `requestId` pendente **prima della chiusura**, anche dopo il rifiuto di una richiesta da parte del worker; `instruction` in esecuzione, `extension` in fase `activities_completed`. Dopo `test.close_requested` non accetta più risposte. Verifica `userId` e `vmId` rispetto a login, test e prenotazione. `409` per fase incompatibile o test terminale. Il worker riceve il messaggio tramite la propria coda comandi. |
| `POST /api/v1/tests/{sessionId}/closure-requests` | `Idempotency-Key` UUID; JSON `{ "confirmedByUser": true, "note"?: string }` | `202` `transportReceipt`, fase `closing` | Il server costruisce `test.close_requested` con `messageId = Idempotency-Key`. Ammesso dopo `test.activities_completed`; consegna al worker, **non** libera la VM. Da qui nessun nuovo chiarimento o istruzione: il worker segnala errori finali e conferma comunque la chiusura se è raggiungibile. |
| `POST /api/v1/tests/{sessionId}/interruption-requests` | `Idempotency-Key` UUID; JSON `{ "reason"?: string }` | `202` `transportReceipt`, fase `interrupt_requested` | Il server costruisce `test.interrupt_requested`. Ammesso prima dello stato terminale, con priorità sulle istruzioni in coda; i comandi superati sono marcati come tali dal server. Il worker conferma con `test.interrupted` se può rispondere. |
| `POST /api/v1/tests/{sessionId}/manual-interruptions` | `Idempotency-Key`; JSON `{ "reason": string, "manualActions": string, "samSituation": string, "workerStoppedConfirmed": true, "missingEvidence"?: [string] }` | `200` `TestView` con `status: "interrupted"`, `terminationOrigin: "manual"`, `workerConfirmed: false` | Recupero quando il worker non conferma l’interruzione o la chiusura è bloccata. Il server crea **audit amministrativo**, congela lo storico e **non fabbrica `test.interrupted`**. La VM resta prenotata; l’utente la libera separatamente quando pronta. `409` se il test è già terminale o il percorso manuale non è pertinente. |
| `GET /api/v1/tests/{sessionId}/report` | — | `200` `ReportView` | Dopo `test.activities_completed` restituisce un riepilogo provvisorio (`isFinal: false`); dopo `test.closed`, `test.interrupted` o interruzione manuale, la fotografia finale congelata (`isFinal: true`), con esiti di logout/upload ed evidenze mancanti. Prima del primo riepilogo: `409 report_not_ready`. |

`transportReceipt` segue `#/$defs/transportReceipt` dello schema: contiene `sessionId`, `userId`, `vmId`, `acknowledgedMessageId`, `status` (`stored` o `duplicate`) e `serverSequence` della registrazione. Una richiesta respinta per dati o stato non validi restituisce invece **Problem Details** con il relativo codice HTTP. La **presa in carico da parte del worker** è un ulteriore messaggio `command.ack`, distinto dalla ricevuta HTTP.

## 5. Worker: presenza, comandi e registrazione degli eventi

Il worker comunica esclusivamente con il server tramite l’ingresso HTTPS delle API worker, usando la propria identità tecnica di VM. Non definisce né modifica la disponibilità manuale della VM nel pool.

Il server recapita **solo le richieste dell’utente** (avvio, risposte, estensioni, chiusura, interruzione), non i passi UI o le decisioni su SAM: l’agente Azure nel worker sceglie autonomamente le azioni. Alla riconnessione il worker verifica lo stato autorevole del test prima di riprendere; se il server non risponde, sospende nuove azioni.

| Metodo e risorsa | Richiesta | Successo | Scopo e regole |
|---|---|---|---|
| `POST /api/v1/worker-instances` | `Idempotency-Key = workerInstanceId`; JSON `{ "workerInstanceId": UUID, "vmId": UUID, "windowsSessionId": string, "status": "ready" \| "unavailable", "desktopCaptureOk": boolean }` | `201` `WorkerInstanceView` | Registrazione dopo login RDP. `ready` ammesso solo dopo verifica desktop e screenshot; il server associa la credenziale tecnica alla VM. Un worker ancora vivo nella stessa sessione impedisce la registrazione concorrente (`409`). |
| `PUT /api/v1/worker-instances/{workerInstanceId}/status` | JSON `{ "status": "ready" \| "busy" \| "paused" \| "unavailable", "desktopCaptureOk": boolean, "samProcessRunning": boolean }` | `200` `WorkerInstanceView` | Aggiorna presenza e prontezza; il server attribuisce `lastSeenAt`. È anche la comunicazione periodica di presenza. Non cambia lo stato manuale della VM. |
| `GET /api/v1/worker-instances/{workerInstanceId}/commands` | `afterDeliverySequence` ≥ 0, `waitSeconds` 0–20 (predefinito 15) | `200` `{ "commands": [{ "deliverySequence": integer, "message": envelope JSON }], "lastDeliverySequence": integer }` | Long polling delle sole richieste dell’utente. `lastDeliverySequence` è il più alto numero consecutivo di consegne **confermate dal server tramite `command.ack` oppure `superseded`**, non l’ultimo comando solo offerto. `afterDeliverySequence` è il checkpoint noto al worker: se maggiore di quello persistito dal server viene ignorato in favore del checkpoint server. Un comando offerto ma non confermato viene riproposto con gli stessi `deliverySequence` e `messageId`; `test.start` viene offerto solo se la password è ancora in memoria volatile, altrimenti resta in attesa del reinserimento dell’utente. L’interruzione scavalca comandi pendenti incompatibili, marcati `superseded` atomicamente; il worker deduplica per `messageId`. |
| `GET /api/v1/worker-instances/{workerInstanceId}/tests/{sessionId}/state` | — | `200` `{ "sessionId": UUID, "status": string, "phase": string \| null, "lastServerSequence": integer }` | Stato autorevole riservato al worker della VM della sessione, accessibile anche dopo il recupero manuale. Il worker lo consulta alla riconnessione **prima** di nuove azioni su SAM; se `completed` o `interrupted` non prosegue. Non aggiunge eventi allo storico. |
| `GET /api/v1/worker-instances/{workerInstanceId}/tests/{sessionId}/sam-login` | — | `200` `{ "userName": string, "password": string }` | Lettura riservata al worker della VM per il login o un successivo chiarimento che richiede di reinserire la password. Disponibile solo finché il valore volatile è presente e il test non è in chiusura o terminale; `409 sam_login_required` se assente. La risposta non è registrata né notificata. Il server elimina il valore volatile dopo l’ack (`accepted` o `rejected`) del relativo `test.start`, oppure dopo l’ack della `clarification` che usa la nuova password. |
| `POST /api/v1/worker-instances/{workerInstanceId}/messages` | `Idempotency-Key = messageId`; envelope JSON completo secondo lo schema con `origin: "worker"` | `200` `transportReceipt` | Registra `command.ack`, `step.event`, `worker.notice`, `test.activities_completed`, `test.closed` o `test.interrupted`. Verifica stato, mittente, `sessionId`, `userId`, `vmId`, progressivo dei completamenti e riferimenti agli artefatti; attribuisce `serverSequence` e applica la transizione nella stessa operazione persistente. Un test terminale rifiuta nuovi messaggi (`409`), salvo risposta idempotente a un evento già registrato. |

`command.ack.status: "accepted"` prova la **presa in carico**, non la riuscita su SAM. Un `rejected` prima della chiusura richiede `reason` e un successivo `worker.notice` con domanda (`requestId`, `requiresResponse: true`): il server registra il rifiuto, mostra la richiesta all’utente e pone il test in attesa; la risposta `clarification` permette al worker di riesaminare e riprovare, senza ripetere automaticamente il vecchio comando. Per `test.start` la ripresa è nello stesso `sessionId` e il piano originale resta immutato; un nuovo avvio non viene creato. Se il problema persiste, l’utente interrompe il test. Dopo `test.close_requested` il worker non rifiuta la chiusura per un problema operativo né chiede chiarimenti: invia `worker.notice` di errore con `requiresResponse: false`, poi `test.closed` con i valori effettivi di `samLogoutCompleted`/`uploadsCompleted`, `manualInterventionRequired` ed eventuali `missingEvidence`. Anche questo test diventa `completed`; l’utente sistema SAM manualmente prima di rilasciare la VM. `test.interrupted` può avere upload incompleti e congela lo storico indicando le mancanze. Se il worker è irraggiungibile e non invia `test.closed`, il test resta in chiusura ed è recuperabile manualmente. Il server non crea mai messaggi del worker al suo posto.

I comandi restano registrati finché non vengono presi in carico; il worker non avvia due volte lo stesso `messageId` dopo una riconnessione. Un comando rifiutato non viene riproposto automaticamente: è una nuova risposta al chiarimento che abilita il worker a riprovare in base al nuovo stato osservato. Se il worker non risponde, il server mantiene il test in fase di recupero e **non** libera la VM. La richiesta di interruzione può aspettare il prossimo polling; non equivale ad arrestare immediatamente un processo Windows non più responsivo.

## 6. Evidenze: upload, lettura e download

Il percorso server dei file è sotto la root configurata per `sessionId`; l’API non accetta né restituisce un percorso fisico assoluto utilizzabile come destinazione di scrittura.

**Solo il worker autenticato può caricare file.** Browser client e UI amministrativa possono consultare e scaricare i file collegati, mai caricarli, neppure nella chat preliminare.

| Metodo e risorsa | Richiesta | Successo | Scopo e regole |
|---|---|---|---|
| `POST /api/v1/worker-instances/{workerInstanceId}/tests/{sessionId}/artifacts` | `Idempotency-Key` UUID; `multipart/form-data`: parte `metadata` (`application/json`, schema `#/$defs/artifactUploadMetadata`) + parte `file` binaria | `201` `#/$defs/artifactUploadReceipt` | I metadati includono **`eventMessageId`** generato dal worker prima dell’upload, `sessionId`, `userId`, `vmId`, nome, MIME, categoria, dimensione e SHA-256. Il server verifica file/hash/dimensione e conserva l’upload *in attesa dell’evento*. Il ritentativo identico restituisce lo stesso `artifactId`. |
| `GET /api/v1/tests/{sessionId}/artifacts` | `limit`, `cursor`, eventuale `eventId` | `200` pagina di `ArtifactView` | Mostra solo allegati già collegati a eventi della sessione; non espone upload temporanei. |
| `GET /api/v1/tests/{sessionId}/artifacts/{artifactId}/content` | — | `200` contenuto binario con MIME appropriato | Solo file della sessione accessibile all’utente; screenshot visualizzabili, altri file scaricabili. `404` se non collegato o assente. |

L’upload avviene **prima** dell’evento che ne cita `artifactId`. Quando il worker pubblica quell’evento con il proprio `messageId`, il server collega atomicamente gli allegati per `eventMessageId` e verifica che appartengano allo stesso `sessionId`. File caricati ma mai collegati non compaiono nello storico congelato. I documenti forniti nella chat preliminare **non** hanno in v1 un upload persistente, poiché prima dell’avvio non esiste `sessionId`.

## 7. UI amministrativa locale: utenti, cruscotto e storico

Tutte le risorse seguenti sono disponibili **esclusivamente** dal listener `localhost` della macchina server. La UI amministrativa usa Razor/HTML/CSS/JavaScript, ma non ha login applicativo nella v1; le richieste che modificano dati richiedono antiforgery. Non vengono esposti questi endpoint sul listener di rete delle API utenti/worker.

| Metodo e risorsa | Richiesta | Successo | Scopo e regole |
|---|---|---|---|
| `GET /admin/api/v1/users` | `limit`, `cursor` | `200` pagina di `UserView` | Elenco utenti della console. |
| `POST /admin/api/v1/users` | `Idempotency-Key`; JSON `{ "userName": string, "password": string }` | `201` `UserView` | Crea un utente abilitato; nome univoco, password conservata **solo come hash con sale**. La risposta non restituisce la password. |
| `GET /admin/api/v1/users/{userId}` | — | `200` `UserView` | Consultazione dell’utente. |
| `PATCH /admin/api/v1/users/{userId}` | JSON `{ "isEnabled": boolean }` | `200` `UserView` | Abilita/disabilita l’accesso; non cancella la cronologia dell’utente. |
| `PUT /admin/api/v1/users/{userId}/password` | JSON `{ "newPassword": string }` | `204` | Sostituisce l’hash precedente con uno nuovo. L’amministratore comunica la nuova password fuori dall’applicazione; nessun endpoint di cambio password per l’utente. |
| `GET /admin/api/v1/dashboard` | — | `200` `{ "activeTests": [TestView], "vmSummary": {...}, "updatedAt": UTC }` | Vista minima dei collaudi in corso, VM, utente e stato. Le notifiche SignalR possono richiedere il ricaricamento di questa rappresentazione. |
| `GET /admin/api/v1/tests` | Filtri `status`, `userId`, `vmId`, date, `limit`, `cursor` | `200` pagina di `TestView` | Storico globale, compresi collaudi completati e interrotti. |
| `GET /admin/api/v1/tests/{sessionId}` | — | `200` `TestView` | Dettaglio storico o in corso. |
| `GET /admin/api/v1/tests/{sessionId}/events` | `afterSequence`, `limit` | `200` pagina di `TestEventView` | Diario ordinato, senza segreti SAM. |
| `GET /admin/api/v1/tests/{sessionId}/report` | — | `200` `ReportView`; `409` se non pronto | Resoconto provvisorio/finale secondo lo stato del test. |
| `GET /admin/api/v1/tests/{sessionId}/artifacts` | `limit`, `cursor` | `200` pagina di `ArtifactView` | Elenco dei file storicizzati. |
| `GET /admin/api/v1/tests/{sessionId}/artifacts/{artifactId}/content` | — | `200` contenuto binario | Recupera un file dalla root server, senza esporre upload non collegati. |

La UI amministrativa è in **lettura** per i collaudi: non cambia uno stato terminale e non produce messaggi del worker. Prenotazione/rilascio della VM e recupero manuale sono operazioni del client dell’utente previste nelle sezioni precedenti.

## 8. SignalR: notifiche ai browser

SignalR non è un endpoint REST e non riceve comandi dei worker. Sono previsti due hub distinti:

- `/hubs/v1/client-events`: ingresso di rete per browser autenticati. Il server invia notifiche solo all’utente proprietario, e per i dettagli solo alle sue schede del collaudo; l’appartenenza non è decisa da un `userId` inviato dal browser.
- `/admin/hubs/v1/monitor`: **solo listener localhost**, senza login applicativo, per il cruscotto amministrativo.

Ogni notifica è un **segnale per aggiornare la UI**, non la copia autorevole di un evento: JSON `{ "notificationId": UUID, "type": string, "occurredAt": UTC, "sessionId"?: UUID, "serverSequence"?: integer, "resourceId"?: UUID }`. `serverSequence` è presente per `TestEventAvailable`; notifiche sulle VM e sui worker non richiedono `sessionId` prima dell’avvio del test.

| `type` della notifica | Quando viene emessa | Destinatari |
|---|---|---|
| `VirtualMachineChanged` | Prenotazione o rilascio registrati | Browser che consultano il pool; amministrazione locale |
| `WorkerReadinessChanged` | Worker pronto/non pronto o presenza aggiornata | Utente che ha prenotato la VM; amministrazione locale |
| `TestEventAvailable` | Evento accettato e numerato dal server | Utente proprietario e amministrazione locale; contiene `sessionId`, `serverSequence` |
| `TestStateChanged` | Cambia stato o fase del collaudo, inclusa interruzione manuale | Utente proprietario e amministrazione locale |
| `InterventionRequested` | Domanda del worker in attesa di risposta | Utente proprietario e amministrazione locale; `resourceId` = `requestId` |
| `ArtifactAvailable` | File collegato a un evento | Utente proprietario e amministrazione locale; `resourceId` = `artifactId` |
| `ReportAvailable` | Riepilogo provvisorio o finale disponibile | Utente proprietario e amministrazione locale |

**Prima si registra nel database, poi si notifica.** Il browser legge il contenuto tramite le API REST del proprio ingresso. Dopo una riconnessione ripete `GET` del test e degli eventi con `afterSequence` pari all’ultimo numero visto; il cruscotto amministrativo ricarica dashboard e liste. Le notifiche possono essere duplicate o mancare senza perdere eventi, poiché SQL Server rimane la fonte autorevole. I file restano accessibili esclusivamente tramite gli endpoint di lettura/download, non via SignalR.

## 9. Regole trasversali del lifecycle

1. **Prenotazione e test sono separati.** Una prenotazione può contenere zero o più collaudi successivi; soltanto l’invio del piano crea il `sessionId` e il primo messaggio del test.
2. **Immutabilità terminale.** `test.closed`, `test.interrupted` o recupero manuale congelano la sessione. Nessun ulteriore `user.reply`, `step.event` o allegato storico viene aggiunto; il rilascio VM aggiorna soltanto prenotazione e audit. Un ritentativo con `messageId` già registrato restituisce invece la ricevuta originaria, senza mutare il test.
3. **Estensioni nello stesso collaudo.** `test.activities_completed` lascia il test `in_progress / activities_completed`; `user.reply` con `kind: "extension"` riprende lo stesso `sessionId` prima della chiusura. Ogni nuovo completamento incrementa `completionNumber`.
4. **Interruzione senza worker.** `test.interrupt_requested` rimane non confermato finché il worker non invia `test.interrupted`. In mancanza di risposta l’utente interviene nella VM e usa `manual-interruptions`; la sessione diventa `interrupted`, `workerConfirmed: false`, **senza** evento `test.interrupted` inventato dal server. La VM resta prenotata fino al rilascio manuale distinto.
