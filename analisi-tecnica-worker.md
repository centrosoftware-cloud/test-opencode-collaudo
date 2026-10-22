# Analisi tecnica del worker — Interazione con SAM ERP

Questo è il **riferimento tecnico ufficiale per il worker C#/.NET 10**. Specifica gli strumenti esterni necessari per osservare e utilizzare SAM ERP nella sessione grafica Windows e le responsabilità del worker durante un collaudo. Si collega all’architettura generale di [analisi-tecnica.md](analisi-tecnica.md), ai [contratti API](contratti-api.md) e al [linguaggio dei messaggi](test-language.md). Qui non vengono descritte funzioni proprie di una singola gestione né le librerie interne con cui SAM è stato compilato: le classi citate sono osservabili **dall’esterno**.

La verifica esplorativa è stata svolta con SAM eseguito nella stessa sessione Windows del processo di automazione, usando operazioni non distruttive: accesso, navigazione, apertura di form, ricerca, lettura di griglia, input temporanei annullati e dialoghi di scelta file/stampa annullati. I risultati indicati come **osservati** derivano dalla UI; le scelte per .NET 10 indicate come **proposte** devono essere confermate nel prototipo del worker, specialmente nella VM via RDP.

## 1. Inventario degli strumenti usati e corrispondenza C#

OpenCode ha guidato gli strumenti di sistema tramite comandi PowerShell 5.1, `Add-Type` e interoperabilità Win32/WinRT. **PowerShell non è un prerequisito di produzione:** il worker può chiamare le stesse API direttamente da C#.

| Funzione | Strumenti effettivamente impiegati nelle prove | Proposta per il worker .NET 10 |
|---|---|---|
| Avvio e controllo del processo | PowerShell `Start-Process`, `Get-Process`, nome, PID, sessione e finestra principale | `System.Diagnostics.Process` con percorso eseguibile e working directory configurati; verifica PID, sessione Windows, risposta e finestre effettive |
| Albero accessibile | Assembly `UIAutomationClient`, `UIAutomationTypes` e `WindowsBase` per il punto geometrico; `AutomationElement.FromHandle`, `FromPoint`, `FindAll`, `FocusedElement`, proprietà e pattern | Windows UI Automation tramite riferimenti Windows Desktop o interop COM `UIAutomationCore`; un wrapper come **FlaUI.UIA3 è opzionale**, da provare sul target .NET 10 |
| Albero nativo | `user32.dll`: `EnumWindows`, `EnumChildWindows`, `GetWindowThreadProcessId`, `GetClassNameW`, `GetWindowTextW`, `GetWindowRect`, `IsWindowVisible`, `IsWindowEnabled` | P/Invoke C# (`LibraryImport`/`DllImport`) con gestione corretta di HWND a 64 bit e stringhe Unicode |
| Focus e geometria | `GetForegroundWindow`, `SetForegroundWindow`, `BringWindowToTop`, `ShowWindow`, `WindowFromPoint`, `GetCursorPos`, `SetCursorPos`; occasionalmente `AttachThreadInput` | Modulo C# per attivazione/controllo del focus e mappatura delle coordinate; `AttachThreadInput` solo se necessario e sempre con sgancio in `finally` |
| Input | `System.Windows.Forms.SendKeys`, `mouse_event`, `SendMessageW`/`PostMessageW`, `BM_CLICK`, `WM_SETTEXT` in prove controllate | Preferire pattern accessibili realmente supportati o **`SendInput`** Win32 per mouse/tastiera nella sessione interattiva; messaggi diretti solo per classi ed effetti verificati |
| Screenshot | `System.Drawing.Bitmap`, `Graphics.CopyFromScreen`, `ImageFormat.Png` | `System.Drawing.Common` **solo Windows** o equivalente di cattura desktop; conservare la schermata originale e produrre ritagli/ingrandimenti per OCR |
| Riconoscimento testo | `System.Runtime.WindowsRuntime` in PowerShell; `Windows.Storage`, `Windows.Graphics.Imaging`, `Windows.Media.Ocr.OcrEngine` | Proiezioni Windows SDK/WinRT disponibili per .NET 10 con `Windows.Media.Ocr`; gestire lingue OCR installate. Fallback da valutare: Tesseract con dati linguistici installati |
| Monitor e DPI | `System.Windows.Forms.Screen.AllScreens`; `user32.dll` `SetProcessDpiAwarenessContext`, `GetSystemMetrics`, `GetDpiForWindow`, geometria delle finestre | Manifest/processo **Per-Monitor DPI Aware V2** dall’avvio; enumerazione di monitor e coordinate fisiche prima di interagire |
| Menu e suggerimenti | Finestre `THintWindow`; popup `#32768`, messaggio `MN_GETHMENU`, `GetMenuItemCount`, `GetMenuStringW`, `GetSubMenu` | Tooltip mediante osservazione/hover; introspezione di menu nativi **solo diagnostica**, con OCR/tastiera come fallback per menu disegnati o personalizzati |
| Finestre Windows comuni | Enumerazione native di `#32770`, controlli `Static`, `Button`, `Edit`, `ComboBox`; dialoghi di file e stampa | Stesso modulo Windows/UIA: lettura del testo esatto, pulsanti, stato e conferma dell’esito. Nessuna dipendenza da coordinate fisse del dialogo |
| Trasferimento delle evidenze | PNG prodotti localmente e testo OCR/controlli usato per interpretare la UI | `System.IO` e `HttpClient` per inviare eventi JSON e upload binari al Console Collaudo Server, con ID del collaudo e del messaggio |

Il modello Azure interpreta i risultati degli strumenti, ma **non sostituisce le API Windows**. Nelle prove il modello non ha ricevuto una lettura visiva diretta delle immagini: la lettura del desktop è avvenuta con proprietà dei controlli e OCR; le schermate sono state conservate come evidenze. L’input visivo diretto nei deployment Azure previsti resta una capacità da verificare separatamente.

## 2. Prerequisito di esecuzione: stessa sessione grafica di SAM

- Il worker parte **dopo il login Windows via RDP**, nella sessione utente che ospita SAM. Non eseguire il motore UI come normale servizio in Session 0.
- Worker e SAM devono operare con privilegi compatibili nella stessa sessione: se SAM è avviato con un livello di elevazione maggiore, input e lettura cross-process possono essere limitati da Windows. Rilevare questa differenza prima di interpretarla come controllo non disponibile; l’eventuale elevazione del worker va verificata nell’ambiente previsto.
- Una sola istanza deve pilotare quel desktop alla volta: due agenti che inviano focus, mouse e tastiera nella stessa sessione interferiscono.
- Separare l’indicazione «processo presente/rispondente» da «desktop pronto». Prima del test verificare finestra principale, eventuali modali, dimensioni dei monitor e capacità di produrre **uno screenshot leggibile**.
- Una finestra principale può cambiare classe, titolo e figli durante caricamenti o apertura di documenti. L’identificazione affidabile combina PID, sessione, classe, titolo, HWND corrente e postcondizione; gli HWND **non sono identificativi persistenti**.
- Non presumere che il worker possa leggere pixel se RDP è bloccata, disconnessa o minimizzata. Va misurato nella VM reale, con finestra `mstsc` piccola e coperta come caso d’uso previsto. Se la cattura fallisce o è obsoleta, mettere in pausa e notificare il server.

## 3. DPI e coordinate: requisito preliminare obbligatorio

**Osservazione riprodotta:** il processo di automazione inizialmente usava coordinate Windows virtualizzate dal DPI. Sullo stesso desktop, `GetSystemMetrics` forniva **1536 × 960** prima di impostare la consapevolezza DPI e **1920 × 1200** dopo `SetProcessDpiAwarenessContext(PER_MONITOR_AWARE_V2)`. Un punto interpretato come appartenente alla toolbar veniva individuato da `WindowFromPoint` come griglia sottostante. Non era una variazione dei dati di SAM: era un disallineamento dello spazio di coordinate del processo chiamante.

**Requisito per il worker:**

1. Dichiarare **Per-Monitor DPI Awareness V2** nel manifest del processo C# o impostarla all’avvio, **prima** di creare finestre, inizializzare componenti grafici o acquisire coordinate. Registrare se l’impostazione non è stata applicata.
2. Scegliere come rappresentazione interna **coordinate fisiche del desktop**. Associare a ogni rettangolo la sorgente e lo spazio di coordinate (UIA, Win32, OCR); fare conversioni esplicite quando necessarie.
3. Confrontare risoluzione/area utile di ogni monitor con `GetWindowRect`, `AutomationElement.BoundingRectangle`, `WindowFromPoint` e screenshot su uno stesso punto di prova. Considerare monitor con origine negativa, cambio risoluzione e scaling differente.
4. Rileggere geometria dopo apertura di modali, spostamenti, massimizzazione, riconnessione RDP e cambio DPI. **Non** memorizzare coordinate assolute come definizione permanente di un pulsante.

La semplice riuscita di `SetForegroundWindow` o di un click **non dimostra** che sia stato colpito il controllo inteso: verificare il controllo sotto il punto e la postcondizione visibile.

## 4. Pattern riscontrati nei componenti Delphi

| Pattern osservato | Conseguenza per il worker |
|---|---|
| Form MDI: più finestre figlie possono comparire nell’albero UIA anche quando una sola è attiva; talvolta risultano `IsOffscreen = false` pur essendo coperte | Identificare **titolo/form attiva** e gerarchia dal processo; non considerare una finestra utilizzabile solo perché `IsWindowVisible` o `IsOffscreen` lo suggeriscono |
| Finestre modali di proprietà dell’app, inclusi wizard e messaggi `#32770`, non sempre compaiono fra i figli diretti di `AutomationElement.RootElement` | Inventariare **`EnumWindows` per PID** dopo ogni azione, quindi usare `AutomationElement.FromHandle` e enumerare i figli nativi |
| `TButton`, `TCSEdit`, `TCheckBox`, `TComboBox` e controlli `TAdv*` possono apparire come `ControlType.Pane` senza Invoke/Value/Toggle/Selection pattern | Fare capability detection per **quel controllo**, usare testo/bounds nativi o OCR e azioni verificabili; non assumere che una libreria UIA possa invocare tutti i pulsanti |
| Le griglie `TSMDBGrid` possono esporre il contenitore ma non celle, intestazioni, righe o `ScrollPattern` | Per leggere usare OCR delle regioni della griglia, conteggi esposti altrove e, se necessario, apertura della singola form; scorrere tramite input reale e verificare che le righe cambino |
| Una scelta in combo può sostituire il controllo edit nello stesso rettangolo, rendendo nascosto il vecchio HWND | Dopo ogni scelta **rileggere HWND, focus e controllo visibile** prima di digitare; non riutilizzare un handle precedente |
| Una listbox disegnata (`TAdvSmoothListBox`) può mostrare voci non rappresentate come `ListItem` accessibili | Usare OCR di listbox/tooltip, scorrimento e click/tastiera reali; controllare la scelta dalla schermata successiva |
| Tooltip disponibili come finestre `THintWindow`; alcuni testi sono generici e pulsanti vicini condividono suggerimenti simili | Hover e lettura del tooltip aiutano a scoprire il comando, ma l’azione effettiva va **confermata dal risultato**, non dall’icona o dal solo tooltip |
| Popup `#32768` spesso elencabili con API menu Win32, ma posizione e voci cambiano a seconda di dove viene aperto | Usare l’elenco nativo quando disponibile; confermare per OCR lo stato del menu e non riprodurre clic usando posizioni di un’apertura precedente |
| Si possono enumerare anche figli nativi di schede `TPageControl` nascoste, con handle validi e controlli abilitati | Filtrare sulla scheda/form effettivamente mostrata. Evitare di confondere figli nascosti con campi correnti |
| Menu ad albero e campi di ricerca generali personalizzati (`TCSVirutalTreeViewMenu`, `TCSMenuSearchEdit`) possono mostrare voci disegnate senza figli accessibili per ogni elemento | Valutare OCR e ricerca tramite digitazione reale; dopo la scelta controllare il titolo della form aperta anziché dedurre il risultato dal solo testo inserito |

**Input e persistenza.** Su un campo personalizzato `WM_SETTEXT` ha cambiato il testo visto da UI Automation e OCR, mentre `GetWindowTextW` del controllo è rimasto vuoto: i diversi canali di lettura non sono intercambiabili. Le precedenti prove di collaudo hanno mostrato che un testo visibile impostato artificialmente può non aggiornare i dati applicativi. La digitazione reale con focus e uscita dal campo è la prima scelta; quando il piano prevede un salvataggio, il worker deve **riaprire o rinfrescare** e confrontare i dati. `WM_CHAR` è soltanto un’alternativa sperimentale da verificare sul controllo specifico. Non inviare `CB_SETCURSEL` e notifiche `CBN_SELCHANGE` costruite artificialmente alle combo come surrogato non verificato della scelta dell’utente.

**Griglie e OCR.** È stato possibile leggere intestazioni, valori visibili e un conteggio esposto nella form tramite crop + OCR, anche quando la griglia non esponeva le celle a UIA. Un dialogo di conteggio ha distinto **righe estratte** da **righe selezionate**. La navigazione tramite tastiera con focus sulla griglia ha cambiato le righe visibili; un `WM_MOUSEWHEEL` postato direttamente al suo HWND non ha prodotto lo stesso risultato nella prova. Il worker deve verificare movimento e focus, usare più risoluzioni di OCR e correggere eventuali ambiguità (`0`/`O`, `1`/`I`, date, separatori degli importi) prima di riportare un dato come certo.

## 5. Sequenza di interazione richiesta al worker

Ogni azione dell’agente deve attraversare queste fasi:

1. **Osserva:** inventaria processo, finestre top-level e modali; acquisisci controlli nativi/UIA, titolo e screenshot della zona pertinente. Preferisci una lettura strutturata quando esiste; usa OCR per contenuti grafici.
2. **Scegli il bersaglio:** processo/sessione, finestra attiva, controllo corrente, bounds, capacità esposte, focus e verifica `WindowFromPoint` se serve input a coordinate.
3. **Agisci una sola volta:** Invoke/Value/Selection soltanto se supportati e verificati; altrimenti input reale (`SendInput`) con focus controllato. Un `PostMessage(BM_CLICK)` è un’alternativa mirata per pulsanti nativi già identificati, non la modalità universale. `SendMessage(BM_CLICK)` sincrono può restare in attesa se apre una modale; non scambiarne il timeout per un blocco di SAM senza inventariare le finestre.
4. **Attendi per condizione:** cerca nuova finestra, dialogo, cambio titolo, variazione dei controlli o dei dati. `TViewLoading` e altri indicatori intermedi non sono il risultato finale. Usa timeout, retry mirato e verifica del processo; non ripetere ciecamente un salvataggio.
5. **Verifica e registra:** confronta stato precedente e successivo, testo esatto dei messaggi, screenshot, dati ed eventuale persistenza. Se il nuovo stato è ambiguo, chiedi chiarimenti anziché inventare la riuscita.

Il worker espone all’agente LLM **strumenti a granularità controllata**, per esempio `ListWindows`, `InspectControls`, `GetFocusedElement`, `CaptureScreen`, `RecognizeText`, `ReadTooltip`, `ClickControl`, `TypeIntoControl`, `SendKey`, `NavigateMenu`, `WaitForState` e `ReadDialogs`. Le azioni devono restituire sia un esito tecnico sia la **postcondizione osservata**; l’LLM sceglie il passo successivo usando quel risultato. `StartSam`, `EndSam` e `UploadArtifact` completano gli strumenti non grafici.

La risposta tecnica interna di un’osservazione dovrebbe comprendere: PID/sessione Windows, HWND e classe **correnti**, titolo, rettangolo con coordinate/DPI, sorgente (`native`, `uia`, `ocr`), pattern disponibili, testo con livello di attendibilità, screenshot collegato e data di acquisizione. Gli HWND restano temporanei; il modello non deve trattarli come identificatori stabili tra sessioni o dopo la chiusura della form.

## 6. Screenshot, OCR, z-order e ripresa

- `Graphics.CopyFromScreen` legge i **pixel visibili del desktop**, non il contenuto garantito di una specifica finestra. Quando un’altra applicazione copriva la zona, l’OCR ha riconosciuto quella applicazione invece di SAM. Verificare foreground, processo sotto il punto e contenuto plausibile prima di attribuire la cattura a SAM.
- **Non lasciare la finestra principale di SAM topmost.** Nel test, renderla sempre in primo piano con `SetWindowPos(HWND_TOPMOST)` ha fatto sì che un wizard proprietà dell’app fosse enumerabile ma risultasse coperto: OCR leggeva la griglia sottostante. Preferire la normale attivazione della form; se è necessario cambiare temporaneamente z-order, ripristinarlo prima di interagire con una modale.
- Se una form appare fuori dall’area di lavoro corrente, **misurare e conservare le bounds originali**, enumerare i monitor e poi riposizionare la finestra su un monitor disponibile tramite Win32 prima di riprendere. Il recupero non deve cancellare l’evidenza della posizione iniziale.
- Nel worker C# conviene offrire crop per form/controllo, ingrandimento per OCR e schermata originale per evidenza. Conservare titolo e geometria insieme allo screenshot per evitare attribuzioni errate.
- Se **SAM è coperta da un’altra applicazione nel desktop remoto**, `CopyFromScreen` può leggere la copertura anziché SAM. La cattura con SAM minimizzata o con una sessione RDP disconnessa **non è dimostrata**. Coprire la sola finestra `mstsc` sul PC dell’utente è un caso diverso, verificato sotto. Windows Graphics Capture o DXGI Desktop Duplication sono candidati da valutare **solo se la cattura desktop ordinaria è insufficiente nella VM**: nessuno dei due garantisce automaticamente un desktop utilizzabile senza sessione grafica.
- **Prova aggiuntiva con RDP ridotta ma non minimizzata:** quando `mstsc` è rimasta connessa, piccola e coperta sul PC dell’utente, le catture `CopyFromScreen` eseguite sulla macchina di SAM sono rimaste leggibili. Un’apertura di finestra e una successiva navigazione hanno prodotto nuovi contenuti OCR, non soltanto hash diversi. Sessione Windows e risoluzione sono rimaste stabili. Protocollo, misure e immagini sono in [prova-rdp.md](prova-rdp.md). È un esito sulla configurazione provata, **da ripetere sulla VM di destinazione**.
- In seguito a pausa o intervento umano, rileggere completamente la UI prima di agire; non riprendere dall’ultimo HWND o dalle coordinate precedenti. Se la cattura non produce dati nuovi/attendibili, sospendere e notificare la console.

## 7. Dialoghi di sistema e periferiche

| Superficie osservata | Capacità necessaria al worker | Limite delle prove |
|---|---|---|
| Dialoghi informativi/di conferma | Leggere finestra top-level e controllo `Static`, acquisire messaggio, distinguere i pulsanti e scegliere soltanto dopo averne compreso la conseguenza. Chiudere tramite `Button` nativo e verificare che la finestra scompaia | Non interpretare automaticamente tutti i messaggi come errori |
| Scelta file Windows `#32770` | Distinguere `Edit`, `ComboBox`, `Button`, percorso/nome file, conferma e Annulla. Il worker può compilare un nome soltanto se l’azione richiede davvero un output | È stato aperto e annullato un Salva con nome; **nessun export** creato nel giro tecnico |
| Selezione report personalizzata | Leggere opzioni, `TListView`, intestazioni e file visibili tramite nativo/OCR; scegliere il modello solo se richiesto | `TListView` non esponeva le voci come `ListItem` nell’albero UIA osservato |
| Stampa Windows `#32770` | Leggere stampante selezionata, pagine, copie e pulsanti. Preferire Annulla nelle sole prove di navigazione | La stampante predefinita osservata era **Remote Desktop Easy Print**, quindi può dipendere dal reindirizzamento RDP; nessuna stampa fisica eseguita |
| Clipboard e input similare | Se necessari, usare API Windows/WinForms in un thread compatibile e verificare focus e formato; evitare di considerare la clipboard Windows equivalente a funzioni applicative di copia | Clipboard, lettori barcode, registratori di cassa e altri dispositivi fisici **non collaudati** in questa esplorazione |

Per collaudare una periferica reale occorreranno dispositivo, driver, configurazione e risultato atteso specifici. **Non è richiesto un SDK di ogni periferica solo per navigare nelle form di SAM**: il worker automatizza prima l’interfaccia Windows/SAM e osserva ciò che il programma espone. Soltanto se un piano richiederà di verificare direttamente una coda di stampa o un dispositivo, si valuterà l’aggiunta delle relative API Windows o del protocollo del dispositivo.

Se il collaudo produce un file, verificare anche **che sia completo e del formato atteso**, non solo che esista: dimensione, firma/formato reale e contenuto pertinente. Per testo e PNG bastano strumenti .NET/Windows; per fogli di calcolo e PDF valutare rispettivamente Open XML SDK o un lettore/renderizzatore PDF compatibile con licenza e deployment scelti. Queste librerie non sono necessarie per la semplice automazione delle form, ma lo diventano per test che richiedono di verificare i documenti prodotti.

## 8. Librerie da prevedere nel progetto .NET 10

**Base necessaria, Windows-specifica:**

1. Progetto `net10.0-windows` nella sessione RDP dell’utente, con manifest Per-Monitor DPI Aware V2.
2. `System.Diagnostics` e `System.IO` per processo, file temporanei, tracce e screenshot; `System.Net.Http` per API server, eventi e upload binari.
3. Windows UI Automation (`AutomationElement` e pattern). Si può usare il riferimento Windows Desktop alle API gestite oppure un’interfaccia COM/il wrapper `FlaUI.UIA3`; **scegliere la variante dopo una prova di compilazione .NET 10 e sui controlli reali**. Un wrapper non aggiunge automaticamente accessibilità alle griglie personalizzate.
4. Interop nativo `user32.dll` e, dove necessario, `kernel32.dll`/`shcore.dll`: enumerazione finestre, PID/sessione, DPI, testo e classi, focus, input, menu e geometria. Incapsularlo in un modulo isolato con firme 64 bit e Unicode verificate.
5. Cattura PNG via API Windows supportate nel processo Windows (`System.Drawing.Common` su Windows oppure equivalente), più OCR Windows `Windows.Media.Ocr` tramite proiezione WinRT compatibile con il progetto .NET 10. In PowerShell 5.1 è stato usato `System.Runtime.WindowsRuntime` come ponte: **non copiarne meccanicamente il codice nel worker moderno**.

**Supporto da valutare, non prerequisito verificato:** WinRT/Windows SDK projection specifica, Tesseract per lingue o ambienti senza OCR Windows, OpenCV per pre-elaborazione delle immagini, Windows Graphics Capture/DXGI per esigenze di rendering. Valutare manutenzione, compatibilità e benefici sul desktop RDP prima di aggiungere dipendenze.

`System.Windows.Forms` può essere utile per monitor, input assistito o clipboard; la funzionalità da realizzare, non il fatto che sia stata usata in uno script PowerShell, decide se aggiungerla al progetto. Nel progetto worker servono inoltre un client Azure per il modello configurato e `HttpClient` per le API della console: la parte di **orchestrazione** è precisata nelle sezioni seguenti e non sostituisce lo strato UI qui definito.

## 9. Processo worker e moduli C#

L’esecuzione è un’applicazione **`net10.0-windows` nella sessione interattiva della VM Windows 11**. Si avvia automaticamente al login Windows via RDP, per esempio tramite attività pianificata configurata per l’utente loggato e **«Esegui solo se l’utente ha effettuato l’accesso»**. Non è un servizio Windows in Session 0. Deve impedire una seconda istanza che controlli lo stesso desktop, anche se riceve richieste concorrenti o il browser dell’utente ha più schede aperte.

| Modulo proposto | Responsabilità |
|---|---|
| `SessionHost` | Avvio al login, configurazione DPI prima della creazione di finestre, identificazione VM/sessione Windows, singola istanza, verifica preliminare di desktop e screenshot |
| `SamProcessController` | Individuare o avviare SAM con working directory configurata, effettuare il login applicativo, riconoscere caricamenti e form pronte, effettuare logout alla chiusura ordinaria; mai equiparare `Process.Responding` a test riuscito |
| `DesktopObserver` | Enumerazione top-level nativa per PID e UI Automation, gerarchia MDI, focus, form modali, tooltip, menu, controlli visibili e pattern supportati |
| `ImageObserver` | Acquisizione PNG nel desktop remoto, ritaglio/ingrandimento, OCR, associazione della cattura a titolo, HWND, monitor, DPI, ora e stato del foreground |
| `InputDriver` | Azioni serializzate su un controllo verificato; input reale, pattern UIA dove funzionano, dialoghi Windows, selezione/tastiera, verifica del bersaglio dopo variazioni della UI |
| `OutputInspector` | Per i flussi che producono file, verifica esistenza, tipo reale, dimensione e contenuto; distingue il file effettivo dall’estensione o dal solo messaggio di successo |
| `AgentRunner` | Dialogo con il modello Azure configurato sul worker, scelta degli strumenti, piano naturale, postcondizioni, classificazione errori, pause, estensioni e interruzioni |
| `ConsoleTransport` | Client HTTP in uscita verso la Console Collaudo Server, registrazione e heartbeat, long polling dei comandi, `command.ack`, eventi JSON, upload binario, ritenti e riconnessione |
| `EvidenceStore` | Coda temporanea locale dei file/eventi non consegnati e tentativo di consegna prima della chiusura; eventuali file mancanti sono indicati nel resoconto finale, copia autorevole nella root del server, non nella VM |

La UI viene pilotata da **un solo esecutore di azioni alla volta**. Screenshot, OCR e chiamate al modello possono essere asincroni, ma non devono generare input concorrente nello stesso desktop. Alcune chiamate cross-process UIA/Win32 potrebbero restare in attesa se SAM non risponde: imporre timeout e diagnosi dello stato; valutare l’isolamento dei provider UIA in un processo ausiliario qualora il prototipo dimostri blocchi non recuperabili. Un timeout **non** giustifica il doppio invio di Salva, Elimina o altri comandi con effetti persistenti.

SAM e worker devono essere nella **stessa sessione Windows e con livelli di privilegio compatibili**. Verificare queste condizioni e la capacità di catturare un’immagine leggibile **prima di segnalare `ready`** al server. Il worker legge una VM prenotata ma **non decide da solo** se essa sia disponibile nel pool.

## 10. Contratto interno degli strumenti offerti al modello

L’agente nel worker usa un deployment Azure configurato sul worker fra quelli inizialmente previsti (`gpt-6-sol`, `gpt-6-luna`): l’utente **non** sceglie un modello in `test.start`. Un eventuale router futuro può cambiare questa decisione senza modificare le azioni UI. La capacità effettiva di leggere direttamente immagini dipende dal deployment e va verificata; **UI Automation e OCR devono funzionare anche senza di essa**. Il worker traduce la descrizione naturale del piano in chiamate a strumenti locali e osservazioni successive; non consegna al modello il potere di assumere che un clic abbia avuto effetto. Il server recapita le richieste dell’utente e conserva i risultati, ma non sceglie le azioni su SAM: queste sono decise dall’agente Azure integrato nel worker.

| Strumento dell’agente | Input essenziale | Output/postcondizione richiesti |
|---|---|---|
| `ListWindows` / `InspectControls` | PID SAM, finestra o controllo corrente | Finestre per PID e sessione, classi, titoli, bounds fisici, visibilità, abilitazione, pattern UIA; indicazione distinta di finestre attive, sottostanti e modali |
| `GetFocusedElement` / `ReadTooltip` | Finestra/posizione corrente | Focus effettivo e suggerimento riconosciuto, senza dedurre dal solo tooltip l’azione della toolbar |
| `CaptureScreen` / `RecognizeText` | Monitor, finestra o regione, eventuale ingrandimento | File PNG originale, regioni OCR con coordinate, titolo e foreground, esito della cattura; testo OCR marcato come tale e non confuso con proprietà strutturate |
| `ClickControl` / `TypeIntoControl` / `SendKey` | Bersaglio risolto dalla UI attuale, azione, eventuale testo | Risultato dell’input **e** nuova osservazione: finestra, focus, valore mostrato, dialogo o dati cambiati; niente riuso cieco di HWND o coordinate |
| `NavigateMenu` / `ReadDialogs` | Menu o finestra modale corrente | Voci/etichette, pulsanti, messaggio esatto e nuovo stato dopo la scelta; stop se il contesto è ambiguo |
| `WaitForState` | Postcondizione attesa e timeout | Stato raggiunto, stato transitorio o motivo del timeout; distinzione tra processo rispondente e UI pronta |
| `StartSam` / `EndSam` / `UploadArtifact` | Sessione di collaudo e azione autorizzata | Login o logout verificato, oppure `artifactId` dopo conferma dell’upload sul server |

Ogni risultato degli strumenti riporta **origine (`native`, `uia`, `ocr`), data UTC e coordinate/DPI** quando applicabili. I risultati OCR incerti devono mantenere il riferimento all’immagine per una verifica successiva. Il worker limita il testo letto dai campi riservati: non invia credenziali SAM a screenshot, diario del collaudo o prompt del modello quando non servono all’operazione di login.

## 11. Dialogo con il server e ciclo di vita del collaudo

Il worker **non dialoga direttamente con il browser**: chiama in uscita le API HTTPS del server usando la propria identità tecnica della VM. Le firme definitive sono in [contratti-api.md](contratti-api.md); i messaggi applicativi sono in [test_language_schema.json](test_language_schema.json).

1. **Registrazione e prontezza:** dopo login Windows, identificare sessione, SAM eventualmente già aperto, capacità UIA/OCR/cattura; registrare l’istanza worker e comunicare heartbeat e stato `ready`, `busy`, `paused` o `unavailable`. Un heartbeat assente non libera automaticamente la VM.
2. **Avvio:** ricevere tramite long polling il comando `test.start` per la prenotazione e il `sessionId` assegnati. Verificare `messageId` per non eseguirlo due volte; inviare `command.ack` **prima** di eseguire il comando. L’ack significa presa in carico, non login riuscito. Se la richiesta non può essere presa in carico, inviare ack `rejected` con motivo e `worker.notice` con domanda e `requestId`, poi attendere chiarimento o intervento manuale; la risposta consente di riprovare nella stessa sessione, senza rieseguire automaticamente il vecchio comando. Se serve una nuova password, il worker la preleva dalla specifica API riservata **prima** dell’ack della risposta chiarificatrice. Il nome utente SAM rimane nello storico, la password è usata solo per il login e non storicizzata negli eventi o negli artefatti; se il server riparte prima della consegna dell’avvio, attende il reinserimento da parte dell’utente.
3. **Esecuzione:** per ciascuna azione osservare → scegliere → agire → attendere → verificare. Inviare `step.event` con `stepId`, testo naturale breve, osservazioni e riferimenti alle evidenze già caricate. Il server assegna `serverSequence` e conserva l’ordine della conversazione.
4. **Intervento e ripresa:** davanti a una situazione che richiede una decisione inviare `worker.notice` con `requiresResponse`, `requestId`, domanda e, se utile, screenshot. Mettere in pausa le azioni. Dopo `user.reply` di tipo `clarification` e descrizione di eventuali azioni manuali, **rileggere da capo la schermata** prima di procedere. Un’`instruction` spontanea può essere applicata durante il test a un punto sicuro.
5. **Attività terminate:** inviare `test.activities_completed` con esito e riepilogo, **senza** fare logout. Attendere `user.reply` di tipo `extension` per riprendere nello stesso collaudo oppure `test.close_requested` per chiudere. Con un’estensione rilegge SAM e produce un nuovo numero di completamento.
6. **Chiusura ordinaria:** accettare la conferma dell’utente e non inviare ulteriori domande o richieste di intervento per il collaudo. Tentare gli upload finali e il logout da SAM. Se falliscono, inviare `worker.notice` di errore con `requiresResponse: false` **prima** di `test.closed`, indicando problemi ed evidenze mancanti; inviare comunque `test.closed` con gli esiti effettivi e `manualInterventionRequired: true`. L’utente sistema manualmente eventuali sessioni SAM rimaste aperte prima di liberare la VM **o avviare un altro test**; il worker torna `ready` per un nuovo test solo dopo una nuova verifica del desktop e dello stato di SAM. Se manca il collegamento al server, non fingere una chiusura confermata: conservare temporaneamente i messaggi e riprovare dopo riconnessione.
7. **Interruzione:** trattare `test.interrupt_requested` con priorità prima di nuove azioni UI, fermarsi al primo punto controllabile e inviare `test.interrupted` con situazione SAM ed evidenze mancanti. Se il worker è bloccato o irraggiungibile, la console può registrare un’interruzione manuale **senza inventare alcun messaggio del worker**: dopo la ripresa del collegamento il worker non deve continuare una sessione ormai terminale.

Il worker verifica tramite l’API di stato a lui riservata l’eventuale chiusura o interruzione manuale **prima di riprendere dopo perdita della connessione** e prima di nuove azioni sulla UI; se il test è terminale non continua. Se il server non è raggiungibile, sospende nuove azioni sulla UI e conserva temporaneamente eventi e file non consegnati; al ripristino sincronizza comandi e stato. La coda long polling trasporta richieste dell’utente, non singole azioni su SAM: il cursore avanza solo per comandi di cui il server ha registrato l’ack o la cancellazione; i comandi non confermati sono riproposti con lo stesso `messageId`. Per `step.event` e comandi usa `messageId` stabile ai ritenti e il relativo `Idempotency-Key`; per gli upload usa un’ulteriore chiave di idempotenza così che un ritentativo non crei un secondo `artifactId`. Il successo HTTP è diverso dall’avvenuta esecuzione dell’azione in SAM.

Gli screenshot sono file **binari separati** dagli eventi JSON. Prima dell’evento che li cita, il worker produce `eventMessageId`, carica file e metadati nella root del server, riceve `artifactId`, poi include l’ID nel messaggio: un file non confermato non è una prova disponibile al client. Il formato delle schermate, il percorso temporaneo e i ritenti non devono richiedere condivisioni di rete tra worker e server.

## 12. Errori, evidenze e controllo dell’affidabilità

| Situazione | Regola esecutiva del worker |
|---|---|
| **Validazione operativa** (dato richiesto o scelta incoerente) | Leggere il testo della modale e il contesto, fornire o correggere un dato consentito dal piano; chiedere all’utente se la scelta non è determinabile. Non classificarla automaticamente come difetto. |
| **Messaggio non bloccante** | Acquisire testo esatto, finestra e schermata **prima di chiuderlo**; verificare eventuali dettagli, scegliere il pulsante appropriato e continuare dai dati realmente presenti. |
| **Blocco applicativo** | Conservare l’ultimo stato attendibile, screenshot e passaggi; sospendere input, inviare `worker.notice` alla console e attendere intervento. Non inviare clic ripetuti per forzare l’avanzamento. |
| **Errore dello strumento/desktop** (focus errato, DPI, finestra coperta, timeout) | Separare la capacità di SAM da quella dell’automazione: verificare PID, sessione, foreground e pixel prima di attribuire un problema a SAM. Ripetere solo dopo avere identificato la precondizione mutata. |
| **Salvataggio ambiguo o copia parziale** | Controllare se l’operazione ha già prodotto effetti persistenti, rinfrescare e riaprire quando consentito. Un avviso o una chiusura con Annulla non costituiscono prova di rollback. |

Gli eventi di errore riportano **passo, gravità, testo naturale breve per la UI, messaggio originale, stato osservato, postcondizione mancata e screenshot associato**. **Il worker non deve inventare cause sulla base del solo errore di un’azione UI**: deve fornire evidenze e contesto alla console. 

Verificare sempre il contenuto delle immagini: hash diverso e screenshot salvato **non bastano** per dire che una funzione sia riuscita. Occorre una differenza semantica attesa — form aperta, dialogo chiuso, riga cambiata, dato riletto — confrontata con il passo svolto. Prima di un’azione potenzialmente persistente, acquisire una baseline; dopo, verificare l’effetto e registrare dati di prova eventualmente creati.

## 13. Verifica minima del prototipo worker

Per considerare consistente lo strato di interazione, ripetere in **una VM Windows 11 con SAM** almeno queste prove, registrando prima/dopo e l’effetto osservato:

1. Login Windows via RDP → avvio worker nella **stessa sessione** → avvio/login SAM → riconoscimento dell’ambiente richiesto.
2. Enumerazione completa di finestra principale, MDI, wizard e dialogo proprietario, confrontando UIA Root/Children con `EnumWindows` per PID.
3. Riconoscimento della griglia personalizzata, intestazioni, prime righe e contatore estratto; navigazione a righe successive con tastiera e verifica del cambiamento.
4. Campo di ricerca con selezione reale di un’opzione, possibile sostituzione del suo HWND, digitazione e conferma; distinzione fra testo visibile e dato effettivamente applicato.
5. Tooltip e popup con submenu; navigazione corretta senza usare coordinate memorizzate dalla precedente apertura.
6. Messaggio applicativo, scelta file e dialogo stampante: lettura completa e annullamento verificato, senza creare output di prova non richiesti.
7. DPI 100%, 125% e un monitor aggiuntivo: stessa azione mirata al medesimo controllo con coordinate coerenti; modali non coperte dalla finestra principale.
8. `mstsc` piccola e coperta sul PC dell’utente: screenshot e OCR acquisiti **nella VM** devono restare leggibili. Verificare rilevazione e pausa se la cattura diventa vuota, statica o riferita alla finestra sbagliata.
9. Dopo ogni azione: attesa della postcondizione, eventi con ID della sessione e screenshot consegnati alla console server; un timeout non deve produrre un secondo salvataggio cieco.

La verifica con RDP minimizzata, disconnessa o sessione Windows bloccata è **distinta** e non è un requisito dimostrato da queste prove. Le librerie definitive e la strategia di cattura possono essere confermate soltanto dopo il prototipo nella VM prevista.
