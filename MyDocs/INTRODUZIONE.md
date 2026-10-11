# KytyPS5 spiegato da zero — e cosa possiamo farne

> Versione in linguaggio semplice, pensata per chi inizia. La versione tecnica completa è in [ARCHITECTURE.md](ARCHITECTURE.md).

## Cosa fa, in parole semplici

Un gioco PS5 è un programma scritto per vivere in una casa precisa: un computer con processore AMD, scheda video AMD e un sistema operativo Sony. KytyPS5 è un programma che prende quel gioco e lo fa vivere nel tuo PC **fingendo di essere quella casa**.

Dentro KytyPS5 non c'è una riga di codice Sony. È tutto scritto da zero, osservando come si comportano i giochi e cosa chiedono al sistema. Il gioco lo porti tu, come copia decriptata di un titolo che possiedi. L'emulatore non sa decriptare nulla e non scarica nulla.

C'è un dettaglio che rende la PS5 un caso speciale. La sua CPU è un normale x86-64, lo stesso tipo di processore del tuo PC. Quindi il codice del gioco **non viene tradotto**: viene eseguito direttamente dal tuo processore, a velocità piena. Nessuna "macchina virtuale" della CPU. Quello che va emulato è tutto il resto: il sistema operativo, la scheda video e i programmini che girano sulla scheda video.

## Come lo fa: la storia di un avvio

Quando lanci un gioco succede questo, nell'ordine.

1. **Il caricamento.** Il file `eboot.bin` è l'equivalente di un `.exe`. L'emulatore lo apre e lo mette in memoria. Dentro c'è una lista della spesa: "mi servono queste funzioni del sistema", tipo apri file, crea thread, leggi il controller, disegna. Ogni voce ha un codice numerico, il **NID**. L'emulatore ha una rubrica che associa ogni codice a una propria funzione C++ e scrive i propri "numeri di telefono" nella lista del gioco. Da quel momento, ogni volta che il gioco crede di chiamare Sony, sta chiamando l'emulatore. Questo è il cuore dell'approccio chiamato **HLE**, High-Level Emulation.

2. **Il sistema operativo finto.** Quando il gioco chiede memoria, thread o file, l'emulatore risponde con memoria del PC, thread del PC e cartelle del PC. La cartella del gioco diventa `/app0`, i salvataggi finiscono in `_SaveData`, e così via. Il gioco non se ne accorge.

3. **La scheda video.** Il gioco non parla con la GPU a voce: scrive i comandi di disegno in un quaderno condiviso, in un linguaggio chiamato **PM4** che è quello delle GPU AMD. Un thread dedicato dell'emulatore legge il quaderno e riscrive ogni comando in **Vulkan**, il linguaggio grafico che la tua scheda video capisce.

4. **I programmi della GPU.** Gli **shader** sono piccoli programmi che girano sulla scheda video, scritti nel codice macchina della GPU PS5 (architettura RDNA 2). L'emulatore contiene un mini compilatore che, la prima volta che incontra uno shader, lo traduce in **SPIR-V**, il formato di Vulkan. È la parte più difficile di tutto il progetto ed è dove l'autore lavora quasi ogni giorno.

Attorno a questo ci sono audio e controller, che passano per la libreria SDL, con il DualSense supportato fino a trigger adattivi e vibrazione fine. Il `launcher` in Qt è solo il menu: avvia l'emulatore come programma separato.

Perché i giochi crashano spesso? Perché l'emulatore è severo per scelta: se un gioco chiede qualcosa che non conosce, si ferma subito invece di tirare a indovinare. La compatibilità cresce funzione dopo funzione.

## Possiamo farne "la nostra console"?

Dipende da cosa si intende.

**Sì, se "console" significa un'esperienza da console costruita attorno a questo motore.** Un mini PC x86-64 con una scheda video Vulkan, Linux o Windows, che si accende direttamente su una nostra interfaccia navigabile col controller, con il nostro nome, il nostro tema, la libreria giochi, i profili per gioco, i trofei, le impostazioni. Il motore di emulazione resta quello upstream, aggiornato spesso. La nostra parte è il guscio. È la strada con il miglior rapporto fatica/risultato ed è adatta a un junior: il launcher attuale è un programma Qt separato di circa seimila righe e il contratto con l'emulatore è una riga di comando. Lo stesso guscio può lanciare anche giochi nativi per PC, inclusi quelli che faremo noi.

**Sì, con più fatica, se significa aggiungere funzioni all'emulatore.** Overlay personalizzato, screenshot, sistema di cheat e patch, mappature controller, impostazioni di sistema. Sono modifiche nei layer librerie e presentazione, dove il codice è più leggibile. Si fanno dopo aver imparato a costruire e lanciare il progetto.

**No, se significa una piattaforma per i nostri giochi.** L'emulatore esegue solo eseguibili PS5, e per produrli serve l'SDK Sony, proprietario e sotto NDA. I nostri giochi li facciamo con un motore normale e li distribuiamo per PC. La console può ospitarli, ma non attraverso l'emulatore.

**No, su hardware ARM.** Niente Raspberry Pi o simili: tutto il progetto presuppone un processore x86-64 nell'host, perché il gioco gira nativo. Su Apple Silicon funziona solo grazie a Rosetta.

## Tre vincoli da tenere a mente

1. **Licenza GPL-2.0.** Se distribuiamo binari del nostro fork dobbiamo pubblicare anche il sorgente con la stessa licenza; per uso personale nessun obbligo. Un launcher che avvia l'emulatore come processo separato resta un programma a sé.
2. **Marchi.** Niente "PlayStation", "PS5" o loghi Sony nel nome e nella grafica del prodotto, come fa il progetto stesso.
3. **Contenuti.** Solo copie di giochi che possediamo, mai distribuire dump o firmware. In Europa l'emulazione in sé è lecita e la distribuzione di materiale protetto no. (Non è un parere legale.)

Il rischio tecnico più grande per un fork è un altro: l'upstream fa decine di commit al giorno sul motore. Se modifichiamo i file del cuore, dopo un mese non riusciamo più ad aggiornarci. Regola: non toccare i file originali e mettere tutto il nostro in cartelle separate, oppure tenere KytyPS5 come dipendenza di un repo "console" tutto nostro.

## Percorso proposto

1. Compilare e avviare l'emulatore in locale con un gioco nostro, per vedere il ciclo dal vivo.
2. Leggere il launcher e fare una modifica piccola ma visibile, per imparare Qt e il contratto con l'emulatore.
3. Progettare la console: cosa aggiunge rispetto al launcher, su quale hardware, con quale interfaccia. La decisione su come usare anche AnyPS5 è in [ADR-0001](ADR-0001-ibrido-anyps5-kyty.md).
4. Solo dopo, se serve, entrare nell'emulatore partendo dall'overlay e dalle librerie.

## Mini glossario

| Termine | Significato |
|---|---|
| Emulatore / translation layer | Programma che fa girare software scritto per un'altra macchina |
| HLE | High-Level Emulation: reimplementare le funzioni di sistema invece dell'hardware |
| ELF / eboot.bin | Il formato dell'eseguibile del gioco, come un `.exe` |
| NID | Codice numerico che identifica una funzione di sistema Sony |
| PRX | Una libreria dinamica Sony, come una `.dll` |
| PM4 | Il linguaggio dei comandi per le GPU AMD |
| Vulkan | API grafica moderna multipiattaforma usata dall'emulatore |
| Shader / SPIR-V | Programmi per la GPU e il formato in cui Vulkan li vuole |
| RDNA 2 | L'architettura della GPU PS5 (famiglia AMD Radeon RX 6000) |
| Dump / .zar | La copia del gioco estratta da una console, in cartella o in archivio |
