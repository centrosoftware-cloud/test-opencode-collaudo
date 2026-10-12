# Clienti > Consensi: finestra aperta fuori dai limiti del desktop corrente

## Ambiente
Collaudo 09/10/2026; Windows, SAM ERP Delphi; eseguibile `C:\OutSamW2_12_Develop\SAM.exe`, build non acquisita. Database collaudo, FILIPPO. Desktop osservato largo **1920 pixel**, altezza 1080; finestra principale 1920 × 1032.
Anagrafica **000219 — SOLE TEST CLIENTE 20261009**.

## Riproduzione osservata
1. Aprire Clienti dal menu **Area BASE e di SISTEMA > Base > Controparti > Clienti**.
2. Aprire il cliente 000219 tramite Modifica, scheda Dati Generali.
3. Premere **Consensi**.
4. Viene creata la finestra `TFrmGridConsensi`, titolo **Lista consensi: 000219 - SOLE TEST CLIENTE 20261009**, ma i controlli si trovano oltre il bordo destro del desktop corrente.

## Evidenza geometrica
Enumerazione nativa dei controlli al momento dell'apertura:
- pannello: x **2657–3437**, y **314–738**;
- toolbar: x **2660–2833**, y **277–313**;
- pulsante Chiudi: x **3342–3417**, y **752–777**.
Coordinate incompatibili con la visibilità sul desktop osservato x 0–1920. La finestra risultava esistente e visibile secondo Windows.

## Atteso / osservato / impatto
Atteso: apertura su un monitor disponibile, con titolo e comandi raggiungibili dall'utente.
Osservato: finestra fuori dall'area del desktop catturabile; interazione visiva impedita fino al riposizionamento. Recupero eseguito dall'automazione con `SetWindowPos` a **550,220**; dopo lo spostamento la lista è stata utilizzata e chiusa normalmente.
Priorità proposta: **media**, problema di usabilità con recupero possibile. Una occorrenza osservata; non verificata la ripetibilità dopo riposizionamento/riavvio.

## Ipotesi non verificate
Possibile posizione salvata da precedente configurazione multi-monitor. Non è stata acquisita la topologia completa dei monitor virtuali, né una prova dopo disconnessione RDP/cambio risoluzione: la causa non è accertata. Verificare prima di attribuire il difetto a una specifica configurazione.

## Evidenze locali
`Sole/registro_collaudo.md` sezione A05 e `Sole/linee_guida_agente.md`.
`Sole/immagini/64-consensi.png` mostra **la finestra dopo il recupero**, non una cattura dell'apertura fuori schermo. File locali non ancora pubblicati su GitHub.

## Verifica richiesta
Controllare ripristino delle coordinate e limitazione all'area di lavoro dei monitor disponibili; provare apertura dopo cambio monitor/risoluzione e su desktop singolo. Una posizione storica non più disponibile non dovrebbe rendere inaccessibile la form.
