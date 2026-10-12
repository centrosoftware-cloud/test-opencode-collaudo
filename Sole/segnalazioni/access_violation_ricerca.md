# Ricerca Clienti: access violation rtl290.bpl durante input automatizzato

## Stato della segnalazione
**Anomalia osservata, riproduzione manuale da confermare.** La prova usa messaggi Windows programmatici: non è accertato che un utente con mouse e tastiera normali incontri lo stesso errore. Non presentare come bug manuale già riprodotto.

## Ambiente
09/10/2026; Windows, SAM ERP desktop Delphi; eseguibile `C:\OutSamW2_12_Develop\SAM.exe`; build non acquisita. Database collaudo; utente FILIPPO. Gestione **Area BASE e di SISTEMA > Base > Controparti > Clienti**, 106 righe al momento della prova.

## Sequenza osservata
1. Aprire **Ricerca** dalla toolbar Clienti (form `TFrmFind`).
2. Tentare di impostare **Ragione Sociale 1** nel Campo di Ricerca (`TComboBox`) inviando `CB_SETCURSEL` con indice 3 e notifica `CBN_SELCHANGE` al parent.
3. Portare la form in primo piano, cliccare il valore da ricercare (`TCSEdit`) e inviare «SOLE» tramite tastiera automatizzata.
4. Compare il messaggio sotto riportato.
5. Chiudere OK; premere OK nella ricerca: compare «Nessuna riga trovata per Partita I.V.A. = …». Il campo effettivo non risulta cambiato come inteso dall'automazione.

## Messaggio esatto
```text
Access violation at address 638625C5 in module 'rtl290.bpl'
(offset 1325C5). Read of address 00000008.
```

## Atteso / osservato / impatto
Atteso: input gestito coerentemente o validazione del valore senza access violation.
Osservato: errore a video, selezione interna non coerente con il tentativo di automazione; ricerca positiva non validata. Chiudendo il messaggio SAM resta utilizzabile; non sono stati salvati dati anagrafici da questo percorso.
Priorità proposta: **media, da triage**. Una occorrenza documentata; non riprodotto con interazione manuale, altri utenti o altre build.

## Evidenze locali
`Sole/immagini/60-access-violation-ricerca.png` e `Sole/registro_collaudo.md` sezione A02. File presenti nella copia di lavoro, non allegati/pubblicati su GitHub.

## Approfondimento richiesto
- Ripetere scegliendo il campo dal dropdown con mouse/tastiera reali, digitando SOLE e provando anche valore vuoto e annullamento.
- Confrontare la stessa sequenza automatizzata e verificare se l'indice della combo è valido e le notifiche sono appropriate.
- Acquisire stack e build; gli indirizzi riportati sono quelli della sessione, non identificatori stabili.
- Se il problema è esclusivamente una notifica artificiale non supportata, classificare l'esito come limite dell'automazione e aggiornare le linee guida anziché confermare un difetto del flusso utente.
