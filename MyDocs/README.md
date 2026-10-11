# MyDocs — materiali di studio del fork BalduPS5_

Cartella **nostra**, separata dal progetto upstream KytyPS5. Qui vanno analisi, decisioni, appunti e documenti di visione. Nessun file del progetto originale viene modificato per il nostro studio.

## Indice

| File | Cosa contiene | Per chi | Stato |
|---|---|---|---|
| [INTRODUZIONE.md](INTRODUZIONE.md) | Cosa fa KytyPS5 e come, in linguaggio semplice; cosa possiamo farne; vincoli; percorso proposto | Chi inizia | Attuale |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Mappa tecnica completa: modello di emulazione, misure, moduli, flussi, sottosistemi, threading, test/CI, decisioni di design, rischi, guida alla lettura, glossario | Sviluppatori | Attuale, al commit `59183db` |
| [CONFRONTO-AnyPS5.md](CONFRONTO-AnyPS5.md) | KytyPS5 vs AnyPS5: stessa idea, due confezioni; differenze tecniche; maturità; implicazioni | Tutti | Snapshot 2026-10-10 |
| [ADR-0001-ibrido-anyps5-kyty.md](ADR-0001-ibrido-anyps5-kyty.md) | Decisione: non unire i due progetti nel codice; console multi-runtime con motori intatti | Decisori | Proposed |
| [ADR-0002-struttura-repository.md](ADR-0002-struttura-repository.md) | Decisione: repository separato per il prodotto, motori come dipendenze fissate, fork solo per studio e patch | Decisori | Proposed |
| [PROGETTO-CONSOLE.md](PROGETTO-CONSOLE.md) | Carta di progetto: obiettivo, cosa costruiamo e cosa usiamo, confini, requisiti, vincoli legali, criteri di successo, tappe, rischi | Tutti | Bozza |

Prossimi documenti previsti (vedi action items di ADR-0001 e ADR-0002): contratto guscio ↔ motori, matrice di compatibilità per gioco e motore. Quando nascerà il repository del prodotto (ADR-0002), questa cartella migrerà lì e qui resterà solo un rimando.

## Convenzioni

- **Lingua**: italiano, termini tecnici in inglese.
- **Ancoraggio al codice**: ogni affermazione sul codice cita `file:riga` e il **commit** a cui si riferisce. L'upstream fa decine di commit al giorno, quindi i numeri di riga invecchiano in fretta; il commit rende l'affermazione verificabile.
- **ADR**: una decisione architetturale per file, numerazione progressiva `ADR-NNNN-titolo.md`, formato con Status / Context / Decision / Options / Trade-off / Consequences / Action Items. Una decisione superata non si cancella: si marca `Superseded` e si linka la nuova.
- **Snapshot esterni**: dati su progetti terzi (stelle, commit, release) riportano sempre la data di rilevazione.
- **Cosa non va qui**: codice del progetto, build, dump di gioco, materiale coperto da copyright.

## Come tenere aggiornata la cartella

1. Dopo un aggiornamento del fork dall'upstream, rileggere ARCHITECTURE.md §3 (misure) e §11 (rischi) e correggere ciò che è cambiato, aggiornando il commit di riferimento in testa.
2. Quando una decisione cambia, nuovo ADR che supersede il precedente.
3. Quando si prova un gioco o un motore, annotare l'esito nella matrice di compatibilità (da creare).

## Nota sul clone cloud

Le sessioni Claude Code in cloud clonano il repo **shallow** (50 commit). Per analisi storiche: `git fetch --deepen=<n>` oppure `git fetch --unshallow`.
