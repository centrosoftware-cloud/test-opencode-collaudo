# Censimento issue — Gestione Clienti SAM ERP

Repository: [centrosoftware-cloud/test-opencode-collaudo](https://github.com/centrosoftware-cloud/test-opencode-collaudo).
Evidenze: collaudo del **09/10/2026**, `C:\OutSamW2_12_Develop\SAM.exe`, Database collaudo, FILIPPO.

## Issue pubblicate

### Opportunità

- [#6 — Prepara Campagna: errore SQL ORDER BY con Tutte le righe](https://github.com/centrosoftware-cloud/test-opencode-collaudo/issues/6), aperta. Ripetuto con filtro SOLE_OPP; non bloccante per SAM.
- [Descrizione locale](segnalazioni/opportunita_campagna_orderby.md); [registro](registro_opportunita.md).
- Screenshot `immagini/opp-18-errore-campagna.png`. **Allegato GitHub incompleto**: browser Incognito non autenticato (Sign in to comment); CLI autenticata, ma non offre upload PNG nell’editor. Necessaria sessione browser autenticata per completare il caricamento. Nessun PNG committato.

### Aggiornamento dopo il secondo giro

| Issue | Stato verificato dopo le prove | Esito |
|---|---|---|
| #1 | Chiusa dall’utente, mantenuta chiusa | Messaggio SQL non ricomparso nei passi riprovati; [commento](https://github.com/centrosoftware-cloud/test-opencode-collaudo/issues/1#issuecomment-6082776160) |
| #2 | Aperta | Ricerca reale riuscita, causa del caso automatizzato ancora da valutare; [commento](https://github.com/centrosoftware-cloud/test-opencode-collaudo/issues/2#issuecomment-6082776574) |
| #3 | Chiusa dall’utente | Comportamento ricorrente coerente con spiegazione; censito in issue_skipped.md |
| #4 | Chiusa dall’agente dopo verifica | Riposizionamento, chiusura e riapertura verificati; [commento](https://github.com/centrosoftware-cloud/test-opencode-collaudo/issues/4#issuecomment-6082771556) |
| [#5](https://github.com/centrosoftware-cloud/test-opencode-collaudo/issues/5) | Aperta | Dettaglio IBAN: etichette interne e mancata chiusura; [descrizione](segnalazioni/dettaglio_iban_placeholder.md) |

Dettagli delle prove: [registro_collaudo_r2.md](registro_collaudo_r2.md). Casi senza nuova segnalazione: [issue_skipped.md](issue_skipped.md). Gli screenshot della nuova issue #5 sono locali; caricamento degli allegati da completare, senza link remoti inesistenti.

### Stato storico alla creazione iniziale

| Evidenza | Issue | Classificazione | Stato alla creazione | Descrizione locale |
|---|---|---|---|---|
| A01 | [#1 — Errore SQL IBANValidation](https://github.com/centrosoftware-cloud/test-opencode-collaudo/issues/1) | Errore ripetuto, impatto alto; causa da verificare | Aperta | [Testo](segnalazioni/ibanvalidation.md) |
| A02 | [#2 — Access violation nella ricerca automatizzata](https://github.com/centrosoftware-cloud/test-opencode-collaudo/issues/2) | Da verificare con interazione manuale; automazione coinvolta | Aperta | [Testo](segnalazioni/access_violation_ricerca.md) |
| A03 | [#3 — Salvataggio parziale con campi personalizzati mancanti](https://github.com/centrosoftware-cloud/test-opencode-collaudo/issues/3) | Verifica funzionale dei requisiti transazionali | Aperta | [Testo](segnalazioni/salvataggio_parziale.md) |
| A05 | [#4 — Consensi fuori dal desktop](https://github.com/centrosoftware-cloud/test-opencode-collaudo/issues/4) | Posizionamento finestra; topologia monitor e ripetibilità da verificare | Aperta | [Testo](segnalazioni/consensi_fuori_schermo.md) |

Ogni descrizione comprende ambiente, sequenza osservata, messaggio, risultato atteso/da chiarire, impatto, recupero, attendibilità e verifiche successive. Non sono stati inventati build, stack, query o cause non acquisiti.

## Deduplicazione

- Prima della creazione sono state consultate **tutte le issue aperte e chiuse**: l'elenco era vuoto.
- Tutte le occorrenze di `IBANValidation` sono centralizzate in **#1**, incluse azioni multiple (sei rami), campagne, Geo Marketing, dettaglio IBAN, selezione/rinfresco, ordinamento, ricerca avanzata, export TXT/XLSX, grafico, somma e destinazioni del Navigatore.
- Gli avvisi di Geo Marketing e di selezione dati con il medesimo errore nei dettagli non hanno generato issue separate.
- Le occorrenze in creazione e modifica dei campi personalizzati sono nella stessa **#3**.
- Per nuove occorrenze verificare anche le issue chiuse e aggiornare la segnalazione esistente quando il fenomeno coincide. Un messaggio simile non dimostra da solo una stessa causa: eventuali separazioni successive devono essere motivate da diagnosi/evidenze.

## Osservazioni senza issue autonoma

| Osservazione | Decisione |
|---|---|
| A04: record 000219 già in uso da Filippo Curati | Controllo di concorrenza: accesso riuscito chiudendo la scheda precedente; nessun difetto accertato |
| Ragione sociale obbligatoria, identificativi fiscali mancanti, campo filtro mancante, etichetta grafico non selezionata | Validazioni applicative; nessuna issue autonoma |
| WM_SETTEXT cambia la visualizzazione senza aggiornare il dataset | Limite dell'automazione, registrato nelle linee guida |
| Copia dalla sorgente diversa da quella inizialmente intesa | Contesto/selezione alterato durante il test, non difetto di copia accertato |
| Rami non esplorati o non verificati | Debito di copertura, non automaticamente difetto applicativo |

## Evidenze e pubblicazione

Le immagini e i documenti `Sole` sono **locali e non ancora pubblicati nel repository**. Le issue indicano i percorsi e distinguono le schermate dopo recupero dalle evidenze del problema; non contengono collegamenti remoti a file inesistenti. Questa attività ha creato le issue, senza commit o push dei documenti.

## Documentazione

- [Registro di collaudo](registro_collaudo.md): traccia tecnica e dettaglio dei tentativi.
- [Linee guida agente](linee_guida_agente.md): regole di interazione e classificazione delle evidenze.
- [Dati generati](dati_generati.md): record/output residui.
- Il manuale utente contiene istruzioni operative, senza censimento di bug o riferimenti alle issue.

Non sono state eseguite nuove interazioni SAM durante la compilazione delle issue: le incertezze delle evidenze storiche sono riportate esplicitamente come verifiche da svolgere, anziché trasformarle in riproduzioni confermate.
