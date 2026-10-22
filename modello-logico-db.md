
## Modello logico delle entità

Le entità seguenti descrivono dati e relazioni, **non ancora tabelle SQL definitive**.

| Entità | Dati principali | Relazioni e regole |
|---|---|---|
| **Utente** | `UserId`, nome utente univoco, hash password, stato account | Può avere più prenotazioni e collaudi; è l’identità autenticata della console client. |
| **MacchinaVirtuale** | `VmId`, nome macchina, destinazione RDP, stato disponibile/prenotata dichiarato | Ha al massimo una prenotazione aperta. Lo stato non è dedotto dalla presenza del worker. |
| **PrenotazioneVM** | `ReservationId`, `VmId`, `UserId`, date di prenotazione e rilascio, stato | Appartiene a una VM e a un utente; contiene zero o più collaudi **successivi**. Ha un audit amministrativo distinto. |
| **SessioneCollaudo** | `SessionId`, `ReservationId`, `UserId`, `VmId`, stato, fase, esito, date di creazione/avvio/fine, origine della terminazione, conferma worker, versione per aggiornamenti concorrenti | Nasce all’invio del piano, non alla prenotazione; una sola sessione attiva alla volta per prenotazione. Dopo `Completata` o `Interrotta` è congelata. |
| **PianoCollaudo** | `SessionId`, descrizione in linguaggio naturale, data di conferma | Fotografia iniziale. Indicazioni successive sono eventi `user.reply`, non modifiche retroattive del piano. |
| **EventoCollaudo** | `EventId`, `SessionId`, `messageId`, `serverSequence`, tipo, origine, date evento/ricezione, testo naturale, payload JSON storico, `stepId`, `requestId`, gravità e `inReplyTo` quando presenti | Più eventi per sessione. `(SessionId, messageId)` evita duplicati; `(SessionId, serverSequence)` preserva l’ordine. Non conserva segreti SAM in chiaro. |
| **AllegatoCollaudo** | `ArtifactId`, `SessionId`, `EventId`, percorso file sulla macchina server, nome, categoria, dimensione, tipo, hash e data | Ogni allegato storico è legato a un evento **della stessa sessione**. Gli upload in attesa del relativo evento sono temporanei e non ancora allegati storici; solo il worker può inviare file. |
| **ResocontoCollaudo** | `SessionId`, esito, riepilogo, esiti di logout SAM e upload, intervento manuale richiesto, evidenze mancanti, data e finalità | Anteprima dopo le attività; alla chiusura o interruzione diventa finale e immutabile, anche se logout o upload sono falliti. |
| **RichiestaIntervento** | `requestId`, `SessionId`, evento domanda, eventuale evento risposta, stato e date | Associa `worker.notice` e `user.reply` di tipo `clarification`; evita risposte alla richiesta sbagliata. |
| **ConsegnaComando** | `SessionId`, `messageId` comando, destinazione worker, stato consegna/presa in carico, numero tentativi e date | Consente ritenti senza riesecuzioni duplicate dopo cadute della rete. |
| **WorkerAttivo** | Identità worker, `VmId`, sessione Windows, prontezza e ultima comunicazione | Descrive chi può eseguire il collaudo; non determina da solo la disponibilità della VM nel pool. |
| **AuditPrenotazione** | Prenotazione, utente, operazione, data ed eventuale motivazione | Registra prenotazione e rilascio della VM, senza riaprire i collaudi già congelati. |
| **AuditRecupero** | `SessionId`, utente, data, motivazione, azioni manuali, conferma worker assente, evidenze mancanti | Formalizza la chiusura manuale forzata quando non esiste `test.interrupted`. È separato dagli eventi del worker. |

La sessione web autenticata serve a instradare UI, chat e notifiche, **ma non coincide con `SessionId`**. Non è prevista una BozzaCollaudo persistita nella prima versione: la conversazione preliminare può andare perduta; dal piano inviato in avanti la cronologia è persistita.


## Vincoli e decisioni da mantenere nella progettazione

1. **Prenotazione esclusiva atomica** nel database; la scelta della VM e il successivo file `.rdp` non devono consentire una seconda prenotazione concorrente.
2. **Creazione del test distinta dalla prenotazione.** Ogni invio di un piano produce un nuovo `SessionId` sulla prenotazione esistente o appena aperta; un reinvio dello stesso avvio non crea un secondo test.
3. **Eventi ordinati e non duplicati**, ciascuno con `SessionId`, `messageId`, `serverSequence`, tipo, origine e testo naturale. Gli allegati storici appartengono allo stesso `SessionId` del loro evento.
4. **Nessuna mutazione dopo lo stato terminale** di piano, eventi, file e stato del collaudo. Il rilascio VM aggiorna solo prenotazione e audit; le informazioni mancanti a un’interruzione sono indicate nel resoconto congelato.
5. **Nessun completamento inventato:** il solo `test.closed` valido prova chiusura ordinaria; il solo `test.interrupted` prova interruzione confermata dal worker. Se il worker non risponde, serve il comando distinto di recupero manuale e l’origine resta tracciabile.
6. **VM e collaudo indipendenti:** lo stesso utente può conservare la VM riservata dopo un test terminale e inviare un nuovo piano per creare una nuova sessione; solo la liberazione manuale rimette la VM nel pool.

La formalizzazione fisica di tabelle, chiavi esterne, indici, campi di audit e transazioni deve conservare questi vincoli. Le firme delle API stabiliranno come esporre invio del piano, comandi del test, upload, recupero degli eventi, rilascio VM e recupero manuale, senza trasformare una chiusura amministrativa in un messaggio inesistente del worker.
