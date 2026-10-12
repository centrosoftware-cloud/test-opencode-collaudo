# Collaudo della gestione Clienti SAM ERP

Sessione del **9 ottobre 2026**, ambiente **Database collaudo**, utente **FILIPPO**.

## Documenti

- [Manuale Opportunità](manuale_opportunita.md).
- [Registro di collaudo Opportunità](registro_opportunita.md): risultati, dati controllati e copertura residua.

- [Linee guida per scrivere il manuale utente](linee_guida_manuale.md).
- [Secondo giro di collaudo](registro_collaudo_r2.md): verifica delle risposte e nuove evidenze.
- [Casi senza nuova segnalazione](issue_skipped.md).

- [Manuale operativo Clienti](manuale_clienti.md): procedure, schermate, accessi duplicati e matrice di copertura.
- [Linee guida per l’agente automatico](linee_guida_agente.md): pattern operativi, lettura UI, input, sincronizzazione e raccolta evidenze.
- [Dati generati](dati_generati.md): cliente 000219, configurazione filtro, prove annullate e output.
- [Registro di collaudo](registro_collaudo.md): sequenze, risultati e anomalie.
- [Censimento issue GitHub](issue_list.md): collegamenti alle quattro segnalazioni, deduplicazione e classificazione delle evidenze.
- [Cartella immagini](immagini/): **68 schermate** della sessione. Alcune catture preliminari non rappresentano la form intesa; non sono usate come evidenze positive e sono segnalate nelle linee guida.

## Risultato e copertura

Creato il cliente **000219 — SOLE TEST CLIENTE 20261009**, rimasto nel database. Totale griglia: da **105** a **106**.

Il manuale è un’**edizione con copertura parziale**, non un manuale esaustivo già validato. Contiene le istruzioni operative e gli accessi disponibili. La copertura tecnica, le anomalie e i punti residui sono conservati nel registro di collaudo e nel censimento issue, separatamente dalle istruzioni per l'utente.

Ulteriori anomalie: access violation durante una ricerca automatizzata da riprodurre manualmente; mancato salvataggio dei campi personalizzati obbligatori; lista Consensi aperta fuori schermo. Il record in uso è stato distinto come controllo di concorrenza.

## Output dell’applicazione

- [Stampa PDF](clienti_stampa_test.pdf).
- [Tentativo export TXT](clienti_export_test.txt).
- [Tentativo export XLSX](clienti_export_test.xlsx).

Gli export sono stati generati con messaggio d’errore successivo: contenuto e completezza non validati. Il PDF è stato salvato, mentre il report è stato letto dalla sua anteprima SAM.

I link locali dei documenti sono stati confrontati con l’inventario dei file della cartella; i percorsi referenziati risultano presenti. Il repository non è stato committato. Le quattro issue sono pubblicate su GitHub; le immagini e i documenti restano nella copia di lavoro locale.
