# M-ERP-03COM-Front End 6.0

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 1 -->
# GUIDA OPERATIVA

# Modulo Front End Clienti

# (Vendita a Banco)

# Area Commerciale

# V.6.0

## Destinato a Clienti, Partner e Consulenti Centro Software

## Team Education

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 2 -->
## Sommario

## Premessa ................................................................................................................. 3

## Impostazioni ............................................................................................................ 4

### Opzioni Vendita a Banco ..................................................................................................4

### Opzioni Front End Negozio ..............................................................................................5

### Listino di Vendita .............................................................................................................6

### Anagrafica articolo ...........................................................................................................7

### Parametri SAM.CFG .........................................................................................................8

## Flusso operativo ....................................................................................................... 9

### Modalità Vendita a banco ...............................................................................................9

### Modalità Front End Negozio ..........................................................................................12

### Generazione documenti ....................................................................................... 19

### Contabilizzazione Documenti ............................................................................... 19

### Generazione Scontrino con stampante fiscale ..................................................... 20

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 3 -->
# Premessa

Le funzioni necessarie per la gestione dei punti vendita comprendono la cosiddetta gestione del “Front-End Clienti” e l’interfacciamento con i registratori di cassa.

Il FRONT-END CLIENTI di SAM ERP2 comprende le funzioni che consentono la compilazione rapida e automatica dei documenti di vendita (ordini, DdT, ricevute e fatture) tramite lettura dei codici a barre di ogni articolo o lotto.

L’interfaccia con i REGISTRATORI di CASSA consente invece la comunicazione diretta tra il sistema centrale e le casse: SAM ERP2 invia al registratore (con sincronia predefinita) i dati relativi agli articoli e ai clienti, e al contempo riceve in automatico i dati relativi agli articoli venduti e agli scontrini cartacei emessi, dati che vengono utilizzati sia per l’area contabile (movimenti prima nota) sia per la movimentazione del magazzino (movimenti merci).

La struttura dei file che vengono trasmessi è configurabile in base ai modelli di registratori usati. L’interfacciamento avviene quindi in entrambe le direzioni; i dati vengono trasferiti sia dal gestionale verso il registratore di cassa che nella direzione opposta.

Dal sistema SAM ERP2 si possono innanzitutto inviare verso il registratore di cassa, e successivamente allineare in modo automatico, i dati relativi agli ARTICOLI; opzionalmente possono anche essere inviati anche i dati relativi ai CLIENTI.

Nella direzione opposta, e cioè dal registratore di cassa verso SAM ERP2, si ottengono i dati relativi alla parte contabile (movimenti di prima nota) e alla movimentazione delle merci (movimenti di magazzino).

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 4 -->
# Impostazioni

Esistono 2 modalità di funzionamento della Vendita a banco:

- Modalità standard con uso di PC -> Vendita a banco

- Modalità che prevede l’utilizzo di monitor touch screen -> Front End Negozio

# Opzioni Vendita a Banco

Area Commerciale/Vendita a Banco/Vendita a banco:

• CODLISDEF - Codice Listino di default: consente di indicare il Codice di Listino di Vendita da utilizzarsi in sede di reperimento prezzo durante l‘inserimento nell’interfaccia ‘Vendita a Banco’; se questa Opzione è valorizzata ha priorità maggiore rispetto al cambio prezzi ivato/netto indicati in Anagrafica Codice Articolo;

• VBSOLOARTINLIST - Utilizza solo articolo presenti nel listino: se indicato il listino all’opzione precedente nella ricerca degli articoli dalla vendita a banco verranno estratti solo gli articoli presenti all’interno del listino indicato;

• VBCONTOCLIDEF - Conto cliente di default: è il conto del cliente che si vuole venga proposto in automatico all’ingresso nella gestione, solitamente si utilizza un conto legato ad un cliente (controparte) generico;

• VBUSAF2ART - Utilizza TF2 Articolo nella Vendita a Banco: tramite l’attivazione è possibile attivare la ricerca dell’articolo con l’oggetto standard a tendina;

• VBUSALOTTI - Gestione Lotti nella Vendita a banco: attivando questa opzione è possibile gestire i lotti anche dalla vendita a banco;

• VBCERCART1/8 - Priorità ricerca articolo: si tratta di una scaletta di 8 posizioni in cui si può indicare come deve essere eseguita la ricerca dell’articolo (ad es.: per barcode, per codice, ecc.);

• PATHVBBEFORE - Percorso salvataggio file XML generato prima della creazione documento: indicare il percorso in cui salvare il file XML utile all’interazione con il registratore di cassa/stampante fiscale. In questo caso la valorizzazione di questa opzione, significa che il file XML viene compilato in funzione di quanto specificato nel corpo del Front End Clienti (ossia in tabella temporanea) e prima della generazione del documento;

• VBTSXMLBEFORE - Aggiungi data ed ora nel file XML generato prima della creazione documento: i file XML riportano nel loro nome anche la data/ora di creazione. L’attivazione di questo parametro consente a ogni transazione di avere il rispettivo file XML;

• PATHVBAFTWER - Percorso salvataggio file XML generato dopo la creazione documento: indicare il percorso in cui salvare il file XML utile all’interazione con il registratore di cassa/stampante fiscale. In questo caso la valorizzazione di questa opzione, significa che il file XML viene compilato dopo la generazione del documento e con il contenuto del documento;

• VBTSXMLAFTER - Aggiungi data ed ora nel file XML generato dopo la creazione documento: i file XML riportano nel loro nome anche la data/ora di creazione. L’attivazione di questo parametro comporta che ogni transazione abbia il rispettivo file XML;

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 5 -->
• VBVISDATITD - Visualizza impostazioni testata e fondo documento in creazione: abilitato di default, in sede di creazione del documento di vendita (Ricevuta, Commessa, DDT, Fattura) è visualizzata la testata ed il fondo documento;

• VBCREADOCDEF - Crea documenti in stato definitivo: i documenti generati tramite l’interfaccia (Ricevute, DDT, Fatture) sono già in stato definitivo;

• VBTOTMISFISC - Calcola totali in modalità misuratore fiscale: calcola i totali in modalità misuratore fiscale

• VBQTAUMPRZ - Proposta quantità riferita UM prezzo: se per l’articolo è definita un’unità di misura prezzo diversa dall’unità di misura di vendita, la quantità in input viene convertita secondo il coefficiente di conversione in essere tra UMV e UM prezzo;

• VBBOLD - Visualizza Caratteri in Grassetto: i caratteri delle etichette dell’interfaccia sono esposti in grassetto;

# Opzioni Front End Negozio

Con la modalità ‘Front End Negozio’ le opzioni sono inferiori perché questa modalità implica una gestione più semplice e immediata.

Area Commerciale/Vendita a Banco/Front end negozio:

• VENBANCONCLIDEF - Conto cliente di default: è il conto del cliente che si vuole venga proposto in automatico all’ingresso nella gestione, solitamente si utilizza un conto legato ad un cliente (controparte) generico;

• VENBANTSXMLBEFORE - Aggiungi data ed ora nel file XML generato prima della creazione documento: i file XML riportano nel loro nome anche la data/ora di creazione. L’attivazione di questo parametro consente a ogni transazione di avere il rispettivo file XML;

• VENBANTSXMLAFTER - Aggiungi data ed ora nel file XML generato dopo la creazione documento: i file XML riportano nel loro nome anche la data/ora di creazione. L’attivazione di questo parametro comporta che ogni transazione abbia il rispettivo file XML;

• PATHVENBANBEFORE - Percorso salvataggio file XML generato prima della creazione documento: indicare il percorso dove salvare il file XML utile all’interazione con il registratore di cassa/stampante fiscale. In questo caso la valorizzazione di questa opzione, significa che il file XML viene compilato in funzione di quanto specificato nel corpo del Front End Clienti (ossia in tabella temporanea) e prima della generazione del documento;

• PATHVENBANAFTER - Percorso salvataggio file XML generato dopo la creazione documento: indicare il percorso in cui salvare il file XML utile all’interazione con il registratore di cassa/stampante fiscale. In questo caso la valorizzazione di questa opzione, significa che il file XML viene compilato dopo la generazione del documento e con il contenuto del documento;

• VENBANRICART1/8 - Priorità ricerca articolo: si tratta di una scaletta di 8 posizioni in cui si può indicare come deve essere eseguita la ricerca dell’articolo (ad es.: per barcode, per codice, ecc.);

• VISIMGART - Visualizza immagine articolo alla selezione del codice: permette la visualizzazione dell’immagine indicata nell’anagrafica dell’articolo.

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 6 -->
# Listino di Vendita

In Area Commerciale -> Listini di Vendita bisogna creare il listino dedicato alla Vendita a banco, che solitamente è un listino ‘IVA compresa’:

Sulle righe di listino dovranno poi essere indicati i vari articoli con prezzo (comprensivo di IVA se spuntato IVA compresa):

Il listino creato dovrà essere poi inserito nell’opzione CODLISDEF– Codice listino di default.

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 7 -->
# Anagrafica articolo

Per poter ricercare correttamente il prezzo, ogni articolo dovrà avere in anagrafica specificato:

In alternativa alla gestione del listino di vendita si possono inserire i prezzi direttamente in anagrafica articolo.

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 8 -->
# Parametri SAM.CFG

Oltre alle opzioni esistono alcuni parametri gestibili nel SAM.CFG che possono modificare il funzionamento della Vendita a banco.

[Vendite]

VenBanNeg=-1

Questo parametro consente di attivare la funzionalità ‘Front End Negozio’ con utilizzo di monitor touch screen (modelli specifici). La vendita a banco e la vendita a banco negozio touch screen sono mutualmente esclusive, ossia non sono utilizzabili contemporaneamente: l’azienda decide quale delle due utilizzare.

### [M_VENDITABANCO]

•

### GEST_IVA_COMP=S

Per gestire prezzi con IVA anche se inseriti manualmente. •

### NONRICHIEDISTAMPA=-1

Alla creazione del documento non compare la domanda se si vuole stampare il documento. •

### DOC_IVA_COMP_GEST=S

Permette di impostare IVA compresa oppure IVA esclusa nelle righe dei documenti creati dal Front End clienti coerentemente con l'impostazione 'GEST_IVA_COMP' oppure con quanto indicato nel listino. •

### ABILITAQTANEG=-1

Permette di impostare quantità negative nelle righe dei documenti creati dal Front End clienti

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 9 -->
# Flusso operativo

# Modalità Vendita a banco

Accedendo alla voce di menu ‘Front End clienti’, senza l’attivazione di alcun parametro Sam.cfg, si avvia la procedura di ‘Vendita a banco’ standard:

Se si entra nell’ERP con l’utente CSADMIN si vedrà attiva l’icona di configurazione finestra che permette di indicare quali campi si vogliono vedere o non vedere a video in questa form.

L’utente potrà stabile quali campi vedere per poterli inserire/modificare e quali invece non sono necessari:

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 10 -->
Cliente : deriva dall’opzione ‘Codice cliente di default’ ma può essere modificato.

Articolo: La ricerca dell’articolo avviene a seconda di come impostata l’opzione ‘Utilizza TF2 articolo’:

• se attiva sarà attivato il TF2 con la ricerca standard dell’ERP,

• se l’opzione non è attiva allora si presenterà una casella in cui inserire:

o il codice a barre come ricerca prioritaria (indicato in anagrafica articolo),

o oppure il codice o la descrizione dell’articolo,

La ricerca è dinamica via via che si digitano codici o descrizione.

Il resto dei campi è visibile o meno a seconda delle scelte fatte, i campi necessari e non opzionabili, sono:

• Quantità

• Prezzo (che può essere digitato o derivare da listino)

L’inserimento della riga avviene automaticamente dopo essere passati sull’ultimo campo disponibile e compilabile: procedere col tabulatore --->I e all’uscita dal campo ‘Rif3’ viene inserita la riga:

Meno campi ci sono visibili, se non necessari, più veloce sarà l’inserimento delle righe.

Per attivare la ricerca ‘per contenuto in’ si può premere F3 (con il medesimo pulsate si attiva e si disattiva).

I campi presenti nella griglia delle righe possono essere spostati con trascinamento non essendo presente la funzionalità di ‘layout’.

Se si vuole lavorare con i barcode è sempre possibile sostituire la tastiera con un lettore di codici a barre.

Terminato l’inserimento degli articoli si può procedere a generare i documenti Ricevuta, Offerta, Commessa, DDT, Fattura, tramite i pulsanti o i relativi tasti funzione, oppure predisporre un nuovo inserimento (F12-NUOVO) o, infine, uscire (Esc - Chiudi).

Generazione Documenti

Le form di generazione documenti sono semplificate e contengono i pochi dati di testata e fondo che può essere utile completare.

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 11 -->
Cliccando sul tasto SALVA il documento verrà creato, salvato nelle relative gestioni, e verrà chiesto se si desidera stamparlo:

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 12 -->
Se viene impostato il relativo parametro nel Sam.cfg non verrà fatta la domanda (ad esempio, se il front end viene usato per un inserimento veloce di commesse o offerte l’utente potrebbe non essere interessato a stampare ogni volta il documento).

È possibile, inoltre, tramite personalizzazione, stampare lo scontrino su stampante fiscale (in questo caso occorre fare un’analisi per definire l’interfacciamento del registratore di cassa o stampate fiscale utilizzata dal cliente).

# Modalità Front End Negozio

Per attivare questa modalità operativa occorre impostare il seguente parametro nel Sam.cfg

[Vendite]

VenBanNeg=-1

Chi sceglie l’utilizzo di questa modalità deve disporre di monitor Touch Screen con determinate caratteristiche tecniche.

Accedendo alla funzionalità di menu ‘Front End Clienti’ con il parametro indicato nel Sam.cfg troverà questa videata:

Nella sezione ‘Configurazione’ sarà riportato il cliente che è indicato nelle opzioni aziendali:

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 13 -->
Di conseguenza il pagamento indicato in anagrafica di questa controparte.

Configurazione Pulsanti e sezioni

La procedura si apre direttamente su 3 Tabsheet che contengono una serie di ‘pulsanti’ che possono essere configurati con l’indicazione di uno specifico articolo:

Per procedere alla configurazione ci si posiziona sulla sezione che si vuole personalizzare e sulla ‘zona’ dei pulsanti e si esegue clic con il tasto destro del mouse:

Cliccando su ‘Configura’ si aprirà una form di configurazione in cui l’utente potrà personalizzare sia il titolo della scheda che il contenuto dei pulsanti:

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 14 -->
Ogni singolo TF2 aprirà l’elenco degli articoli permettendo di abbinare un articolo e un colore ad ogni pulsante.

Sarà possibile anche personalizzare la descrizione che apparirà sul pulsante e il colore che si vuole dare al pulsante

Terminata la configurazione desiderata e confermado con OK si potrà uscire, vedere il risultato ottenuto e procedere a configurare altro Tabsheet:

Utilizzo pulsanti e tastierina

Per elencare i prodotti acquistati dall’utente si potrà quindi ‘premere’ sui pulsanti creati:

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 15 -->
L’articolo verrà inserito automaticamente, con un prezzo (derivante da un listino) e nel riquadro in basso a destra si potrà vedere l’immagine dell’articolo (se desiderato e se presente in anagrafica)

oppure il semplice elenco degli articoli.

In alternativa è sempre possibile utilizzare la lettura del codice a barre direttamete nel campo ‘Articolo’:

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 16 -->
E’ possibile anche la digitazione di descrizione o codice articolo, tenendo sempre presente la scaletta di priorità di ricerca che si è impostata nelle opzioni.

Il campo relativo alla quantità è impostato direttamente a ‘1’ e risulta disattivato.

E’ possibile cancellare una delle righe inserite , selezionadola e tasto CANC

La tastierina numerica sulla destra riporta tasti simili a quelli che si trovano sui normali registratori di cassa:

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 17 -->
• Q: Quantità

• PR: Prezzo

• SR: Sconto Riga

• ST: Sconto Testata

• PA: Pagamento

• R: Reso

Per inserire le operazioni occorre rispettare alcune regole e ordine di inserimento.

Sconto: SR

Occorre inseire il valore dello sconto che si desidera e poi premere il tasto ‘SR’ (per lo Sconto di Riga) :

Successivamenre bisogna selezionare l’articolo:

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 18 -->
Il prezzo indicato, che apparirà nella griglia in basso a destra, sarà già comprensivo dello sconto.

Reso: R

Nel caso di un reso si dovrà premere prima il tasto relativo al reso ‘R’ e a seguire l’articolo

Ottenendo quindi la riga negativa.

Pagamento: PA

Inserendo la cifra di contante ricevuta e cliccando sul tasto PA, si otterrà il conteggio del resto:

Quantità: Q

Se si desidera modificare la quantità basterà indicare la quantità in numero e premere il tasto ‘Q’

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 19 -->
Inserendo l’articolo si otterrà la quantità desiderata.

# Generazione documenti

È possibile generare documenti con i tasti presenti nella sezione sottostante:

La generazione è più veloce e non richiede l’inserimento di nessun dato.

Il tasto ‘Ann. Sc.’

serve per annullare lo scontrino in corso, ossia resettare eventuali righe già

inserite.

# Contabilizzazione Documenti

Considerato che dalla vendita a banco, qualsiasi modalità, è possibile generare dei documenti di tipo Ricevuta Fiscale – Scontrino e quindi la contabilizzazione deve generare delle registrazioni di corrispettivi, sarà necessario compilare anche la sezione delle opzioni relativa alla gestione dei Corrispettivi:

<!-- source: M-ERP-03COM-Front End 6.0.pdf; page: 20 -->
# Generazione Scontrino con stampante fiscale

Per l’utilizzo della modalità Front End negozio e la stampa degli scontrini fiscali occorre integrare la procedura standard con la parte di personalizzazione che può variare in funzione della stampante fiscale scelta o che si desidera utilizzare.

Per questo quindi si rimanda ad apposita analisi e quotazione.
