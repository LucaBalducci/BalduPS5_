# Progetto "Console personale" — carta di progetto

**Stato:** bozza, 2026-10-11
**Owner:** Luca Balducci
**Nome del prodotto:** da scegliere. Vincolo: nessun marchio Sony (niente "PlayStation", "PS5", "DualSense" nel nome o nella grafica).
**Decisioni collegate:** [ADR-0001](ADR-0001-ibrido-anyps5-kyty.md) (non fondere i motori), [ADR-0002](ADR-0002-struttura-repository.md) (dove vive il progetto)

---

## 1. Obiettivo

Avere nel mio PC un **sistema da console**: si accende, mostra la libreria dei miei giochi PS5 con le copertine, si naviga col controller, si sceglie un gioco e si gioca. I giochi girano tramite un motore di emulazione esistente; il sistema attorno ai giochi è nostro.

In una frase: **KytyPS5 come motore, non modificato e aggiornato spesso, più un guscio nostro, più la mia libreria catalogata e provata gioco per gioco.**

## 2. Cosa costruiamo noi e cosa usiamo

| Strato | Chi lo fa | Note |
|---|---|---|
| Far girare i giochi (kernel finto, librerie Sony, GPU, shader) | **Usiamo** KytyPS5 | Non si riscrive e non si fonde con altro (ADR-0001). Si aggiorna seguendo l'upstream |
| Secondo motore opzionale | **Usiamo** AnyPS5, in futuro | Solo per titoli che vi girano meglio; oggi aggiunge quasi nulla |
| Guscio: avvio, libreria, navigazione, profilo, trofei, impostazioni | **Costruiamo noi** | Partendo dal launcher Qt di KytyPS5 o da zero; parla ai motori solo via riga di comando e file |
| Libreria: catalogo, prove, matrice di compatibilità | **Facciamo noi** | Lavoro continuo, titolo per titolo |
| Copie dei giochi | **Fuori perimetro** | Vedi §5 |

## 3. Cosa il sistema può essere e cosa non può essere

Può essere: una console da giocatore singolo, offline, con la mia libreria, la mia interfaccia, il mio profilo e i trofei, su un PC x86-64 con GPU Vulkan.

Non può essere:
- **Una replica dell'interfaccia Sony.** Il sistema operativo PS5 è software protetto; il guscio è disegnato da noi.
- **Online.** La rete del motore è forzata offline: nessun account, store, multiplayer.
- **Completa oggi.** La compatibilità la decide il progresso dell'emulatore. Una parte della libreria partirà, una parte avrà glitch, una parte resterà in attesa. Riferimento: la lista comunitaria su kytyps5.github.io.
- **Una piattaforma per giochi nostri via emulatore.** Servirebbe l'SDK Sony. I giochi nostri si fanno con un motore normale e il guscio li ospita come giochi PC.
- **Su ARM.** Il motore richiede un host x86-64.

## 4. Requisiti hardware e software

- CPU x86-64 (un processore recente a 6 o più core è un buon punto di partenza).
- GPU con driver **Vulkan 1.3** aggiornati.
- RAM: il motore riserva circa 13.5 GB per riprodurre la memoria della console. 16 GB sono il minimo pratico, 32 GB consigliati.
- Windows 10 1803+ o una distribuzione Linux corrente. Windows è la piattaforma più testata dall'upstream.
- Controller: DualSense consigliato (trigger adattivi, haptics), qualunque pad SDL funziona.
- Modalità "console": account del PC con autologin che avvia il guscio a schermo intero.

## 5. Prerequisiti esterni

- **Copie decriptate dei giochi che possiedo.** L'emulatore non decripta e non scarica nulla. Le copie si ottengono da una console modificata con firmware specifici: è il passo pratico più difficile dell'intero progetto, è fuori dal perimetro del software, e si colloca in un'area legalmente grigia. Senza questo passo la libreria non esiste.
- Formati accettati dal motore: cartella con `eboot.bin`, oppure archivio ZArchive `.zar` con `eboot.bin` alla radice (montato in sola lettura, senza estrazione).

## 6. Vincoli legali

1. **Licenze.** KytyPS5 e AnyPS5 sono GPL-2.0-only. Il guscio, se avvia i motori come processi separati, è un programma a sé con licenza a nostra scelta; qualsiasi modifica ai motori resta GPL-2.0 e, se distribuita in binario, va distribuita col sorgente. Per uso personale non c'è alcun obbligo di pubblicazione.
2. **Marchi.** Nessun nome o logo Sony nel prodotto.
3. **Contenuti.** Solo copie di giochi posseduti. Mai distribuire copie, firmware, o binari convertiti da AnyPS5 (sono copie del gioco).
4. Niente di quanto sopra è un parere legale.

## 7. Criteri di successo

| Versione | Criterio misurabile |
|---|---|
| **v0 "sembra una console"** | Il PC si accende nel guscio; con il solo controller vedo la griglia delle copertine, avvio un gioco, lo chiudo e torno al menu |
| **v1 "sistema"** | Profilo e nome utente, lingua, impostazioni per gioco, visualizzatore trofei, tema e suoni nostri; matrice di compatibilità aggiornata |
| **Libreria** | Percentuale dei titoli posseduti che arrivano in-game, misurata e tracciata nel tempo (obiettivo iniziale: tutti i titoli segnati "playable" nella lista comunitaria) |
| **Manutenzione** | Aggiornare il motore all'ultima build upstream senza toccare il guscio |

## 8. Tappe

| # | Tappa | Risultato | Stima |
|---|---|---|---|
| 1 | **Motore** | KytyPS5 compilato o scaricato dalle build CI, un gioco mio che parte | 1–2 settimane |
| 2 | **Libreria** | Copie catalogate, ogni titolo provato, matrice di compatibilità in questa cartella | 2–4 settimane, poi continuo |
| 3 | **Guscio v0** | Autostart, navigazione solo controller, griglia copertine (icone `icon0.png` già presenti nelle copie), avvio e ritorno | 3–6 settimane |
| 4 | **Guscio v1** | Profilo, lingua, impostazioni per gioco, trofei, tema | 1–3 mesi |
| 5 | **Opzionali** | AnyPS5 come secondo motore, giochi PC nativi in libreria, contributi all'emulatore per titoli che mi interessano | in prospettiva |

Le stime presuppongono il supporto dell'agente e una persona part-time. La variabile non controllabile è quante copie l'emulatore fa girare oggi.

## 9. Rischi e mitigazioni

| Rischio | Mitigazione |
|---|---|
| L'upstream cambia la riga di comando o le cartelle del motore | Il guscio parla al motore attraverso un solo modulo "adapter"; test di avvio a ogni aggiornamento |
| Compatibilità insufficiente per i titoli che mi interessano | Matrice aggiornata; priorità ai titoli "playable"; pazienza: l'upstream migliora ogni settimana |
| Stutter da compilazione shader | Accettarlo; in prospettiva, cache shader su disco a monte (idea da AnyPS5, ADR-0001 opzione C) |
| Tentazione di modificare il motore | Regola fissa: motore intatto; patch solo in un branch dedicato del fork, piccole e rebasate (ADR-0002) |
| Le copie dei giochi non si ottengono | Il progetto si ferma alla tappa 1 con giochi di prova; nessun aggiramento |

## 10. Cosa NON faremo

- Riscrivere o fondere i motori.
- Copiare l'interfaccia o asset Sony.
- Qualsiasi funzione online o di rete.
- Distribuire il sistema con giochi dentro.
