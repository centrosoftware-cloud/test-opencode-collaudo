# Clienti: anagrafica salvata anche quando fallisce il salvataggio dei campi personalizzati obbligatori

## Stato
**Comportamento da verificare rispetto ai requisiti transazionali.** SAM avverte esplicitamente del mancato salvataggio dei campi personalizzati; non è accertato se il salvataggio indipendente dell'anagrafica sia previsto. La segnalazione richiede una valutazione funzionale.

## Ambiente e dati
Collaudo 09/10/2026, Windows, eseguibile `C:\OutSamW2_12_Develop\SAM.exe`, build non acquisita; Database collaudo, FILIPPO.
Menu **Area BASE e di SISTEMA > Base > Controparti > Clienti**.
Cliente di prova: **000219 — SOLE TEST CLIENTE 20261009**; ragione sociale 2 **Collaudo operativo agente**; nessun codice fiscale o partita IVA. Il cliente rimane nel database.

## Riproduzione osservata
1. In Clienti premere Nuovo.
2. Scegliere Annulla precompilazione per compilazione manuale.
3. Compilare Ragione Sociale 1, lasciare non compilati i campi personalizzati obbligatori; non compilare identificativi fiscali.
4. Premere OK; alla richiesta relativa all'assenza di codice fiscale/partita IVA scegliere Yes.
5. SAM assegna il codice **000219**, ma mostra il messaggio sotto.
6. Chiudere OK e rinfrescare Clienti: totale **105 → 106**; il cliente è presente e riapribile.
7. Il medesimo avviso è stato osservato anche in modifica del cliente.

## Messaggio
```text
ATTENZIONE: Non tutti i campi personalizzati indicati come obbligatori sono stati compilati.

Salvataggio Campi personalizzati NON effettuato
```

## Atteso da chiarire / osservato / impatto
Osservato: l'anagrafica principale è persistita mentre i campi personalizzati non vengono salvati; nessun annullamento globale. Possibilità di record incompleto rispetto alla configurazione dei campi obbligatori. La form non identifica nel messaggio i campi mancanti.
Atteso da confermare con responsabile funzionale: validazione prima del salvataggio complessivo oppure salvataggio parziale esplicitamente previsto e gestito. Non assumere atomicità obbligatoria senza requisiti.
Priorità proposta: **media**; messaggio non bloccante per il processo. Non verificati i nomi esatti dei campi obbligatori configurati, né il ramo con tali campi completamente valorizzati.

## Attendibilità dei dati
Il primo tentativo con WM_SETTEXT non aggiornava sempre il dataset: tale limite è distinto dalla presente segnalazione. La ragione sociale è stata poi inserita con tastiera e uscita dal campo; persistenza e totale verificati riaprendo/rinfrescando. Non è attribuito a SAM il mancato aggiornamento dovuto a WM_SETTEXT.

## Evidenze locali
`Sole/immagini/06-avviso-campi-personalizzati.png`, `39-classificatori.png`, `63-cliente-test-verificato.png`; `Sole/registro_collaudo.md` A03; `Sole/dati_generati.md`. File locali non ancora pubblicati su GitHub.

## Verifica richiesta
Identificare i campi obbligatori effettivi, ripetere creazione/modifica con campi mancanti e completi, verificare transazioni e persistenza di entrambe le sezioni. Definire il comportamento atteso e l'eventuale indicazione dei campi da correggere.
