# KytyPS5 — Mappa dell'architettura

> Documento di studio del fork `LucaBalducci/BalduPS5_` (upstream: `KytyPS5/KytyPS5`).
> Non fa parte del progetto originale: vive in `MyDocs/`, la cartella dei nostri materiali.

| Campo | Valore |
|---|---|
| Commit analizzato | `59183db` ("shader: add missing F64 VOPC compares"), 2026-10-09 |
| Branch di lavoro | `ccr-bec8a371-t8ufmi`, identico a `main` al momento dell'analisi |
| Versione dichiarata in CMake | `0.3.0` |
| Linguaggio / standard | C++20, solo Clang (GCC tollerato su Linux, MSVC `cl.exe` rifiutato) |
| Backend grafico | Vulkan 1.3 (MoltenVK su macOS) |
| Licenza | GPL-2.0-only; la base Kyty originale è MIT (`LICENSES/Kyty-MIT.txt`) |
| Metodo | Lettura diretta di entry point, build, CI, launcher e layer di piattaforma; esplorazione dei quattro sottosistemi grossi (loader/kernel, libs, GPU, shader) con verifica a campione delle affermazioni non ovvie. Conteggi con `wc -l` su `.cpp/.h/.inc` |

**Avvertenze.** Il clone usato è *shallow* (50 commit): la storia visibile copre solo l'8–9 ottobre 2026. I numeri di riga citati valgono per il commit sopra; l'upstream fa decine di commit al giorno, quindi i riferimenti `file:riga` vanno presi come punti d'ingresso, non come verità stabili.

---

## 0. In una pagina

KytyPS5 è un emulatore PlayStation 5 che **non emula la CPU**. La PS5 è un x86-64 (AMD Zen 2) con GPU AMD RDNA 2 e kernel derivato da FreeBSD, quindi l'ELF del gioco viene caricato nello spazio di indirizzi del processo `kyty_emulator` ed eseguito nativamente. Viene emulato tutto ciò che sta attorno al codice del gioco, in **HLE** (High-Level Emulation):

1. **Kernel**: memoria virtuale, thread, sincronizzazione, filesystem (`src/kernel/`).
2. **Librerie di sistema Sony** (`sceKernel*`, `scePad*`, `sceAgc*`, audio, savedata, trofei, rete…): reimplementate in C++ host e collegate al gioco dal runtime linker tramite NID (`src/libs/`, `src/loader/`).
3. **GPU**: i command buffer PM4 che il gioco scrive in memoria vengono parsati da un thread GPU e tradotti in Vulkan (`src/graphics/guest_gpu/`, `src/graphics/host_gpu/`).
4. **Shader**: i binari RDNA 2 vengono ricompilati a runtime in SPIR-V (`src/graphics/shader/`).

Le parti difficili, e quelle dove si concentrano codice e commit recenti, sono quattro: il **linking** (NID, patch di istruzioni, TLS), il **modello di memoria unificata CPU/GPU** (coerenza via protezione pagine), la **GPU** (un thread che emula 57 code hardware) e il **ricompilatore shader** (in particolare il *resource binding* tramite tabelle SRT lette dalla memoria).

---

## 1. Contesto

- **Origine**: fork pesantemente modificato di [Kyty](https://github.com/InoriRus/Kyty) (InoriRus). Riferimenti dichiarati a shadPS4 per memoria PS4, aliasing GPU, coerenza cache, AvPlayer; il patcher di istruzioni è dichiaratamente preso da shadPS4.
- **Piattaforme**: Windows e Linux primarie (Windows è il target di riferimento per le PR). macOS sperimentale, build x86-64 eseguita sotto Rosetta 2, Vulkan via MoltenVK.
- **Stato**: boot di giochi 2D e di una selezione di 3D (UE4/5, Unity, engine custom). Compatibility list comunitaria separata.
- **Attività upstream**: nel clone, 50 commit in due giorni, autore principale `nmzik`. 27 su 50 hanno prefisso `shader:`.
- **Policy AI**: l'uso di AI è ammesso per ricerca e sviluppo, ma PR, commenti e issue devono venire dall'umano, con disclosure dello scopo dell'assistenza e descrizione di review e test effettuati.

---

## 2. Il modello di emulazione

### 2.1 Cosa gira nativo e cosa è emulato

| Componente PS5 | Approccio in KytyPS5 |
|---|---|
| Codice x86-64 del gioco | Esecuzione nativa nel processo host |
| ISA Zen 2 non disponibile sull'host | Riscrittura statica al load (Zydis + Xbyak) oppure trap `#UD` + emulazione software |
| TLS via segmento FS | Riscrittura statica: `mov reg, fs:[0]` diventa una call a uno stub; gli store a `fs:[0x28]` diventano NOP |
| Kernel (memoria, thread, equeue, fs) | HLE in `src/kernel/` esposto via `libs/libKernel.cpp` |
| Librerie `libSce*` | HLE in `src/libs/`, una funzione C++ per NID |
| `libc` | Si carica la `libc.prx` del gioco se presente; `libs/libC.cpp` fornisce ~54 NID di fallback |
| GPU (comandi PM4) | Parser PM4 su thread dedicato, renderer Vulkan |
| Shader RDNA 2 | Ricompilazione a SPIR-V con IR propria |
| Memoria unificata | Un unico backing store condiviso con viste fisse; coerenza CPU/GPU via protezione pagine |

### 2.2 Perché serve comunque un emulatore di istruzioni

Tutto in `src/loader/`:

- **Static rewrite al caricamento** (`PatchProgram`, `runtimeLinker.cpp:940`): accesso TLS e store su `fs:[0x28]`, perché Windows non può ospitare il segmento FS del guest.
- **Rewrite opzionale con Zydis/Xbyak** (`guestInstructionPatcher.cpp`, attivato da `--amd-cpu` e `--redzone`): i confini delle funzioni vengono da `.eh_frame_hdr`. `VRSQRTPS` (tabelle di approssimazione AMD e Intel differiscono), `EXTRQ`, `INSERTQ`, `RDPID`, `CLWB` vengono spostate in trampolini quando l'host non le ha; se impossibile, l'istruzione viene trasformata in un trap `#UD` forzato. Su Windows la protezione della *red zone* SysV (128 byte) sposta le istruzioni di memoria mentre la red zone è viva, perché altrimenti il vectored exception handler la sovrascriverebbe.
- **Fallback a runtime** (`KytyExceptionHandler`, `runtimeLinker.cpp:659` → `X64InstructionEmulator::TryEmulate`, `x64InstructionEmulator.cpp:802`): emula RSQRT, MONITORX/MWAITX, SSE4a EXTRQ/INSERTQ, SHA-NI, RDPID, CLWB. Su macOS (Rosetta senza SSE4a) il launcher nasconde l'opzione AMD e si usa solo il percorso a trap.
- Un **access violation** viene prima offerto a `HandleGpuFault` (tracking pagine GPU, vedi §6.4); se non è un fault GPU, è fatale.

Le syscall raw non vengono mai intercettate: il guest chiama solo gli export HLE di libkernel.

---

## 3. Misure

### 3.1 Dimensione per sottosistema (`.cpp` + `.h` + `.inc`)

| Directory | Righe | Quota |
|---|---:|---:|
| `src/libs/` | 55 493 | 34 % |
| `src/graphics/shader/` | 35 698 | 22 % |
| `src/graphics/host_gpu/` | 18 640 | 11 % |
| `src/kernel/` | 14 589 | 9 % |
| `src/graphics/guest_gpu/` | 10 691 | 7 % |
| `src/common/` | 8 154 | 5 % |
| `src/graphics/presentation/` | 7 418 | 5 % |
| `src/loader/` | 7 366 | 5 % |
| `src/launcher/` | 6 547 | 4 % |
| **Totale `src/` (solo `.cpp`/`.h`)** | **160 509** | 211 file `.cpp`, 181 file `.h` |

### 3.2 File più grandi

| File | Righe | Cosa contiene |
|---|---:|---|
| `src/libs/agc.cpp` | 4 580 | HLE dell'API grafica AGC: encoder PM4, init driver, shader setup |
| `src/kernel/pthread.cpp` | 4 435 | pthread guest su host thread, stack guest, TLS cleanup |
| `src/kernel/memory.cpp` | 4 409 | Memoria diretta/flessibile/pooled, mapping, rollback |
| `src/libs/network.cpp` | 4 368 | Socket, DNS, HTTP, SSL, NetCtl |
| `src/graphics/guest_gpu/command_processor/pm4Handlers.cpp` | 3 930 | Handler dei pacchetti PM4 |
| `src/libs/libKernel.cpp` | 3 453 | Tabella NID di libkernel (392 NID) e wrapper |
| `src/libs/libNet.cpp` | 2 993 | Tabelle NID di 17 librerie di rete/NP |
| `src/libs/ngs2.cpp` | 2 602 | Motore audio NGS2 (rack/voice) |
| `src/libs/libAmpr.cpp` | 2 517 | Ampr/Apr: I/O asincrono a command buffer |
| `src/graphics/shader/recompiler/ir/passes/ResourceTracking.cpp` | 2 493 | Tracking risorse e provenienza scalare |
| `src/libs/libFont.cpp` | 2 337 | Font via stb_truetype |
| `src/loader/runtimeLinker.cpp` | 2 192 | Loader ELF, relocazioni, risoluzione NID, patch |
| `src/libs/avPlayer.cpp` | 2 084 | AvPlayer su FFmpeg |
| `src/graphics/host_gpu/renderer/cache/textureCache.cpp` | 2 063 | Cache texture, overlap, detiling, GC |
| `src/graphics/shader/recompiler/frontend/decode/VectorAluOps.cpp` | 2 011 | Tabelle decode VALU |
| `src/graphics/presentation/videoOut.cpp` | 2 010 | VideoOut, vblank, flip queue |

### 3.3 Test, CI, dipendenze

| Voce | Valore |
|---|---|
| File in `tests/` | 32 (30 `.cpp` + 2 `.inc` inclusi dai test shader) |
| Eseguibili di test registrati in `kyty_tests` | 29 (+ `macos_sse4a_tests` solo su macOS) |
| Test CTest registrati | ~50 (molti eseguibili espongono sotto-test via flag `--xxx-only`) |
| Test eseguiti in CI Windows | 12 (subset: audio_out2_port, pad_haptics, controller_settings, ime_dialog, lru_cache, resource_tracking, scalar_provenance, resource_materialization, shader_vertex_metadata, archive_file, avplayer_file, virtual_memory_allocation) |
| Framework di test | Nessuno: `Check(bool, msg)` + `abort()`; i test shader parlano con Vulkan reale |
| CI | GitHub Actions, 3 job (windows-2022, ubuntu-24.04, macos-15), Qt 6.10.3, sccache, release tag `KytyPS5-YYYY-MM-DD-sha7` |
| Submodule git | 14 (SDL3 3.4.x, fmt, spdlog, tracy, imgui, magic_enum, xxHash, nlohmann_json, Vulkan-Headers, SPIRV-Tools/Headers, VMA, LibAtrac9, ffmpeg-core prebuilt) |
| Fetch a configure-time | opus, xbyak, zydis, zstd, ZArchive (serve rete) |
| Vendored in `3rdparty/` | cpuinfo, renderdoc API header, stb, winpthread, patches |
| Review bot | CodeRabbit (`.coderabbit.yaml`, output ridotto) |
| Formattazione | clang-format v22 via pre-commit su `src/**/*.{cpp,h,inc}` |

### 3.4 Storia git (clone shallow)

| Voce | Valore |
|---|---|
| Commit visibili | 50 (2026-10-08 → 2026-10-09) |
| Autore | `nmzik` |
| Prefissi commit | `shader:` 27, `kernel:` 3, `agc:` 3, `ampr:` 2, poi `tests`, `sysmodule`, `renderer`, `net`, `launcher`, `font`, `cmake` 1 ciascuno |

Convenzione dei messaggi: `area: descrizione imperativa breve`.

---

## 4. Mappa dei moduli

```
src/
├── main.cpp, emulator.cpp     CLI + sequenza di boot
├── common/                    base condivisa (log, config, profiler, piattaforma, archivi)
│   ├── logging/               spdlog wrapper
│   └── platform/              sysWindows*/sysLinux*: memoria virtuale, file I/O, debug/eccezioni
├── loader/                    ELF/SELF, runtime linker, NID, patcher x64, cheat, param.json
├── kernel/                    HLE kernel: memory, pthread, equeue, eventFlag, semaphore, futex, fs
├── libs/                      HLE librerie Sony (una lib*.cpp per tabella NID)
│   └── ajm/                   decoder audio (AAC/MP3/Opus via FFmpeg, ATRAC9 via LibAtrac9)
├── graphics/
│   ├── guest_gpu/             lato PS5: PM4, stato registri, tiling, formati
│   │   └── command_processor/ dispatch + handler PM4
│   ├── host_gpu/              lato Vulkan: memoria, tracking, renderer, cache, shader compute embedded
│   │   ├── renderer/          draw, sync, scheduler
│   │   │   ├── cache/         texture/buffer cache, fault manager
│   │   │   ├── image/         tiler, image info
│   │   │   └── pipeline/      pipeline cache, descriptor, shaders
│   │   └── shaders/           GLSL compute (detiling, fault buffer, blit) compilati a build time
│   ├── shader/                ricompilatore RDNA2 → SPIR-V
│   │   └── recompiler/
│   │       ├── frontend/      decode/, cfg/, translate/
│   │       ├── ir/            Value, Block, opcodes/*.inc, passes/
│   │       └── backend/spirv/ emitter SPIR-V (builder proprio)
│   └── presentation/          window (SDL3), swapchain, videoOut, overlay ImGui, RenderDoc
└── launcher/                  GUI Qt 6 (processo separato)
tests/                         test di regressione mirati
3rdparty/                      submodule + vendored
```

### 4.1 Responsabilità e confini

| Modulo | Possiede | Non possiede |
|---|---|---|
| `loader` | Formato ELF/SELF, layout dei moduli in memoria guest, relocazioni, database simboli, patch istruzioni | L'esecuzione dei thread guest (kernel/pthread) |
| `kernel` | Spazio di indirizzi guest, backing store, pthread/TLS, primitive di sync, mount point | Le tabelle NID (stanno in `libs/libKernel.cpp`) |
| `libs` | Semantica delle API Sony, **encoding PM4**, ciclo di vita del sottosistema grafico (`Libs::Graphics::Initialize` crea finestra, VideoOut, renderer, compilatore shader) | Tutto ciò che succede dopo il submit |
| `graphics/guest_gpu` | Parsing PM4, modello registri, matematica del tiling | Oggetti Vulkan |
| `graphics/host_gpu` | Oggetti Vulkan, cache, coerenza memoria, detiling su GPU | Significato delle API AGC |
| `graphics/shader` | Da binario RDNA 2 a SPIR-V, piano risorse, interprete SRT | Creazione pipeline (host_gpu) |
| `graphics/presentation` | Swapchain, vblank virtuale, overlay di sistema | Rendering dei frame |
| `launcher` | Configurazioni, lista giochi, mapping input, trofei, update checker | Qualsiasi logica di emulazione |

Il **punto di contatto unico** fra `libs` e `graphics` è `g_renderer->GetGpu()` (`agc.cpp:46`, `4321`, `4380`).

---

## 5. Binari, build e configurazione

### 5.1 Due eseguibili

- **`kyty_emulator`**: l'emulatore vero. Si lancia da CLI con `--game <dir|elf|zar>` e ~40 opzioni (`src/main.cpp`). Non dipende da Qt.
- **`launcher`**: GUI Qt 6 (Concurrent, Network, Widgets). Avvia `kyty_emulator` come **processo separato** via `QProcess`, dentro un terminale (su Linux genera uno script bash; su Windows usa `cmd /K` con nuova console) — `launcher/src/mainDialog.cpp:405`. Opzione `KYTY_BUILD_LAUNCHER=OFF` per build senza Qt.

### 5.2 Target CMake rilevanti

- `kyty_emulator`, `launcher`, `kyty_tests` (aggregato), `common` (libreria).
- Gli shader GLSL in `host_gpu/shaders/` vengono compilati a **build time** con `glslang -V --target-env vulkan1.3 -Os` e incorporati come header `*_spv.h` tramite `embed_spirv.cmake`: 12 tiler (`standard256/4/4_3d/64/64_3d`, `prt/_3d`, `render_target`, `depth`, `promote/demote_d16`, `color_transform`), `fault_buffer_process` (con `-DCACHING_PAGEBITS=14 -DMAX_PAGE_FAULTS=1024`), tre shader di blit/overlay.
- Provenienza build: `KYTY_BUILD_ORIGIN` ∈ {Source, Official, Fork}. La CI setta `Fork` per qualsiasi repo diverso da `KytyPS5/KytyPS5`, quindi **le build del tuo fork si auto-identificano come Fork** nei log diagnostici.
- `kytyGitVersion.h` generato a ogni build da `generate_version.cmake`.
- macOS: `macosGuestAddressSpace.cpp` + flag linker (`-segaddr`, `-pagezero_size`, `-image_base 0x700000000000`) per ancorare lo spazio guest con sezioni `.zerofill`; firma con entitlement JIT.

### 5.3 Configurazione a runtime

`Config::ConfigOptions` (`common/emulatorConfig.h`) copre: dimensioni finestra, utente, lingua console (0–29), present mode (Fifo/Mailbox/Immediate, default Mailbox), indice GPU, vblank virtuale (default 60), validazioni Vulkan/GPU-assisted/shader, tipo di ottimizzazione shader, log shader e dump command buffer, printf guest, profiler Tracy, RenderDoc, patch AMD, red zone (solo Windows), keymap DualSense, trofei, PlayGo hack, skip notice screen.

Cartelle di lavoro create accanto all'eseguibile: `_SaveData/<TITLE_ID>/`, `_DownloadData/<TITLE_ID>/`, `_TempData/<TITLE_ID>/`, `_PipelineCache/<title>.bin`, `_Shaders/`, `_Buffers/`, `_Textures/`, `_RenderDoc/`, `_kyty.txt`.

---

## 6. Flussi principali

### 6.1 Boot (`src/emulator.cpp`)

1. `main` → `VirtualMemory::Init()`, `InitializeThreads()`, parsing argomenti, `Emulator::Run`.
2. `Common::Subsystems` inizializza in ordine esplicito, distruzione automatica inversa: **Config → Log → Timer → Pthread → Profiler → Network → Memory → FileSystem → Controller → Audio → Graphics**. Ogni sottosistema espone una `struct Lifecycle { name, initialize, shutdown[, emergency_shutdown] }`.
3. Lettura di `sce_sys/param.json` (non `param.sfo`): TITLE_ID, APP_VER, titolo localizzato, `kernel.flexibleMemorySize` → `SetFlexibleMemorySize`.
4. `atexit(KytyClose)` e `at_quick_exit(EmergencyShutdownActive)`.
5. Mount: `/app0` e `/hostapp` → cartella gioco (o radice del `.zar`); `/download0` → `_DownloadData/<id>`; `/temp0`, `/temp` → `_TempData/<id>`.
6. `Libs::InitAll(rt->Symbols())`: tutte le `lib*.cpp` registrano i loro NID nel `SymbolDatabase`.
7. `RuntimeLinker::LoadProgram` dell'eboot (vedi §7.1).
8. `Execute`: un `Common::Thread` guest chiama `RuntimeLinker::Execute` (carica `libc.prx`, `RelocateAll`, applica cheat, chiama `DT_INIT` di libc, poi `RunEntry` con switch di stack in inline asm). Il main thread entra in `Libs::Graphics::WindowRun()` (loop SDL). All'uscita `std::quick_exit(0)`.

### 6.2 Risoluzione di un import (NID)

1. Nel gioco un import ha nome `NID#libId#modId` (id base64-like, decodificati da `EncodeId64`, `runtimeLinker.cpp:723`).
2. Lato host, `libs/libs.h` offre i macro `LIB_VERSION(lib, lv, mod, maj, min)` e `LIB_FUNC(nid, fn)`; la chiave memorizzata è `NID[lib_vN][mod_vM.m][Func]`, prefisso `libSce` rimosso (`symbolDatabase.cpp:16`).
3. `Resolve` (`runtimeLinker.cpp:1327`) prova in ordine: (a) match esatto nel database host; (b) export dei PRX caricati; (c) **fallback per solo NID** con scansione lineare, loggato come "PS5 NID fallback".
4. `RelocateRecord` scrive l'indirizzo host nello slot di import. Relocazioni gestite: `R_X86_64_64`, `GLOB_DAT`, `JUMP_SLOT`, `RELATIVE`, `DTPMOD64`.
5. Le funzioni host sono `KYTY_SYSV_ABI` (`__attribute__((sysv_abi))`, `common/abi.h`): il `call` del gioco atterra **direttamente** nel C++ anche su Windows, senza trampolino.
6. **Import non risolto**: una funzione riceve un thunk di 34 byte che logga "Unresolved import stub called" (max 1024 volte) e ritorna 0 in `rax`/`xmm0`; un oggetto dati punta a una pagina no-access a `0x840000000`, il cui accesso è fatale.

Esempio end-to-end: `scePadOpen` → import `xk0AcarP3V4#…` → `libPad.cpp:13` `LIB_VERSION("Pad",1,"Pad",1,1)` + `libPad.cpp:169` `LIB_FUNC("xk0AcarP3V4", Controller::PadOpen)` → chiave `xk0AcarP3V4[Pad_v1][Pad_v1.1][Func]` → slot patchato con `&Controller::PadOpen` (`controller.cpp:1009`).

### 6.3 Un frame, dall'API al monitor

1. **Encoding**: il gioco chiama le API AGC. `libs/agc.cpp` scrive **pacchetti PM4 reali** (DRAW_INDEX_2, DISPATCH_DIRECT, SET_SH_REG…) nel `CommandBuffer` di proprietà del gioco; se lo spazio finisce, l'host richiama la *grow callback* del gioco (host → guest). I comportamenti solo-emulatore viaggiano in pacchetti `IT_NOP` estesi: `R_FLIP`, `R_RELEASE_MEM`, `R_ACQUIRE_MEM`, `R_WAIT_FLIP_DONE`, `R_CONTEXT_STATE` (`pm4.h:81-96`).
2. **Submit**: `AgcDriverSubmitDcb/Acb` → `GuestGpu::Submit/SubmitCompute` (`graphicsRun.cpp:137-161`). **Zero copy**: in coda va uno span sulla memoria guest. Queue id `0x20–0x58` = compute.
3. **Thread GPU** (`Thread_Gpu`, `graphicsRun.cpp:442-537`): 1 command processor grafico + fino a 56 compute (7 pipe × 8 code, creati on-demand), serviti round-robin. `ProcessPm4` dispatcha header Type-3 su tabelle `constinit` da 256 voci (`pm4Dispatch.cpp`). Pacchetti bloccanti (`WAIT_REG_MEM`, predicazione) sospendono la submission e passano ad altre code; se tutte bloccate, attesa fino a 100 ms.
4. **Stato registri**: `hardwareContext.h` tiene uno stato **già decodificato** (non un register file grezzo): `HW::Context` (render target, blend, depth, viewport, shader), `HW::UserConfig`, `HW::Shader` con 32 user SGPR tipizzati; push/pop del contesto supportato.
5. **Draw** (`renderDraw.cpp:1184-1296` → `ExecutePreparedDraw`): mutex del renderer → shortcut per clear/copy di metadati → lookup programmi shader da stato registri → risoluzione target colore/depth → binding → lookup pipeline → `vkCmdBeginRendering` + draw (o `drawMeshTasksEXT`: i geometry shader NGG diventano **mesh shader**).
6. **Flip**: `AgcDcbSetFlip` emette `R_FLIP` → `CpOpFlip` → `PrepareVideoOutFlip` → `FlipQueue::Prepare`; oppure flip CPU via `VideoOutSubmitFlip` instradato come submission `FlipPreparation` per ordinarlo dopo il lavoro GPU già inviato.
7. **Present thread** (`videoOut.cpp:260, 855-923`): emula vblank e pre-vblank alla frequenza configurata, solleva eventi `EVFILT_VIDEO_OUT`, presenta al massimo un flip group pronto per tick, ripresenta l'ultimo frame se in pausa o se l'overlay va aggiornato. Il secondo bus VideoOut viene blendato sopra (`gpu_video_out_overlay.frag`).

### 6.4 Coerenza memoria CPU/GPU

La PS5 ha memoria unificata: il gioco scrive liberamente in buffer che la GPU legge, e la GPU scrive in memoria che la CPU rilegge. L'emulatore lo risolve con la **protezione pagine dell'host**:

- Granularità: pagine tracker da **4 KiB**, regioni da **4 MiB**, spazio 2^44 (`regionDefinitions.h`).
- Per regione, `RegionManager` tiene bitmap `cpu_dirty` / `gpu_dirty` che **non devono mai sovrapporsi** (enforced con `EXIT`). Le pagine in sync con la GPU sono **write-protected**; le pagine sporcate dalla GPU sono **NoAccess**.
- `PageManager` tiene watcher ref-counted per pagina e batcha le `mprotect`.
- Fault path: handler SIGSEGV/VEH (`runtimeLinker.cpp:659`) → `HandleGpuFault` (`kernel/memory.cpp:929`) → `RenderContext::HandleFault`. Su **write**: buffer e texture cache invalidano la pagina (CPU-dirty, sprotetta). Su **read** di memoria GPU-dirty: `ReadMemory` salta sul thread GPU con `SendCommandSync`, aspetta la GPU, scrive i dati nel backing tramite un alias non protetto (`WriteBacking`). Il thread GPU drena queste richieste **tra un pacchetto e l'altro**, così il thread guest non resta bloccato dietro un command stream lungo.
- Upload: solo i range CPU-dirty vengono caricati prima dell'uso; letture piccole di dati CPU-dirty bypassano la cache via stream buffer.
- Texture: osservano pagine intere; un'immagine che condivide solo una pagina di bordo con una scrittura CPU è marcata "forse sporca" e **rivalidata hashando i byte di bordo**, per ridurre false invalidazioni.
- **Shader con puntatori raw** (FLAT/global via Buffer Device Address): consultano una page table lato GPU a **16 KiB** (`CACHING_PAGEBITS=14`, ~1.5 TiB coperti, **~768 MiB** di memoria device). Pagina assente → bit in una bitmap di fault da 12 MiB e lettura di zero. A GC time il compute `fault_buffer_process.comp` converte la bitmap in max **1024** indirizzi (`MAX_PAGE_FAULTS`); un callback differito crea i buffer, validi dal submit successivo.
- **Attenzione**: le label EOP/release-mem vengono scritte in memoria guest **al parse**, non al completamento GPU (`graphicsRun.cpp:1085-1088`); solo interrupt e flip sono differiti. È corretto solo perché le pagine GPU-dirty sono NoAccess (vedi §11).

---

## 7. Sottosistemi in dettaglio

### 7.1 Loader (`src/loader/`)

- **Formati** (`elf.cpp`): `Elf64::Open` legge prima l'header SELF; `IsSelf` accetta i magic `4F153D1D` (PS4) e `5414F5EE` (PS5), altrimenti ELF grezzo. `IsValid` richiede OSABI FreeBSD, x86-64, `EI_ABIVERSION` 0 o 2, tipo `ET_DYNEXEC` (0xfe10) o `ET_DYNAMIC` (0xfe18). Solo SELF **decriptati e non compressi**: un segmento compresso termina con `EXIT_NOT_IMPLEMENTED`.
- **Layout** (`LoadProgramToMemory`, `runtimeLinker.cpp:1647`): un'unica regione guest per immagine + area trampolini da 8 MiB + pagina handler TLS. Moduli piazzati da `0x900000000` in passi di 256 MiB. Segmenti `PT_LOAD`/`PT_OS_RELRO` copiati, registrati `PT_TLS`, `PT_OS_PROCPARAM`, `PT_GNU_EH_FRAME`; poi patch dei segmenti eseguibili e protezione. Eseguibili PS4 rifiutati; accettati solo moduli shared o eseguibili PS5 (ABI 2).
- **Startup** (`RuntimeLinker::Execute`, `:1250`; `RunEntry`, `:234`): stack guest, `libc.prx`, relocazioni, cheat, `DT_INIT`, poi switch RSP/RBP e `entry(EntryParams*, atexit)`. Su Windows azzera i bound dello stack nel TEB (`gs:0x08`, `gs:0x10`).
- **Unload**: le scritture di relocazione sono tracciate in bitmap così `UnloadProgram` può ri-bindare gli importatori.
- **Cheat** (`gamePatch.cpp`): file JSON stile GoldHEN/ETAHen (`--game-patch`). Verifica TITLE_ID, APP_VER, nome processo; trova il modulo cercando il primo byte pattern; può allocare code-cave; patch pendenti applicate quando un PRX successivo matcha.
- **System content** (`systemContent.cpp`): `param.json` mappato a chiavi stile SFO, `icon0.png`, conteggio chunk PlayGo da `playgo-chunk.dat` / `playgo-chunkdefs.xml` / `chunkmanifest`.

### 7.2 Kernel (`src/kernel/`)

**Memoria** (`memory.cpp`, `memoryAddressSpace.inc`):

| Elemento | Valore |
|---|---|
| Range guest riservati sull'host | sistema `0x40000–0xFFFFFFFFF`; utente `0x1000000000–0xFBFFFFFFFF`; 512 GiB a `0x80000000000` |
| Meccanismo di riserva | Windows `VirtualAlloc2` placeholder; Linux `PROT_NONE MAP_NORESERVE` nei buchi di `/proc/self/maps`; macOS `.zerofill` fissati al link |
| "RAM fisica" | un singolo oggetto condiviso da **13.5 GiB** (memfd / shm / section), `GuestBackingStore` |
| Memoria diretta | offset `[0, totale − flessibile)` del backing |
| Memoria flessibile | coda del backing, **1 GiB** default (override da `param.json`) |
| Mapping | viste fisse del backing (`MapBacking`): l'**aliasing funziona**; l'emulatore scrive tramite un alias scrivibile separato (`TryWriteBacking`) |
| Base default dei mapping | `0x200000000` |
| Altro | pooled memory, aperture PRT; protezione guest tracciata in `VirtualRanges`; il codice programma resta RW/RWX sull'host |
| GPU | bit GPU read/write → protezione host; `MapGpuRange` notifica il `RenderContext` |

**Thread e sync** (`pthread.cpp`, `eventFlag/eventQueue/semaphore/syncOnAddress.cpp`):

- 1 thread guest = 1 pthread host (`PthreadCreate`, `:3234`). Stack guest allocati top-down da `0x7efff8000` e riciclati. `RunOnGuestStack` cambia RSP; `PthreadExit` ripristina i registri host salvati.
- Handle pthread guest = puntatori a oggetti `*Private` host; inizializzatori statici creati lazy al primo uso.
- Mutex = `std::mutex` + condvar, con **polling ogni 10 ms** per consegnare i segnali guest.
- Event flag, semafori, equeue su `Common::Mutex/CondVar`; filtri equeue: graphics, video-out, timer, user, AMPR.
- `SyncOnAddress` e `_umtx_op`: futex su Linux, waiter portabile altrove.
- **TLS**: eseguibile principale → un TCB per host thread (immagine TLS prima del TCB, `tcb[0]` = self), chiave = thread id host (`runtimeLinker.cpp:1579`); PRX → `DTPMOD64 = Program*` + `__tls_get_addr`; liberazione in `CleanupThread`.

**Filesystem** (`fileSystem.cpp`): `MountPoints` con lookup longest-prefix da path guest a directory host. Un mount può essere un `.zar` (ZArchive): path `archive!/member`, lettura read-only via `Common::OpenArchive`. Lookup **case-insensitive su POSIX**. I path non montati **non** cadono mai sul filesystem host.

### 7.3 Librerie HLE (`src/libs/`)

**Pattern a due file**: `foo.cpp` contiene l'implementazione (usa `LIB_NAME` solo per il logging), `libFoo.cpp` contiene `LIB_VERSION` + tabella `LIB_FUNC`. Funzioni "ombrello" raggruppano più librerie: `InitAudio_1` (12 lib), `InitNet_1` (17 lib, incluse NpTrophy2, UDS, Json2), `InitLibKernel_1` (FS, Mem, Equeue, Pthread, Posix, Apr). `Libs::InitAll` (`libs.cpp:108-154`) le chiama tutte.

**Inventario** (T = con test):

| Area | Mature | Parziali / sottili | Stub |
|---|---|---|---|
| Kernel/sistema | libkernel + Posix, Fiber, Coredump, write_throttling (3.4k righe, 392 NID, T); Ampr/Apr (2.5k) | libc/LibcInternal (fallback ~54 NID), SystemService, Rtc, Ult, Json2 (nlohmann), Ces (T), Sysmodule (solo refcount, non carica PRX), UserService (utente singolo da config) | DbgAddressSanitizer, RazorCpu, DeviceService, Usbd, Random, Psml, ConvertKeycode |
| Grafica/video | Graphics5 (AGC) + Graphics5Driver (4.6k + 2.9k di default registri, 162 NID) | VideoOut (codice in `presentation/`), Videodec2 (H.264/HEVC/VP9 via FFmpeg), PngDec/PngEnc (stb, T), Font/FontFt (stb_truetype) | Hmd2 (PSVR2) |
| Audio | AudioOut/AudioOut2 (T), Ngs2 (2.6k, T), AvPlayer (2.1k, T), Ajm | AudioIn, Audiodec (AT9/MP3/AAC) | Audio3d (silente), Acm, Voice, VoiceQoS, AudioPropagation |
| Input | Pad + DualSense (1.2k + 0.9k, T), Mouse, Keyboard | | BluetoothHid |
| Storage/save | SaveData (T), NpTrophy2 + UDS (T) | AppContent (DLC → `DRM_NO_ENTITLEMENT`), TemporaryData, PlayGo | ContentExport/Search/Delete (solo init) |
| Rete | Net, Ssl, Http, Http2, NetCtl (4.4k; URI parsing T) | | Rudp, tutti gli Np*, GameUpdate, Remoteplay, GameLiveStreaming, SharePlay, Share (offline/signed-out) |
| UI | ImeDialog/Ime (overlay, T), SaveDataDialog (T) | MsgDialog (sempre OK), ErrorDialog, SigninDialog | Login, WebBrowser, TextToSpeech2 |

**AGC** (`agc.cpp`, `libAgcDriver.cpp`, `agcRegisterDefaults.inc`): l'HLE **non traduce mai API in Vulkan**; codifica PM4 nel buffer del gioco (vedi §6.3). In più: `AgcInit` mappa la memoria driver a `0xfe0000000` e pianta uno shader validatore GPU; i default dei registri per versione SDK vengono da `agcRegisterDefaults.inc`; gestisce creazione shader, fusione shader e registri di stato primitive. Esistono helper per patchare pacchetti in place.

**Audio**: solo AudioOut/AudioOut2/AudioIn parlano con SDL3 (uno stream per porta, S16/F32, max 8 canali; output bloccante emulato con priming + pacing a sleep). Le altre librerie producono PCM che il gioco poi sottomette ad AudioOut. Porte PADSPK e VIBRATION → haptics DualSense. Ngs2: grafo rack/voice, sampler PCM/ATRAC9 via LibAtrac9, biquad. AJM: factory di decoder (FFmpeg per MP3/AAC/Opus, LibAtrac9 per AT9), **decodifica sincrona** alla registrazione del job, `BatchStart/Wait` no-op, encode ritorna zeri. AvPlayer: demux FFmpeg con reader custom sul filesystem guest, worker come pthread guest, video → NV12 (swscale), audio → S16 (swresample), pull dal gioco.

**Input**: eventi SDL3 letti in `presentation/window/window.cpp` e `hostInput.cpp` → `Controller::SetButton/SetAxis/SetTouchPad/SetSensor`. Giroscopio/accelerometro via sensori SDL, light bar via `SDL_SetGamepadLED`, rumble via `SDL_RumbleGamepad`. **Trigger adattivi** supportati (`trigger_effect_to_dualsense`, `SDL_SendGamepadEffect` solo su pad PS5). **Haptics USB**: trova il device audio USB a 4 canali del DualSense (speaker davanti, attuatori dietro). **Haptics Bluetooth**: HID raw via `SDL_hid`, report 0x39 con blocchi haptic a 3 kHz, speaker codificato Opus 48 kHz stereo, CRC32 per report, thread sender dedicato; protocollo accreditato a DS5Dongle.

**Storage**: savedata in `_SaveData/<TITLE_ID>/<dir>`, mount in slot, **scritture atomiche** (temp + rename), SaveDataMemory2 in `memory.dat`. Trofei: `NpUniversalDataSystemPostEvent` intercetta `_UnlockTrophy` e regole UDS basate su statistiche, persistenza in `common/trophies`, toast nell'overlay + suono `assets/sounds/trophy-unlock.wav`.

**Rete**: socket BSD/Winsock reali con traduzione errori, P2P → UDP, DNS via `getaddrinfo`; ma NetCtl è **forzato offline**, `HttpSendRequest` ritorna timeout, NP ritorna SIGNED_OUT.

**libc/printf/errno**: `guestPrintf.cpp` è un port del printf di mpaland; `vaContext.h` ricostruisce l'area di salvataggio registri SysV da `rdi–r9` e `xmm0–7`. `errno` è un `thread_local` esportato come `__error`; `POSIX_CALL` converte errori SCE in errno.

### 7.4 GPU lato guest (`src/graphics/guest_gpu/`)

- `pm4.h/.cpp`: definizioni pacchetti e pseudo-pacchetti NOP.
- `command_processor/pm4Dispatch.cpp`: tabelle per opcode e per registro (context, SH, UCONFIG). `pm4Handlers.cpp`: handler (es. `CpOpSetContextReg` `:2372`, indirect buffer su stack di cursori `:1916-1959`).
- `graphicsRun.cpp`: il front-end GPU. Draw/dispatch (`DrawIndex` → `RenderExecutor::DrawIndex`), espansione indirect/multi-draw, EOP/EOS, eventi e barriere, DMA, flip. **Occlusion query falsate come "sempre visibile"** (`:1325-1348`).
- `tile.cpp`: solo **matematica di layout su CPU** (offset mip, padding, mip tail). Modalità: Linear, Standard256B, Standard4KB, Standard64KB, PRT, Depth, RenderTarget → 9 famiglie di blocchi incluse le varianti 3D.
- `gpu_format.cpp`: formati PS5 ↔ Vulkan.

### 7.5 GPU lato host (`src/graphics/host_gpu/`)

- **Memoria e coerenza**: `pageManager`, `memoryTracker`, `regionManager`, `rangeSet` (vedi §6.4). `hostMemory.*` **non** fa parte della coerenza: risponde solo a "questo indirizzo è leggibile?" per l'HLE di libc.
- **Tiling su GPU**: lo swizzling degli indirizzi gira in compute shader (template comune `gpu_tiler_common.inc` + un `block_offset()` per famiglia). Pipeline create lazy, specializzate per byte/elemento e direzione (tile/detile) via specialization constant (`renderer/image/tiler.cpp`). Passi extra: promote/demote D16 (16-bit depth ↔ D24/D32 host), `color_transform` (swap canali BGRA16, packing 10_11_11). Motivazione: i dati guest sono già in un buffer Vulkan, detilare lì evita swizzling CPU di superfici grandi e resta sulla stessa coda.
- **Pipeline cache** (`pipeline/pipelineCache.*`): chiave = formati attachment + id programmi shader + vertex input + `PipelineStaticParameters` da 116 byte (cull, blend, topologia, MSAA). Quasi tutto il resto è **dynamic state** (viewport/scissor con count, depth/stencil, bias, color-write enable, feedback loop). **Dynamic rendering** ovunque, niente render pass. Su disco solo il `VkPipelineCache` del driver (`_PipelineCache/<title>.bin`); nessuna cache SPIR-V/IR.
- **Descriptor** (`pipeline/descriptors.cpp`): push descriptor quando ci stanno, altrimenti set da pool riciclati per tick della timeline (`descriptorHeap.h`). User data in push constant. **Niente bindless**: solo non-uniform indexing di sampled image; l'accesso "bindless-like" passa da BDA.
- **Texture cache** (`cache/textureCache.*`): slot vector + page table multilivello, risoluzione overlap e mip/layer, GC LRU, binding VideoOut per scanout.
- **Sync** (`renderer/sync.cpp`, `commandScheduler.cpp`, `masterSemaphore.cpp`): una sola `VkQueue` dietro `queue_mutex`, una timeline semaphore "master tick", un thread "priority operations" per scheduler che aspetta i tick e pubblica interrupt, flip completati e readback.

### 7.6 Presentazione (`src/graphics/presentation/`)

- `window/*`: finestra SDL3 + Vulkan, loop eventi sul main thread, FPS solo nel titolo della finestra.
- `window/swapchain.cpp`: `present_scheduler` proprio, present mode da config (FIFO default lato swapchain, Mailbox/Immediate opzionali), ricreazione su out-of-date/suboptimal.
- `videoOut.cpp`: thread di presentazione, vblank virtuale, flip queue, eventi `EVFILT_VIDEO_OUT`.
- `systemOverlay.cpp`: ImGui con backend Vulkan e dynamic rendering. **È la UI di sistema PS5, non un HUD di debug**: tastiera IME, dialoghi di sistema, toast trofei.
- `renderDoc.cpp`: carica la libreria via `dlopen`/registro, cattura su **F1** di 2 flip guest sotto i lock di render e coda, output in `_RenderDoc/`; nomi di debug sugli oggetti Vulkan.

### 7.7 Ricompilatore shader (`src/graphics/shader/`)

**Entry point** (`recompiler/ShaderRecompiler.h:63-74`): `PrepareShaderSource` → `TranslateProgram` (decode, CFG, IR, passi, tracking risorse) → `CompileProgram` (specializzazione, info, emit SPIR-V). `RefreshShaderSource` solo per programmi SWAPPC. Il renderer li chiama da `pipelineCache.cpp:320-457`.

**Pipeline di compilazione**:

| Stadio | Dove | Note |
|---|---|---|
| Decode | `frontend/decode/` | Famiglia dai bit alti della word; tabelle opcode `consteval MakeOpcodeTable`; output `Decoder::Instruction` con operandi espliciti e campi SDWA/DPP/NSA/MIMG. Gli stage fusi (LS+HS, ES+GS) diventano un programma unico riscrivendo l'`s_setpc` di hand-off in `s_branch` |
| CFG | `frontend/cfg/ShaderCFG.cpp` | `BuildGraph` trova label e terminatori, risolve jump table `S_GETPC`+`S_SETPC` in `IndirectBranch`, marca irriducibili. `Structurize` = goto-elimination modellata sul GotoPass di shadPS4. Fallback **"dispatcher"**: loop + `OpSwitch` sugli id di blocco con SSA spillato in variabili |
| Traduzione | `frontend/translate/` | Blocco di entry sintetico; seed di user SGPR, EXEC, registri ABI per stage; registri/SCC/EXEC/VCC/M0 nascono come Get/Set e `RewriteToSsa` li rende SSA vera |
| IR | `ir/` | SSA con use list e Phi; ogni nodo tiene **un id SPIR-V per metà wave**; blocchi = `std::list<Inst>` + terminatore strutturato (merge/continue). ~330 opcode tipizzati in X-macro (`ir/opcodes/ValueOpcodes.inc`). `Program : ResourcePlan` |
| Passi | `ir/passes/` | ordine: SSA rewrite → const prop → cleanup CF/identità → DCE → **ReadLaneElimination** (poi ripete 2–4) → `LowerTessellationMemory` → **TrackResources** |
| Backend | `backend/spirv/` | builder SPIR-V **proprio** (non sirit/glslang); `EmitProgram` |

**Stage supportati**: CS, VS, PS, Mesh, LS, HS, TES. I **GS NGG** diventano Vulkan **mesh shader**. Tessellazione: tripla LS/HS/TES preparata in `shader.cpp:795-848`, memoria abbassata in `Tessellation.cpp`. "**Ray tracing**" = emulazione software dell'istruzione RDNA 2 `IMAGE_BVH_INTERSECT_RAY` in SPIR-V che legge memoria guest (`spirvEmitterBvh.cpp`); nessuno stage RT Vulkan. PS5-specifici: `shaderVertexMetadata.cpp` valida l'header user-data AGC che localizza vertex buffer e attribute table; `shaderPixelParameter.cpp` mappa gli interpolanti PS; `rectListShader.cpp` genera TCS/TES che espandono RECTLIST in quad; l'alpha logico dei PS export passa da MRT1.

**Resource binding e provenienza scalare** (il tema dei commit recenti):

- Su PS5 lo shader legge i descrittori dalla memoria con load scalari (**SRT**, shader resource table). In IR ogni accesso costruisce `Get{Buffer,Image,Sampler}Resource` da 4/8/4 word SGPR; `S_LOAD` diventa `ReadConstBuffer`/`LoadAddressU32` per dword. Ogni descrittore è quindi un **albero SSA radicato nei user data + letture scalari concatenate**: la "provenienza scalare".
- `ReadLaneElimination`: forwarda `v_writelane` → `v_readlane` a lane costante attraverso le Phi, annullando gli spill di SGPR in lane VGPR così la provenienza sopravvive.
- `ResourceTracking` (`Tracker::Run`): `PlanScalarReads` fonde letture equivalenti in slot di una **SRT appiattita** che l'host riempie; `Collect` assegna indici densi a buffer/image/sampler; i buffer scelti dalla GPU a runtime ripiegano su **BDA**.
- `SrtWalker`: **interprete host** del grafo di valori mantenuto; `EvaluateRawRead` legge memoria guest; `RefreshFlatBuffer` cammina il CFG rilevante usando predicati uniformi; `SrtReadCapture` registra i range letti per l'invalidazione.
- `ResourceMaterialization`: `ExtractResourcePlan` copia solo i grafi di descrittori/SRT/predicati; `MaterializeResources` **per draw** produce `ResourceSnapshot` (dword dei descrittori + SRT piatta: dati, **mai** ricompilazione) e `ResourceSpecialization` (formati, stride, dimensione, mip, cube, FMASK: **solo questa** crea nuove permutazioni SPIR-V).
- `ShaderInfoCollection`: input/output, user-data usati, feature flag richiesti (ballot, shuffle, BVH, gather…).
- Commit recenti in chiaro: **SWAPPC** = chiamate `s_swappc_b64` via puntatore in SGPR; a draw time l'indirizzo è valutato con SrtWalker, il callee (≤64 word, solo scalare, foglia) è letto dalla memoria guest, inlinato e ri-linkato se cambia; solo compute; permutazioni cache keyed sul codice del callee. **Finite descriptor planning** = descrittori scelti via Phi (if/else che sceglie texture) enumerati in un insieme finito; la Phi diventa Phi di ordinali e l'host binda tutti i candidati; i sampler inline letterali seguono lo stesso percorso. **GPU-selected tables** = quando la chiave la calcola la GPU (bit-scan, loop limitati, workgroup id, material key) l'host enumera la tabella e scrive una mappa ordinata chiave→slot nella SRT piatta; lo shader fa binary search. **Memory-dependent predicates** = solo le condizioni di branch dipendenti da memoria guest vengono catturate per l'invalidazione cache.

**Interfaccia col renderer**: `ShaderParams` (codice, 40 word user-data incluso il prefisso s0:s7 degli stage fusi, hash, back half); chiavi statiche da `BuildStageStaticKey`. Layout: un descriptor set con binding kind fissi, offset per stage 56; array di buffer; 48 binding immagine divisi per sampled/comparison/storage, classe numerica e dimensione; sampler; GDS, page table BDA, fault buffer, SRT piatta, ShaderData, LDS. Push constant da 128 byte (user-data usati, extent di dispatch, offset buffer) con fallback a storage buffer. **Nessuna specialization constant Vulkan**: specializzare = ricompilare. Cache: `ProgramKey` (stage, hash XXH3, user-data count, code size, static key) → `SourceEntry` (plan, snapshot, permutazioni); permutazione matchata su specialization, push-data start e codice callee linkato; validazione SPIRV-Tools opzionale.

**Scelte e limiti noti**:
- Divergenza **modellata, non ricostruita**: EXEC è un booleano SSA per lane; le op di memoria hanno un operando guard; i branch scalari diventano any/all uniformi su ballot (si assume subgroup convergente).
- **Wave64 su subgroup a 32** emulato eseguendo due lane per invocazione: ogni istruzione è emessa due volte.
- FP: flush denormali FP32 disabilitato per performance, preservazione denormali FP64 disabilitata, contratto signed-zero/Inf/NaN dichiarato; workaround per moltiplicazioni 64-bit (fallimento di compilazione NVIDIA); compare F64 VOPC aggiunte solo nel commit analizzato.
- LDS in workgroup/storage/function memory; GDS limitato a partizione da 48 KiB; `DS_ORDERED_COUNT` approssimato.
- Nessun binding storage image sint; FMASK mappato su MSAA nativo; gather con LOD esplicito limitato.
- Fail-fast (`EXIT_NOT_IMPLEMENTED`) sui casi non supportati.

### 7.8 Common e launcher

- `common/`: `Subsystems` (lifecycle), `Singleton`, `Thread/Mutex/CondVar`, `VirtualMemory` (Alloc/Reserve/Commit/Protect/FlushInstructionCache), `HostException` (installazione handler, `ExceptionInfo` con registri e contesto nativo; signal stack per thread su Linux), `File` e `archive` (ZArchive), `emulatorConfig`, `profiler` (Tracy), `trophies`, `lruCache`, `slotVector`, `bitArray`, `dateTime`, `stringUtils`, `systemInfo` (cpuinfo).
- `launcher/`: `MainDialog`, lista giochi (scansione ricorsiva di cartelle con `eboot.bin` e `.zar`), `Configuration*` (profili per gioco e globali), `InputMappingDialog` + `ControllerPreview` + `DualsenseWidget`, `TrophyViewerDialog`, `PatchesDialog` + `CheatRepository`, `CompatibilityDatabase`, `UpdateChecker`, `LauncherTheme`.

---

## 8. Modello di threading complessivo

| Thread | Chi lo crea | Cosa fa |
|---|---|---|
| Main (host) | processo | Init, poi loop eventi SDL (`WindowRun`) |
| Thread guest principale | `Emulator::Execute` | `RuntimeLinker::Execute` → entry point del gioco |
| Thread guest N | `PthreadCreate` HLE | 1:1 con pthread host, stack in memoria guest |
| `Thread_Gpu` | `GuestGpu` ctor | Parsing PM4 di 57 code, registrazione comandi Vulkan, drenaggio richieste di readback |
| Priority operations (per `CommandScheduler`) | renderer | Attende tick timeline, pubblica interrupt/flip/readback |
| Present thread (VideoOut) | `videoOut.cpp` | Vblank virtuale, presentazione |
| Sender Bluetooth DualSense | `dualSenseBluetooth.cpp` | Invio report HID |
| Worker AvPlayer | `avPlayer.cpp` | Demux/decode, creati come pthread guest |

Sincronizzazione chiave: i thread guest sottomettono senza bloccare; `AgcSuspendPoint` blocca su un semaforo binario, così c'è circa **un frame di comandi in volo**. Una sola `VkQueue` (async compute serializzato). `RenderContext::m_mutex` attorno a draw e present. Gli eventi GPU arrivano al guest tramite le equeue del kernel.

---

## 9. Test, CI e tooling di sviluppo

- **Test unitari mirati** sulle parti più rischiose: `MemoryTrackerTests`, `PageManagerTests`, `ImagePageTableTests`, `VirtualMemoryAllocationTests` (+ red zone patcher), `SyncOnAddressTests`, `EventQueueLifetimeTests`, `KernelFileSystemTests`, `ArchiveFileTests`, `ResourceTrackingTests`, `ResourceMaterializationTests`, `ScalarProvenanceTests` (IR costruita a mano + SrtWalker su memoria finta), `ShaderVertexMetadataTests`, audio/pad/ime/savedata/trofei/png/ces/http.
- **Test shader**: `shaderCfgTests.cpp` (~210 casi) alimenta word RDNA 2 codificate a mano nell'intera pipeline, valida con SPIRV-Tools, controlla i parent delle Phi, impone budget di dimensione SPIR-V e asserisce che **non** venga emesso il dispatcher/`OpSwitch`. `ShaderRecompilerComputeTests.cpp` costruisce casi con helper `Encode*`, buffer iniziale e risultato atteso, e li **esegue su un device Vulkan reale** alla subgroup size del device (così si esercita wave64-su-32); include i test GPU BVH/FLAT. Sotto-test via flag: alignbyte, polygon-mode, draw-offset, scheduler, stream-buffer, gpu-command-lane, context-state, gpu-tiler, layered-image, host-image-allocation, image-view-cache, storage-sampled, depth-readback, compute-meta-clear, buffer-cache-gc, rt1d, image-overlap, htile-clear, buffer-cache-range.
- `add_kyty_full_emulator_test`: linka l'intero emulatore nel test (shader_cfg, http_uri_parse, kernel_file_system, shader_recompiler_compute, virtual_memory_allocation).
- **Tooling**: Tracy (`--profile`), RenderDoc (`--rd`, F1), dump command buffer (`--command-buffer-dump`), dump texture (`_Textures/`), log shader (`--shader-log-direction`), SPIR-V debug printf, validation Vulkan e GPU-assisted, printf guest su file, clang-tidy opzionale (`KYTY_ENABLE_CLANG_TIDY`), IWYU, sccache/ccache auto-rilevati, dev shell Nix, setup VS Code (Windows).

Comandi di riferimento (Linux, senza Qt):

```bash
git submodule update --init --recursive
cmake -S . -B _Build/linux-no-qt -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DKYTY_BUILD_LAUNCHER=OFF
cmake --build _Build/linux-no-qt --target kyty_emulator kyty_tests --parallel
ctest --test-dir _Build/linux-no-qt --output-on-failure
./_Build/linux-no-qt/kyty_emulator --game /path/al/gioco
```

---

## 10. Decisioni di design (in stile ADR compatto)

| # | Decisione | Perché | Conseguenze |
|---|---|---|---|
| D1 | Esecuzione nativa + HLE, niente CPU emulata | PS5 è x86-64; shadPS4 ha mostrato che funziona | Velocità nativa; dipendenza dall'ISA host (Zen 2 vs Intel vs Rosetta); linking e ABI diventano il problema centrale |
| D2 | Funzioni HLE con ABI SysV anche su Windows | Il guest è FreeBSD/SysV | Nessun trampolino; attenzione a TEB, red zone, FS segment |
| D3 | Linking lassista (thunk per import mancanti, fallback per solo NID) | Far bootare più giochi possibile | Errori silenziosi fino alla chiamata; fallback a scansione lineare |
| D4 | Backing store unico con viste fisse | Riprodurre memoria fisica e aliasing PS5 | 13.5 GiB riservati; rollback complessi in `memory.cpp` |
| D5 | Coerenza via protezione pagine, dirty CPU/GPU mutuamente esclusivi | Memoria unificata senza copie | Fault handler nel percorso caldo; invarianti rigide (`EXIT`) |
| D6 | AGC resta a livello PM4 | Un solo parser per codice AGC compilato nel gioco e helper HLE | Zero copy; i buffer guest devono vivere fino al consumo; pseudo-pacchetti in NOP |
| D7 | Un thread GPU, una VkQueue, 57 code cooperative | Semplicità, determinismo | Niente async compute reale; polling 100 ms se tutto bloccato |
| D8 | Vulkan 1.3 "moderno": dynamic rendering, dynamic state, push descriptor, mesh shader, timeline | Chiavi pipeline piccole, meno oggetti | Richiede driver recenti; MoltenVK come limite su macOS |
| D9 | Detiling su GPU con compute specializzati | Superfici grandi, dati già in VRAM | Pipeline extra; D16 promosso/demotato |
| D10 | IR SSA propria + builder SPIR-V proprio | Controllo totale su divergenza e wave64 | Nessuna dipendenza da sirit/glslang; più codice da mantenere |
| D11 | Risorse: piano a compile time + interprete host per draw; ricompila solo su specializzazione | Descrittori letti da memoria (SRT) non bindabili staticamente | `ResourceTracking.cpp` da 2.5k righe di prove pattern-specifiche; SrtWalker nel percorso per-draw |
| D12 | Fail-fast (`EXIT_NOT_IMPLEMENTED`) | Scoprire subito i casi non gestiti | Crash invece di glitch; ottimo in sviluppo, duro in uso |

---

## 11. Rischi e debito tecnico osservati

1. **Fail-fast ovunque**: qualsiasi pacchetto PM4, stato o encoding shader sconosciuto termina il processo.
2. **Un solo thread GPU e una sola coda**: niente async compute; caso "tutte le code bloccate" a polling.
3. **Pipeline compilate in modo sincrono sul thread GPU**; solo la cache driver su disco → stutter al primo incontro di ogni shader/permutazione.
4. **Label EOP scritte al parse** (`graphicsRun.cpp:1085-1088`): corrette solo grazie alle pagine NoAccess; dipendenza implicita fragile.
5. **Bitmap fault BDA aggiornata con read-modify-write non atomico**: fault persi sotto contesa; prima lettura di una pagina non cachata ritorna 0 fino al submit successivo.
6. **Registrazione librerie dispersa e duplicata**: `Share` è registrata sia in `libShare.cpp:13` che in `libNet.cpp:2694`, `WebBrowserDialog` idem; per i duplicati il lookup esatto prende l'**ultima** registrazione (`insert_or_assign`), il fallback NID la **prima**. Trophy e Json2 registrati in `libNet.cpp`; VideoOut vive in `graphics/`.
7. **Logging quirk**: `PRINT_NAME_ENABLED` è `thread_local` **per file** (`libs.h:22`) e viene attivato dal file di registrazione sul thread di init, quindi non accende mai il log dei nomi nei file di implementazione come `agc.cpp`.
8. **Stub silenziosi**: Audio3d e ACM silenti, AJM sincrono con encode a zeri, MsgDialog sempre OK, occlusion query sempre "visibile", rete sempre offline nonostante socket reali, DLC senza entitlement.
9. **Footprint memoria**: backing da 13.5 GiB (committato in anticipo su Windows), page table GPU ~768 MiB, bitmap fault 12 MiB.
10. **Mutex guest con polling a 10 ms** per la consegna dei segnali.
11. **Percorsi `EXIT_NOT_IMPLEMENTED` noti**: SELF compressi, flag di `LoadStartModule`, eseguibili PS4, pacchetti/registri sconosciuti.
12. **Build esigente**: Clang obbligatorio, Qt 6, glslang, Python, fetch di rete a configure-time (nessun `nix build` sandboxed senza vendoring).
13. **Test selettivi in CI**: su Windows girano 12 test su ~50; i test shader su GPU reale non girano in CI.

---

## 12. Guida alla lettura

Ordine consigliato per entrare nel codice:

1. `src/emulator.cpp` — boot, mount, ordine dei sottosistemi.
2. `src/loader/runtimeLinker.cpp` — `LoadProgramToMemory`, `Resolve`, `PatchProgram`, `Execute`, `RunEntry`.
3. `src/libs/libs.h` + `src/libs/libPad.cpp` + `src/libs/controller.cpp` — il pattern HLE su un esempio piccolo e concreto.
4. `src/kernel/memoryAddressSpace.inc` + `src/kernel/memory.cpp` — layout guest e backing store.
5. `src/libs/agc.cpp` (encoder PM4) → `src/graphics/guest_gpu/graphicsRun.cpp` (thread GPU) → `src/graphics/host_gpu/renderer/renderDraw.cpp` (draw).
6. `src/graphics/host_gpu/regionManager.h` + `pageManager.cpp` + `renderer/cache/bufferCache.cpp` — coerenza.
7. `src/graphics/shader/recompiler/ShaderRecompiler.cpp` → `frontend/cfg/ShaderCFG.cpp` → `ir/passes/ResourceTracking.cpp` → `backend/spirv/SpirvEmitter.cpp`.
8. `tests/shaderCfgTests.cpp` e `tests/ScalarProvenanceTests.cpp` — per vedere la pipeline shader "dal basso".

Riferimenti esterni indispensabili: AMD **RDNA 2 ISA Reference Guide** (doc 70648) per decode/semantica, specifiche **Vulkan 1.3** e **SPIR-V**, codice **shadPS4** per i pattern di memoria e il GotoPass.

### Glossario

| Termine | Significato |
|---|---|
| **HLE** | High-Level Emulation: reimplementare le API di sistema invece dell'hardware |
| **NID** | Identificatore hash di un simbolo esportato da una libreria Sony |
| **PRX** | Modulo dinamico Sony (shared library), es. `libc.prx` |
| **SELF / ELF** | Formato eseguibile Sony firmato / ELF standard; l'emulatore richiede SELF decriptati |
| **AGC** | API grafica PS5 (successore di GNM/PS4), compilata nel gioco e in parte HLE |
| **PM4** | Formato dei pacchetti del command processor AMD |
| **DCB / ACB** | Draw Command Buffer / Async Compute Buffer |
| **RDNA 2** | Architettura GPU AMD della PS5; ISA con SGPR (scalari), VGPR (vettoriali), EXEC mask, wave64 |
| **SGPR / VGPR / EXEC** | Registri scalari per wave, registri vettoriali per lane, maschera delle lane attive |
| **SRT** | Shader Resource Table: descrittori letti dalla memoria dallo shader stesso via load scalari |
| **V# / T# / S#** | Descrittori AMD di buffer / texture / sampler |
| **BDA** | Vulkan Buffer Device Address: puntatori raw lato shader |
| **EOP / release_mem** | Eventi di fine pipe con scrittura di label in memoria e/o interrupt |
| **Memoria diretta / flessibile / pooled** | Classi di memoria del kernel PS5: fisica mappata esplicitamente, heap gestito dal kernel, pool riservato |
| **Equeue / EVFILT** | Event queue stile kqueue di FreeBSD e i suoi filtri |
| **PlayGo** | Sistema di chunk per installazione progressiva |
| **ZArchive (.zar)** | Formato archivio (da Cemu) usato per i dump di gioco, montato in sola lettura |
| **Rosetta 2** | Traduttore x86-64 → ARM di Apple; su macOS gioco ed emulatore girano entrambi tradotti |

---

## Appendice — Come sono state ottenute le misure

```bash
# struttura e dimensioni
find src -name '*.cpp' | wc -l ; find src -name '*.h' | wc -l
find src -name '*.cpp' -o -name '*.h' | xargs wc -l | tail -1
for d in src/kernel src/loader src/libs src/graphics/guest_gpu src/graphics/host_gpu \
         src/graphics/presentation src/graphics/shader src/common src/launcher; do
  printf "%-35s %s\n" "$d" "$(find $d -name '*.cpp' -o -name '*.h' -o -name '*.inc' | xargs cat | wc -l)"
done
find src -name '*.cpp' -o -name '*.h' | xargs wc -l | sort -rn | head -40

# storia (clone shallow!)
git rev-parse --is-shallow-repository
git rev-list --count HEAD
git log --format='%s' | sed -n 's/^\([a-z0-9_]*\):.*/\1/p' | sort | uniq -c | sort -rn

# build e test
grep -c "add_test(" CMakeLists.txt
sed -n '/add_custom_target(kyty_tests/,/)/p' CMakeLists.txt
```

Le affermazioni sui sottosistemi derivano dalla lettura del codice al commit indicato; quelle marcate con `file:riga` sono state verificate a campione (duplicati di registrazione, flag `thread_local`, scrittura label al parse, thread GPU unico).
