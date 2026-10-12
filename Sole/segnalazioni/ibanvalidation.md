# Clienti: errore SQL «IBANValidation non è valido» in più funzioni

## Ambiente
- Collaudo del 09/10/2026, SAM ERP desktop Delphi su Windows.
- Eseguibile: `C:\OutSamW2_12_Develop\SAM.exe`. Numero build/file version non acquisito.
- Database indicato dalla UI: **Database collaudo**; utente **FILIPPO** (Filippo Curati).
- Connessione indicata dall'errore: FireDAC / ODBC Driver 18 for SQL Server.
- Gestione: menu sinistro **Area BASE e di SISTEMA > Base > Controparti > Clienti**.
- 105 clienti iniziali; 106 dopo inserimento del cliente di prova **000219 — SOLE TEST CLIENTE 20261009**.

## Problema e messaggio esatto
```text
[FireDAC][Phys][ODBC][Microsoft][ODBC Driver 18 for SQL Server][SQL Server]
Il nome di oggetto 'IBANValidation' non è valido.
```
In alcuni percorsi compare anche «Errore durante la selezione dei dati, se il problema persiste contattare l'amministratore»; Geo Marketing mostra «Non è stato possibile accedere ai servizi di geo marketing», con lo stesso errore SQL in Dettagli.

## Passi base
1. Avviare SAM, accedere al Database collaudo come FILIPPO.
2. Aprire Clienti dal menu sinistro.
3. Scegliere una riga e attivare uno dei percorsi sotto elencati; per le azioni multiple selezionare **Solo righe selezionate**, poi OK.
4. Leggere il messaggio; chiuderlo con OK per continuare il collaudo.

## Percorsi accorpati nella stessa segnalazione
| Percorso | Risultato osservato |
|---|---|
| Griglia > selezione riga / Rinfresca | Uno o più errori; talvolta griglia aggiornata comunque |
| Popup > Ordinamento decrescente | Ordinamento applicato visivamente, poi Warning/errore |
| Popup > Ricerca Avanzata | Errore prima della maschera successiva |
| Configura / Filtri dopo selezione dati | Warning in alcuni tentativi; in altri maschera aperta |
| Azioni Multiple > Azione, Mail, Appuntamento, Allegato, Work Flow, NewsLetters > Solo righe selezionate > OK | Ogni ramo tentato: stesso errore prima del flusso operativo |
| Prepara Campagna > Solo righe selezionate > OK | Stesso errore |
| Geo Marketing > Solo righe selezionate > OK > Dettagli | Stesso errore nei dettagli |
| Visualizza dettaglio IBAN con riga scelta | Stesso errore |
| Esporta > TXT > Salva con nome > Salva | File presente, errore successivo; integrità non validata |
| Esporta > XLSX (MSExcel non installato) > Salva con nome > Salva | File presente, errore successivo; integrità non validata |
| Grafico > Ragione Sociale 1 + conteggio > Crea Grafico | Finestra aperta con errore; Numero elementi mostra 96, significato/contenuto non validati |
| Popup > Somma valori campo selezionato | Errore, nessun totale validato |
| Navigatore > Clienti / Schede Contabili / Estratto conto per partita | Errore nei tentativi |
| Navigatore > Scheda Credito, Scoperto e Rischio Cliente | Errore |
| Navigatore > Interrogazioni O.V. e Offerte / Spedizioni / Fatture | Errore |

## Atteso e impatto
Atteso: interrogazioni e funzioni utilizzabili senza errore SQL; esportazioni con esito coerente e verificabile.
Osservato: numerosi flussi non completabili, messaggi ripetuti, output di esportazione non verificati. **Alta priorità proposta**: impedisce una parte significativa del collaudo Clienti. Il processo non si arresta: OK consente di recuperare e altre gestioni funzionano.

## Riproducibilità e diagnosi
Ripetuto in diversi percorsi nella stessa sessione. Non verificato in altra installazione o con altri utenti. Nome database SQL, query, stack applicativo e definizione dell'oggetto non acquisiti. L'errore non dimostra da solo che manchi una tabella: verificare oggetto/schema, database effettivo e query. Non è accertata una causa comune a livello di codice; i messaggi identici sono centralizzati qui per evitare duplicazioni.
Alcuni primi click avvennero con modali non correttamente enumerate: la prima attribuzione al report è ambigua e non è usata come riproduzione autonoma.

## Evidenze locali
Disponibili nella cartella `Sole` della copia di lavoro; **non ancora pubblicate su GitHub**:
- `Sole/immagini/08-errore-report.png`, `12-errore-selezione-dati.png`.
- `Sole/immagini/19-errore-geomarketing.png`, `20-dettagli-geomarketing.png`.
- `Sole/registro_collaudo.md`, sezione A01.
- `Sole/clienti_export_test.txt` e `.xlsx`: output non validati.

## Verifica dopo correzione
Ripetere i percorsi elencati con riga corrente e selezione multipla; verificare assenza del messaggio, apertura delle maschere, correttezza dei dati e completezza degli export. Aggiornare questa issue con ulteriori occorrenze dello stesso messaggio.
