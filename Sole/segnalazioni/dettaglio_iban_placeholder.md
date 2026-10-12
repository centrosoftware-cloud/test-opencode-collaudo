# Clienti: Dettaglio IBAN mostra etichette interne e non si chiude

## Ambiente e prerequisiti
- Nuovo giro di collaudo del 09/10/2026 dopo l’aggiornamento del database comunicato nella issue #1.
- Eseguibile `C:\OutSamW2_12_Develop\SAM.exe`, Windows, Database collaudo, FILIPPO. Versione file non acquisita.
- 106 clienti. Filtro di prova `SOLE_R2` applicato in parte delle prove; apertura iniziale anche prima dell’applicazione del filtro.
- Cliente corrente **000219 — SOLE TEST CLIENTE 20261009**, senza partita IVA e codice fiscale, con indirizzo Via del Collaudo 9 / 40100 Bologna.

## Riproduzione
1. Avviare SAM e aprire **Area BASE e di SISTEMA > Base > Controparti > Clienti**.
2. Individuare il cliente 000219 e tornare alla griglia.
3. Premere **Visualizza dettaglio IBAN — Ctrl+D** sulla toolbar.
4. Si apre **Clienti - Dettaglio IBAN** (`TFrmCheckIBANDetail`), ma al posto di codice, descrizione e identificativi sono visualizzate le etichette interne sotto riportate.
5. Provare la chiusura: ESC/Alt+F4 e richiesta standard di chiusura finestra non hanno chiuso la form nelle prove. Non sono stati trovati pulsanti di chiusura nell’enumerazione dei controlli; UI Automation indica finestra non modale, pronta all’interazione, senza massimizzazione/minimizzazione.
6. La form resta presente. È stato possibile spostarla su un altro monitor e proseguire il collaudo nella griglia. Una successiva attivazione del pulsante riporta in vista la stessa finestra con gli stessi testi.

## Testi esatti osservati
```text
lblCodiceControparte
lblDescrizioneControparte
Codice Fiscale:
lblCFControparte2
Partita IVA:
lblPIVAControparte2
```

## Atteso / osservato / impatto
Atteso: dati del cliente corrente, gestione chiara dell’assenza di identificativi/IBAN e possibilità di chiudere la finestra.
Osservato: nomi di componenti visualizzati al posto dei dati e finestra persistente dopo i tentativi di chiusura. Non compare il vecchio errore SQL `IBANValidation`.
Impatto medio: consultazione non utilizzabile e finestra residua; **non bloccante per tutto SAM**, che resta rispondente e permette altre operazioni.

## Evidenze e limiti
Print-screen acquisiti prima del recupero e dopo nuova attivazione:
- `Sole/immagini/r2-07-iban-placeholder.png`;
- `Sole/immagini/r2-20-iban-conferma.png`.

**Disponibilità immagini:** i file sono nella copia di lavoro, non ancora pubblicati. Non è disponibile in questa sessione uno strumento di caricamento degli allegati GitHub; non sono stati eseguiti commit/push dei file. Completamento necessario: pubblicare/allegare le immagini prima di considerare completa la distribuzione delle evidenze. Non vengono inseriti collegamenti remoti a immagini inesistenti.

Per la chiusura sono stati provati anche `WM_CLOSE` e `WindowPattern.Close`: nessun cambiamento osservato; tali tentativi sono aggiuntivi e distinti da mouse/tastiera. Non è stato forzato l’arresto del processo. Non verificato con cliente dotato di IBAN o in altra installazione.

## Deduplicazione
Consultate tutte le issue aperte/chiuse e commenti. #1 riguarda l’errore SQL, non ripresentato in questa form; #2 la ricerca; #3 i campi personalizzati; #4 la geometria Consensi. Il nuovo problema è distinto e centralizza dati-placeholder e mancata chiusura osservati nella stessa finestra.

## Verifica richiesta
Ripetere con e senza IBAN/identificativi, verificare inizializzazione dei dati e comandi di chiusura. Distinguere un’eventuale condizione del record dall’errore di presentazione della form.
