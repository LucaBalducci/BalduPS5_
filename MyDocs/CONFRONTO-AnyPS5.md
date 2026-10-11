# KytyPS5 vs AnyPS5 — confronto

> Documento di studio. Snapshot al 2026-10-10. Fonti: README, note di release e documentazione per sviluppatori di AnyPS5, pagina GitHub di KytyPS5, articoli di stampa. **Il codice sorgente di AnyPS5 non è stato letto**: dove la documentazione non dice nulla, qui c'è scritto "non documentato".

## In una frase

Fanno la stessa cosa con la stessa idea di fondo, reimplementare le librerie Sony e far girare nativamente il codice x86-64 del gioco, e si differenziano per **quando e dove avviene il collegamento** tra gioco e librerie. Lo slogan di AnyPS5 "nessuna emulazione" è più marketing che tecnica: nemmeno KytyPS5 emula la CPU, e su GitHub si descrive come "PlayStation 5 translation layer".

## Stessa idea, due confezioni

Il gioco ha una lista di funzioni di sistema che gli servono, identificate da un codice NID, e qualcuno deve scriverci dentro gli indirizzi delle funzioni reimplementate.

- **KytyPS5** compila la rubrica **a ogni avvio**, dentro il processo `kyty_emulator`. Il file del gioco su disco non viene toccato. L'eseguibile dell'emulatore contiene tutte le librerie reimplementate come tabelle C++, un loader ELF proprio, un kernel finto, una memoria finta e un thread GPU.
- **AnyPS5** compila la rubrica **una volta per tutte** con il *relinker*: prende `eboot.bin` e `sce_module/*` e produce `app.exe`/`app.elf` più `libs/*.prx`, librerie condivise del sistema operativo con gli export rinominati al NID. Il gioco diventa un normale programma lanciato dal loader di Windows o Linux.

## Differenze tecniche

| Aspetto | KytyPS5 | AnyPS5 |
|---|---|---|
| Quando si collega il gioco | A runtime, nel processo emulatore | Offline, con il relinker; poi il loader del sistema operativo |
| File del gioco | Mai modificato | Convertito in un nuovo binario nativo |
| Istruzioni AMD-only (Zen 2) | Riscrittura al caricamento + trap `#UD` ed emulazione software di fallback | Passo `--to-intel` del relinker che le abbassa in anticipo |
| Librerie Sony | Tabelle `LIB_FUNC(nid, fn)` compilate in un unico eseguibile | Una shared library per libreria, export rinominati al NID da `nid_patcher`, `APS5_EXPORT` per i casi speciali |
| Import mancanti | Thunk che logga e ritorna zero; il gioco prova comunque a partire | Non documentato; il loader del sistema operativo è severo per natura |
| GPU | Submit DCB/ACB → parser PM4 su thread dedicato → Vulkan | Submit via `libSceAgcDriver/Submit` → `Execution/Pm4` → Vulkan |
| Shader | Decode → CFG → IR SSA → passi → SPIR-V, builder proprio | `RdnaDecoder` → `ControlFlow` → traduzione in IR → ottimizzazione (SSA, risorse, binding) → `SpirvBackend` |
| Cache shader | Solo `VkPipelineCache` del driver su disco | Varianti in memoria + `ShaderDiskCache` su disco |
| Compilazione shader | Sincrona sul thread GPU | Pool in background (`hardware_concurrency()/2`, min 2 thread) |
| Vulkan | 1.3 obbligatorio, dynamic rendering senza fallback | Pipeline library + dynamic state quando disponibili, fallback a render pass; la stampa parla di 1.1 |
| Errori | `EXIT_NOT_IMPLEMENTED`, processo terminato | `std::runtime_error` stampato su stderr, processo terminato |
| Memoria, address space, TLS | Backing store da 13.5 GiB con viste fisse, range guest riservati, patch TLS nel loader | **Non documentato** |
| Input | SDL3, DualSense completo (trigger adattivi, haptics USB e Bluetooth) | SDL, controller con binding configurabili, tastiera e mouse via `anyps5-input.ini` |
| Piattaforme | Windows, Linux, macOS x86-64 via Rosetta | Windows, Linux |

## Maturità e comunità (snapshot 2026-10-10)

| | KytyPS5 | AnyPS5 |
|---|---|---|
| Origine | Riscrittura di Kyty (2021) | Progetto nuovo, prime release 28 settembre 2026 |
| Stelle / fork | 7.1k / 850 | 24.2k / 2.0k |
| Commit | 1 306 | 4 195 |
| Issue aperte / PR aperte | 549 / 233 | 68 / 164 |
| Release | Nessuna sulla pagina; build CI per commit | 2 (v0.1.0, v0.1.1) |
| Giochi | 2D e selezione di 3D (UE4/5, Unity, engine custom); screenshot di Astro Bot, Demon's Souls, Saros, UFC 6 | Un titolo verificato (Dreaming Sarah, 2D); Demon's Souls "al primo livello" nelle note di v0.1.0 |
| Licenza | GPL-2.0-only | GPL-2.0-only |
| Tooling | Tracy, RenderDoc, ~50 test CTest, test shader eseguiti su GPU reale | HW oracle (esegue assembly RDNA su GPU AMD reale per fissare la semantica), mappa di progresso automatica per libreria |
| Documentazione dev | README ricco, nessun documento di architettura | `ARCHITECTURE`, `BUILD`, `CONVENTIONS`, `TechnicalDebt`, `HW_ORACLE`, `PROGRESS` |

Letture da fare con attenzione: le stelle misurano la popolarità, non la maturità. AnyPS5 ha avuto copertura stampa virale per l'angolo "senza emulazione" e ha un solo gioco verificato; commit e PR dicono però che la comunità è molto attiva. KytyPS5 ha meno rumore ma più giochi che partono e un autore che spinge decine di commit al giorno sul ricompilatore shader.

## Cosa significa per il nostro progetto

- **Studio**: KytyPS5 resta il riferimento principale. La documentazione dev di AnyPS5 è materiale complementare: la sua ARCHITECTURE.md descrive la stessa pipeline shader che abbiamo mappato, e HW_ORACLE.md mostra un metodo rigoroso di verifica delle istruzioni GPU che KytyPS5 non ha.
- **Console**: un launcher stile console può lanciare sia `kyty_emulator --game ...` sia un gioco convertito da AnyPS5. Entrambi GPL-2.0, stessi obblighi se si distribuiscono binari. La decisione su come unirli è in [ADR-0001](ADR-0001-ibrido-anyps5-kyty.md).
- **Attenzione legale specifica ad AnyPS5**: il binario convertito è una copia del gioco. Per uso personale è indifferente; distribuirlo significa distribuire il gioco, cosa vietata. KytyPS5 non modifica mai il file su disco.
- **Instabilità**: AnyPS5 è alla v0.1; CLI, formato di output e API cambieranno. Da ricontrollare tra qualche mese.

## Fonti

- [AnyPS5 README](https://github.com/boykopovar/AnyPS5)
- [AnyPS5 releases](https://github.com/boykopovar/AnyPS5/releases)
- [AnyPS5 docs/dev/ARCHITECTURE.md](https://github.com/boykopovar/AnyPS5/blob/main/docs/dev/ARCHITECTURE.md)
- [AnyPS5 docs/dev/HW_ORACLE.md](https://github.com/boykopovar/AnyPS5/blob/main/docs/dev/HW_ORACLE.md)
- [AnyPS5 docs/dev/PROGRESS.md](https://github.com/boykopovar/AnyPS5/blob/main/docs/dev/PROGRESS.md)
- [KytyPS5 repository](https://github.com/KytyPS5/KytyPS5)
- [Notebookcheck](https://www.notebookcheck.net/Playing-PS5-titles-on-PC-might-be-possible-using-a-different-route-altogether-with-AnyPS5.1409700.0.html)
- [FandomWire](https://fandomwire.com/anyps5-tool-ps5-games-natively-on-pc/)
- [iXBT](https://ixbt.games/en/news/2026/09/17/modder-rabotaet-nad-nativnym-zapuskom-playstation-5-igr-na-pk-s-windows-i-linux.html)
- [pasqualepillitteri.it](https://pasqualepillitteri.it/es/news/21733/anyps5-juegos-ps5-windows-linux)
