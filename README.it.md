<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/basalt-lockup-dark.png">
    <img src="assets/basalt-lockup-light.png" alt="Basalt" width="480">
  </picture>
</p>

# Basalt Win+GUI: questo fork

[English](README.md) · *Italiano*

**Basalt Win+GUI** è un fork di **[Basalt](https://github.com/jesdga95/basalt)** di jesdga95 (licenza MIT), che a sua
volta è un fork di **[Strata](https://github.com/Niko1221/Strata)** di Niko1221 (licenza MIT). Il motore, il formato
`.basalt`, il convertitore e il server vengono da Basalt; le note di copyright dei due progetti e del codice che
includono (ggml, llama.cpp e altri) sono in [LICENSE](LICENSE) e [THIRD_PARTY_LICENSES](THIRD_PARTY_LICENSES). I file
del modello derivano da Qwen3.8-Flash-Next e sono sotto la
[Qwen Community License 1.0](https://huggingface.co/Qwen/Qwen3.8-Flash-Next/blob/main/LICENSE), non sotto MIT.

Cosa aggiunge questo fork: un port nativo del motore per **Windows 10/11** (il codice per Linux è invariato) e quello
che serve per usarlo su un desktop: un'app nativa che si apre sul Loader web, che prevede memoria e velocità prima di
caricare; una configurazione guidata che trova i valori migliori per ogni PC; i profili di caricamento; una libreria
dei modelli che li trova, li verifica e li converte; gli script di avvio e arresto. Il README originale di Basalt, in
inglese, è in fondo a [README.md](README.md#upstream-basalt-readme).

## Come funziona

- **Un modello.** Basalt esegue Qwen3.8-Flash-Next (architettura GGUF `qwen4exp`): 48 strati, 512 esperti per strato,
  10 attivi per token. A 3 bit i soli esperti occupano circa 50 GiB, più di quanti ne tenga una GPU da 32 GB.
- **Esperti su tre livelli.** Una **cache degli esperti in VRAM** ne tiene quanti ne entrano (`--expert-cache auto`:
  tutta la VRAM che resta dopo la riserva), riempita secondo un profilo d'uso, così i più usati stanno sulla GPU; una
  **seconda GPU** può fare da livello in più (`--gpu 0,1`); il resto lo calcola la **CPU** (kernel AVX2/AVX-512). Con
  `--resident-experts` gli esperti che non sono in VRAM vengono copiati in RAM una volta sola, invece di essere letti
  dal file quando servono. L'instradamento è sbilanciato: una cache con un terzo degli esperti risponde già a circa il
  98% delle richieste.
- **Il file `.basalt`.** Un file per modello: una tabella di 64 KiB di sezioni con nome (ognuna con il suo SHA-256),
  poi pesi, tokenizer, template della chat, drafter, profilo degli esperti e proiettore per le immagini. Lo costruisce
  `tools/convert/convert.py` dai file GGUF, copiando i tensori byte per byte (nessuna riquantizzazione).
- **Il drafter MTP.** Lo strato di predizione multi-token del modello propone i token successivi e il modello
  principale li verifica in un solo passaggio; il testo è identico a quello senza drafter, perché ogni token lo decide
  la verifica. `spec 7` è una finestra di verifica di 7 token (fino a 6 proposte).
- **Prestito per il prompt.** Un prompt lungo si legge a blocchi grandi, i cui buffer sono presi in prestito dagli
  ultimi posti della cache degli esperti per la durata del prompt e poi riempiti di nuovo (`--prefill auto`).

## Cosa cambia questo fork

- **Port nativo per Windows** (dettagli in [docs/WINDOWS.md](docs/WINDOWS.md)): build MSVC + CUDA 13.1, uno strato di
  piattaforma per file, mappature e letture posizionali, `VirtualLock` e limiti di memoria che tengono conto del
  **commit** (la memoria che Windows ha promesso ai programmi: ci conta ogni allocazione, VRAM compresa), arene
  bloccate secondo il budget di memoria condivisa di WDDM, letture non bufferizzate con una I/O completion port, la
  topologia della CPU con i core performance per primi sui processori ibridi, `basalt serve` in un job object che si
  chiude con lui. `--no-pdl` lancia ogni kernel senza il lancio dipendente programmatico, se un driver si comporta male.
- **L'app, BasaltWinGUI.exe («Basalt Win+GUI»), è l'interfaccia principale.** Avvia il server senza caricare un
  modello e si apre sul Loader web nella sua finestra (WebView2): app e browser hanno esattamente le stesse
  impostazioni; un'icona vicino all'orologio tiene acceso Basalt Win+GUI quando la finestra è chiusa.
- **Interfaccia web** (`/`, in inglese o in italiano, vedi [Lingua](#lingua)): Chat, Monitor, Hardware, **Carica
  modello** (il Loader) e Impostazioni. Ogni impostazione sta in una sola scheda:
  - **Loader**: tutto ciò che si decide al caricamento del modello (il modello, contesto massimo, cache KV, lettura del
    prompt, drafter MTP e la sua finestra, cache degli esperti, esperti residenti, **VRAM da lasciare libera**, tabella
    PLE, thread CPU, slot paralleli, immagini, porta), i profili di caricamento e il **profilo predefinito** (★,
    Machine Default, Valori sicuri: è qui che stanno i valori di caricamento predefiniti), la configurazione guidata,
    la calibrazione e Carica.
  - **Impostazioni**: le preferenze dell'app: lingua, cartelle dei modelli, **modello predefinito per il server**, quota
    di RAM e commit lasciata agli altri programmi, se proporre la configurazione guidata all'avvio. Una riga
    *Parametri di caricamento* mostra cosa gira adesso (per esempio "256K · q4_0 · spec 7 · 4 GB di VRAM liberi") e il
    profilo che corrisponde, con **Apri il Loader** per cambiarlo.
  - **Chat → Impostazioni** (il pannello della chat): i valori di inferenza (campionamento, ragionamento, contesto
    pieno, output strutturato) e i loro preset. **Monitor** e **Hardware** mostrano soltanto.
  - Il Loader modifica ogni impostazione del motore, mostra VRAM, RAM e commit che il piano richiede accanto a quelli
    liberi ora (una quota di RAM e del limite di commit resta sempre libera per gli altri programmi, il 15% se non ne
    scegli un'altra; il file di paging non deve mai crescere) e la velocità prevista; carica il modello, o ricarica il
    motore dentro il server acceso. La calibrazione misura le velocità vere per modello e impostazioni.
  - La **configurazione guidata** trova i valori migliori per questo PC e li salva come **Machine Default** (sotto).
  - **Profili di caricamento** (impostazioni del motore, porta, modello) e **preset di inferenza** (impostazioni della
    chat), condivisi da interfaccia, script e app; «Valori sicuri» è un profilo incorporato **calcolato per questo PC e
    il modello scelto**.
  - Chat con i **valori di campionamento del modello** (dal GGUF d'origine), **politiche di contesto** (ferma al
    limite, finestra scorrevole, o compattazione fatta dal modello) e **output strutturato** (`response_format` JSON /
    JSON Schema), controllato e rigenerato con l'errore come spiegazione (`structured_retries`, predefinito 2).
  - **Libreria dei modelli**: una o più cartelle, una scansione delle sole intestazioni che segna ogni modello
    *Pronto*, *Da convertire* (un GGUF supportato) o *Non compatibile* (con il motivo), conversione e verifica in
    sottofondo, e il modello predefinito del server.
- **Server**: porta predefinita 7232, `--listen-first` (la pagina risponde mentre il modello si carica), `--standby`
  (la pagina risponde e non si carica nulla finché non premi Carica nel Loader), `--relaunch-on-model-change` (caricare
  un altro modello chiude il server con codice d'uscita 75, così chi l'ha avviato lo riavvia con quel modello),
  `/api/settings`, `/api/loader/*`, `/api/wizard/*`, `/api/profiles`, `/api/models/*`.
- **Script** `Avvia-Basalt.ps1` / `Ferma-Basalt.ps1` con gli stessi nomi di opzione di Strata: `-gui` (il predefinito:
  l'app), `-browser`, `-server` (per gli agenti), `--model`, `--config "<profilo>"`, `--port`, `--vram-libera GB`,
  `-riusa` / `-riavvia`, `--lang it|en`.

### Cosa è cambiato dalla versione precedente di questo README

- **Un solo insieme di opzioni** per script e app (`-gui` predefinito, `-browser`, `-server`, `-prova`, `-converti`,
  `--config`, `--model`, `--port`, `--vram-libera`), con gli stessi nomi di Strata; i vecchi `-Sfondo` / `-SoloServer`
  funzionano ancora.
- **Prima l'app**: l'app avvia il server **in attesa** (standby: nulla caricato) e si apre sul Loader web; le cartelle
  si scelgono con la finestra di Windows; un pannello nativo fa da riserva quando il server non parte. Caricare un altro
  modello riavvia il server con quello (codice 75) e la pagina si ricollega.
- **Valori predefiniti veri**: la chat parte dai **valori di campionamento del modello** (dal GGUF d'origine); i valori
  di caricamento vengono dalle regole collaudate di «Valori sicuri», calcolate per questo PC e modello.
- **Configurazione guidata e Machine Default** (profilo di caricamento + preset della chat), con i pulsanti
  **Ripristina**, una versione da riga di comando per gli agenti, e l'ordine dei predefiniti: scelto a mano > Machine
  Default > Valori sicuri.
- **Monitor principale o secondario**: la riserva di VRAM dipende da se la GPU NVIDIA comanda il monitor principale
  (1,5 GB in più), uno secondario (circa 0,4 GB) o nessuno; quello che usano gli altri programmi si mostra, non si somma.
- **Memoria misurata**: RAM e commit presi davvero da un caricamento si conservano per modello e si sommano ai piani
  successivi.
- **Cartelle dei dati**: le impostazioni in `%APPDATA%\BasaltPannello\` (copiate una volta da `~\.cache\basalt`, mai
  spostate).
- **Nome**: l'app si chiama **Basalt Win+GUI** (`BasaltWinGUI.exe`, icona nuova); motore, API, file e script tengono i
  loro nomi.
- **`-server` con un server già acceso** chiede se riusarlo o riavviarlo (`-riusa`, `-riavvia`, 60 s, senza risposta
  si riusa).
- **Inglese come lingua predefinita**, italiano a richiesta, per app, pagina, script e strumenti da riga di comando.
- **Conversione e verifica** dalla libreria dei modelli (GGUF → `.basalt`, controllo SHA-256), in sottofondo.

## Requisiti

- Una **GPU NVIDIA Blackwell** (compute capability 10.x o 12.x: GeForce RTX serie 50, RTX PRO 6000, B200). I kernel
  del motore sono compilati solo per Blackwell; con un'altra GPU il Loader, la configurazione guidata, lo script e l'app
  lo dicono prima di partire, e non si carica nulla.
- **Windows 10 o 11 x64** (driver WDDM) o **Linux x86-64** per motore, server e interfaccia web. Script e app sono solo
  per Windows.
- RAM sufficiente per gli esperti che non entrano in VRAM (circa 50 GiB per i file a 3 bit, con gli esperti residenti).
- Python 3.10 o più recente con i pacchetti di `requirements.txt`, in un ambiente virtuale `.venv` nella cartella del
  repository (`python -m venv .venv` poi `.venv\Scripts\pip install -r requirements.txt`).
- Per l'app: .NET 8 Desktop Runtime e Microsoft Edge WebView2 Runtime (senza WebView2 l'app apre l'interfaccia web nel
  browser predefinito).
- Compilare: Windows in [docs/WINDOWS.md](docs/WINDOWS.md) (strumenti Visual Studio v143, CUDA 13.1, CMake, Ninja),
  Linux nel README originale ([Build](README.md#build)). L'app: `dotnet publish tools\windows\desktop -c Release -o
  tools\windows\desktop\publish`.

## Avviare Basalt Win+GUI

```powershell
.\Avvia-Basalt.ps1                                             # l'app (-gui): prima il Loader, nulla caricato
.\Avvia-Basalt.ps1 -browser                                    # il server legato a questa finestra, l'interfaccia nel browser
.\Avvia-Basalt.ps1 -server --config "Machine Default"          # staccato, senza interfaccia: torna quando il modello è caricato
.\Avvia-Basalt.ps1 -server --model "D:\Modelli\Qwen3.8-Flash-Next-IQ3_XXS.basalt" --port 7240 --vram-libera 2
.\Avvia-Basalt.ps1 -prova                                      # tutti i controlli, nulla avviato, nulla scritto
.\Ferma-Basalt.ps1 --port 7240                                 # ferma solo quel server (il suo albero di processi)
```

Ogni opzione si accetta con uno o due trattini e in maiuscolo o minuscolo (`-gui`, `--gui`, `-GUI`).

| opzione | significato |
|---|---|
| *(nessuna)* = `-gui` | apre `BasaltWinGUI.exe` passandogli `--config`, `--model`, `--port` e `--vram-libera`; il browser **non** si apre, l'indirizzo si stampa e resta disponibile. Senza l'app compilata lo script dice come compilarla e usa `-browser` |
| `-browser` | il server legato a questa finestra (Ctrl+C o chiudere la finestra ferma server e motore e libera la RAM), l'interfaccia aperta nel browser |
| `-server` | il server da solo, staccato, senza app né browser; lo script torna quando `/health` dice che il modello è caricato. Per gli agenti; lo ferma `Ferma-Basalt.ps1` |
| `--config "<profilo>"` | il profilo di caricamento; senza, il profilo predefinito (sotto) |
| `--model "<file .basalt>"` | il modello da servire, con il percorso completo |
| `--port N` | la porta; vince su quella del profilo (predefinita 7232) |
| `--vram-libera GB` | la VRAM da lasciare libera per gli altri programmi, in GB (`1.5` o `"1,5"`), al posto di quella del profilo; `--vram-reserve-mib N` è lo stesso in MiB |
| `-converti` | con `-browser` / `-server`, quando il modello è un GGUF da convertire: mostra il piano, chiede, converte, poi serve il file nuovo (nell'app: **Converti…** nel Loader) |
| `-riusa` / `-riavvia` | con `-server`, la risposta data in anticipo quando un server è già acceso con altri parametri di caricamento: riusarlo, o fermarlo e riavviarlo con i nuovi (sotto) |
| `--lang it\|en` | la lingua dei messaggi per questo avvio (imposta anche `BASALT_LANG` per il server e l'app che avvia) |
| `-prova` | tutti i controlli, nulla avviato; `-browser` / `-server` lavorano su una copia del file di configurazione |
| `-file FILE` | il file di configurazione (predefinito `basalt_config.json` accanto allo script; l'app ne crea uno senza modello in un'installazione nuova) |

`-Sfondo` e `-SoloServer`, i vecchi nomi, funzionano ancora come `-server` e stampano una nota. `Ferma-Basalt.ps1`
accetta `--port N` e `--file FILE` allo stesso modo.

**Quando un server è già acceso** (uno per installazione, su qualunque porta):

- `-gui` apre l'app, che chiede **Collegati al server in esecuzione** o **Cambia modello** (ferma solo l'albero di
  processi di quel server, poi si sceglie nel Loader).
- `-browser` apre il browser su quello; un `--model` o `--config` diverso non si applica, e lo script lo dice.
- `-server` risolve i parametri di caricamento chiesti (`--config`, altrimenti il profilo predefinito; `--model`;
  `--vram-libera`; `--port`) esattamente come per un avvio nuovo e li confronta con quelli del server acceso (file del
  modello, ogni impostazione di caricamento, porta).
  - **Uguali**: stampa indirizzo, modello e profilo e torna (codice d'uscita 0).
  - **Diversi**: elenca le differenze (`engine.max-context: 65536 -> 131072`) e chiede **Riusa il server avviato** o
    **Riavvia con i nuovi parametri**: in una finestra quando c'è un desktop, altrimenti nella console (`R` / `A`).
    Senza risposta entro **60 secondi** (si cambiano con `BASALT_ASK_TIMEOUT_S`), o senza nessuno a cui chiedere (un
    agente con l'input rediretto), il server acceso **si riusa**: nessun server viene fermato senza una risposta.
    Riavviare ferma solo l'albero di processi di quel server (`Ferma-Basalt.ps1 --port N`) e lo riavvia sulla stessa
    porta, se `--port` non ne chiede un'altra.
  - **`-riusa`** e **`-riavvia`** danno la risposta in anticipo, per agenti e attività pianificate; con `-prova` lo
    script dice soltanto cosa farebbe.

**Quale modello parte** con `-browser` / `-server`: il primo impostato fra: `--model`; il modello salvato nel profilo
di caricamento; il **modello predefinito del server** (scelto in **Impostazioni → Modello predefinito per il
server**); il modello già nel file di configurazione. Se nessuno è impostato lo script si ferma con codice 1 e dice
come sceglierne uno. Il controllo della memoria usa lo stesso piano del Loader; un avvio che farebbe crescere il file
di paging viene rifiutato, come una GPU che il motore non supporta (codice 1).

**Il profilo di caricamento predefinito**: quello scelto con **Rendi predefinito** nel Loader, altrimenti **Machine
Default** dopo che la configurazione guidata l'ha salvato, altrimenti **Valori sicuri**.

Direttamente, su Windows o Linux: `bin\basalt.exe serve --config basalt_config.json [--port N]` (`bin/basalt serve` su
Linux; ogni opzione è nel README originale, [Settings](README.md#settings---config), e in `python -m serve.server
--help`).

### L'app

`tools\windows\desktop\publish\BasaltWinGUI.exe` (finestra e icona: «Basalt Win+GUI»; il nome del file è diverso da
quello del motore, `basalt.exe`, di proposito; le stesse opzioni dello script: `--config`, `--model`, `--port`,
`--vram-libera`, `--file`, `--lang`). All'apertura avvia il server nel suo job (finisce con l'app) **senza caricare un
modello** e mostra il **Loader** web:

1. la prima volta, le cartelle dei modelli (**Sfoglia…** apre la finestra di Windows per scegliere una cartella; sono
   proposti i posti abituali trovati su questo PC);
2. il modello, il suo stato, **Verifica** e **Converti…**;
3. finché questo PC non ha un Machine Default, viene proposta la **configurazione guidata** (si può saltare, o
   spegnere);
4. il profilo di caricamento e ogni valore, la VRAM da lasciare libera, la previsione, poi **Carica**.

Caricare un modello diverso da quello con cui il server è partito lo salva e riavvia il server con quello; la pagina si
ricollega da sola. Chiudere la finestra la nasconde nell'icona vicino all'orologio: **Apri**, **Ricarica con altri
parametri…** (il Loader), **Avvia solo server (per gli agenti)…** (come `Avvia-Basalt.ps1 -server`, al posto del server
dell'app), **Spegni**. Se il server non parte (una porta occupata, una build mancante), l'app mostra il motivo, la fine
del registro e un pannello di avvio nativo come riserva. Con il server acceso la stessa interfaccia è a
`http://127.0.0.1:7232/` in qualunque browser.

### Configurazione guidata e Machine Default

**Configurazione guidata** nel Loader (anche `python -m serve.loader wizard` per gli agenti, sotto) trova i valori
migliori per questo PC; l'app distribuita su altri PC la usa per adattarsi a ogni macchina. Perché: i valori giusti
dipendono da VRAM, RAM, limite di commit, core e da quale scheda mostra il desktop. Valori fissi sprecherebbero
velocità su un PC grande o non si caricherebbero su uno piccolo (o farebbero crescere il file di paging). La
configurazione guidata calcola ogni valore da quello che rileva, dice perché, e può verificarlo con una misura vera.

1. **Questo PC**: RAM, limite di commit e quanto è usato, file di paging (quelli gestiti da Windows possono crescere:
   Basalt Win+GUI non li fa mai crescere), core performance ed efficienza, la GPU NVIDIA, e le schede video con quella
   che mostra il **monitor principale**. Un PC può comandare il monitor da un'altra scheda (una GPU integrata, per
   esempio) e tenere la GPU NVIDIA per il calcolo; la GPU può avere anche un monitor **secondario**, acceso o spento.
   La configurazione mostra cosa ha rilevato e chiede di confermarlo o correggerlo.
2. **Cosa lasciare agli altri programmi**: la quota di RAM e commit (15% proposto, dal 5 al 50%, mai meno di 2 GB;
   diventa la regola di questo PC per ogni piano, si cambia poi in **Impostazioni → Memoria e avvio**) e la VRAM: 0,5 GB per il driver, più 1,5 GB quando la GPU comanda il
   monitor principale (o circa 0,4 GB per uno secondario), arrotondati per eccesso a 256 MiB. Quello che gli altri
   programmi usano ora sulla GPU si mostra, non si somma: la cache degli esperti si dimensiona sulla VRAM libera al
   momento del caricamento, che lo lascia a loro.
3. **Il modello** e i suoi valori per la chat.
4. **I valori calcolati**, con i motivi e la previsione: cache KV q4_0, prefill auto, il drafter a spec 7 e min-p 0,5
   quando il file ce l'ha, esperti residenti e tabella PLE in RAM quando ci stanno, i thread della CPU dal processore,
   il contesto più lungo che ci sta.
5. **Misura** (facoltativa, circa 5-10 minuti, solo con meno del 75% del limite di commit in uso): i valori si
   caricano, la memoria vera dopo il caricamento si confronta con il piano (un contesto più corto, poi niente esperti
   residenti, finché è sopra), si calibra la velocità, si prova la soglia del drafter per richiesta (0,3, 0,5, 0,7) e
   le finestre di verifica 5, 7 e 8 (si fissano all'avvio del motore: un ricaricamento ciascuna); si tiene la più
   veloce (a parità, la finestra più piccola) e si mostrano tutti i risultati.
6. **Salva**: il profilo di caricamento **Machine Default** (con il modello; diventa il predefinito) e il preset della
   chat **Machine Default** (i valori di campionamento del modello). **Ripristina Machine Default** li rimette: nel
   Loader accanto a **Ripristina valori sicuri**, e nelle impostazioni della chat accanto a **Valori del modello**.

Senza domande, per gli agenti (le risposte rilevate e proposte, salvo quelle date; la misura richiede il Loader):

```powershell
.venv\Scripts\python -m serve.loader wizard --model "D:\Modelli\Qwen3.8-Flash-Next-IQ3_XXS.basalt" `
    --monitor none --free-share 0.15 --vram-libera 2 --save        # --json per un solo oggetto JSON
```

**Le impostazioni della chat** restano di ogni client: l'app e ogni browser hanno le loro. Un client si apre con il
preset usato per ultimo, se esiste ancora; altrimenti segue il predefinito di adesso (Machine Default, altrimenti i
valori del modello). Le impostazioni cambiate a mano restano, e il pannello dice quale preset è in uso.

### Modelli: cartelle, conversione, verifica

**Impostazioni → Cartelle dei modelli** contiene una o più **cartelle dei modelli** (il Loader le chiede la prima
volta), per esempio quella dei tuoi file `.basalt` e quella dei download di LM Studio; **Impostazioni → Modello
predefinito per il server** è il modello che gli script e l'app avviano senza `--model`. Il Loader sceglie fra questi
il modello da caricare. Ogni `.basalt` e `.gguf` al loro interno viene descritto dalla sola
intestazione e messo in cache; nessuno legge i pesi. Ogni modello ha uno stato:

- **Pronto**: un file `.basalt` che il motore può aprire. **Verifica** controlla lo SHA-256 di ogni sezione in
  sottofondo.
- **Da convertire**: un insieme GGUF di Qwen3.8-Flash-Next con tutte le parti presenti e formati degli esperti che i
  kernel della GPU coprono (per esempio IQ3_XXS, IQ3_S, Q2_0, Q8_0 e le quantizzazioni UD di Unsloth). **Converti…**
  mostra prima una finestra con la dimensione del file nuovo, il disco e il suo spazio libero; parte solo dopo quella
  conferma, con spazio sufficiente e il commit sotto il 75% del limite. Il proiettore per le immagini (`mmproj`) nella
  stessa cartella entra nel file; il drafter MTP viene da una cartella già sul disco o si scarica dal checkpoint Qwen
  (`mtp_fetch.py`). I valori di campionamento consigliati dal GGUF d'origine (`general.sampling.*`) diventano i
  predefiniti della chat del file; per i file convertiti prima, il server li legge dalla copia dell'intestazione
  d'origine conservata nel file.
- **Non compatibile**: tutto il resto, con il motivo (un'altra architettura, formati degli esperti senza kernel come
  gli esperti Q2_K statici o i down Q4_0, una parte mancante, un download non finito). I file del proiettore e del
  drafter sono elencati come allegati del loro modello.

Lo stesso da un terminale: `python -m serve.models list | json | dirs | set-dirs DIR ... | default --set FILE |
resolve [--model FILE] [--profile NOME] | convert --model GGUF [--out DIR]`.

### VRAM da lasciare libera

**VRAM da lasciare libera** (nel Loader e nella configurazione guidata; `engine.vram-reserve-mib`,
salvata nei profili) è la VRAM che il motore non tocca, per desktop, browser, driver e altri programmi. Con la cache
degli esperti su `auto`, tutto il resto contiene esperti. Spostare il controllo ricalcola, dal vivo: la VRAM che resta
per gli esperti, quanti esperti ci stanno (per esempio «13.066 di 24.576 esperti in VRAM (53%)»), la percentuale
attesa di esperti trovati in cache, le velocità previste di risposta e di lettura del prompt, e il commit che il
caricamento richiede.

- **Il compromesso**: più VRAM libera vuol dire meno esperti sulla GPU (un po' più lento, perché quelli mancanti li
  calcola la CPU) e, su Windows, meno commit (la VRAM ci conta). Meno VRAM libera è più veloce, finché il desktop o un
  altro programma non resta a corto.
- **Come scegliere**: la regola della configurazione guidata qui sopra; la pagina dice quali programmi usano la GPU e
  avvisa quando resta meno di 1 GB libero su una GPU che comanda il monitor principale. Lasciane di più mentre giochi o
  monti video sulla stessa GPU.

**Memoria misurata.** Dopo ogni caricamento vero il server confronta RAM e commit presi davvero (le letture del sistema
prima e dopo) con il piano degli stessi valori, e conserva la differenza con le calibrazioni del modello; da lì in poi
i piani di quel modello aggiungono quello che era in più (mostrato come «misurato all'ultimo caricamento»).

### «Valori sicuri»: calcolati, non fissi

Il profilo incorporato si calcola ogni volta dall'hardware trovato e dal modello scelto, con le stesse regole di ogni
piano e della configurazione guidata: la quota di RAM e commit di questo PC resta libera; la riserva di VRAM è quella
della configurazione guidata (o la sua regola, con 1 GB per il monitor finché la configurazione non conferma dov'è:
1,5 GB in tutto); la cache KV è q4_0 e il contesto il più lungo che ci sta (fino a 262.144); il drafter è acceso (spec 7,
min-p 0,5) solo se il file ce l'ha, le immagini solo se ha un proiettore; esperti residenti e tabella PLE in RAM solo se
ci stanno; i thread della CPU sono i core performance più metà dei core efficienza, meno uno (su un processore senza
core efficienza, i core fisici meno uno). Ogni valore ha il suo motivo nel Loader. Per esempio, su una RTX 5090 (32 GB,
1 GB usato da altri programmi) con 128 GB di RAM e un processore con 8 core performance e 16 efficienza, il file a 3 bit
dà: 262.144 token, q4_0, spec 7 a min-p 0,5, cache esperti auto, esperti residenti, 1,5 GB lasciati liberi, 15 thread
CPU, immagini dynamic.

### Dove stanno le impostazioni

Per utente, mai nel repository, nella **cartella dei dati dell'utente**: `%APPDATA%\BasaltPannello\` su Windows,
`~/Library/Application Support/Basalt/` su macOS, `$XDG_DATA_HOME/basalt/` (`~/.local/share/basalt/`) altrove, oppure
`BASALT_DATA`. Contiene `profiles.json` (profili di caricamento, preset di inferenza, il profilo predefinito),
`models.json` (le cartelle dei modelli, il modello predefinito del server, l'ultima cartella di uscita), `machine.json`
(le risposte della configurazione guidata: quota di RAM e commit, VRAM da lasciare libera, dov'è il monitor),
`calibration\<modello>.json` (velocità e memoria misurate), `prefs.json` (la lingua) e `desktop.json` dell'app. Le cache
che si possono ricostruire restano in `%USERPROFILE%\.cache\basalt\` (`~/.cache/basalt/`, o `BASALT_CACHE`): la
scansione delle intestazioni, i registri dei lavori di conversione e verifica, i file del tokenizer e delle immagini
estratti da un modello, il registro del motore. Le versioni precedenti tenevano le impostazioni nella cartella della
cache: la prima volta vengono copiate (non spostate) nella cartella dei dati. `basalt_config.json` accanto agli script
lo scrivono loro e il Loader dal profilo scelto, e non è tracciato da git.

## Usarlo come server (API)

Il server parla le API di OpenAI e di Anthropic a `http://127.0.0.1:<porta>` (porta predefinita **7232**):

| endpoint | API |
|---|---|
| `POST /v1/chat/completions` | OpenAI Chat Completions: messaggi, strumenti (function calling), immagini se la visione è accesa, `response_format`, `stream` |
| `POST /v1/responses` | OpenAI Responses, **senza stato**: si manda tutta la conversazione ogni volta (`previous_response_id` e `GET /v1/responses/<id>` non sono conservati) |
| `POST /v1/messages` | Anthropic Messages: strumenti, blocchi di ragionamento (thinking), immagini, `stream` |
| `POST /v1/messages/count_tokens` | conteggio dei token di Anthropic: lo stesso prompt che leggerebbe `/v1/messages`, senza eseguirlo |
| `GET /v1/models` | il modello servito e ogni altro modello delle cartelle dei modelli che potrebbe servire (forma OpenAI o Anthropic) |
| `GET /v1/models/{id}` | uno di questi |
| `GET /health` | `{"service": "basalt", "loaded": true, ...}` quando il modello è caricato; `"standby": true` mentre un server avviato dall'app aspetta il Loader |

**Avviarlo per gli agenti**: `.\Avvia-Basalt.ps1 -server` (torna quando il modello è caricato; negli script senza
nessuno davanti aggiungi `-riusa`), fermarlo con `.\Ferma-Basalt.ps1`. `basalt.pid` contiene PID, porta e profilo di un
server avviato da uno script o dall'app.

**Nome del modello.** L'id del modello servito è l'`--alias` (o `"alias"` nella sezione `server` del file di
configurazione), altrimenti il nome scritto nel file del modello. Il campo `model` di una richiesta può essere
qualunque cosa (il server esegue un modello alla volta) e la risposta riporta il nome servito, con un'eccezione: un
nome che `GET /v1/models` elenca come un altro modello, non caricato, viene rifiutato (sotto).

**L'elenco dei modelli.** `GET /v1/models` elenca per primo il modello servito (`"status": {"value": "loaded"}`, o
`"unloaded"` mentre si carica alla prima richiesta), poi ogni file `.basalt` delle cartelle dei modelli che il Loader
mostra come **pronto** (un GGUF da convertire o un file che Basalt non può eseguire non è elencato), ciascuno con
`"status": {"value": "unloaded"}`. L'id di un modello così è il nome del file senza `.basalt` (con un breve hash in
coda quando due cartelle hanno lo stesso nome di file): il nome da scegliere in un client. Ogni voce ha `created` (la
data del file), `owned_by`, `display_name`, la lunghezza del contesto quando il file la dice (`context_length`; per il
modello servito `meta.n_ctx` è il contesto con cui gira, per un altro modello `meta.n_ctx_train` il più lungo con cui
è stato addestrato), `architecture.input_modalities` (`image` quando il file ha un proiettore visivo) e, solo su un
server in ascolto su 127.0.0.1, il `path` del file. Le cartelle si leggono dalle intestazioni dei file, su un thread, al
massimo ogni 30 secondi: né l'elenco né una richiesta di chat aspettano il disco; l'elenco funziona anche in standby
(nessun modello caricato: solo i modelli delle cartelle). Una richiesta che nomina un modello che l'elenco dà come non
caricato (per id, nome del file o percorso) riceve un 404 che dice come caricarlo, invece della risposta del modello
caricato: OpenAI `{"error": {"code": "model_not_found", ...}}`, Anthropic
`{"type": "error", "error": {"type": "not_found_error", ...}}`.

```sh
# forma OpenAI
curl http://127.0.0.1:7232/v1/models
# {"object": "list", "data": [
#   {"id": "Qwen3.8 Flash Next Uncensored", "object": "model", "created": 1791904651, "owned_by": "basalt",
#    "display_name": "Qwen3.8 Flash Next Uncensored", "status": {"value": "loaded"}, "meta": {"n_ctx": 131702},
#    "context_length": 131702, "architecture": {"input_modalities": ["text"], "output_modalities": ["text"]},
#    "path": "D:/Modelli/Qwen3.8-Flash-Next-Uncensored-IQ3_XXS.basalt"},
#   {"id": "OrcaRouter-IQ3_XXS", "object": "model", "created": 1791904651, "owned_by": "basalt",
#    "display_name": "OrcaRouter", "status": {"value": "unloaded"}, "context_length": 262144,
#    "meta": {"n_ctx_train": 262144}, "quant": "IQ3_XXS", "path": "D:/Modelli/OrcaRouter-IQ3_XXS.basalt", ...}]}

# forma Anthropic: con l'header anthropic-version (o x-api-key senza Authorization)
curl http://127.0.0.1:7232/v1/models -H "anthropic-version: 2023-06-01"
# {"data": [{"type": "model", "id": "Qwen3.8 Flash Next Uncensored", "display_name": "Qwen3.8 Flash Next Uncensored",
#            "created_at": "2026-10-13T15:17:31Z"},
#           {"type": "model", "id": "OrcaRouter-IQ3_XXS", "display_name": "OrcaRouter",
#            "created_at": "2026-10-13T15:17:31Z"}],
#  "has_more": false, "first_id": "Qwen3.8 Flash Next Uncensored", "last_id": "OrcaRouter-IQ3_XXS"}

# un modello, nelle due forme (un id con spazi va codificato nell'URL); altrimenti 404 model_not_found / not_found_error
curl http://127.0.0.1:7232/v1/models/OrcaRouter-IQ3_XXS
curl http://127.0.0.1:7232/v1/models/OrcaRouter-IQ3_XXS -H "anthropic-version: 2023-06-01"

# una chat che nomina il modello non caricato
curl http://127.0.0.1:7232/v1/chat/completions -H "Content-Type: application/json" \
  -d '{"model": "OrcaRouter-IQ3_XXS", "messages": [{"role": "user", "content": "ciao"}]}'
# 404 {"error": {"message": "The model 'OrcaRouter-IQ3_XXS' is not loaded (this server is serving 'Qwen3.8 Flash
#   Next Uncensored'). Load it in the web page's Loader (choose it in the model list, then Load), or start the server
#   with it: .\Avvia-Basalt.ps1 --model \"D:/Modelli/OrcaRouter-IQ3_XXS.basalt\"", "type": "invalid_request_error",
#   "param": "model", "code": "model_not_found"}}
```

TODO (possibile in futuro: caricare su richiesta): il server non cambia modello da solo quando una richiesta ne nomina
un altro dell'elenco; dice solo come caricarlo. Caricare il modello chiesto alla richiesta resta per dopo.

**Chiave API.** Il server ascolta solo su 127.0.0.1 e non chiede chiavi: ai client che ne vogliono una per forza si
può dare un valore qualunque. `--api-key CHIAVE` (solo da riga di comando, mai nel file di configurazione) la rende
obbligatoria su `/v1/*`, come `Authorization: Bearer CHIAVE` o `x-api-key: CHIAVE`; serve per ascoltare su un altro
indirizzo (`--host 0.0.0.0`) e per lasciar usare il server a pagine web di altre origini. Script e app avviano il
server su 127.0.0.1 senza chiave.

**Streaming.** `"stream": true` manda eventi server-sent nel formato di ciascuna API (OpenAI `chat.completion.chunk`,
eventi Responses, Anthropic `message_start` / `content_block_delta` / ...), ragionamento compreso, come
`reasoning_content` (OpenAI) o blocchi `thinking` (Anthropic).

**Ragionamento.** Il modello ragiona per impostazione predefinita (il livello alto del template). OpenAI:
`"reasoning_effort": "none" | "low" | "medium" | "high"` (Responses: `"reasoning": {"effort": ...}`). Anthropic:
`"thinking": {"type": "disabled"}`, oppure `"enabled"` con `budget_tokens` (sotto 2048: basso, sotto 8192: medio,
altrimenti alto). `reasoning_budget_tokens: N`, su qualunque endpoint, chiude il ragionamento dopo N token e fa
rispondere il modello.

```sh
curl http://127.0.0.1:7232/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "basalt", "messages": [{"role": "user", "content": "Saluta in tre lingue."}],
  "max_tokens": 512, "reasoning_effort": "low"}'

curl -N http://127.0.0.1:7232/v1/messages -H "Content-Type: application/json" -H "anthropic-version: 2023-06-01" -d '{
  "model": "basalt", "max_tokens": 1024, "stream": true,
  "messages": [{"role": "user", "content": "Scrivi un haiku sul basalto."}]}'

curl http://127.0.0.1:7232/v1/responses -H "Content-Type: application/json" -d '{
  "model": "basalt", "input": "Quanto fa 17 * 23?", "reasoning": {"effort": "none"}}'
```

(In PowerShell usa `curl.exe` e metti il JSON in un file: `-d "@corpo.json"`.)

Python, con gli SDK ufficiali (`pip install openai anthropic`):

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:7232/v1", api_key="none")   # chiave qualunque, salvo --api-key
r = client.chat.completions.create(model="basalt", messages=[{"role": "user", "content": "Ciao!"}],
                                   max_tokens=512, reasoning_effort="low")
print(r.choices[0].message.content)

for chunk in client.chat.completions.create(model="basalt", stream=True,
                                            messages=[{"role": "user", "content": "Conta fino a cinque."}]):
    print(chunk.choices[0].delta.content or "", end="", flush=True)

print(client.responses.create(model="basalt", input="Nomina tre rocce vulcaniche.").output_text)
```

```python
import anthropic

client = anthropic.Anthropic(base_url="http://127.0.0.1:7232", api_key="none")
msg = client.messages.create(model="basalt", max_tokens=1024,
                             messages=[{"role": "user", "content": "Ciao!"}])
print(msg.content[-1].text)                      # prima può esserci un blocco di ragionamento

with client.messages.stream(model="basalt", max_tokens=1024, thinking={"type": "disabled"},
                            messages=[{"role": "user", "content": "Conta fino a cinque."}]) as s:
    for text in s.text_stream:
        print(text, end="", flush=True)

print(client.messages.count_tokens(model="basalt", messages=[{"role": "user", "content": "Ciao!"}]).input_tokens)
```

**Claude Code** usa l'endpoint di Anthropic:

```powershell
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:7232"
$env:ANTHROPIC_AUTH_TOKEN = "none"                 # valore qualunque; la chiave data con --api-key, se c'è
$env:ANTHROPIC_MODEL = "basalt"                    # nome qualunque: il server ha un solo modello
$env:ANTHROPIC_SMALL_FAST_MODEL = "basalt"
claude
```

(Su Linux/macOS: `export ANTHROPIC_BASE_URL=http://127.0.0.1:7232` e così via.) Gli strumenti per agenti mandano prompt
lunghi: scegli nel profilo di caricamento un contesto di almeno 65.536 token, e tieni `--parallel 1` a meno che più
agenti lavorino insieme (il contesto si divide fra i posti, vedi
[`--parallel`](README.md#concurrent-requests---parallel-n)).

**Output strutturato** (`/v1/chat/completions`): `"response_format": {"type": "json_object"}` oppure `{"type":
"json_schema", "json_schema": {"name": ..., "schema": {...}}}`. La risposta viene controllata; se non è JSON valido, o
non rispetta lo schema, la richiesta si rigenera con l'errore come spiegazione, fino a `structured_retries` volte (da
0 a 5, predefinito 2; 0 lo spegne). Il nuovo tentativo è la stessa conversazione più un messaggio che dice cosa non
andava e dove (l'errore, il percorso JSON in cui il testo si è rotto, una voce scritta più volte, una sola volta, col
suo numero di ripetizioni); il testo rifiutato non viene mai rimandato indietro, così un modello che girava a vuoto non
rivede il suo giro. Solo la risposta validata resta trattenuta fino al controllo. Il ragionamento scorre mentre viene
scritto, anche quello dei tentativi rifiutati; un tentativo rifiutato è annunciato da un chunk con `delta` vuoto e
`"structured_output": {"attempt": 1, "discarded": true, "reason": "...", "retrying": true}`, la sua risposta non viene
mai inviata, e l'ultimo chunk porta solo la risposta validata, con `"structured_output": {"validated": true,
"attempts": 2, "discarded": [{"attempt": 1, "reason": "..."}]}` (una risposta senza streaming restituisce
ragionamento e risposta dell'ultimo tentativo, con lo stesso campo). `GET /v1/status` lo dichiara in
`structured_output`: `content_buffered` e `reasoning_streamed` sono veri, e `stream_buffered` (lasciato per i client
vecchi) significa che la risposta validata è trattenuta fino al controllo. `/v1/responses` con un `text.format` JSON
trasmette anche il ragionamento e controlla la risposta una volta, senza ritentare; `/v1/messages` non ha output
strutturato.

**Occupato ed errori.** Con `--parallel 1` le richieste aspettano il loro turno in ordine d'arrivo. Mentre il motore
parte, o mentre un server avviato dall'app è in attesa (nessun modello caricato), una richiesta riceve un 503 che dice
perché: riprova più tardi, o carica un modello nel Loader. Un prompt più lungo del contesto viene rifiutato con un 400
che dice la lunghezza del contesto.

## Impostazioni di caricamento, una per una

Si impostano nel Loader (ognuna con il suo motivo e la previsione dal vivo), si salvano nei profili di caricamento e
si scrivono nelle sezioni `server` / `engine` di `basalt_config.json`. Valgono quando il motore si (ri)carica.

| impostazione | cosa fa | come sceglierla |
|---|---|---|
| **Modello** (`server.model`) | il file `.basalt` da servire | dalla libreria dei modelli; un GGUF prima si converte |
| **Porta** (`server.port`) | la porta HTTP, predefinita 7232 | cambiala solo per due server insieme o se la 7232 è occupata |
| **Richieste in parallelo** (`server.parallel`, 1-8) | richieste servite insieme nei posti (slot) del motore | 1 per un utente o un agente (la più veloce per richiesta); di più solo per più agenti insieme. Il contesto si divide: ogni posto ha `max-context / N` |
| **Contesto intero per posto** (`server.slot-full-context`) | ogni posto ha tutto il contesto invece di una parte | costa N volte la cache KV, presa dalla cache degli esperti: lascialo spento salvo che ogni agente abbia bisogno di tutta la lunghezza |
| **Immagini** (`server.vision`) | `off`, `dynamic` (prende in prestito posti della cache degli esperti solo mentre legge un'immagine: nessuna VRAM fissa), `on` (encoder sempre sulla GPU, circa 2,2 GiB), `peer` (su una seconda GPU), `cpu` (niente VRAM, lento) | `dynamic` se il file ha un proiettore, altrimenti `off` |
| **Contesto** (`engine.max-context`) | la conversazione più lunga, in token (fino a 262.144) | il più lungo che ci sta (lo scelgono «Valori sicuri» e la configurazione guidata); uno più corto lascia più VRAM agli esperti (un po' più veloce) e chiede meno commit |
| **Cache KV** (`engine.kv`) | la precisione della memoria dell'attenzione: `fp16`, `int8`, `q4_0`, `k8v4` (chiavi INT8, valori Q4_0) | `q4_0`: un quarto della memoria di fp16, così i contesti lunghi ci stanno; `int8` o `k8v4` per un'attenzione un po' più precisa con più memoria |
| **Blocchi del prompt** (`engine.prefill`) | come si leggono i prompt lunghi: `auto` (il blocco più grande che la cache degli esperti può prestare), `auto:8192/16384/32768` (un tetto), o una misura fissa | `auto` |
| **Finestra di verifica** (`engine.spec`, 0-8) | il drafter MTP propone fino a N-1 token, il modello li controlla in un passaggio; 0 = spento | 7 se il file ha un drafter (la configurazione guidata misura 5, 7 e 8); il testo è lo stesso a ogni valore, cambia solo la velocità |
| **Soglia del drafter** (`engine.spec-min-p`, 0-1) | una proposta si fa solo finché il drafter ne è sicuro almeno così | 0,5; la configurazione guidata prova 0,3, 0,5 e 0,7. Più bassa propone di più (più lavoro sprecato sul testo difficile), più alta di meno |
| **Cache degli esperti** (`engine.expert-cache`) | posti per esperti tenuti in VRAM: `auto` (tutta la VRAM che resta dopo la riserva) o un numero | `auto` |
| **Esperti residenti** (`engine.resident-experts`) | gli esperti che non sono in VRAM si copiano in RAM una volta al caricamento, invece di leggerli dal file | acceso se ci stanno in RAM tenendo libera la quota (circa 50 GiB per il file a 3 bit); spento sui PC con meno RAM, a costo di velocità |
| **VRAM da lasciare libera** (`engine.vram-reserve-mib`) | VRAM che il motore non tocca (desktop, driver, altri programmi) | vedi [VRAM da lasciare libera](#vram-da-lasciare-libera); la regola della configurazione guidata: 0,5 GB + 1,5 GB se la GPU comanda il monitor principale (circa 0,4 GB per uno secondario) |
| **Tabella PLE** (`engine.ple-io`) | la tabella n-gram per strato: `direct` (letta dall'SSD quando serve, mai in RAM), `ram` (bloccata in RAM all'avvio: RAM ma niente commit), `mmap` (una variante di prova) | `ram` se ci sta tenendo libera la quota, altrimenti `direct`; su un disco lento o meccanico meglio `ram` |
| **Thread CPU** (`engine.pool-workers`) | i thread che calcolano gli esperti che non sono in VRAM | i core performance più metà dei core efficienza, meno uno (un core resta per il server e il desktop) |

Una regola in più vale per il PC, non per il profilo: la **quota di RAM e commit lasciata libera** agli altri
programmi (15% se la configurazione guidata non ha avuto un'altra risposta, dal 5 al 50%, almeno 2 GB). Ogni piano,
«Valori sicuri» e la configurazione guidata la tengono libera, e un caricamento che farebbe crescere il file di paging
viene rifiutato.

## Impostazioni della chat (inferenza), una per una

Nel pannello delle impostazioni della chat; si salvano per client e come **preset di inferenza** con un nome,
condivisi tramite il server (il preset **Machine Default** è quello della configurazione guidata). Una richiesta porta
solo i campi impostati, così tutto quello che resta vuoto usa i **valori del modello** (da `general.sampling.*` del
GGUF d'origine, mostrati in grigio nei campi), poi quelli del motore. I client delle API mandano gli stessi campi nelle
richieste.

| impostazione | campo della richiesta | cosa fa | consiglio |
|---|---|---|---|
| **Thinking** (accanto al messaggio; effort) | `reasoning_effort` | quanto il modello ragiona prima di rispondere: High (predefinito), Medium, Low, Off | High per i problemi difficili, Low/Off per risposte rapide e chiamate di servizio (molto più veloce) |
| **Budget di ragionamento** | `reasoning_budget_tokens` | dopo N token di ragionamento il ragionamento si chiude e il modello risponde; vuoto o 0 = nessun limite | 2.000-8.000 per limitare le risposte lente; troppo basso taglia il ragionamento |
| **Token massimi in uscita** | `max_tokens` | la risposta più lunga, ragionamento compreso | vuoto per lo spazio del contesto; impostalo per risposte brevi e prevedibili |
| **Prompt di sistema** | il primo messaggio `system` | istruzioni tenute in testa a ogni conversazione | breve e preciso |
| **Stringhe di stop** | `stop` | la risposta finisce prima di una di queste stringhe | per formati che finiscono con un segnale |
| **Temperatura** | `temperature` | casualità: 0 prende sempre il token più probabile | quella del modello (1,0 per Qwen3.8-Flash-Next); 0,6-0,8 per codice più stabile |
| **Top-p** | `top_p` | sceglie solo fra i token più probabili che insieme arrivano a questa probabilità | quella del modello (0,95) |
| **Top-k** | `top_k` | sceglie solo fra i k token più probabili (1-64) | quella del modello (20) |
| **Min-p** | `min_p` | scarta i token meno probabili di questa frazione del primo | 0-0,05 |
| **Penalità di ripetizione** | `repetition_penalty` | rende meno probabili i token recenti (1 = spenta) | 1-1,1; più alta peggiora codice ed elenchi |
| **Penalità di presenza** | `presence_penalty` | una penalità fissa per ogni token già usato (0-2) | 0; fino a 1,5 se le risposte girano in tondo |
| **Penalità di frequenza** | `frequency_penalty` | una penalità che cresce a ogni uso (0-2) | 0 |
| **Finestra delle penalità** | `penalty_last_n` | quanti token recenti guardano le penalità | 64 (quella del motore) |
| **Seme** | `seed` | lo stesso seme con le stesse impostazioni ripete una risposta | vuoto per risposte varie |
| **Politica del contesto** | (pagina della chat) | cosa fa la chat quando la conversazione supera il contesto: **Ferma al limite**, **Finestra scorrevole** (toglie i turni più vecchi), **Compatta** (alla soglia il modello riassume i turni di mezzo; prompt di sistema, primo messaggio e i due turni più recenti restano parola per parola) | Compatta per le sessioni lunghe; Ferma al limite quando ogni parola deve restare |
| **Compatta al (% del contesto)** | (pagina della chat) | la percentuale del contesto che fa partire la compattazione (30-100%, predefinita 80%) | 80% |
| **Output strutturato** | `response_format` | Testo, oggetto JSON, o schema JSON (con un nome e lo schema) | per i programmi che leggono la risposta |
| **Tentativi dopo una risposta rifiutata** | `structured_retries` | quante volte una risposta JSON non valida si rigenera, con l'errore come spiegazione (0-5, predefinito 2) | 2 |

**Con quale preset si apre un client**: quello usato per ultimo, se esiste ancora; altrimenti segue il predefinito di
adesso (Machine Default, altrimenti i valori del modello). Le impostazioni cambiate a mano restano, e il pannello dice
quale preset è in uso. **Valori del modello** rimette i valori del modello; **Ripristina Machine Default** quelli della
configurazione guidata.

## Calibrazione

Il Loader prevede le velocità di risposta e di lettura del prompt di ogni piano. La previsione parte da una misura di
riferimento e la scala secondo la percentuale di esperti trovati in cache (quanti esperti il piano tiene in VRAM), la
finestra di verifica e i blocchi del prompt. **Calibra** nel Loader (che parte anche da solo al primo caricamento di un
modello) misura la configurazione in uso con tre richieste: una risposta senza ragionamento (512 token), una con
ragionamento (1.024 token) e un prompt lungo (circa 8.000 token). Il risultato va in `calibration\<modello>.json` nella
cartella dei dati, legato al modello e alle impostazioni. Da lì in poi le previsioni di quel modello partono dalla
calibrazione delle stesse identiche impostazioni, altrimenti dalla più recente del modello, invece che dal riferimento
incorporato, e il Loader dice quale ha usato. La misura della configurazione guidata è la stessa calibrazione, ripetuta
per le soglie del drafter e le finestre di verifica che prova. Accanto alle velocità, il file conserva RAM e commit
presi davvero dal caricamento (vedi «Memoria misurata»).

## Lingua

Inglese come predefinito; italiano a richiesta. Una sola scelta vale per l'app, la pagina web, gli script e gli
strumenti da riga di comando, in quest'ordine: `--lang it|en` (script e app, per quell'avvio) > la variabile
d'ambiente `BASALT_LANG` > la scelta salvata in **Impostazioni → Lingua** nella pagina web (o con `python -m serve.lang
set it|en|auto`, conservata in `prefs.json` nella cartella dei dati) > inglese. La lingua di Windows non si usa. Le
pagine Monitor e Hardware e alcuni messaggi tecnici del server sono solo in inglese.

Per avere tutto in italiano una volta per tutte:

```powershell
.venv\Scripts\python -m serve.lang set it
```
