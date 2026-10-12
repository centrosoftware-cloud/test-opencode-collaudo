# Registro di collaudo — Scheda opportunità

Ambiente: Database collaudo, FILIPPO. Eseguibile: `C:\OutSamW2_12_Develop\SAM.exe`. Data: 09/10/2026.

## Avvio

- Nuova sessione SAM, accesso riuscito.
- Menu laterale: Area C.R.M. > CRM-I Marketing > Opportunità > Scheda Opportunità.
- Letti storico issue aperte/chiuse e linee guida agente/manuale prima delle prove.

Le prove, i dati letti e i limiti di copertura vengono registrati durante l’esplorazione. Nessun risultato è considerato verificato sulla sola apertura di una form.

## Griglia e primo inserimento

- Conteggio iniziale: 65; Proposte 18, Confermate 46, Annullate 1; Esportate 34, Non Esportate 31. Somme coerenti con totale.
- Popup rilevato: ordinamenti, ricerca contenuto/avanzata, selezione totale, numero righe, somma, grafico, CRUD, copia, filtro/layout/export/anteprima/stampa/rinfresco e Cambia Stato > Proposto/Confermato/Annullato.
- Numero Righe mostra 65, coerente con riepilogo.
- Aperto primo record: Trattativa commerciale 2026/2, CERRI FRATELLI, stato Proposta, Totale 0; riga descrittiva preesistente «pppppppppppp» senza valore. Si tratta di contenuto del database di prova, non automaticamente errore.
- Aperte Testata, Righe, Scheda commerciale, Tracking; immagini opp-03–06.
- Nuovo 2026/3: OK senza contatto mostra «Selezionare un contatto intestatario della trattativa». Validazione operativa, recuperata.
- Digitazione nel selettore del codice 000219 e Tab risolve contatto e indirizzo; F2 o parte destra del selettore ha aperto creazione controparte, annullata senza salvare.
- Riga descrittiva 2 × 100, successo 50%: dopo uscita dai campi, importo 200, pesato 100, prezzo netto 100. Coerenza aritmetica verificata.
- OK chiede memorizzazione riga pendente; Sì. Avviso assenza classificatori articolo; Sì. Nuovo OK salva testata. Rinfresco aggiorna totale 66, Proposte 19, Non Esportate 32.
- Alcuni tentativi di input da tastiera hanno attivato scorciatoie con focus non atteso: recuperate selezioni report senza pubblicare bug attribuibili a SAM. Input alternativo verificato con calcolo e successivo salvataggio; persistenza dettagli ancora da rileggere.

## Verifiche successive

- Riaperta 2026/3: contatto, nota, riga e importi persistiti. Sconto globale 10: totale documento 180 e pesato 90, coerenti con riga 200/pesato 100.
- Stati modificati dalla combo della testata: Proposta > Confermata > Annullata > Proposta. Riepiloghi filtrati aggiornati e coerenti. Cambio stato da popup tentato, ma avviso selezione non presente: ramo multiplo non completato.
- Duplica riga: ottenute due righe 2 × 100. Provati spostamenti e cancellazione; eliminate entrambe durante l’esplorazione. Ricreata una riga 2 × 100, successo 50, e salvato documento. Ultimi importi riletti in griglia: 200 / sconto 10 / totale 180 / pesato 100 / pesato documento 90.
- Scheda commerciale: tentati validità, note validità e riferimento cliente. Una voce interpretata come testo era un campo data e ha provocato «La data inserita non è valida». Modifiche scartate tramite Annulla > No; nessun bug attribuito alla validazione.
- Ricerca per Descr. Contatto = SOLE: risultato positivo senza access violation. Filtro SOLE_OPP creato su denominazione completa; estrazione singola verificata. Layout aperto/letto e annullato. Grafico su Descr. Contatto + conteggio: una opportunità rappresentata.
- Somma sulla colonna Numero dell’estrazione singola: 3, coerente con numero opportunità; non interpretata come totale economico.
- Annotazioni riga: aperta form Nota con sezioni Note/Allegati; chiusa senza modifica intenzionale.
- Copia > Nuovo > Incolla: scelta Testata e Righe, messaggio «Copiate 1 Righe». Il tentativo di uscita con No non ha impedito la presenza del duplicato 2026/4 dopo rinfresco. Non confermata la causa né una riproduzione controllata: possibile persistenza già durante Incolla. Duplicato identificato e **eliminato**, conteggio filtrato tornato a uno. Osservazione da approfondire, non segnalata come bug accertato.
- Esporta Opportunità: selezionata l’estrazione tramite Seleziona Tutto, aperta configurazione verso documenti di vendita; Testata/Righe/Proposte/Tracciabilità. Inserisci richiede campi obbligatori; scelta tipo documento non completata correttamente. Esci; nessun documento di vendita verificato come creato.
- Export TXT: opportunita_test.txt generato, contenuto non validato. Invio e-Mail/Fax aperto fino al criterio, poi annullato.
- Anteprima filtrata iniziale: Nessun dato da visualizzare. In quel momento le righe di prova erano state eliminate; non attribuito a bug di report. Anteprima Tutte le righe genera una sequenza di documenti distinti. Ripetere Alt+F4 apre l’anteprima successiva; recupero riuscito con **Interrompi i processi in corso e esci**, confermato Sì. Non è un blocco applicativo.
- Colori aperti e chiusi senza regole salvate. Ricerca Avanzata nel popup rilevata, ma clic ripetuti non hanno aperto una maschera: abilitazione/contesto da verificare.

## Problema segnalato

**#6 — Prepara Campagna, ORDER BY.** Con filtro SOLE_OPP attivo e Tutte le righe, errore SQL prima della preparazione. Ripetuto una seconda volta con stesso esito. OK permette di continuare. Non esistevano issue equivalenti nello storico.
Screenshot: immagini/opp-18-errore-campagna.png. Tentato accesso al browser per allegarlo: Chrome Incognito, GitHub mostra Sign in to comment. **Upload non completato per sessione browser non autenticata**, distinto dall’autenticazione CLI.

## Dati preesistenti e precisione

- Conteggi iniziali coerenti: 18+46+1=65; 34+31=65. Dopo nuovo record: 66, Proposte 19; Non Esportate 32.
- Dati preesistenti con descrizioni di prova e importi zero: rilevati, ma senza requisiti o dati attesi non classificati come imprecisioni applicative.
- Le percentuali aggregate della testata non sono la probabilità della singola riga: il test mostra riga 50% con indicazione testata 100%; non dimostrata incoerenza. Da chiarire definizione del riepilogo prima di segnalarla.
- Calcoli del record di prova controllati e coerenti; nessuna imprecisione aritmetica accertata.

## Copertura residua

Il collaudo **non è esaustivo**: da completare articoli codificati/unità/valute alternative, classificatori e tutti i lookup, allegati effettivi, compilazione valida commerciale, tracking con documenti generati, conversione OV completa, cambio stato multiplo, tutte le varianti layout/colori/grafici/export, stampa PDF e invio effettivo. Alcuni pulsanti aperti o tentati non equivalgono a un flusso completo validato.
Stato finale documentato: SAM rispondente, gestione Opportunità, record prova 2026/3 mantenuto; copia 2026/4 cancellata. I limiti restano nel registro e non nella guida utente.
