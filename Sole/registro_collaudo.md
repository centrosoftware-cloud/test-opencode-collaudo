# Registro di collaudo — Clienti SAM ERP

Data: 9 ottobre 2026. Eseguibile: `C:\OutSamW2_12_Develop\SAM.exe`.
Ambiente indicato dalla UI: **Database collaudo**. Utente: **FILIPPO**.

Questo registro raccoglie percorsi, esiti, duplicazioni e anomalie. Le segnalazioni GitHub sono censite in [issue_list.md](issue_list.md); il manuale utente contiene le sole istruzioni operative.

## Sessione

- Avvio di una nuova sessione SAM e navigazione richiesta dal menu laterale.
- Apertura Clienti: 105 righe, stati 98 attivi / 4 in contenzioso / 3 bloccati.
- Popup griglia rilevato e acquisito in `02-popup-griglia.png`; accessi duplicati con toolbar identificati.
- Popup > Numero Righe: messaggio «Numero di righe estratte: 105». Chiuso.
- Nuovo: precompilazione tramite partita IVA o codice fiscale; scelto «Annulla precompilazione» per l’inserimento manuale.
- OK senza ragione sociale: validazione «È necessario specificare una Ragione Sociale». Non è un bug; chiusa e proseguito.
- Inserimento iniziale tramite WM_SETTEXT: testo visibile ma validazione della ragione sociale ancora negativa. Ripetuto tramite tastiera e Tab: validazione superata. Problema dello strumento di automazione, non attribuito a SAM.
- OK con ragione sociale ma senza identificativi fiscali: conferma «ATTENZIONE!! Non è stato specificato né il Codice Fiscale né la Partita IVA. Continuare?». Scelto Yes.
- Assegnato codice 000219; avviso «Non tutti i campi personalizzati indicati come obbligatori sono stati compilati. Salvataggio Campi personalizzati NON effettuato». Chiuso con click su OK; ritorno alla griglia, ancora 105 righe prima di rinfresco. Persistenza in verifica.

## Esiti successivi e dati persistiti

- Rinfresco: totale **106**, con **99 attivi / 4 in contenzioso / 3 bloccati**. Il cliente **000219** è presente.
- Prima riapertura del cliente: i valori impostati tramite WM_SETTEXT non erano tutti persistiti. Successiva modifica tramite messaggi di input ai controlli: salvati indirizzo, CAP, comune ed e-mail; ragione sociale 2 rilevata nella griglia come «Collaudo operativo agente».
- Riapertura finale del cliente: verificati ragione sociale principale, codice 000219, Via del Collaudo 9, CAP 40100, Bologna. Ragione sociale 2 rilevata nella griglia; e-mail immessa ma persistenza finale non riletta come valore strutturato. Non tutti i campi sono stati esportati in un dataset strutturato verificato.
- Apertura di tutte le schede principali del cliente e delle dieci sottoschede commerciali. Acquisite schermate 29–49; nessun inserimento nelle griglie figlie.
- Consensi: finestra inizialmente fuori schermo, spostata alle coordinate 550,220. Provato storico/attivi sui generici e sugli specifici per cliente. Nessun consenso creato.
- Storico Ragione Sociale: form aperta sul cliente di prova, nessuna variazione creata.
- Trascodifica Articoli: aperta Codifiche Articoli Clienti, nessuna codifica inserita.
- Copia: trasferimento tramite Copia > Nuovo > Incolla. Sorgente effettivamente visualizzata dopo Incolla: **AMAZON EU S.A R.L.**, non il cliente di prova inizialmente inteso. La form è stata annullata, evitando di salvare un duplicato. La selezione iniziale era stata alterata da rinfreschi/errori: verificare il titolo della sorgente prima di copiare.
- Eliminazione: conferma aperta; scelto No. Nessuna eliminazione confermata.

## Anomalie e riproduzione

### A01 — Errore SQL: IBANValidation

Testo nativo:

```text
[FireDAC][Phys][ODBC][Microsoft][ODBC Driver 18 for SQL Server][SQL Server]
Il nome di oggetto 'IBANValidation' non è valido.
```

**Non bloccante per il processo**, ma impedisce diversi flussi. Chiusura con OK; SAM resta rispondente. Non è stata verificata la struttura del database e non è attribuita una causa certa.

| Sequenza osservata | Risultato / recupero |
|---|---|
| Griglia Clienti > rinfresco o selezione di riga | Uno o più messaggi A01; OK consente di proseguire, griglia talvolta aggiornata |
| Popup > Ordinamento decrescente | Ordinamento visibile (Zucchini Garden / ZOCCARATO…); Warning sulla selezione dati, poi possibile messaggio A01 |
| Popup > Ricerca Avanzata | A01 prima della form successiva |
| Configura o Filtri dopo selezione dati | Warning «Errore durante la selezione dei dati…» in alcuni tentativi; in altri la form si apre |
| Azioni Multiple > Azione > Solo righe selezionate > OK | A01 |
| Azioni Multiple > Mail > Solo righe selezionate > OK | A01 |
| Azioni Multiple > Appuntamento > Solo righe selezionate > OK | A01 |
| Azioni Multiple > Allegato > Solo righe selezionate > OK | A01 |
| Azioni Multiple > Work Flow > Solo righe selezionate > OK | A01 |
| Azioni Multiple > NewsLetters > Solo righe selezionate > OK | A01 |
| Prepara Campagna > Solo righe selezionate > OK | A01 |
| Geo Marketing > Solo righe selezionate > OK | Messaggio «Non è stato possibile accedere ai servizi di geo marketing»; Dettagli mostra A01; OK recupera |
| Dettaglio IBAN con riga scelta | A01; senza selezione adeguata viene mostrata la normale validazione «Selezionare una controparte» |
| Esporta > TXT > Salva con nome > Salva | File generato, poi A01; contenuto non validato |
| Esporta > XLSX senza Excel > Salva con nome > Salva | File generato, poi A01; contenuto non validato |
| Grafico > etichetta Ragione Sociale 1 + conteggio > Crea Grafico | Form Grafico: Clienti aperta, valore 96 nel controllo Numero elementi; A01 e grafico non validato |
| Popup > Somma valori campo selezionato | A01; nessun totale numerico validato |
| Navigatore > Clienti, Schede Contabili, Estratto conto per partita | A01 nei tentativi |
| Navigatore > Scheda Credito, Scoperto e Rischio Cliente | A01 |
| Navigatore > Interrogazioni O.V. e Offerte | A01 |
| Navigatore > Interrogazioni su Spedizioni / Fatture | A01 |

Alcuni tentativi iniziali di toolbar sono stati eseguiti mentre esistevano modali non enumerate da UI Automation Root/Children: l’attribuzione precisa del primo messaggio A01 di selezione report è meno affidabile delle riproduzioni successive. La tabella usa i percorsi verificati in seguito; non ricava automaticamente un bug distinto per ogni pulsante.

### A02 — Access violation durante ricerca automatizzata

Sequenza: aprire Ricerca dalla toolbar > tentare selezione programmatica della combo Campo di Ricerca mediante CB_SETCURSEL e notifica CBN_SELCHANGE > portare la form in primo piano > click nel valore > invio tastiera «SOLE».

```text
Access violation at address 638625C5 in module 'rtl290.bpl'
(offset 1325C5). Read of address 00000008.
```

Schermata `60-access-violation-ricerca.png`. OK chiude l’errore; successiva conferma ricerca mostra nessuna riga trovata per Partita I.V.A. Il campo non era stato realmente cambiato come inteso dall’automazione. **Da riprodurre manualmente prima di attribuirlo a un normale percorso utente.** Non bloccante.

### A03 — Campi personalizzati obbligatori

Nuovo > dati generali > OK > conferma senza identificativi fiscali > avviso di mancato salvataggio campi personalizzati. L’anagrafica principale viene salvata. Ripetuto in modifica. Potenziale comportamento da valutare rispetto ai requisiti transazionali; l’avviso in sé è esplicito e non viene classificato automaticamente come bug.

### A04 — Record in uso

Modifica di 000219 mentre un’altra sua scheda è aperta > messaggio di record in uso da Filippo Curati. Chiusura con Annulla > chiusura della scheda precedente > Modifica: accesso riuscito. Successivo Riprova senza chiusura della scheda ripropone la stessa richiesta. Classificato come controllo di concorrenza.

### A05 — Posizionamento Consensi fuori schermo

Anagrafica 000219 > Consensi: apertura a coordinate x oltre 2600, desktop corrente largo 1920. Spostata sul desktop disponibile e utilizzata. Possibile stato geometrico multi-monitor precedente, non verificato.

## Altri percorsi verificati

- Configura: Campi Disponibili / Campi Visualizzati e ricerche; Annulla. Nessun layout salvato.
- Colori: letti condizioni, inchiostro, sfondo, stili e priorità; chiusura senza salvataggio.
- Filtro: immesso nome SOLE_CLIENTI_TEST; riapertura mostra il nome. Tentativo di aggiungere condizione senza campo realmente selezionato > normale validazione campo obbligatorio. Nessun filtro operativo con condizione validato.
- Anteprima: selezione report clienti.csrx > generazione > Anteprima di stampa (Clienti), 1/13. Toolbar anteprima non esplorata completamente.
- Stampa: report anagrafica > Microsoft Print to PDF > Salva output > `clienti_stampa_test.pdf`. File esistente.
- Etichette: parametri predefiniti, 3 colonne > conferma assenza di selezione > report eticontr3.csr/.csrx > Annulla. Nessuna stampa etichette generata.
- Cruscotto: aperto su CERRI FRATELLI S.R.L.; lette campagne e sezioni. Non modificati dati.
- Dossier: browser aperto su BPMWEB; non proseguita esplorazione web. Cattura 17-dossier non valida come schermata della form SAM.
- Navigatore > Scadenzario: aperto su Zucchini Garden, Records: 1; 1.608,20 Dare, scadenza 31/12/2023.
- Navigatore > Interrogazioni su ricevute fiscali: aperta form, nessuna riga visibile; totale non verificato.
- Navigatore > Rubrica: aperta form, nessuna riga visibile; totale non verificato.

## Limiti di copertura

La matrice tecnica della prima edizione distingueva percorsi conclusi, parziali, impediti e rilevati soltanto; il manuale è stato successivamente ripulito per uso finale. Questo registro conserva gli esiti e il censimento issue conserva le segnalazioni. Non sono stati testati tutti i rami, tutti i valori possibili e tutti i popup delle griglie figlie. Gli errori chiudibili non hanno imposto l’arresto del processo; i relativi rami sono rimasti incompleti.

Stato finale: SAM PID 42652, rispondente, griglia Clienti aperta con 106 righe. Nessun messaggio di errore attivo al termine della sessione documentata. Alcune form collegate e un’anagrafica di prova restano aperte; nessun secondo cliente salvato e nessuna eliminazione eseguita.
