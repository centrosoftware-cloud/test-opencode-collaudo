# Tracciato SQL Server — Sistema di collaudo SAM ERP

Questo documento definisce una proposta di **schema fisico SQL Server** per la prima versione. I nomi delle tabelle sono in inglese e restano rivedibili prima delle migrazioni; la semantica deriva da [modello-logico-db.md](modello-logico-db.md), [test-life-cycle.md](test-life-cycle.md) e [contratti-api.md](contratti-api.md).

## 1. Convenzioni del tracciato

- Schema SQL: `dbo`. Chiavi applicative: `UNIQUEIDENTIFIER`; identificativi generati dal server o, dove previsto dalle API, dal mittente. Tutte le date: `DATETIME2(3)` **UTC**; `CreatedAtUtc` e simili vengono valorizzati dal server, non dal browser.
- Nomi delle colonne in `PascalCase`, corrispondenti ai campi JSON `camelCase` delle API. `NOT NULL` salvo dove indicato `NULL`. Tipi `NVARCHAR(n)` per testi, `VARCHAR(n)` per codici controllati; testi lunghi e JSON in `NVARCHAR(MAX)` con `CHECK (ISJSON(...)=1)` per i JSON non nulli.
- Entità modificabili concorrenti: `ROWVERSION` per confronto ottimistico. L’API espone la versione della VM come `ETag`. I vincoli di unicità, i controlli sul lifecycle e le transazioni sul server restano necessari anche con `ROWVERSION`.
- La disponibilità visualizzata di una VM è **calcolata** da abilitazione e prenotazione aperta, secondo la dichiarazione manuale degli utenti. Il heartbeat del worker **non** libera la VM né ne determina lo stato nel pool.
- Password della Console Client: solo hash con sale nel DB, nel formato del componente standard usato dall’applicazione. Credenziali SAM: il **nome utente** resta in piano e storico; la **password non viene persistita** in SQL Server, file, eventi, log, report, allegati, ricevute o risposte storiche. È presente soltanto in memoria volatile per la consegna al worker e nel worker per il login; dopo riavvio del server, se non ancora consegnata, l’utente la reinserisce.
- File fisici nella root configurata sul server: il DB conserva un **percorso relativo**, che insieme alla root individua il percorso fisico effettivo. Non salva PNG, PDF e altri file come `VARBINARY(MAX)` nelle entità storiche.
- Una sessione `Completed` o `Interrupted` e i suoi dati operativi sono immutabili. Rilascio VM e audit amministrativi hanno un ciclo di vita separato.

Le tabelle elencate sono il nucleo fisico previsto; gli esempi DDL successivi illustrano vincoli particolarmente importanti, senza sostituire le future migrazioni complete.

## 2. Utenti, sessioni web e macchine virtuali

### `Users`

| Colonna | Tipo | Vincoli e significato |
|---|---|---|
| `UserId` | `UNIQUEIDENTIFIER` | PK |
| `UserName` | `NVARCHAR(128)` | NOT NULL; nome scelto dall’amministratore |
| `NormalizedUserName` | `NVARCHAR(128)` | NOT NULL; normalizzazione applicativa, univoca anche con collation non case-insensitive |
| `PasswordHash` | `NVARCHAR(512)` | NOT NULL; rappresentazione prodotta da `PasswordHasher`, con sale incorporato, non reversibile |
| `IsEnabled` | `BIT` | NOT NULL, default 1 |
| `CreatedAtUtc`, `UpdatedAtUtc` | `DATETIME2(3)` | NOT NULL |
| `RowVersion` | `ROWVERSION` | Controllo degli aggiornamenti concorrenti |

**Indici:** UNIQUE (`NormalizedUserName`). Nessun endpoint restituisce `PasswordHash`. Una reimpostazione della password sostituisce soltanto l’hash e revoca le sessioni web esistenti dell’utente nella stessa operazione. Disabilitare un account ne impedisce nuovi accessi e revoca le sessioni web esistenti.

### `ClientLoginSessions`

| Colonna | Tipo | Vincoli e significato |
|---|---|---|
| `LoginSessionId` | `UNIQUEIDENTIFIER` | PK; identificativo tecnico del browser autenticato, **diverso da `SessionId` del collaudo** |
| `UserId` | `UNIQUEIDENTIFIER` | NOT NULL, FK → `Users` |
| `CreatedAtUtc`, `ExpiresAtUtc` | `DATETIME2(3)` | NOT NULL |
| `RevokedAtUtc` | `DATETIME2(3) NULL` | Logout o revoca da parte del server |

**Indici:** (`UserId`, `ExpiresAtUtc`). Il cookie protetto identifica questa sessione; il DB consente revoca e controllo dell’utente anche dopo un riavvio del server. Login, logout o chiusura del browser non mutano le prenotazioni VM.

### `VirtualMachines`

| Colonna | Tipo | Vincoli e significato |
|---|---|---|
| `VmId` | `UNIQUEIDENTIFIER` | PK, stabile anche se cambiano nome o host RDP |
| `MachineName` | `NVARCHAR(128)` | NOT NULL |
| `NormalizedMachineName` | `NVARCHAR(128)` | NOT NULL, univoco |
| `RdpHost` | `NVARCHAR(255)` | NOT NULL; nome o indirizzo inserito nel file `.rdp` |
| `DisplayName` | `NVARCHAR(160) NULL` | Nome nella UI |
| `Description` | `NVARCHAR(1000) NULL` | Descrizione del pool |
| `IsEnabled` | `BIT` | NOT NULL, default 1; se 0 non può essere prenotata |
| `LastDeliverySequence` | `BIGINT` | NOT NULL, default 0; numerazione persistente dei comandi destinati alla VM |
| `CreatedAtUtc`, `UpdatedAtUtc` | `DATETIME2(3)` | NOT NULL |
| `RowVersion` | `ROWVERSION` | Concorrenza e `ETag` delle API |

**Indici:** UNIQUE (`NormalizedMachineName`); indice su (`IsEnabled`, `MachineName`) per il pool. Non è prevista verifica automatica dello stato di alimentazione o delle sessioni RDP.

### `VmReservations`

| Colonna | Tipo | Vincoli e significato |
|---|---|---|
| `ReservationId` | `UNIQUEIDENTIFIER` | PK |
| `VmId` | `UNIQUEIDENTIFIER` | NOT NULL, FK → `VirtualMachines` |
| `UserId` | `UNIQUEIDENTIFIER` | NOT NULL, FK → `Users` |
| `Status` | `VARCHAR(16)` | NOT NULL; `active` o `released` |
| `ReservedAtUtc` | `DATETIME2(3)` | NOT NULL |
| `ReleasedAtUtc` | `DATETIME2(3) NULL` | Valorizzato solo al rilascio dichiarato dall’utente |
| `RowVersion` | `ROWVERSION` | Concorrenza |

**Vincoli:** CHECK sulla coppia (`Status`, `ReleasedAtUtc`): `active` richiede data nulla, `released` data presente. UNIQUE filtrato su `VmId` con `ReleasedAtUtc IS NULL`; UNIQUE (`ReservationId`, `UserId`, `VmId`) per verificare mediante FK composita che una sessione utilizzi la prenotazione dello stesso utente e della stessa VM. **Indici:** (`UserId`, `Status`, `ReservedAtUtc DESC`). Una prenotazione può esistere senza collaudi e può comprendere più collaudi **successivi**.

### `ReservationAudit`

`ReservationAuditId UNIQUEIDENTIFIER` PK; `ReservationId UNIQUEIDENTIFIER` FK → `VmReservations`; `UserId UNIQUEIDENTIFIER` FK → `Users`; `Action VARCHAR(16)` (`reserved`, `released`); `ConfirmedDisconnected BIT NULL` (richiesto `true` per `released`, NULL per `reserved`); `Note NVARCHAR(2000) NULL`; `OccurredAtUtc DATETIME2(3)` NOT NULL. Indice (`ReservationId`, `OccurredAtUtc`). Non è un `TestingEvent`: può registrare il rilascio dopo la chiusura di tutte le sessioni senza mutarle. Un secondo audit `released` per la stessa prenotazione è impedito dalla transazione di rilascio e da un indice UNIQUE filtrato sull’azione.

## 3. Sessioni, piano e cronologia dei collaudi

### `TestingSessions`

| Colonna | Tipo | Vincoli e significato |
|---|---|---|
| `SessionId` | `UNIQUEIDENTIFIER` | PK, generato al momento dell’invio del piano |
| `ReservationId`, `UserId`, `VmId` | `UNIQUEIDENTIFIER` | NOT NULL; FK composita (`ReservationId`, `UserId`, `VmId`) → `VmReservations` |
| `Status` | `VARCHAR(20)` | NOT NULL; `created`, `in_progress`, `completed`, `interrupted` |
| `Phase` | `VARCHAR(32) NULL` | Per `in_progress`: `awaiting_worker`, `executing`, `awaiting_user`, `activities_completed`, `closing`, `interrupt_requested`, `interrupt_unconfirmed`; NULL se terminale |
| `Outcome` | `VARCHAR(16) NULL` | Ultimo esito dichiarato (`passed`, `partial`, `failed`); non sostituisce `Status` |
| `TerminationOrigin` | `VARCHAR(16) NULL` | `worker` o `manual` per sessioni terminali |
| `WorkerConfirmed` | `BIT NULL` | 1 quando è pervenuto `test.closed`/`test.interrupted`, 0 nella chiusura manuale |
| `CreatedAtUtc`, `StartedAtUtc` | `DATETIME2(3)` | NOT NULL; l’avvio e `test.start` sono registrati nella stessa operazione di creazione |
| `EndedAtUtc`, `FrozenAtUtc` | `DATETIME2(3) NULL` | Entrambe valorizzate nella transizione terminale |
| `LastServerSequence` | `BIGINT` | NOT NULL, default 0; progressivo per gli eventi della sessione |
| `LastCompletionNumber` | `INT` | NOT NULL, default 0; incrementato da `test.activities_completed` |
| `RowVersion` | `ROWVERSION` | Versione per transizioni concorrenti |

**Vincoli:** CHECK sugli stati/fasi e su `EndedAtUtc`/`FrozenAtUtc`: per `completed`/`interrupted` fase NULL e date presenti; per `in_progress` fase ammessa e date NULL. L’eventuale `created` resta transitorio all’interno della transazione di avvio; in una sessione persistita e avviata lo stato è `in_progress / awaiting_worker`. UNIQUE filtrato su `ReservationId` con `EndedAtUtc IS NULL`: **un solo test non terminale per prenotazione**. Indici (`UserId`, `CreatedAtUtc DESC`), (`Status`, `CreatedAtUtc DESC`) e (`VmId`, `CreatedAtUtc DESC`) per client e cruscotto amministrativo.

La sessione non ha `ReleasedAtUtc`: **il rilascio è una proprietà della prenotazione**, non una transizione del collaudo.

### `TestingPlans`

`SessionId UNIQUEIDENTIFIER` PK e FK → `TestingSessions`; `Description NVARCHAR(MAX)` NOT NULL (piano naturale concordato); `SamUserName NVARCHAR(128)` NOT NULL (nome utente usato per il login SAM); `ConfirmedAtUtc DATETIME2(3)` NOT NULL; `PlanFormatVersion VARCHAR(16)` NOT NULL. Una sola fotografia iniziale per sessione. Ulteriori istruzioni sono eventi `user.reply`, **non** aggiornamenti del piano. In v1 la chat preliminare è temporanea nel browser e non esiste una tabella per bozze o allegati pre-collaudo; il client non può caricare file.

### `TestingEvents`

| Colonna | Tipo | Vincoli e significato |
|---|---|---|
| `EventId` | `UNIQUEIDENTIFIER` | PK generata dal server |
| `SessionId` | `UNIQUEIDENTIFIER` | NOT NULL, FK → `TestingSessions` |
| `MessageId` | `UNIQUEIDENTIFIER` | NOT NULL; ID dell’envelope JSON originale, stabile nei ritenti |
| `ServerSequence` | `BIGINT` | NOT NULL, ≥ 1; ordine attribuito dal server nella sessione |
| `EventType` | `VARCHAR(64)` | NOT NULL; tipo previsto in `test_language_schema.json` |
| `Origin` | `VARCHAR(16)` | NOT NULL; `client` o `worker` |
| `OccurredAtUtc`, `ReceivedAtUtc` | `DATETIME2(3)` | NOT NULL; data del mittente e data registrata dal server |
| `DisplayText` | `NVARCHAR(2000)` | NOT NULL; testo naturale breve per il diario e, dove pertinente, la notifica |
| `PayloadJson` | `NVARCHAR(MAX)` | NOT NULL, JSON sanificato; `test.start` contiene solo `plan.description` e `samLogin.userName`, **mai la password SAM**; il servizio ne valida la forma oltre al CHECK `ISJSON` |
| `StepId` | `NVARCHAR(128) NULL` | Identificativo logico del passaggio |
| `RequestId` | `UNIQUEIDENTIFIER NULL` | Riferimento a una richiesta di chiarimento |
| `Severity` | `VARCHAR(16) NULL` | Livello del messaggio, se pertinente |
| `InReplyToMessageId` | `UNIQUEIDENTIFIER NULL` | Causazione/conversazione |

**Vincoli:** UNIQUE (`SessionId`, `MessageId`), UNIQUE (`SessionId`, `ServerSequence`), UNIQUE (`SessionId`, `EventId`) per FK composite degli allegati; CHECK JSON su `PayloadJson`, CHECK su `Origin` e sui codici ammessi. **Indici:** (`SessionId`, `ServerSequence`) per il diario e il recupero dopo riconnessione; (`SessionId`, `RequestId`) filtrato per richieste non nulle; (`SessionId`, `OccurredAtUtc`) se utile al resoconto.

L’evento `test.start` storico contiene descrizione del piano e **nome utente SAM**, non la password o l’envelope completo di trasporto. Il server valida separatamente tale proiezione sanificata; quando deve consegnare il comando completo usa il nome utente dal piano e la password soltanto dalla memoria volatile della sessione. Se questa non è più disponibile, non consegna `test.start` finché l’utente non reinserisce la password senza creare un nuovo evento. Per ogni tipo di evento produce `DisplayText` dai dati pertinenti: descrizione del worker, motivo dell’avviso o risposta dell’utente. Eventi di presa in carico e comandi hanno un riepilogo breve generato dal server. `ServerSequence` è attribuito dal server: un valore fornito dal mittente è rifiutato.

### `TestingReports`

`SessionId UNIQUEIDENTIFIER` PK/FK → `TestingSessions`; `IsFinal BIT` NOT NULL; `Outcome VARCHAR(16) NULL`; `Summary NVARCHAR(MAX)` NOT NULL; `SamLogoutCompleted BIT NULL`, `UploadsCompleted BIT NULL`, `ManualInterventionRequired BIT NULL` (esiti finali del worker, NULL se non ancora confermati); `ReportJson NVARCHAR(MAX)` NOT NULL, JSON con conteggi ed esiti; `MissingEvidenceJson NVARCHAR(MAX)` NOT NULL, JSON array (anche vuoto); `GeneratedAtUtc DATETIME2(3)` NOT NULL. Prima della conclusione, dopo `test.activities_completed`, può essere aggiornato come anteprima; nella transizione terminale diventa `IsFinal = 1` e **non viene più modificato**. Anche una chiusura ordinaria con logout o upload falliti congela il report con l’intervento manuale necessario ed eventuali evidenze mancanti; l’incompletezza non quantificabile è segnalata.

## 4. Comandi, richieste di intervento e worker

### `CommandDeliveries`

| Colonna | Tipo | Vincoli e significato |
|---|---|---|
| `CommandDeliveryId` | `UNIQUEIDENTIFIER` | PK |
| `EventId`, `SessionId` | `UNIQUEIDENTIFIER` | NOT NULL; FK composita (`SessionId`, `EventId`) → `TestingEvents`, per `test.start`, `user.reply`, `test.close_requested`, `test.interrupt_requested` |
| `VmId` | `UNIQUEIDENTIFIER` | NOT NULL, FK → `VirtualMachines`; corrisponde alla VM del test |
| `DeliverySequence` | `BIGINT` | NOT NULL, ≥ 1; progressivo persistente **per VM**, indipendente da `ServerSequence` |
| `WorkerInstanceId` | `UNIQUEIDENTIFIER NULL` | Istanza che ha ricevuto o preso in carico il comando; FK composita (`WorkerInstanceId`, `VmId`) → `WorkerInstances` quando valorizzata |
| `Priority` | `SMALLINT` | NOT NULL; interruzione prima delle altre istruzioni pendenti |
| `Status` | `VARCHAR(20)` | NOT NULL; `pending`, `offered`, `accepted`, `rejected`, `superseded` |
| `AttemptCount` | `INT` | NOT NULL, default 0 |
| `CreatedAtUtc`, `LastOfferedAtUtc`, `AcknowledgedAtUtc` | `DATETIME2(3)` | Prima NOT NULL, altre NULL |

**Vincoli:** UNIQUE (`EventId`) e UNIQUE (`VmId`, `DeliverySequence`). Indice (`VmId`, `Status`, `Priority`, `DeliverySequence`) per il polling dei worker. La registrazione dell’evento e della consegna avviene nella stessa transazione. Comandi offerti ma non confermati possono essere ripresentati con gli stessi `MessageId` e numero anche se il worker presenta un cursore successivo; il checkpoint di polling è il maggiore `DeliverySequence` **consecutivo** il cui stato è `accepted`, `rejected` o `superseded`, mai il massimo numero solo offerto. Un `rejected` resta registrato e non viene riproposto automaticamente: una successiva `clarification` è una nuova consegna. L’interruzione marca `superseded` i comandi pendenti incompatibili nella stessa transazione in cui si registra il comando prioritario, senza cancellare gli eventi già storicizzati.

### `Interventions`

`RequestId UNIQUEIDENTIFIER` PK; `SessionId UNIQUEIDENTIFIER` FK → `TestingSessions`; `NoticeEventId UNIQUEIDENTIFIER` NOT NULL, FK composita → `TestingEvents` della stessa sessione; `ReplyEventId UNIQUEIDENTIFIER NULL`, FK composita → `TestingEvents` della stessa sessione; `Question NVARCHAR(2000)` NOT NULL; `Status VARCHAR(16)` (`pending`, `answered`, `superseded`); `ResumePhase VARCHAR(32)` NOT NULL (`awaiting_worker` o `executing`); `CreatedAtUtc DATETIME2(3)` NOT NULL; `AnsweredAtUtc DATETIME2(3) NULL`. UNIQUE (`NoticeEventId`), UNIQUE filtrato su `ReplyEventId` non nullo; indice (`SessionId`, `Status`, `CreatedAtUtc`). Il `worker.notice` che segue un avvio rifiutato imposta `ResumePhase=awaiting_worker`; le altre domande ordinarie usano `executing`. La risposta deve usare un `requestId` ancora pendente; il worker rilegge SAM prima della ripresa. Nessuna domanda viene creata dopo `test.close_requested`.

### `WorkerInstances`

`WorkerInstanceId UNIQUEIDENTIFIER` PK; `VmId UNIQUEIDENTIFIER` FK → `VirtualMachines`; `WindowsSessionId NVARCHAR(64)` NOT NULL; `Status VARCHAR(16)` (`ready`, `busy`, `paused`, `unavailable`); `DesktopCaptureOk BIT` NOT NULL; `SamProcessRunning BIT` NOT NULL; `RegisteredAtUtc`, `LastSeenAtUtc DATETIME2(3)` NOT NULL; `EndedAtUtc DATETIME2(3) NULL`. UNIQUE (`WorkerInstanceId`, `VmId`) per la verifica della VM nelle consegne; indice (`VmId`, `LastSeenAtUtc DESC`). Una sola istanza operativa per VM: UNIQUE filtrato su `VmId` con `EndedAtUtc IS NULL`; un’istanza precedente non responsiva richiede riconciliazione prima di registrarne un’altra. La scadenza del heartbeat non rilascia la prenotazione VM.

### `WorkerCredentials`

`WorkerCredentialId UNIQUEIDENTIFIER` PK; `VmId UNIQUEIDENTIFIER` FK → `VirtualMachines`; `TokenHash VARBINARY(32)` NOT NULL; `IsEnabled BIT` NOT NULL; `CreatedAtUtc DATETIME2(3)` NOT NULL; `RevokedAtUtc DATETIME2(3) NULL`. Indici (`VmId`, `IsEnabled`) e UNIQUE (`TokenHash`). L’identità Bearer per VM viene predisposta durante l’installazione; **il token in chiaro non è storicizzato**. Distribuzione e rotazione potranno essere dettagliate quando si definisce l’installazione delle VM.

## 5. File, upload temporanei e allegati storici

Il trasferimento binario e i metadati JSON di un file sono distinti dagli eventi. `eventMessageId` nei metadati indica **l’evento ancora da inviare**; solo dopo la registrazione di tale evento il file diventa un allegato storico.

### `PendingArtifactUploads`

`ArtifactId UNIQUEIDENTIFIER` PK, generato dal server; `SessionId UNIQUEIDENTIFIER` FK → `TestingSessions`; `EventMessageId UNIQUEIDENTIFIER` NOT NULL; `StepId NVARCHAR(128) NULL`; `Category VARCHAR(16)` (`screenshot`, `document`, `export`, `other`); `OriginalFileName NVARCHAR(260)` NOT NULL; `ContentType NVARCHAR(128)` NOT NULL; `SizeBytes BIGINT` NOT NULL, ≥ 0; `Sha256 BINARY(32)` NOT NULL; `StoredRelativePath NVARCHAR(1024)` NOT NULL; `ReceivedAtUtc DATETIME2(3)` NOT NULL. Indici (`SessionId`, `EventMessageId`) e (`ReceivedAtUtc`) per associazione e pulizia. La riga esiste solo per upload completo e file verificato; non compare negli elenchi storici o nei download della UI.

### `TestingArtifacts`

| Colonna | Tipo | Vincoli e significato |
|---|---|---|
| `ArtifactId` | `UNIQUEIDENTIFIER` | PK; conserva l’ID già restituito dall’upload |
| `SessionId`, `EventId` | `UNIQUEIDENTIFIER` | NOT NULL; FK composita (`SessionId`, `EventId`) → `TestingEvents` |
| `StepId` | `NVARCHAR(128) NULL` | Passaggio cui il file si riferisce |
| `Category` | `VARCHAR(16)` | NOT NULL; `screenshot`, `document`, `export`, `other` |
| `OriginalFileName` | `NVARCHAR(260)` | NOT NULL |
| `ContentType` | `NVARCHAR(128)` | NOT NULL |
| `SizeBytes` | `BIGINT` | NOT NULL, ≥ 0 |
| `Sha256` | `BINARY(32)` | NOT NULL |
| `StoredRelativePath` | `NVARCHAR(1024)` | NOT NULL; percorso sotto la root server organizzata per `SessionId` |
| `UploadedAtUtc` | `DATETIME2(3)` | NOT NULL |

Indici (`SessionId`, `EventId`) e (`SessionId`, `Category`, `UploadedAtUtc DESC`). Nessuna FK ammette un allegato di una sessione collegato all’evento di un’altra. Il servizio risolve e verifica il percorso fisico mediante la root configurata; **non** usa come destinazione un percorso assoluto inviato dalla VM.

### Sequenza di persistenza file

1. Il worker sceglie il `MessageId` dell’evento futuro e invia file e metadati con `EventMessageId` e un’`Idempotency-Key` di upload.
2. Il server scrive su un file temporaneo nella root della sessione, controlla dimensione e hash, quindi lo rende disponibile nel percorso fisico assegnato. Registra `PendingArtifactUploads` e restituisce `ArtifactId`. File fisici privi di una riga DB a causa di interruzione fra filesystem e SQL sono **orfani temporanei da riconciliare**, mai artefatti storici visibili.
3. Nella transazione che inserisce `TestingEvents` con quel `MessageId`, il server verifica ogni `ArtifactId` citato: medesima sessione, `EventMessageId` corrispondente e file presente. Inserisce `TestingArtifacts` e rimuove le relative righe temporanee. Un rollback lascia gli upload ancora in attesa per un ritentativo.
4. Il worker tenta di consegnare e associare tutti gli upload prima di `test.closed`. Se alcuni falliscono, `uploadsCompleted=false` e il report finale segnala le evidenze mancanti, senza impedire la transizione a `completed`. Lo stesso vale per `test.interrupted` o recupero manuale: upload non associati restano fuori dallo storico e vengono rimossi o riconciliati separatamente, senza aggiungere file a un test terminale.

La root dei file e il backup devono essere gestiti insieme al database: ripristinare soltanto SQL Server produrrebbe riferimenti a file assenti, e ripristinare solo la root perderebbe l’associazione agli eventi.

## 6. Idempotenza, report e audit di recupero

### `ApiIdempotencyRecords`

`IdempotencyRecordId UNIQUEIDENTIFIER` PK; `ActorKind VARCHAR(16)` (`client`, `worker`, `local_admin`); `ActorId UNIQUEIDENTIFIER` (utente, VM o identificativo amministrativo locale); `OperationKey NVARCHAR(200)` (operazione e risorsa bersaglio); `IdempotencyKey UNIQUEIDENTIFIER`; `RequestFingerprint BINARY(32)` (impronta con chiave della richiesta, **senza password in chiaro**; per la sola creazione del test esclude anche la password SAM dall’impronta); `HttpStatusCode SMALLINT NULL`; `ResponseJson NVARCHAR(MAX) NULL` (solo risposta senza segreti); `State VARCHAR(16)` (`processing`, `completed`); `CreatedAtUtc DATETIME2(3)` NOT NULL; `CompletedAtUtc DATETIME2(3) NULL`.

UNIQUE (`ActorKind`, `ActorId`, `OperationKey`, `IdempotencyKey`). Per la stessa chiave e stessa impronta si restituisce l’esito registrato; per contenuto diverso `409`. Per la creazione del test l’impronta comprende prenotazione, piano e nome utente SAM, **non la password**: un ritentativo con la stessa chiave può rifornire la password volatile soltanto se l’avvio non è ancora stato preso in carico, senza creare un altro test. Il reinserimento della password su un test esistente non crea eventi né record di idempotenza. La conservazione per i test attivi non deve scadere prima che sia impossibile riapplicare un comando; il tempo di eliminazione degli idempotency record storici sarà definito con la retention.

### `ManualRecoveryAudit`

`ManualRecoveryId UNIQUEIDENTIFIER` PK; `SessionId UNIQUEIDENTIFIER` FK → `TestingSessions` e UNIQUE; `UserId UNIQUEIDENTIFIER` FK → `Users`; `Reason NVARCHAR(2000)` NOT NULL; `ManualActions NVARCHAR(MAX)` NOT NULL; `SamSituation NVARCHAR(MAX)` NOT NULL; `WorkerStoppedConfirmed BIT` NOT NULL, richiesto 1; `WorkerConfirmed BIT` NOT NULL, fissato a 0; `MissingEvidenceJson NVARCHAR(MAX)` NOT NULL, JSON array; `LastServerSequenceAtRecovery BIGINT` NOT NULL; `RecoveredAtUtc DATETIME2(3)` NOT NULL. Inserimento e transizione della sessione a `interrupted` **nella stessa transazione**, senza aggiungere un falso evento `test.interrupted`.

`ManualRecoveryAudit` è storico amministrativo autonomo: può essere consultato con il collaudo, ma non appartiene alla sequenza dei messaggi del worker. Il rilascio della VM resta un’ulteriore operazione di `VmReservations` e `ReservationAudit`.

## 7. Vincoli SQL e indici irrinunciabili

Esempi del tipo di vincoli da includere nelle migrazioni (nomi e schema definitivo da validare prima della creazione fisica):

```sql
-- Una sola prenotazione aperta per VM.
CREATE UNIQUE INDEX UX_VmReservations_OpenVm
ON dbo.VmReservations (VmId)
WHERE ReleasedAtUtc IS NULL;

-- Una sola sessione non terminale per prenotazione.
CREATE UNIQUE INDEX UX_TestingSessions_ActiveReservation
ON dbo.TestingSessions (ReservationId)
WHERE EndedAtUtc IS NULL;

-- Ritenti e ordine degli eventi per sessione.
CREATE UNIQUE INDEX UX_TestingEvents_SessionMessage
ON dbo.TestingEvents (SessionId, MessageId);

CREATE UNIQUE INDEX UX_TestingEvents_SessionSequence
ON dbo.TestingEvents (SessionId, ServerSequence);

-- Esempio di vincolo JSON, oltre a quelli sui valori di Status e Phase.
ALTER TABLE dbo.TestingEvents
ADD CONSTRAINT CK_TestingEvents_PayloadJson
CHECK (ISJSON(PayloadJson) = 1);
```

Ulteriori vincoli richiesti: UNIQUE su nome utente/macchina normalizzati, `ArtifactId`, `RequestId`, evento del comando e (`VmId`, `DeliverySequence`); CHECK sui codici `Status`, `Phase`, `Category`, `Origin`, `EventType`, numeri non negativi e coerenza delle date. FKs senza cancellazione a cascata dello storico per utenti, VM, prenotazioni, collaudi ed eventi. Le FKs composite sopra indicate vincolano proprietario della prenotazione e sessione degli allegati.

**Immutabilità terminale:** le FKs e i CHECK non bastano a vietare da soli nuovi figli di una sessione terminale. Gli inserimenti di evento/upload e le transizioni devono essere eseguiti sotto lock della riga `TestingSessions` con verifica dello stato; prevedere anche controlli persistenti lato database (procedure o trigger sulle scritture operative) se esistono percorsi che possono bypassare tali transazioni. Eventuali ritenti di un messaggio già registrato restituiscono la ricevuta precedente **prima** di tentare una nuova scrittura.

## 8. Operazioni transazionali del lifecycle

| Operazione | Letture/lock e scritture nella medesima transazione SQL |
|---|---|
| **Prenotare VM** | Bloccare `VirtualMachines`, verificare `IsEnabled` e assenza di prenotazioni aperte; inserire `VmReservations`, `ReservationAudit` e risultato idempotente. L’indice filtrato impedisce la seconda prenotazione anche sotto concorrenza. **Nessuna** `TestingSession` nasce qui. |
| **Creare collaudo** | Bloccare prenotazione, verificare proprietario, stato aperto, worker pronto e assenza di test attivo; attribuire `SessionId`, inserire `TestingSessions`, `TestingPlans` con nome utente SAM, evento `test.start` con `ServerSequence=1` e payload sanificato (piano e nome utente, non password), `CommandDeliveries` e risposta idempotente. Al commit la fase è `awaiting_worker`. Solo dopo il commit il server associa temporaneamente la password **in memoria volatile** al `SessionId` per la consegna; se il processo termina prima, lo stato resta recuperabile con reinserimento della password. Fallimento prima del commit: nessun test parziale. |
| **Aggiungere evento/risposta** | Cercare prima `(SessionId, MessageId)` per i ritenti; bloccare `TestingSessions`, verificare fase e messaggio JSON senza `serverSequence` fornito dal mittente, incrementare `LastServerSequence`, inserire `TestingEvents`, eventuali `Interventions`/`CommandDeliveries`/`TestingArtifacts`, aggiornare la fase e confermare. `command.ack` rifiutato prima della chiusura resta `rejected`; il successivo `worker.notice` apre l’intervento e porta in `awaiting_user`. La `clarification` usa `ResumePhase` e non crea un secondo `test.start`. Pubblicare su SignalR **dopo** il commit. |
| **Concludere normalmente** | Accettare `test.closed` con i valori effettivi di logout SAM e upload anche se falsi; collegare solo i file consegnati; registrare evento, errori ed evidenze mancanti nel report, fissare `TestingReports.IsFinal=1`, impostare `completed`, `EndedAtUtc` e `FrozenAtUtc` nella stessa transazione. Eliminare l’eventuale password volatile dopo il commit. **Nessuna** modifica alla prenotazione: l’utente sistema SAM prima di rilasciare la VM. |
| **Interrompere con worker** | Registrare `test.interrupted`, fotografare nel report ciò che manca, impostare `interrupted`/`TerminationOrigin=worker`/`WorkerConfirmed=1` e congelare nella stessa transazione; eliminare l’eventuale password volatile dopo il commit. VM ancora prenotata. |
| **Interrompere manualmente** | Bloccare test non terminale, inserire `ManualRecoveryAudit`, fissare resoconto finale con assenze note, impostare `interrupted`/`TerminationOrigin=manual`/`WorkerConfirmed=0` e congelare; eliminare l’eventuale password volatile dopo il commit. **Nessun** evento inventato per conto del worker. VM ancora prenotata. |
| **Liberare VM** | Bloccare prenotazione e verificare assenza di test non terminali; registrare `ReleasedAtUtc`, `Status=released`, `ReservationAudit` e risposta idempotente. Le sessioni già terminali non vengono aggiornate. |

Per l’assegnazione di `ServerSequence` e `DeliverySequence` usare l’aggiornamento serializzato delle rispettive righe `TestingSessions` e `VirtualMachines` nella stessa transazione degli eventi/comandi. Evitare la generazione mediante `MAX(sequence)+1` senza lock. Definire un ordine stabile nell’acquisizione dei lock per ridurre i deadlock, oltre alla gestione dei retry su conflitti e versione.

## 9. Dati non persistiti e decisioni successive

- **Chat preliminare:** non esistono tabelle per conversazione o bozza prima dell’invio del piano. Il browser trasmette al server il contesto a ogni turno; la cronologia persistita del test comincia da `test.start`.
- **Password SAM:** solo memoria volatile per il tempo necessario alla consegna al worker, mai persistita né inclusa nella chat; dopo riavvio del server richiede nuovo inserimento se non ancora consegnata. Il nome utente SAM rimane nello storico. Password della console: hash non reversibile e mai in chiaro nel DB.
- **Versione/edition SQL Server, retention di idempotenza e staging, dimensioni massime dei file e scelta delle procedure/trigger:** da fissare prima delle migrazioni eseguibili. Non modificano separazione fra prenotazione e collaudo, unicità degli eventi e congelamento terminale qui definiti.
