# Secondo giro di collaudo — Clienti

Data: 09/10/2026. Nuovo processo SAM: PID 30704. Database collaudo, FILIPPO. Eseguibile `C:\OutSamW2_12_Develop\SAM.exe`.

## Risposte lette prima delle prove

- #1 chiusa dall’utente: database aggiornato, errore IBANValidation risolto.
- #3 chiusa con spiegazione: campi personalizzati non allineati, avviso compatibile con prosecuzione dell’operatività.
- #4 aperta con indicazione: controllare monitor e riposizionare la finestra.
- #2 aperta senza risposta: non duplicare; verifica aggiuntiva tramite interazione reale, senza notifiche artificiali.

## Ambiente grafico

Monitor principale: (0,0,1920,1080), area utile 1920 × 1032. Monitor secondario: (-1920,0,1920,1080). Il desktop esteso non coincide con il solo monitor principale.

## Prove eseguite

| Funzione | Sequenza e risultato | Evidenza |
|---|---|---|
| Avvio/accesso/menu laterale | Login riuscito; Clienti aperta dal menu sinistro; 106 righe | r2-01-clienti.png |
| Rinfresca | Nessun messaggio IBANValidation osservato | Stato UI |
| Ricerca | Dropdown aperto, click su Ragione Sociale 1, digitazione SOLE, OK: individuato 000219; nessuna access violation | r2-02-ricerca.png |
| Modifica | Riaperto 000219; anagrafica e recapiti leggibili | Letture UI |
| Consensi | Apertura a (550,220,784,543); riposizionata a 600,240, chiusa e riaperta con posizione mantenuta | r2-03-consensi.png |
| Azioni Multiple | Provati i sei accessi del menu e il pulsante principale fino all’apertura della form Azione - Nuovo, quindi annullati. La selezione singola della riga è stata effettuata sul margine sinistro. Prima della selezione: normale avviso «Non non ci sono righe selezionate» | r2-04-azione.png |
| Compilazione azione | Tentati scelta tipologia e testo di prova; validazione «È necessario specificare la tipologia di azione». Selezione non completata; annullato senza salvare | Letture UI |
| Prepara Campagna | Aperta Preparazione campagna con Contatto e Campagna; annullata | r2-05-campagna.png |
| Dettaglio IBAN | Nuova anomalia: etichette interne e mancata chiusura. Recupero: finestra spostata sul secondo monitor, continuate altre prove. Nuova attivazione la riporta sul principale ancora con gli stessi testi | r2-07-iban-placeholder.png, r2-20-iban-conferma.png |
| Geo Marketing | Aperta mappa; elemento SOLE TEST CLIENTE 20261009, «elementi trovati: 1». Chiusa normalmente | r2-08-geomarketing.png |
| Ricerca Avanzata | Aperta con Seleziona il Classificatore / Seleziona il record; nessun dato da visualizzare; annullata | r2-09-ricerca-avanzata.png |
| Layout | Aperto dal popup; annullato senza salvataggio | Stato UI |
| Filtro | Creato SOLE_R2; selezionata Ragione Sociale 1 tramite dropdown/tastiera; uguaglianza al nome completo; OK. Risultato: solo 000219, Attivo (1), Filtro Attivo SOLE_R2 | r2-10-filtro.png, r2-11-filtro-risultato.png |
| Export TXT | Con estrazione filtrata: file r2-clienti.txt salvato, nessun errore SQL osservato; contenuto non letto dal tool Read | File locale |
| Export XLSX senza Excel | r2-clienti.xlsx salvato; messaggio «Esportazione terminata correttamente»; verifica ZIP fallita: formato non validato, selezione finale del formato da confermare prima di attribuire un difetto | r2-12-export-esito.png |
| Navigatore: Schede Contabili | Aperta selezione con date, conti, registrazioni, ordinamenti e valuta; annullata | r2-13-schede-contabili.png |
| Navigatore: Estratto conto | Aperta selezione conti, partite, ritenute, scaduto e valuta; annullata | r2-14-estratto-conto.png |
| Navigatore: Credito | Aperta selezione credito/scoperto/rischio; annullata | r2-15-credito.png |
| Navigatore: O.V./Offerte | Aperta gestione senza errore SQL | r2-16-offerte.png |
| Navigatore: Spedizioni/Fatture | Aperte gestioni senza errore SQL; nessun salvataggio | Titoli UI |
| Ordinamento | Tentati crescente/decrescente da popup; nessun errore SQL. Con estrazione singola non validata differenza di ordinamento | Stato UI |
| Colori | Aperta finestra e chiusa; nessuna regola salvata | Stato UI |
| Grafico | Etichetta Ragione Sociale 1, Includi conteggio dati; validazione iniziale assenza dato, corretta. Grafico con cliente e Numero Righe; nessun errore SQL | r2-17-grafico.png |
| Anteprima | Criterio e report, aperta anteprima 1/1 del cliente filtrato; recapiti leggibili; chiusa | r2-18-anteprima.png |
| Modifica/salvataggio #3 | Ragione Sociale 2 aggiornata a «Collaudo operativo agente R2», confermata assenza identificativi fiscali. Avviso campi personalizzati ricomparso; chiuso e tornati alla griglia | r2-19-campi-personalizzati.png |
| Cruscotto | Aperto su 000219; ritorno a Clienti | Titolo UI |
| Etichette | Parametri e conferma selezione, apertura report; annullato prima della stampa | Stato UI |

## Esito issue

- #1: nessuna ricorrenza del messaggio nei passi osservati; commento di verifica aggiunto, resta chiusa.
- #2: uso reale riuscito; commento aggiunto; resta aperta, causa dell’episodio automatizzato non accertata.
- #3: stessa casistica già spiegata dall’utente, non riaperta; censita in issue_skipped.md.
- #4: recupero e riapertura con posizione mantenuta verificati; commentata e chiusa. La causa storica del posizionamento non è stata dimostrata.
- #5: creata per Dettaglio IBAN, distinto dal vecchio errore SQL. Screenshot locali; pubblicazione degli allegati da completare.

## Limiti del giro

Questo giro amplia la verifica delle segnalazioni ma **non completa un collaudo esaustivo di tutti i rami**: non completati salvataggi delle sei azioni, campagne, P.d.C., griglie figlie, tutte le combinazioni filtro/layout/colori/grafico, tutti i formati export, stampa effettiva etichette, cancellazione e dossier. La mancanza di errori nell’apertura non valida il flusso completo.

Non sono state generate nuove anagrafiche. Le azioni e la campagna sono state annullate. Il cliente 000219 è stato modificato; filtro e output registrati in dati_generati.md.

Stato finale: SAM resta rispondente; Dettaglio IBAN resta aperto e presenta la nuova anomalia. Non è stato forzato l’arresto. Tutte le altre prove documentate sono state recuperabili.
