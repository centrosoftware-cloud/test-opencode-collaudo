## Problema
La preparazione di una campagna dall’elenco opportunità si interrompe con un errore SQL prima di aprire la maschera di preparazione.

## Ambiente e precondizioni
SAM ERP, `C:\OutSamW2_12_Develop\SAM.exe`, Database collaudo, utente FILIPPO. Gestione Area C.R.M. > CRM-I Marketing > Opportunità > Scheda Opportunità. Filtro attivo `SOLE_OPP`: Descr. Contatto uguale a `SOLE TEST CLIENTE 20261009`; una opportunità, Trattativa commerciale 2026/3.

## Riproduzione
1. Aprire Scheda Opportunità e applicare il filtro indicato.
2. Premere **Prepara Campagna**.
3. Scegliere **Tutte le righe** e premere OK.

## Osservato
```text
[FireDAC][Phys][ODBC][Microsoft][ODBC Driver 18 for SQL Server][SQL Server]
La clausola ORDER BY non è consentita in viste, funzioni inline, tabelle derivate,
sottoquery ed espressioni di tabella comuni a meno che non sia specificata anche
la clausola TOP, OFFSET o FOR XML.
```
La campagna non viene preparata. OK chiude il messaggio e consente di continuare a usare SAM.

## Atteso
Apertura della preparazione della campagna per le opportunità estratte, senza errore SQL.

## Screenshot
Acquisito il messaggio con il contesto della gestione. Caricamento nella issue incompleto: il browser disponibile è Incognito e GitHub mostra Sign in to comment. È necessaria una sessione browser autenticata per allegare il PNG; la sola autenticazione CLI non effettua il caricamento.
