# ADR-0001: Unire AnyPS5 e KytyPS5 per la "console personale"

**Status:** Proposed
**Date:** 2026-10-11
**Deciders:** Luca Balducci (owner del fork)
**Riferimenti al codice:** fork `LucaBalducci/BalduPS5_`, commit `91def0c` (sorgenti `src/` identici all'upstream `59183db`)
**Documenti collegati:** [ARCHITECTURE.md](ARCHITECTURE.md), [CONFRONTO-AnyPS5.md](CONFRONTO-AnyPS5.md), [INTRODUZIONE.md](INTRODUZIONE.md)

---

## Context

L'obiettivo del progetto personale è una **"console" funzionante**: un'esperienza da console costruita attorno a software che fa girare giochi PS5 su PC. Esistono due progetti open source, entrambi GPL-2.0-only, che perseguono lo stesso scopo con la stessa idea di fondo e un packaging diverso:

- **KytyPS5** (il progetto che abbiamo forkato): un processo `kyty_emulator` che carica l'`eboot.bin` del gioco nel proprio spazio di indirizzi, lo collega a runtime alle librerie Sony reimplementate (HLE) e fa da kernel finto, GPU finta (PM4 → Vulkan) e compilatore shader (RDNA 2 → SPIR-V). Il file del gioco su disco non viene toccato. Boota giochi 2D e una selezione di 3D.
- **AnyPS5**: un tool chiamato *relinker* converte **offline** l'`eboot.bin` e i moduli `sce_module/*` in un eseguibile nativo (`app.exe` / `app.elf`) più librerie condivise `libs/*.prx` con export rinominati al NID. È poi il loader del sistema operativo a collegare il gioco al runtime di AnyPS5. Un solo gioco verificato alla v0.1.1.

La proposta in esame: **usare la conversione di AnyPS5 e il runtime di KytyPS5**, cioè produrre un eseguibile nativo con il relinker di AnyPS5 e farlo appoggiare alle librerie HLE, alla GPU e al compilatore shader di Kyty impacchettati come `.prx`.

Forze in gioco:

- Il team è una persona, junior, in fase di studio.
- Entrambi gli upstream si muovono molto velocemente (Kyty: decine di commit al giorno; AnyPS5: 4 195 commit in poche settimane alla data di analisi).
- L'esperienza utente desiderata è "seleziono un gioco e parte", su hardware x86-64 con GPU Vulkan.
- Vincoli legali: niente giochi o firmware distribuiti, niente marchi Sony, obblighi GPL se si distribuiscono binari.

---

## Decision

**Respingere l'ibrido a livello di codice (Opzione A). Adottare la "console multi-runtime" (Opzione B) con i due motori usati intatti, e tenere il "travaso di idee" (Opzione C) come obiettivo di crescita a medio termine.**

---

## Options Considered

### Option A: Ibrido nel codice (relinker AnyPS5 + runtime Kyty come libreria)

Flusso: `eboot.bin` → relinker AnyPS5 → `app.exe` → loader del sistema operativo → `libkyty` (HLE, GPU, shader di Kyty esposti come `.prx`).

| Dimension | Assessment |
|-----------|------------|
| Complexity | Molto alta |
| Cost | Mesi di lavoro da esperto, su due basi di codice non proprie |
| Scalability (manutenibilità) | Insostenibile da una persona: va tenuto al passo con due upstream che corrono |
| Team familiarity | Nulla, sia su Kyty che su AnyPS5 |
| Valore per la console | Basso: l'utente non vede differenza tra un processo emulatore e un eseguibile convertito |

**Pros:**
- Il gioco diventa un eseguibile autonomo, aggiungibile a qualsiasi launcher come un gioco PC.
- Esercizio tecnico formativo su linking, ABI e memoria.

**Cons (con le prove nel codice):**

1. **La giuntura che non tiene: il controllo dello spazio di indirizzi.** Kyty riproduce fedelmente la memoria PS5 e deve farlo **prima** che il gioco sia in memoria:
   - `ReserveGuestRegions()` riserva interi range di indirizzi guest sull'host (`src/kernel/memoryAddressSpace.inc:1143`, invocata durante l'inizializzazione della memoria a `:693`).
   - La memoria diretta parte da una base fissa: `GUEST_DEFAULT_MAP_BASE = 0x200000000` (`src/kernel/memory.cpp:833`; stessa costante `DEFAULT_PS5_BASE` a `:2266`).
   - Tutta la RAM "fisica" è un unico backing store da 13.5 GiB con viste fisse, su cui si appoggiano aliasing e coerenza CPU/GPU (vedi ARCHITECTURE.md §6.4 e §7.2).
   In un modello "il gioco è l'eseguibile", il loader del sistema operativo mappa gioco, stack e librerie dove decide lui e solo **dopo** esegue i costruttori di `libkyty`: i range possono essere già occupati e i segmenti del gioco non sono viste del backing store, quindi il meccanismo di coerenza basato su alias scrivibili non li copre. Per restituire a Kyty il controllo della memoria serve un processo che parta prima del gioco: quel processo **è** `kyty_emulator`. L'ibrido, portato fino in fondo, ricostruisce Kyty.

2. **Patch del codice guest oggi fatte dal loader di Kyty.** `PatchProgram()` riscrive gli accessi TLS via segmento FS e sostituisce con NOP gli store a `fs:[0x28]` (`src/loader/runtimeLinker.cpp:940`, pattern a `:948`). Nell'ibrido questo lavoro deve migrare nel relinker, offline, su un tool che non controlliamo.

3. **Le librerie HLE di Kyty non sono autonome.** 16 file in `src/libs/` includono direttamente header di `kernel/` o `graphics/` (conteggio al commit `91def0c`), e tutto passa per singleton globali (`Common::Singleton<Loader::RuntimeLinker>`, `g_renderer`). Esportarle come `.prx` significa una sola grande `libkyty` che deve rispettare il contratto di inizializzazione di AnyPS5, contratto che la documentazione di AnyPS5 (`docs/dev/ARCHITECTURE.md`) non descrive.

4. **Tolleranza agli import mancanti.** Kyty boota più giochi anche perché un import sconosciuto diventa un thunk che logga e ritorna zero (`src/loader/runtimeLinker.cpp:145-215`). Il loader del sistema operativo è severo: simbolo mancante, processo che non parte. Servirebbe una libreria di stub generata per ogni NID conosciuto.

5. **Due upstream da seguire** modificandone il cuore: non mantenibile da una persona.

6. Il guadagno concreto (eseguibile nativo aggiungibile a Steam) si ottiene con uno script di una riga che lancia `kyty_emulator --game`.

### Option B: Console multi-runtime (motori intatti)

Un guscio nostro, separato, che sa lanciare `kyty_emulator --game <dir|zar>`, sa invocare il relinker di AnyPS5 e poi lanciare l'eseguibile prodotto, sa lanciare giochi PC nativi, e per ogni titolo ricorda quale motore funziona meglio. È il modello di RetroArch con i suoi core e di Lutris/Heroic con i loro runner.

| Dimension | Assessment |
|-----------|------------|
| Complexity | Bassa |
| Cost | Settimane per un prototipo utile |
| Scalability (manutenibilità) | Minima: i motori si aggiornano da soli, il guscio parla solo per riga di comando e file |
| Team familiarity | Adatta a un junior: UI, processi, file di configurazione |
| Valore per la console | Alto: è esattamente l'esperienza desiderata |

**Pros:**
- Prende il meglio di entrambi i motori senza toccare una riga dei loro core.
- Ogni passo avanti degli upstream arriva gratis.
- Confine di processo: il guscio è un programma a sé, con libertà di licenza e di stack tecnologico.
- Può ospitare anche giochi nativi, inclusi quelli che faremo noi con un motore normale.

**Cons:**
- Dipende dalla stabilità delle interfacce a riga di comando dei motori; AnyPS5 è alla v0.1 e cambierà.
- Due installazioni da gestire (Kyty e AnyPS5) e due famiglie di cartelle di salvataggio/cache da mappare.
- Non "unisce" i motori a livello tecnico: è un'orchestrazione.

### Option C: Travaso di idee (non di codice) da AnyPS5 a un fork di Kyty

Portare in Kyty, come feature isolate, le idee buone di AnyPS5. Nessuna di esse è il relinker:

- cache degli shader su disco (Kyty salva solo il `VkPipelineCache` del driver, ARCHITECTURE.md §7.5);
- preparazione degli shader su un pool in background invece che in modo sincrono sul thread GPU (rischio #3 in ARCHITECTURE.md §11);
- test guidati da un "HW oracle" che misura il comportamento reale delle istruzioni RDNA;
- fallback a render pass per driver senza dynamic rendering.

| Dimension | Assessment |
|-----------|------------|
| Complexity | Media-alta |
| Cost | Settimane per la prima feature, con guida |
| Scalability (manutenibilità) | Media: va seguito l'upstream, idealmente contribuendo a monte |
| Team familiarity | Richiede prima i passi 1-2 del percorso di studio |
| Valore per la console | Alto ma indiretto: migliora il motore principale |

**Pros:** valore per tutta la comunità; apprendimento profondo del motore; se accettato a monte, zero manutenzione nostra.
**Cons:** richiede competenze C++/Vulkan non ancora acquisite; la policy dell'upstream richiede PR scritte, revisionate e testate da un umano, con disclosure dell'assistenza AI.

---

## Trade-off Analysis

- **Valore vs costo.** A ha il costo massimo e il valore minimo per l'obiettivo "console": l'utente non distingue un processo emulatore da un eseguibile convertito. B ha costo minimo e valore massimo perché realizza direttamente l'esperienza desiderata.
- **Controllo vs dipendenza.** A dà il massimo controllo tecnico ma crea una dipendenza strutturale da due upstream instabili. B accetta la dipendenza ma la confina a un contratto sottile (riga di comando, cartelle), che è il tipo di dipendenza più facile da adattare.
- **Apprendimento.** A insegna molto ma su un problema che non ha una soluzione pulita (vedi punto 1 dei cons: la soluzione corretta è il processo-contenitore, cioè Kyty). C insegna altrettanto su problemi che hanno una soluzione e un valore riconosciuto.
- **Legale.** Nessuna differenza di licenza tra le opzioni (tutto GPL-2.0-only). Attenzione specifica ad AnyPS5: il binario convertito **è** una copia del gioco e non va mai distribuito; B lo gestisce tenendo la conversione locale e per-utente.

---

## Consequences

Cosa diventa più facile:
- Cambiare motore per singolo gioco e aggiornare i motori senza rischio.
- Aggiungere un terzo tipo di contenuto (giochi PC nativi, nostri inclusi).
- Spiegare il progetto: "un guscio da console sopra motori esistenti".

Cosa diventa più importante:
- Progettare bene il **contratto del guscio verso i motori**: come lanciarli, dove stanno salvataggi e cache (Kyty: `_SaveData/<TITLE_ID>`, `_PipelineCache`, ecc.), come leggere esito e log, come gestire crash e aggiornamenti.
- Tenere una **matrice di compatibilità per gioco e per motore**, aggiornata a ogni upgrade.

Cosa andrà rivisto:
- Quando AnyPS5 stabilizzerà la propria interfaccia (oltre la v0.1): rivalutare quanto del flusso di conversione automatizzare.
- Se un giorno la scelta sarà contribuire a Kyty (Opzione C): aprire un ADR dedicato per la prima feature scelta.
- Questo ADR va riletto se Kyty dovesse offrire una modalità "eseguibile autonomo" o se AnyPS5 documentasse il proprio modello di memoria: il punto 1 dei cons di A cambierebbe.

---

## Action Items

1. [ ] Compilare e avviare KytyPS5 in locale con un gioco legalmente posseduto (percorso Linux senza Qt in CLAUDE.md, oppure Windows da README).
2. [ ] Provare AnyPS5 sullo stesso gioco e annotare esito e differenze in `MyDocs/`.
3. [ ] Scrivere `MyDocs/VISIONE-CONSOLE.md`: cosa fa il guscio, su quale hardware, con quale interfaccia, quali motori.
4. [ ] Definire il contratto guscio ↔ motori (riga di comando, cartelle, esito) in un documento breve.
5. [ ] Prototipo del guscio: una lista giochi che lancia i due motori, con scelta del motore per titolo.
6. [ ] Rivalutare questo ADR tra tre mesi o alla prima release stabile di AnyPS5.

---

## Appendice: cosa è stato verificato e cosa no

Verificato nel codice del fork (commit `91def0c`):
- `ReserveGuestRegions` in `src/kernel/memoryAddressSpace.inc:1143`, chiamata a `:693`.
- `GUEST_DEFAULT_MAP_BASE = 0x200000000` in `src/kernel/memory.cpp:833`.
- `PatchProgram` in `src/loader/runtimeLinker.cpp:940`, pattern dello store `fs:[0x28]` a `:948`.
- 16 file di `src/libs/` includono header di `kernel/` o `graphics/`.

Letto ma non verificato nel codice: README, note di release e `docs/dev/{ARCHITECTURE,HW_ORACLE,PROGRESS}.md` di AnyPS5. In particolare **il modello di memoria e la gestione di TLS/address space di AnyPS5 non sono documentati**, quindi non è noto come il suo runtime eviti il problema del punto 1; è una domanda aperta da chiarire leggendo il suo codice se mai si volesse riaprire l'Opzione A.
