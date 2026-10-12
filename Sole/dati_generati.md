# Dati generati durante il collaudo

## Gestione Opportunità

- Creata opportunità **Trattativa commerciale / anno 2026 / numero 3**, data 09/10/2026, stato Proposta, contatto **000219 — SOLE TEST CLIENTE 20261009**.
- Nota testata: **SOLE OPPORTUNITA TEST 20261009** (immessa, persistenza da verificare riaprendo).
- Riga descrittiva **SOLE SERVIZIO DIMOSTRATIVO**, quantità **2**, prezzo lordo **100**, successo riga **50%**; importo riga visualizzato **200**, pesato **100**.
- Confermata memorizzazione riga e prosecuzione senza classificatori articolo. Salvata testata. Dopo Rinfresca: opportunità **65 → 66**, Proposte **18 → 19**, Non Esportate **31 → 32**.
- Nessuna nuova controparte salvata: le aperture Controparti - Nuovo durante esplorazione del selettore sono state annullate.
- Persistenza nota e riga verificata riaprendo. Sconto globale **10%**, totale **180**, pesato documento **90**. Stato finale **Proposta**, dopo prove Confermata/Annullata.
- Duplicazione riga e cancellazioni provate; ricreata e mantenuta una sola riga descrittiva 2 × 100 con successo 50%.
- Copia opportunità **2026/4** risultata persistita durante prova Incolla, poi **eliminata**. Rimane solo il record prova 2026/3 per il contatto selezionato.
- Filtro **SOLE_OPP**, Descr. Contatto uguale a SOLE TEST CLIENTE 20261009, mantenuto nella configurazione.
- File **opportunita_test.txt**, export generato ma non validato nel contenuto.
- Nessuna campagna creata; conversione in documento di vendita interrotta su validazioni, nessuna creazione verificata. Nessun invio mail/fax eseguito.

## Secondo giro (09/10/2026)

- Riutilizzato cliente **000219**, nessuna nuova anagrafica creata.
- Ragione Sociale 2 immessa e salvata: **Collaudo operativo agente R2**. La verifica dell’esito completo dei campi personalizzati resta distinta dalla modifica anagrafica.
- Creato filtro **SOLE_R2**: Ragione Sociale 1 uguale a SOLE TEST CLIENTE 20261009. Estratto un cliente. Filtro disattivato e riapplicato in vari passaggi; configurazione residua da conservare come prova.
- File **r2-clienti.txt** e **r2-clienti.xlsx** generati sull’estrazione filtrata. Nessun errore SQL; secondo export mostra esito corretto. Contenuto non validato: il file con estensione XLSX non è risultato leggibile come ZIP; scelta finale del formato da confermare prima di dedurre un difetto.
- Testo temporaneo **SOLE COLLAUDO R2 NEWSLETTER** immesso in una maschera azione; validazione tipologia, poi annullamento. Nessuna azione, mail, newsletter, appuntamento, allegato, workflow o campagna salvata.
- Nessuna cancellazione confermata e nessuna stampa etichette eseguita nel secondo giro.
- Schermate **immagini/r2-*.png**, collegate al registro R2 e al manuale quando pertinenti.

Data: 9 ottobre 2026. Ambiente: Database collaudo.

## Cliente di prova creato

- Codice assegnato da SAM: **000219**.
- Ragione sociale 1: **SOLE TEST CLIENTE 20261009**.
- Indirizzo visualizzato all’inserimento: **Via del Collaudo 9**.
- CAP visualizzato: **40100**; comune visualizzato: **Bologna**.
- E-mail visualizzata: **sole.test@example.com**.
- Codice fiscale e partita IVA: non compilati; confermata la prosecuzione nell’apposito avviso.
- Ragione sociale 2: **Collaudo operativo agente**, rilevata nella griglia dopo salvataggio e rinfresco.
- Il salvataggio ha assegnato il codice ma ha mostrato un avviso relativo a campi personalizzati obbligatori non compilati. Il cliente è stato ritrovato nella griglia; totale passato da 105 a **106**.
- Riapertura: verificati **Via del Collaudo 9**, **40100**, **Bologna**. L’e-mail è stata reinserita nella successiva modifica; la sua persistenza finale non è stata riletta come valore strutturato.
- Stato: cliente **Attivo**. Nessun identificativo fiscale, pagamento, IBAN, listino, nominativo, indirizzo alternativo o consenso creato per questo record.
- **Il cliente rimane nel database. Non è stato eliminato.**

Nota tecnica: alcuni valori sono stati inizialmente inseriti con `WM_SETTEXT`, che aggiorna la visualizzazione ma può non aggiornare il dataset Delphi. La ragione sociale 1 è stata poi immessa tramite tastiera e uscita con Tab. Gli altri valori devono essere verificati riaprendo il record.

La verifica successiva e la reinserzione dei valori sono descritte sopra: la distinzione evita di presentare come salvati tutti i valori del primo tentativo.

## Configurazione filtro di prova

- Nome immesso: **SOLE_CLIENTI_TEST**.
- Il nome è stato rilevato alla riapertura della finestra Filtro.
- Nessuna condizione operativa validata; elenco iniziale «Tutte».
- Non verificato se la memorizzazione sia soltanto locale o condivisa/persistita nel database. **Verificare e rimuovere se non necessario.**

## Valori di ricerca fittizi non salvati come anagrafica

- `99999999999`: partita IVA usata nella precompilazione. Risultato: nessuna azienda trovata; nuovo record successivo annullato.
- `SOLE`, `SOLE TEST` e frammenti intermedi: valori di ricerca temporanei; nessuna anagrafica aggiuntiva generata.

## Copia annullata

- Eseguito Copia > Nuovo > Incolla.
- Sorgente effettivamente visualizzata: **AMAZON EU S.A R.L.**, con indirizzo AVENUE JOHN F. KENNEDY 38 e ruoli Cliente/Fornitore.
- Il nuovo record è stato **annullato**; nessun duplicato salvato.

## File generati dall’applicazione

| File in Sole | Operazione | Esito |
|---|---|---|
| [clienti_export_test.txt](clienti_export_test.txt) | Esporta TXT > Salva | File presente; errore SQL IBANValidation dopo generazione; contenuto non validato |
| [clienti_export_test.xlsx](clienti_export_test.xlsx) | Esporta XLSX senza Excel > Salva | File presente; errore SQL IBANValidation dopo generazione; contenuto non validato |
| [clienti_stampa_test.pdf](clienti_stampa_test.pdf) | Report anagrafica > Microsoft Print to PDF | File presente; contenuto non letto direttamente dal modello |
| `immagini/*.png` | Catture del collaudo | Schermate incluse o referenziate nel manuale; possono contenere dati preesistenti del database |

## Operazioni non effettuate

Nessuna cancellazione confermata, campagna, azione CRM, mail, appuntamento, newsletter, workflow, allegato, consenso o trascodifica articolo creata. Nessun dato salvato nelle griglie figlie. Nessuna regola colore o nuovo layout confermati.
