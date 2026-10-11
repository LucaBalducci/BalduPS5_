# ADR-0002: Dove vive il progetto "Console personale"

**Status:** Proposed
**Date:** 2026-10-11
**Deciders:** Luca Balducci
**Documenti collegati:** [PROGETTO-CONSOLE.md](PROGETTO-CONSOLE.md), [ADR-0001](ADR-0001-ibrido-anyps5-kyty.md)

## Context

Oggi tutto il nostro materiale sta in `MyDocs/` dentro un **fork** di KytyPS5 (`LucaBalducci/BalduPS5_`). Il progetto però non è una modifica di KytyPS5: è un **prodotto nuovo** (il guscio da console) che **usa** KytyPS5, e in futuro forse AnyPS5, come motori non modificati. Entrambi gli upstream si muovono molto velocemente. Serve decidere dove sviluppare il prodotto, come tenere i motori aggiornati e a cosa serve il fork.

Un fork su GitHub è pensato per due usi: proporre modifiche all'upstream (pull request) oppure mantenere una copia modificata di quel progetto. Non è pensato per ospitare un prodotto diverso che lo usa come dipendenza.

## Decision (proposta)

**Creare un repository separato per il prodotto. I motori entrano come dipendenze fissate a una versione, non come codice nostro. Il fork resta per lo studio e per un'eventuale coda di patch da proporre a monte. `MyDocs/` migra nel nuovo repository.**

Struttura proposta del nuovo repository (nome da scegliere, senza marchi Sony):

```
<console>/
├── README.md              cos'è, come si installa, come si aggiornano i motori
├── CLAUDE.md              direttive dell'agente per questo progetto
├── docs/                  ← contenuto attuale di MyDocs/ (carta di progetto, confronto, intro)
│   └── engines/           ← ARCHITECTURE.md di KytyPS5 come documentazione del motore
├── adr/                   ← ADR-0001, ADR-0002, successivi
├── engines/
│   ├── kytyps5/           submodule git fissato a un commit, oppure build CI scaricata da script
│   └── anyps5/            opzionale, stesso schema
├── shell/                 il guscio (il nostro codice)
├── scripts/               fetch/build/install dei motori, avvio in modalità console
└── compat/                matrice di compatibilità per gioco e per motore (yaml o csv)
```

Il fork `BalduPS5_` resta con due soli ruoli:
1. **Studio**: leggere il codice al commit che ci interessa.
2. **Coda di patch**: se un giorno serve una modifica al motore prima che l'upstream la accetti, vive in un branch `patches/*` del fork, piccola, rebasata a ogni aggiornamento e, se possibile, proposta a monte come PR scritta e testata da un umano (policy dell'upstream).

Il `main` del fork resta un mirror dell'upstream, sincronizzato con il pulsante "Sync fork" di GitHub o con `git fetch upstream && git merge --ff-only upstream/main`.

## Options Considered

### Option A: Tutto nel fork

| Dimension | Assessment |
|-----------|------------|
| Complexity | Bassa all'inizio, alta dopo |
| Cost | Zero oggi; crescente a ogni sync con l'upstream |
| Manutenibilità | Scarsa: il nostro codice e quello upstream si mescolano nella stessa storia git |
| Team familiarity | È lo stato attuale |

**Pros:** nessun passo da fare; tutto in un posto.
**Cons:** ogni sync con l'upstream porta conflitti o rumore nella nostra storia; il prodotto eredita per forza la licenza GPL-2.0 e la struttura di build di KytyPS5; una PR a monte partirebbe da un albero pieno di nostri file; il fork resterebbe legato a un solo motore, mentre il prodotto ne prevede due; il nome del repo è quello dell'upstream.

### Option B: Repository separato con i motori come dipendenze fissate (raccomandata)

| Dimension | Assessment |
|-----------|------------|
| Complexity | Bassa |
| Cost | Mezza giornata per crearlo e migrare `MyDocs/` |
| Manutenibilità | Alta: aggiornare un motore è spostare un puntatore di versione e rilanciare i test di avvio |
| Team familiarity | Richiede di imparare submodule o uno script di fetch, entrambi standard |

**Pros:** confine netto tra nostro e loro; licenza del guscio a nostra scelta (processo separato); due motori allo stesso livello; storia git pulita; il nome e il README sono del prodotto; è il modello di RetroArch (frontend) e dei core libretro (repo separati), e di Lutris/Heroic con i loro runner.
**Cons:** un repository in più da gestire; i submodule hanno una curva di apprendimento; in cloud il nuovo repo va aggiunto all'ambiente di Claude Code.

### Option C: Monorepo con copie dei motori incollate dentro (vendoring)

| Dimension | Assessment |
|-----------|------------|
| Complexity | Media |
| Cost | Alto a ogni aggiornamento |
| Manutenibilità | Pessima con upstream che fanno decine di commit al giorno |
| Team familiarity | Bassa |

**Pros:** un solo clone, nessun submodule.
**Cons:** 160k righe copiate da aggiornare a mano; si perde la storia e la tracciabilità delle versioni; repo enorme; invita a modificare il motore "perché tanto è qui".

## Trade-off Analysis

- **Pulizia vs comodità immediata.** A è comodo oggi e costoso domani; B costa mezza giornata oggi e resta pulito. Con due upstream così veloci, "domani" arriva in settimane.
- **Dipendenza esplicita vs implicita.** B rende esplicito che usiamo i motori a una versione precisa, cosa che serve comunque per la matrice di compatibilità ("il gioco X funziona con KytyPS5 al commit Y").
- **Licenza.** Solo B permette di scegliere la licenza del guscio; in A e C il repo intero è di fatto GPL-2.0.

## Consequences

Cosa diventa più facile: aggiornare i motori; aggiungere AnyPS5 o giochi PC nativi; spiegare il progetto a terzi; eventualmente aprire il guscio a contributi.
Cosa diventa più impegnativo: mantenere due repository (fork e prodotto) e ricordare quale fa cosa; imparare i submodule o scrivere lo script di fetch.
Cosa andrà rivisto: se il prodotto dovesse richiedere patch stabili al motore, valutare se il branch `patches/*` del fork basta o se serve un fork "downstream" dichiarato.

## Action Items

1. [ ] Scegliere il nome del prodotto (senza marchi Sony).
2. [ ] Creare il repository su GitHub e aggiungerlo all'ambiente delle sessioni Claude Code.
3. [ ] Migrare `MyDocs/` in `docs/` e `adr/` del nuovo repo; nel fork lasciare un `MyDocs/README.md` che rimanda al nuovo repo.
4. [ ] Aggiungere KytyPS5 come submodule fissato al commit testato, oppure uno script che scarica la build CI di quel commit.
5. [ ] Scrivere lo script di avvio in modalità console e il primo test "il motore parte".
6. [ ] Riportare il `main` del fork a mirror puro dell'upstream.
