# Ciclo di vita dei collaudi e modello logico delle entità

Questo documento formalizza il ciclo di vita concordato per le VM e per i collaudi SAM ERP. Integra [analisi-funzionale.md](analisi-funzionale.md), [analisi-tecnica.md](analisi-tecnica.md) e il linguaggio dei messaggi descritto in [test-language.md](test-language.md). **Prenotazione della VM e sessione di collaudo hanno cicli di vita distinti.** Tabelle SQL, vincoli fisici e firme delle API saranno definiti separatamente.

## 1. Prima della sessione di collaudo

1. L’utente accede alla Console Collaudo Client con le credenziali assegnate dall’amministratore.
2. L’utente e l’agente della console concordano in chat il piano in linguaggio naturale. **Non esiste ancora un `SessionId` di collaudo.** In questa versione la conversazione preliminare non viene storicizzata: può andare persa chiudendo la pagina.
3. Quando ritiene pronto il piano, l’utente sceglie una VM dichiarata disponibile. Se possiede già una prenotazione aperta per la VM da un test precedente, può riutilizzarla.
4. Il server registra **atomicamente** la prenotazione esclusiva della VM, se nuova. La VM risulta occupata, **ma non è ancora stato creato alcun collaudo**. L’utente scarica il file `.rdp`, accede alla VM e attende che il worker avviato nella sessione Windows comunichi di essere pronto.
5. Soltanto quando l’utente **invia il piano alla VM scelta** il server genera `SessionId`, crea la sessione e conserva piano e nome utente SAM. La password SAM è usata solo in memoria volatile per consegnare l’avvio al worker: se il server riparte prima della consegna, la UI richiede all’utente di reinserirla nello stesso test. La UI passa dalla chat preliminare alla vista del collaudo; da questo momento la conversazione persistita parte dal piano e dai messaggi della nuova sessione.

La prenotazione non implica l’avvio automatico di un test. L’utente che abbandona la preparazione prima dell’invio del piano conserva la VM prenotata finché non la libera manualmente.

## 2. Ciclo di vita della prenotazione VM

```text
VM disponibile ── prenotazione atomica ──► VM prenotata dall’utente
                                          ├─ nessun test ancora avviato
                                          ├─ un test attivo alla volta
                                          ├─ più test successivi, ognuno con SessionId distinto
                                          └─ rilascio manuale ──► VM disponibile
```

- La disponibilità della VM è **dichiarata dagli utenti e registrata nel database**; non è dedotta automaticamente da RDP, alimentazione della VM o presenza del worker.
- Una VM può avere **una sola prenotazione aperta**. Prenotazione e rilascio sono operazioni amministrative del pool, distinte dai messaggi JSON di collaudo.
- Una prenotazione appartiene a un utente e può contenere **zero, una o più sessioni di collaudo successive**, ma mai due test attivi contemporaneamente sulla stessa VM.
- Il completamento o l’interruzione di una sessione **non libera la VM**. La UI ricorda all’utente di sistemare eventuali errori residui su SAM prima di avviare un nuovo test sulla VM ancora prenotata o di liberarla quando non gli serve più.
- Dopo il test l’utente si disconnette dalla VM e la marca manualmente disponibile. Chiudere la pagina o disconnettere `mstsc` non cambia da solo lo stato del pool.
- Se un test è ancora in corso, in chiusura o con interruzione non confermata, la prenotazione non viene liberata con il normale percorso. Il recupero manuale di un worker bloccato è descritto sotto.

Il rilascio della VM **non modifica** le sessioni già terminali: i loro stati restano `Completata` o `Interrotta` anche se la VM viene successivamente prenotata da altri utenti.

## 3. Stati e fasi della sessione

Gli **stati principali** sono `Creata`, `InCorso`, `Completata` e `Interrotta`. La **fase operativa** dettaglia un test `InCorso` senza trasformare ogni passaggio in un nuovo stato terminale.

| Stato | Fase, se applicabile | Significato |
|---|---|---|
| `Creata` | — | Sessione e piano sono stati creati per l’invio. L’avvio viene registrato nella stessa operazione logica. Può essere uno stato transitorio non percepibile nella UI. |
| `InCorso` | `AttesaWorker` | `test.start` registrato e destinato al worker, in attesa di presa in carico. |
| `InCorso` | `Esecuzione` | Il worker esegue gli step e invia eventi e osservazioni. |
| `InCorso` | `AttesaUtente` | Il worker ha chiesto un chiarimento oppure non ha potuto prendere in carico una richiesta prima della chiusura; attende l’utente. |
| `InCorso` | `AttivitaTerminate` | Ricevuto `test.activities_completed`; si attendono ulteriori istruzioni o la conferma di chiusura. |
| `InCorso` | `ChiusuraRichiesta` | Ricevuto `test.close_requested`; il worker tenta l’invio finale e il logout da SAM senza chiedere ulteriori chiarimenti. |
| `InCorso` | `InterruzioneRichiesta` | Ricevuto `test.interrupt_requested`; il worker deve fermarsi e riferire lo stato lasciato in SAM. |
| `InCorso` | `InterruzioneNonConfermata` | Il worker non ha risposto; il test non è dichiarato terminato e la VM rimane prenotata. |
| `Completata` | — | Ricevuto `test.closed` valido con gli esiti effettivi di upload e logout. Eventuali errori richiedono intervento manuale sulla VM. Stato terminale. |
| `Interrotta` | — | Interruzione confermata dal worker oppure chiusura manuale di recupero confermata dall’utente. Stato terminale; conserva l’origine della chiusura. |

**`test.activities_completed` non conclude la sessione.** Il worker può emetterlo più volte, con `completionNumber` crescente, quando l’utente chiede approfondimenti tramite `user.reply` di tipo `extension`.

### Transizioni

| Stato/fase iniziale | Evento o comando | Stato/fase risultante | Regola |
|---|---|---|---|
| Nessuna sessione, VM prenotata | Invio del piano | `Creata` → `InCorso / AttesaWorker` | Creazione di sessione, piano e registrazione di `test.start` coordinate atomicamente; `SessionId` assegnato prima di inoltrare il messaggio. |
| `InCorso / AttesaWorker` | `command.ack` accettato per `test.start`, oppure per la `clarification` successiva a un avvio rifiutato, e avvio effettivo | `InCorso / Esecuzione` | L’ack prova soltanto la presa in carico; non si invia un nuovo `test.start` né si modifica quello già registrato. |
| `InCorso / AttesaWorker` o `InCorso / Esecuzione` | `command.ack` rifiutato e `worker.notice` con `requiresResponse: true` | `InCorso / AttesaUtente` | La domanda rende visibile il motivo all’utente; il worker non esegue la richiesta rifiutata. Non vale dopo la conferma di chiusura. |
| `InCorso / Esecuzione` | `step.event` | `InCorso / Esecuzione` | Ogni passaggio è persistito e collegato al medesimo `SessionId`. |
| `InCorso / Esecuzione` | `worker.notice` con `requiresResponse: true` | `InCorso / AttesaUtente` | La richiesta è collegata a `requestId`; il worker sospende le azioni pertinenti. |
| `InCorso / AttesaUtente` | `user.reply` con `kind: clarification` e `requestId` corrispondente | `InCorso / Esecuzione` oppure `InCorso / AttesaWorker` | Il worker rilegge SAM prima di riprendere. Se era stato rifiutato l’avvio resta in `AttesaWorker` fino alla ripresa dell’avvio, senza creare un secondo `test.start`. Una diversa `instruction` non chiude la richiesta pendente. |
| `InCorso / Esecuzione` | `user.reply` con `kind: instruction` | `InCorso / Esecuzione` | L’istruzione entra nella conversazione e viene applicata dal worker in un punto controllabile. |
| `InCorso / Esecuzione` | `test.activities_completed` | `InCorso / AttivitaTerminate` | Esito e riepilogo disponibili; SAM e prenotazione VM restano aperti. |
| `InCorso / AttivitaTerminate` | `user.reply` con `kind: extension` | `InCorso / Esecuzione` | Stesso `SessionId`; lo storico precedente non è riscritto. |
| `InCorso / AttivitaTerminate` | `test.close_requested` | `InCorso / ChiusuraRichiesta` | Non si accettano nuove istruzioni per il test. |
| `InCorso / ChiusuraRichiesta` | `worker.notice` con `requiresResponse: false` | `InCorso / ChiusuraRichiesta` | Segnala eventuali errori di logout o upload; non apre nuove domande. |
| `InCorso / ChiusuraRichiesta` | `test.closed` | `Completata` | Riporta esiti effettivi, intervento manuale necessario ed evidenze mancanti. La VM resta prenotata. |
| `InCorso`, fase non terminale | `test.interrupt_requested` | `InCorso / InterruzioneRichiesta` | Comando prioritario, nessun nuovo step da avviare. |
| `InCorso / InterruzioneRichiesta` | `test.interrupted` | `Interrotta` | Risultati parziali congelati, anche se `uploadsCompleted: false`; assenze segnalate. |
| `InCorso / InterruzioneRichiesta` | Worker bloccato o irraggiungibile | `InCorso / InterruzioneNonConfermata` | Il server non inventa un messaggio `test.interrupted` né libera automaticamente la VM. |
| `InCorso / InterruzioneNonConfermata` o chiusura bloccata | Conferma dell’interruzione manuale di recupero da parte dell’utente | `Interrotta` | Azione amministrativa distinta dai messaggi del worker; `workerConfirmed: false`. |

Dopo la conferma di chiusura dell’utente il worker accetta la richiesta senza inviare ulteriori domande. Anche se logout o upload finali falliscono, invia un avviso di errore non bloccante e `test.closed`; l’utente sistema manualmente SAM prima di rilasciare la VM. Se il worker è irraggiungibile e non può confermare la chiusura, il test resta non terminale e l’utente ricorre all’interruzione e, se necessario, al recupero manuale. Prima della chiusura, una richiesta rifiutata dal worker apre un chiarimento; se il problema persiste l’utente interrompe il test.

## 4. Interruzione richiesta e recupero manuale

**Percorso con worker rispondente:** l’utente richiede l’interruzione; il server registra `test.interrupt_requested` e lo consegna con priorità. Il worker non avvia altre azioni, interrompe ciò che può in modo controllato, verifica gli effetti parziali e invia `test.interrupted`. La sessione diventa immediatamente `Interrotta`, **senza aspettare che la VM venga liberata**.

**Percorso con worker bloccato o comunicazione interrotta:** la richiesta resta non confermata. Dalla UI l’utente dispone di un apposito **comando di interruzione forzata di recupero**. Prima verifica direttamente la VM via RDP, arresta il worker se necessario e riferisce che cosa ha fatto e quale situazione è rimasta in SAM. Il comando nella console:

1. Registra la spiegazione dell’utente e l’assenza di conferma del worker in un audit di recupero, **senza fabbricare `test.interrupted`**.
2. Marca la sessione `Interrotta` con `terminationOrigin: manual` e `workerConfirmed: false`, riportando gli step e i file mancanti. Da questo momento congela il suo storico.
3. Mantiene la VM prenotata. L’utente la libera manualmente con l’operazione distinta del pool quando è pronto, oppure la trattiene per un nuovo test.

Il pulsante della console **non può garantire di arrestare da remoto** un processo worker che non comunica più: certifica il recupero e la chiusura amministrativa sulla base dell’intervento dichiarato dall’utente. Per un nuovo collaudo la sessione conclusa **non viene riaperta**: ne viene creata una nuova, dopo aver verificato che il worker sia nuovamente pronto nella VM.

Se `test.interrupted` arriva con `uploadsCompleted: false`, lo storico viene congelato **indicando esplicitamente le evidenze mancanti**. Non si attendono file indefinitamente e non si aggiungono nuovi eventi al collaudo terminale; eventuali recuperi tecnici successivi richiedono una registrazione amministrativa separata, non la riscrittura del test.

## 5. Immutabilità e ordine della conversazione

- Ogni messaggio del collaudo accettato dal server genera un evento legato a **uno e un solo `SessionId`**. L’ID utente deriva dalla sessione autenticata; VM e prenotazione devono appartenere allo stesso utente e collaudo.
- `messageId` identifica univocamente l’invio anche in caso di ritentativo. `serverSequence` è progressivo per sessione e consente di ricostruire la conversazione in ordine. Risposte e richieste di intervento mantengono anche `requestId` quando applicabile.
- Ogni evento contiene un **testo naturale breve** utilizzabile per il diario e, se pertinente, come testo mostrato al client: il worker spiega il passo o il motivo della richiesta; l’evento `user.reply` contiene la risposta dell’utente. Restano disponibili anche tipo, gravità, payload strutturato e riferimenti ai file.
- Il `test.start` memorizzato espone il piano e **il nome utente SAM**, non la password: il comando completo al worker include anche la password, disponibile soltanto in memoria volatile durante la consegna. La copia storica sanificata non è un envelope di trasporto completo conforme allo schema del comando; il reinserimento della password dopo riavvio del server non aggiunge eventi al test.
- `Completata` e `Interrotta` sono **terminali**: il server non accetta ulteriori messaggi di collaudo con quel `SessionId`, non modifica piano, eventi, resoconto o allegati e non cambia più lo stato della sessione. La successiva operazione di rilascio VM si riferisce alla prenotazione; può menzionare gli ID dei collaudi già terminati soltanto nell’audit amministrativo separato.
- Richieste concorrenti di estensione, chiusura e interruzione sono accettate o respinte secondo lo stato già registrato dal server, nell’ordine assegnato alla sessione. Un messaggio respinto non modifica lo storico operativo.

## 6. File e associazione agli eventi

I file fisici vengono trasferiti **separatamente dai messaggi JSON** e salvati sulla macchina server sotto una root organizzata per `SessionId`. SQL Server conserva soltanto i metadati dell’allegato, inclusi `SessionId`, `EventId`, nome, categoria, percorso sul server, dimensione, tipo di contenuto, hash e stato del trasferimento.

Per consentire al worker di **caricare prima il file e poi inviare l’evento JSON che ne cita l’`artifactId`**, si applica la sequenza concordata:

1. Il worker genera in anticipo il `messageId` dell’evento cui intende associare il file.
2. Invia file binario e metadati JSON con `SessionId` e **`eventMessageId`**. Il server conserva temporaneamente l’upload come *in attesa di collegamento* e restituisce `artifactId`.
3. Il worker invia l’evento con quel `messageId` e l’`artifactId`; il server lo registra, attribuisce `EventId` e collega definitivamente l’allegato all’evento **nella stessa sessione**.
4. Prima di emettere `test.closed`, il worker tenta di consegnare e collegare tutti gli upload. Se alcuni falliscono, li segnala come mancanti nel resoconto e conferma ugualmente la chiusura; dopo la chiusura nessun file viene aggiunto allo storico congelato. Anche in un’interruzione o se il worker è stato arrestato, gli allegati non consegnati o rimasti senza evento vengono segnalati come mancanti; gli upload temporanei senza evento non diventano evidenze dello storico congelato.

La definizione `artifactUploadMetadata` di [test_language_schema.json](test_language_schema.json) comprende `eventMessageId`; le firme di trasferimento sono descritte in [contratti-api.md](contratti-api.md). **Solo il worker carica file al server**; il browser non ha endpoint di upload né per la chat preliminare né per i messaggi del test.
