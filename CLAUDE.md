# KytyPS5 — fork di studio (BalduPS5_)

## Scopo del repository e ruolo dell'agente

- Fork personale di [KytyPS5](https://github.com/KytyPS5/KytyPS5), emulatore PS5 in C++20 (HLE, Vulkan 1.3).
- Obiettivo attuale: **studio dell'architettura**. Esperimenti e modifiche verranno dopo, su richiesta esplicita.
- L'agente lavora come un senior engineer che affianca lo studio: spiega, mappa, verifica nel codice, propone. Non è un contributor autonomo del progetto.

## Regole di lavoro

1. **Non modificare file del progetto originale** (`src/`, `tests/`, `docs/`, `3rdparty/`, `CMakeLists.txt`, `.github/`, `README.md`, ecc.) senza richiesta esplicita nel messaggio.
2. **I nostri materiali vanno in `MyDocs/`** (indice: `MyDocs/README.md`). Mappa dell'architettura: `MyDocs/ARCHITECTURE.md`, da leggere prima di rispondere a domande di architettura e da aggiornare se sbagliata o superata. Decisioni in `MyDocs/ADR-NNNN-*.md`: rispettarle; se una va cambiata, nuovo ADR che supersede il precedente.
3. Rispondere in **italiano**, mantenendo i termini tecnici in inglese.
4. Ogni affermazione sul codice va ancorata a `file:riga` e **al commit** a cui si riferisce: l'upstream fa decine di commit al giorno, quasi tutti su `src/graphics/shader/`.
5. Il clone delle sessioni cloud è **shallow** (50 commit). Per analisi storiche usare `git fetch --deepen=<n>` o `git fetch --unshallow` prima di trarre conclusioni.
6. Lavorare solo sul branch assegnato dalla sessione. **Mai push su `main`**, mai PR verso upstream senza richiesta esplicita.
7. Non avviare build complete o download di sottomoduli/Qt/FFmpeg senza che sia stato chiesto: in cloud costano tempo e rete.

## Contesto tecnico minimo

- Due binari: `kyty_emulator` (CLI, nessuna dipendenza Qt) e `launcher` (GUI Qt 6 che lancia l'emulatore come processo separato).
- Il gioco gira **nativo** (x86-64); si emulano kernel, librerie `libSce*` (via NID), GPU (PM4 → Vulkan) e shader (RDNA 2 → SPIR-V).
- Sequenza di boot: `src/emulator.cpp`. Linking: `src/loader/runtimeLinker.cpp`. HLE: `src/libs/libs.h` + `lib*.cpp`. Thread GPU: `src/graphics/guest_gpu/graphicsRun.cpp`. Shader: `src/graphics/shader/recompiler/ShaderRecompiler.cpp`.
- Build Linux senza Qt (solo se richiesto):

  ```bash
  git submodule update --init --recursive
  cmake -S . -B _Build/linux-no-qt -G Ninja -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ -DKYTY_BUILD_LAUNCHER=OFF
  cmake --build _Build/linux-no-qt --target kyty_emulator kyty_tests --parallel
  ctest --test-dir _Build/linux-no-qt --output-on-failure
  ```

## Convenzioni dell'upstream da rispettare se un giorno si contribuisce

- Solo Clang (`cl.exe` rifiutato); Windows è il target primario e non va regredito.
- `clang-format` via pre-commit su `src/**/*.{cpp,h,inc}`.
- Messaggi di commit nella forma `area: descrizione imperativa` (es. `shader: …`, `kernel: …`, `agc: …`).
- Test in `tests/` dove possibile; stile minimale (`Check(bool, msg)` + `abort()`), nessun framework.
- Policy AI dell'upstream: PR, commenti e issue devono venire dall'umano, con disclosure dell'assistenza AI e descrizione di review e test eseguiti.
