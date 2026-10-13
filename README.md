<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/basalt-lockup-dark.png">
    <img src="assets/basalt-lockup-light.png" alt="Basalt" width="480">
  </picture>
</p>

# Basalt Win+GUI: this fork

*English* · [Italiano](README.it.md)

**Basalt Win+GUI** is a fork of **[Basalt](https://github.com/jesdga95/basalt)** by jesdga95 (MIT license), which is itself
a fork of **[Strata](https://github.com/Niko1221/Strata)** by Niko1221 (MIT license). The engine, the `.basalt` model
format, the converter and the server come from Basalt; the copyright notices of both projects and of the code they
include (ggml, llama.cpp and others) are kept in [LICENSE](LICENSE) and [THIRD_PARTY_LICENSES](THIRD_PARTY_LICENSES).
The model files are derived from Qwen3.8-Flash-Next and are under the
[Qwen Community License 1.0](https://huggingface.co/Qwen/Qwen3.8-Flash-Next/blob/main/LICENSE), not under MIT.

What this fork adds: a native **Windows 10/11** port of the engine (the Linux code paths are unchanged), and the
pieces that make it usable on a desktop: a native app that opens on a web Loader which predicts memory and speed
before loading, a configuration wizard that finds the best values for each PC, load profiles, a model library that
finds, verifies and converts models, and start/stop scripts. The upstream README follows below this section, unchanged.

## How it works

- **One model.** Basalt runs Qwen3.8-Flash-Next (GGUF architecture `qwen4exp`): 48 layers, 512 routed experts per
  layer, 10 of them active per token. At 3 bits the experts alone are about 50 GiB, more than a 32 GB GPU holds.
- **Expert tiers.** The experts live in three places. An **expert cache in VRAM** holds as many as fit (`--expert-cache
  auto`: all the VRAM left after the reserve, see below), filled in the order of an expert profile so the most used ones
  are on the GPU; a **second GPU** can serve as a peer tier (`--gpu 0,1`); the rest are computed by the **CPU**
  (AVX2/AVX-512 kernels). With `--resident-experts` the experts that are not in the VRAM cache are copied into RAM once,
  instead of being read from the file when needed. Routing is skewed, so a cache holding a third of the experts already
  answers about 98% of the lookups.
- **The `.basalt` file.** One file per model: a 64 KiB table of named sections (each with its SHA-256), then the
  weights, tokenizer, chat template, drafter, expert profile and vision projector. It is built from the GGUF shards by
  `tools/convert/convert.py`, which copies the tensors byte for byte (no requantization).
- **The MTP drafter.** The model's own multi-token-prediction layer proposes the next tokens and the main model
  verifies them in one pass; the output is identical to plain decoding because verify decides every token. `spec 7`
  is a verify window of 7 tokens (up to 6 drafts). The ready-made packs carry an MTP layer tuned on the model's own
  output (self-distilled), which raises the acceptance.
- **Prompt lend.** A long prompt is read in large chunks whose buffers are borrowed from the last slots of the expert
  cache for the length of the prompt; those slots are refilled afterwards (`--prefill auto`).

## What this fork changes

- **Native Windows port** (details in [docs/WINDOWS.md](docs/WINDOWS.md)): MSVC + CUDA 13.1 build, a platform layer
  for files, mappings and positional reads, `VirtualLock` and commit-aware memory budgets (Windows charges every
  allocation, VRAM included, to the commit), pinned arenas sized to WDDM's shared-memory budget, unbuffered reads with
  an I/O completion port, the CPU pool's topology from `GetLogicalProcessorInformationEx` with performance cores
  first on hybrid processors, `basalt serve` in a kill-on-close job object. `--no-pdl` launches every kernel without
  programmatic dependent launch, should a driver misbehave.
- **The app, BasaltWinGUI.exe ("Basalt Win+GUI"), is the main interface.** It starts the server without loading a
  model and opens on the web Loader in its own window (WebView2), so the app and the browser have exactly the same
  settings; a tray icon keeps Basalt Win+GUI running when the window is closed.
- **Web UI** (`/`, English or Italian, see [Language](#language)): Chat, Monitor, Hardware, the **Loader** (*Carica
  modello* in Italian) and Settings.
  - The Loader edits every engine setting, shows the VRAM, RAM and commit the plan needs next to what is free now
    (a share of the RAM and of the commit limit always stays free for other programs, 15% unless you choose another;
    the page file must never grow) and the predicted speed; it loads the model, or reloads the engine inside the
    running server. Calibration measures the real speeds per model and setting.
  - The **configuration wizard** finds the best values for this PC and saves them as **Machine Default** (below).
  - **Load profiles** (engine settings, port, model) and **inference presets** (chat settings), shared by the UI, the
    scripts and the app; "Valori sicuri" is a built-in profile **computed from this PC and the chosen model**.
  - Chat with the **model's own sampling values** (from the source GGUF), **context policies** (stop at the limit,
    sliding window, or compaction by the model itself) and **structured output** (`response_format` JSON / JSON
    Schema) validated and generated again with the error fed back (`structured_retries`, default 2).
  - **Model library**: one or more models folders, a header-only scan that marks every model *Ready*,
    *To convert* (a supported GGUF) or *Not compatible* (with the reason), conversion and verification in the
    background, and the default server model.
- **Server**: default port 7232, `--listen-first` (the page answers while the model loads), `--standby` (the page
  answers and nothing is loaded until the Loader's Load), `--relaunch-on-model-change` (a Load of another model ends
  the server with exit code 75 so its owner starts it again with that model), `/api/settings`, `/api/loader/*`,
  `/api/wizard/*`, `/api/profiles`, `/api/models/*`.
- **Scripts** `Avvia-Basalt.ps1` / `Ferma-Basalt.ps1` with the same option names as Strata's: `-gui` (the default:
  the app), `-browser`, `-server` (for agents), `--model`, `--config "<profile>"`, `--port`, `--vram-libera GB`,
  `-riusa` / `-riavvia`, `--lang it|en`.

### Changes since the previous version of this README

- **One set of options** for the scripts and the app (`-gui` default, `-browser`, `-server`, `-prova`, `-converti`,
  `--config`, `--model`, `--port`, `--vram-libera`), the same names as Strata's; the old `-Sfondo` / `-SoloServer`
  still work.
- **The app first**: the app starts the server **in standby** (nothing loaded) and opens on the web Loader; folders are
  chosen with the Windows folder dialog; a native panel is the fallback when the server cannot start. Loading another
  model restarts the server with it (exit code 75) and the page reconnects.
- **Real defaults**: the chat starts from the **model's own sampling values** (from the source GGUF); the load values
  come from the tested rules of "Valori sicuri", computed for this PC and model.
- **Configuration wizard and Machine Default** (load profile + chat preset), with **Restore** buttons, a
  command-line version for agents, and the default precedence: chosen by hand > Machine Default > Valori sicuri.
- **Main vs secondary monitor**: the VRAM reserve depends on whether the NVIDIA GPU drives the main monitor (1.5 GB more),
  a secondary one (about 0.4 GB) or none; what other programs use is shown, not added.
- **Measured memory**: the RAM and commit a real load took are kept per model and added to later plans.
- **Data folders**: settings in `%APPDATA%\BasaltPannello\` (copied once from `~\.cache\basalt`, never moved).
- **Name**: the app is **Basalt Win+GUI** (`BasaltWinGUI.exe`, new icon); the engine, API, files and scripts keep their
  names.
- **`-server` with a server already running** asks whether to reuse it or restart it (`-riusa`, `-riavvia`, 60 s, no
  answer = reuse).
- **English by default**, Italian on request, for the app, the page, the scripts and the command-line tools.
- **Conversion and verification** from the model library (GGUF → `.basalt`, SHA-256 check), in the background.

## Requirements

- An **NVIDIA Blackwell GPU** (compute capability 10.x or 12.x: GeForce RTX 50 series, RTX PRO 6000, B200). The
  engine's kernels are built only for Blackwell; on any other GPU the Loader, the wizard, the start script and the app
  say so before starting, and nothing is loaded.
- **Windows 10 or 11 x64** (WDDM driver) or **Linux x86-64** for the engine, the server and the web UI. The scripts and
  the app are Windows only.
- Enough RAM for the experts that do not fit in VRAM (about 50 GiB for the 3-bit files, with resident experts).
- Python 3.10 or newer with the packages of `requirements.txt`, in a virtual environment `.venv` in the repository
  folder (`python -m venv .venv` then `.venv\Scripts\pip install -r requirements.txt`).
- For the app: the .NET 8 Desktop Runtime and Microsoft Edge WebView2 Runtime (without WebView2 the app opens the
  web UI in the default browser).
- Building: Windows in [docs/WINDOWS.md](docs/WINDOWS.md) (Visual Studio v143 tools, CUDA 13.1, CMake, Ninja), Linux
  in [Build](#build) below. The app: `dotnet publish tools\windows\desktop -c Release -o tools\windows\desktop\publish`.

## Starting Basalt Win+GUI

```powershell
.\Avvia-Basalt.ps1                                             # the app (-gui): the Loader first, nothing loaded yet
.\Avvia-Basalt.ps1 -browser                                    # the server tied to this window, the UI in the browser
.\Avvia-Basalt.ps1 -server --config "Machine Default"          # detached, no UI: returns once the model is loaded
.\Avvia-Basalt.ps1 -server --model "D:\Models\Qwen3.8-Flash-Next-IQ3_XXS.basalt" --port 7240 --vram-libera 2
.\Avvia-Basalt.ps1 -prova                                      # every check, nothing started, nothing written
.\Ferma-Basalt.ps1 --port 7240                                 # stop only that server (its process tree)
```

Every option is accepted with one or two dashes and in any case (`-gui`, `--gui`, `-GUI`).

| option | meaning |
|---|---|
| *(none)* = `-gui` | opens `BasaltWinGUI.exe`, passing it `--config`, `--model`, `--port` and `--vram-libera`; the browser does **not** open, the address is printed and stays available. Without the built app the script says how to build it and uses `-browser` |
| `-browser` | the server tied to this window (Ctrl+C or closing the window stops server and engine, and frees the RAM), the web UI opened in the browser |
| `-server` | the server alone, detached, no app and no browser; the script returns when `/health` says the model is loaded. For agents; `Ferma-Basalt.ps1` stops it |
| `--config "<profile>"` | the load profile; without it, the default profile (below) |
| `--model "<file .basalt>"` | the model to serve, as a full path |
| `--port N` | the port; wins over the profile's (default 7232) |
| `--vram-libera GB` | the VRAM to leave free for other programs, in GB (`1.5` or `"1,5"`), instead of the profile's; `--vram-reserve-mib N` is the same in MiB |
| `-converti` | with `-browser` / `-server`, when the model is a GGUF to convert: show the plan, ask, convert, then serve the new file (in the app: the Loader's **Convert…**) |
| `-riusa` / `-riavvia` | with `-server`, the answer given in advance when a server is already running with other loading parameters: reuse it, or stop it and start again with the new ones (below) |
| `--lang it\|en` | the language of the messages for this run (it also sets `BASALT_LANG` for the server and the app it starts) |
| `-prova` | every check, nothing started; `-browser` / `-server` work on a copy of the config file |
| `-file FILE` | the config file (default `basalt_config.json` beside the script; the app creates one without a model on a new installation) |

`-Sfondo` and `-SoloServer`, the old names, still work as `-server` and print a note. `Ferma-Basalt.ps1` takes
`--port N` and `--file FILE` the same way.

**When a server is already running** (one per installation, on any port):

- `-gui` opens the app, which asks **Connect to the running server** or **Change model** (stop only that server's
  process tree, then choose in the Loader).
- `-browser` opens the browser on it; a `--model` or `--config` that differs is not applied, and the script says so.
- `-server` resolves the loading parameters asked for (`--config`, else the default profile; `--model`;
  `--vram-libera`; `--port`) exactly as for a new start, and compares them with the running server's (the model file,
  every load setting, the port).
  - **The same**: it prints the address, model and profile and returns (exit code 0).
  - **Different**: it lists the differences (`engine.max-context: 65536 -> 131072`) and asks **Reuse the running
    server** or **Restart with the new parameters**: in a window when there is a desktop, else at the console (`R` /
    `S`). With no answer within **60 seconds** (`BASALT_ASK_TIMEOUT_S` changes it), or with nobody to ask (an agent with
    redirected input), the running server is **reused**: a server is never stopped without an answer. Restarting stops
    only that server's process tree (`Ferma-Basalt.ps1 --port N`) and starts it again on the same port unless `--port`
    asks for another.
  - **`-riusa`** and **`-riavvia`** give the answer in advance, for agents and scheduled tasks; with `-prova` the
    script only says what it would do.

**Which model starts** with `-browser` / `-server`: the first of these that is set: `--model`; the model saved in the
load profile; the **default server model** (chosen in the Loader with **Use as the default server model**);
the model already in the config file. When none is set, the script stops with exit code 1 and says how to choose one.
The memory check uses the Loader's own plan; a start that would need the page file to grow is refused, as is a GPU the
engine does not support (exit code 1).

**The default load profile**: the one chosen with **Make default** in the Loader, else **Machine Default** once
the wizard has saved it, else **Valori sicuri**.

Directly, on Windows or Linux: `bin\basalt.exe serve --config basalt_config.json [--port N]` (`bin/basalt serve` on
Linux; every option is in [Settings](#settings---config) below and in `python -m serve.server --help`).

### The app

`tools\windows\desktop\publish\BasaltWinGUI.exe` (window and tray: "Basalt Win+GUI"; the file name differs from the
engine's `basalt.exe` on purpose; the
same options as the script: `--config`, `--model`, `--port`, `--vram-libera`, `--file`). On opening it starts the
server in its own job (it ends with the app) **without loading a model** and shows the web **Loader**:

1. the first time, the models folders (**Browse…** opens the Windows folder dialog; the usual places found on this
   PC are offered);
2. the model, its status, **Verify** and **Convert…**;
3. while this PC has no Machine Default, the **configuration wizard** is offered (it can be skipped, or turned off);
4. the load profile and every value, the VRAM to leave free, the prediction, then **Load**.

Loading another model than the one the server was started with saves it and starts the server again with it; the page
reconnects by itself. Closing the window hides it to the tray icon: **Open**, **Reload with other settings…** (the
Loader), **Start the server alone (for agents)…** (the same as `Avvia-Basalt.ps1 -server`, in place of the app's server),
**Turn off**. If the server cannot start (a port in use, a missing build), the app shows the reason, the end of the log
and a native start panel as a fallback. With the server running, the same UI is at `http://127.0.0.1:7232/` in any
browser.

### Configuration wizard and Machine Default

**Configuration wizard** in the Loader (also `python -m serve.loader wizard` for agents, below) finds the best
values for this PC; the app distributed on other PCs uses it to adapt to each machine. Why: the right values depend
on the VRAM, the RAM, the commit limit, the cores and on which card shows the desktop. Fixed values would either waste
speed on a large PC or fail to load on a small one (or make the page file grow). The wizard computes each value from what
it detects, says why, and can check it with a real measurement.

1. **This PC**: RAM, the commit limit and its use, the page file (Windows-managed files may grow: Basalt Win+GUI
   never makes them grow), the performance and efficiency cores, the NVIDIA GPU, and the video adapters with the one that shows
   the **main monitor**. A PC may drive its monitor from a separate card (an integrated GPU, for example) and keep the
   NVIDIA GPU for compute; the GPU may also have a **secondary** monitor, on or switched off. The wizard shows what it
   detected and asks you to confirm or correct it.
2. **What to leave to the other programs**: the share of RAM and commit (15% proposed, 5 to 50%, never less than 2 GB;
   it becomes this PC's rule for every plan) and the VRAM: 0.5 GB for the driver, plus 1.5 GB when the GPU drives
   the main monitor (or about 0.4 GB for a secondary one), rounded up to 256 MiB. What other programs use on the GPU
   now is shown, not added: the expert cache sizes itself to the VRAM free at the load, which leaves it to them.
3. **The model** and its own chat values.
4. **The computed values**, with their reasons and the prediction: KV cache q4_0, prefill auto, the drafter at spec 7
   and min-p 0.5 when the file has one, resident experts and the PLE table in RAM when they fit, CPU workers from the
   processor, the longest context that fits.
5. **Measure** (optional, about 5-10 minutes, only with less than 75% of the commit limit in use): the values are
   loaded, the real memory after the load is compared with the plan (a shorter context, then no resident experts,
   while it is above), the speed is calibrated, the drafting floor is tried per request (0.3, 0.5, 0.7) and the
   verify windows 5, 7 and 8 (set at the engine's start: one reload each); the fastest is kept (a tie keeps the lower
   window) and every result is shown.
6. **Save**: the load profile **Machine Default** (with the model; it becomes the default) and the chat preset
   **Machine Default** (the model's sampling values). **Restore Machine Default** puts them back: in the Loader next
   to **Restore safe values**, and in the chat settings next to **Model defaults**.

Without questions, for agents (the detected and proposed answers unless given; the measurement needs the Loader):

```powershell
.venv\Scripts\python -m serve.loader wizard --model "D:\Models\Qwen3.8-Flash-Next-IQ3_XXS.basalt" `
    --monitor none --free-share 0.15 --vram-libera 2 --save        # --json for one JSON object
```

**The chat's settings** stay per client: the app and each browser keep their own. A client opens with the preset it
used last, if it still exists; otherwise it follows the current default (Machine Default, else the model's values).
Settings changed by hand are kept, and the panel says which preset is in use.

### Models: folders, conversion, verification

The Loader's model section asks for one or more **models folders**, for example the folder of your `.basalt` files and
LM Studio's download folder. Every `.basalt` and `.gguf` under them is described from its header alone and cached;
nothing reads the weights. Each model gets a status:

- **Ready**: a `.basalt` file the engine can open. **Verify** checks every section's SHA-256 in the background.
- **To convert**: a GGUF set of Qwen3.8-Flash-Next with every shard present and expert formats the GPU kernels
  cover (for example IQ3_XXS, IQ3_S, Q2_0, Q8_0 and the Unsloth UD quants). **Convert…** first shows a dialog with the
  size of the new file, the disk and its free space; it runs only after that confirmation, with enough free space and
  the commit below 75% of its limit. The vision projector (`mmproj`) in the same folder goes into the file; the MTP
  drafter comes from a drafter folder already on disk or is fetched from the Qwen checkpoint (`mtp_fetch.py`). The
  source GGUF's recommended sampling values (`general.sampling.*`) become the file's chat defaults; for files
  converted before, the server reads them from the copy of the source header kept in the file.
- **Not compatible**: anything else, with the reason (another architecture, expert formats without kernels such as
  static Q2_K experts or Q4_0 expert downs, a missing shard, an unfinished download). Projector and drafter files are
  listed as attachments of their model.

The same is available from a terminal: `python -m serve.models list | json | dirs | set-dirs DIR ... | default --set
FILE | resolve [--model FILE] [--profile NAME] | convert --model GGUF [--out DIR]`.

### VRAM to leave free

**VRAM to leave free** (in the Loader, in Settings, in the wizard; `engine.vram-reserve-mib`, saved in profiles) is
the VRAM the engine leaves untouched for the desktop, the browser, the driver and other programs. With the expert
cache on `auto`, everything else holds experts. Moving the control recomputes, live: the VRAM left for experts, how
many experts fit (for example "13,066 of 24,576 experts in VRAM (53%)"), the expected expert-cache hit rate, the
predicted answer and prompt speeds, and the commit the load needs.

- **The trade-off**: more VRAM left free means fewer experts on the GPU (a little slower, since the misses run on the
  CPU) and, on Windows, a smaller commit (VRAM counts toward it). Less VRAM left free is faster until the desktop or
  another program runs short.
- **How to choose**: the wizard's rule above; the page names the programs on the GPU and warns when less than 1 GB
  stays free on a GPU that drives the main monitor. Leave more while you game or edit video on the same GPU.

**Measured memory.** After every real load the server compares the RAM and commit the load really took (the system's
readings before and after) with the plan of the same values, and keeps the difference with the model's calibrations;
from then on the plans of that model add what was above (shown as "measured at the last load").

### "Valori sicuri" (safe values): computed, not fixed

The built-in profile is computed each time from the hardware found and the chosen model, with the same rules as
every plan and the wizard: this PC's share of RAM and commit stays free; the VRAM reserve is the wizard's (or its rule,
with 1 GB for the monitor until the wizard confirms where it is, 1.5 GB in all); the KV cache is q4_0 and the context the longest that
fits (up to 262,144); the drafter is on (spec 7, min-p 0.5) only when the file has one, image input only when it has a
projector; resident experts and the PLE table in RAM only when they fit; the CPU workers are the performance cores plus
half the efficiency cores, less one (on a processor without efficiency cores, the physical cores less one). Each value
comes with its reason in the Loader. As an example, on an RTX 5090 (32 GB, 1 GB used by other programs) with 128 GB of
RAM and a processor with 8 performance and 16 efficiency cores, the 3-bit file gives: 262,144 tokens, q4_0, spec 7 at
min-p 0.5, expert cache auto, resident experts, 1.5 GB left free, 15 CPU workers, image input dynamic.

### Where the settings live

Per user, never in the repository, in the **user data folder**: `%APPDATA%\BasaltPannello\` on Windows,
`~/Library/Application Support/Basalt/` on macOS, `$XDG_DATA_HOME/basalt/` (`~/.local/share/basalt/`) elsewhere, or
`BASALT_DATA`. It holds `profiles.json` (load profiles, inference presets, the default profile), `models.json` (the
models folders, the default server model, the last output folder), `machine.json` (the wizard's answers: the share of
RAM and commit, the VRAM to leave free, where the monitor is), `calibration\<model>.json` (the measured speeds and
memory), `prefs.json` (the language) and the app's `desktop.json`. Caches that can be rebuilt stay in `%USERPROFILE%\.cache\basalt\` (`~/.cache/basalt/`, or
`BASALT_CACHE`): the header scan, the logs of conversion and verification jobs, the tokenizer and vision files
unpacked from a model, the engine log. Versions before this one kept the settings in the cache folder: they are copied
(not moved) to the user data folder the first time. `basalt_config.json` beside the scripts is written by them and by
the Loader from the chosen profile and is not tracked.

## Using it as a server (API)

The server speaks the OpenAI and the Anthropic APIs at `http://127.0.0.1:<port>` (default port **7232**):

| endpoint | API |
|---|---|
| `POST /v1/chat/completions` | OpenAI Chat Completions: messages, tools (function calling), images when vision is on, `response_format`, `stream` |
| `POST /v1/responses` | OpenAI Responses, **stateless**: send the whole conversation each time (`previous_response_id` and `GET /v1/responses/<id>` are not kept) |
| `POST /v1/messages` | Anthropic Messages: tools, thinking blocks, images, `stream` |
| `POST /v1/messages/count_tokens` | Anthropic token count: the same prompt `/v1/messages` would read, not run |
| `GET /v1/models` | the served model (its name, context, state) |
| `GET /health` | `{"service": "basalt", "loaded": true, ...}` once the model is in; `"standby": true` while a server started by the app waits for the Loader |

**Start it for agents** with `.\Avvia-Basalt.ps1 -server` (it returns once the model is loaded; add `-riusa` in
unattended scripts) and stop it with `.\Ferma-Basalt.ps1`. `basalt.pid` holds the PID, port and profile of a server
started by a script or the app.

**Model name.** `/v1/models` reports one model: the `--alias` (or `"alias"` in the config file's `server` section),
else the name stored in the model file. The `model` field of a request may be anything (the server has one model);
the answer carries the served name.

**API key.** The server listens on 127.0.0.1 only and needs no key: clients that insist on one can send any value.
`--api-key KEY` (command line only, never in a config file) makes `/v1/*` require it as `Authorization: Bearer KEY`
or `x-api-key: KEY`; it is needed to listen on another address (`--host 0.0.0.0`) and to let web pages of other
origins use the server. The scripts and the app start the server on 127.0.0.1 without a key.

**Streaming.** `"stream": true` streams server-sent events in each API's own format (OpenAI `chat.completion.chunk`,
Responses events, Anthropic `message_start` / `content_block_delta` / ...), reasoning included as
`reasoning_content` (OpenAI) or `thinking` blocks (Anthropic).

**Reasoning.** The model thinks by default (the template's high level). OpenAI: `"reasoning_effort": "none" | "low" |
"medium" | "high"` (Responses: `"reasoning": {"effort": ...}`). Anthropic: `"thinking": {"type": "disabled"}`, or
`"enabled"` with `budget_tokens` (under 2048: low, under 8192: medium, else high). `reasoning_budget_tokens: N` on
any endpoint closes the thinking after N tokens and lets the model answer.

```sh
curl http://127.0.0.1:7232/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "basalt", "messages": [{"role": "user", "content": "Say hello in three languages."}],
  "max_tokens": 512, "reasoning_effort": "low"}'

curl -N http://127.0.0.1:7232/v1/messages -H "Content-Type: application/json" -H "anthropic-version: 2023-06-01" -d '{
  "model": "basalt", "max_tokens": 1024, "stream": true,
  "messages": [{"role": "user", "content": "Write a haiku about basalt."}]}'

curl http://127.0.0.1:7232/v1/responses -H "Content-Type: application/json" -d '{
  "model": "basalt", "input": "What is 17 * 23?", "reasoning": {"effort": "none"}}'
```

(In PowerShell use `curl.exe` and put the JSON in a file: `-d "@body.json"`.)

Python, with the official SDKs (`pip install openai anthropic`):

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:7232/v1", api_key="none")   # any key, unless --api-key was given
r = client.chat.completions.create(model="basalt", messages=[{"role": "user", "content": "Hello!"}],
                                   max_tokens=512, reasoning_effort="low")
print(r.choices[0].message.content)

for chunk in client.chat.completions.create(model="basalt", stream=True,
                                            messages=[{"role": "user", "content": "Count to five."}]):
    print(chunk.choices[0].delta.content or "", end="", flush=True)

print(client.responses.create(model="basalt", input="Name three volcanic rocks.").output_text)
```

```python
import anthropic

client = anthropic.Anthropic(base_url="http://127.0.0.1:7232", api_key="none")
msg = client.messages.create(model="basalt", max_tokens=1024,
                             messages=[{"role": "user", "content": "Hello!"}])
print(msg.content[-1].text)                      # a thinking block may come first

with client.messages.stream(model="basalt", max_tokens=1024, thinking={"type": "disabled"},
                            messages=[{"role": "user", "content": "Count to five."}]) as s:
    for text in s.text_stream:
        print(text, end="", flush=True)

print(client.messages.count_tokens(model="basalt", messages=[{"role": "user", "content": "Hello!"}]).input_tokens)
```

**Claude Code** talks to the Anthropic endpoint:

```powershell
$env:ANTHROPIC_BASE_URL = "http://127.0.0.1:7232"
$env:ANTHROPIC_AUTH_TOKEN = "none"                 # any value; the key given with --api-key, if any
$env:ANTHROPIC_MODEL = "basalt"                    # any name: the server has one model
$env:ANTHROPIC_SMALL_FAST_MODEL = "basalt"
claude
```

(On Linux/macOS: `export ANTHROPIC_BASE_URL=http://127.0.0.1:7232` and so on.) Agent tools send long prompts:
choose a context of at least 65,536 tokens in the load profile, and keep `--parallel 1` unless several agents work at
once (the context is shared between the slots, see [`--parallel`](#concurrent-requests---parallel-n)).

**Structured output** (`/v1/chat/completions`): `"response_format": {"type": "json_object"}` or `{"type":
"json_schema", "json_schema": {"name": ..., "schema": {...}}}`. The answer is checked; when it is not valid JSON, or
does not match the schema, the request is generated again with the error fed back, up to `structured_retries` times
(0 to 5, default 2; 0 turns it off).

**Busy and errors.** With `--parallel 1` requests wait their turn in arrival order. While the engine is starting, or
while a server started by the app is in standby (no model loaded yet), a request gets a 503 that says why: try again
later, or load a model in the Loader. A prompt longer than the context is refused with a 400 that names the context.

## Loading settings, one by one

Set in the Loader (each with its reason and the live prediction), saved in load profiles, written to the `server` /
`engine` sections of `basalt_config.json`. They take effect when the engine is (re)loaded.

| setting | what it does | how to choose |
|---|---|---|
| **Model** (`server.model`) | the `.basalt` file to serve | from the model library; a GGUF is converted first |
| **Port** (`server.port`) | the HTTP port, default 7232 | change it only to run two servers or when 7232 is taken |
| **Parallel requests** (`server.parallel`, 1-8) | requests served at once in the engine's batch slots | 1 for one user or one agent (fastest per request); more only for several agents at once. The context is shared: each slot gets `max-context / N` |
| **Full context per slot** (`server.slot-full-context`) | every slot gets the whole context instead of a share | costs N times the KV cache, taken from the expert cache: leave off unless each agent really needs the full length |
| **Image input** (`server.vision`) | `off`, `dynamic` (borrows expert-cache slots only while it reads an image: no standing VRAM), `on` (encoder always on the GPU, about 2.2 GiB), `peer` (on a second GPU), `cpu` (no VRAM, slow) | `dynamic` when the file has a projector, else `off` |
| **Context** (`engine.max-context`) | the longest conversation, in tokens (up to 262,144) | the longest that fits ("Valori sicuri" and the wizard pick it); a shorter one leaves more VRAM for experts (a little faster) and needs less commit |
| **KV cache** (`engine.kv`) | the precision of the attention cache: `fp16`, `int8`, `q4_0`, `k8v4` (INT8 keys, Q4_0 values) | `q4_0`: a quarter of fp16's memory, so long contexts fit; `int8` or `k8v4` for slightly more precise attention at more memory |
| **Prompt chunks** (`engine.prefill`) | how long prompts are read: `auto` (the largest chunk the expert cache can lend), `auto:8192/16384/32768` (a ceiling), or a fixed size | `auto` |
| **Verify window** (`engine.spec`, 0-8) | the MTP drafter proposes up to N-1 tokens, the model checks them in one pass; 0 = off | 7 when the file has a drafter (the wizard measures 5, 7 and 8); the output is the same at any value, only the speed changes |
| **Drafting floor** (`engine.spec-min-p`, 0-1) | a draft is proposed only while the drafter is at least this sure of it | 0.5; the wizard tries 0.3, 0.5 and 0.7. Lower drafts more (more wasted work on hard text), higher drafts less |
| **Expert cache** (`engine.expert-cache`) | expert slots kept in VRAM: `auto` (all the VRAM left after the reserve) or a number | `auto` |
| **Resident experts** (`engine.resident-experts`) | the experts not in VRAM are copied into RAM once at load, instead of being read from the file | on when they fit in RAM with the free share kept (they take about 50 GiB for the 3-bit file); off on PCs with less RAM, at a cost in speed |
| **VRAM to leave free** (`engine.vram-reserve-mib`) | VRAM the engine never touches (desktop, driver, other programs) | see [VRAM to leave free](#vram-to-leave-free); the wizard's rule: 0.5 GB + 1.5 GB if the GPU drives the main monitor (about 0.4 GB for a secondary one) |
| **PLE table** (`engine.ple-io`) | the per-layer n-gram table: `direct` (read from the SSD as needed, never in RAM), `ram` (locked in RAM at the start: RAM but no commit), `mmap` (a test arm) | `ram` when it fits with the free share kept, else `direct`; on a slow or rotational disk prefer `ram` |
| **CPU workers** (`engine.pool-workers`) | the threads that compute the experts not in VRAM | the performance cores plus half the efficiency cores, less one (one core stays for the server and the desktop) |

One more rule is per PC, not per profile: the **share of RAM and commit kept free** for other programs (15% unless the
wizard was told otherwise, 5-50%, at least 2 GB). Every plan, "Valori sicuri" and the wizard keep it free, and a load
that would need the page file to grow is refused.

## Chat (inference) settings, one by one

In the chat's settings panel; saved per client, and as named **inference presets** shared through the server (the
**Machine Default** preset is the wizard's). A request carries only the fields that are set, so everything left empty
uses the **model's own values** (from the source GGUF's `general.sampling.*`, shown as placeholders), then the
engine's. API clients send the same fields in their requests.

| setting | request field | what it does | guidance |
|---|---|---|---|
| **Thinking** (beside the message box; effort) | `reasoning_effort` | how much the model reasons before answering: High (default), Medium, Low, Off | High for hard problems, Low/Off for quick replies and helper calls (much faster) |
| **Reasoning budget** | `reasoning_budget_tokens` | after N thinking tokens the thinking is closed and the model answers; empty or 0 = no limit | 2,000-8,000 to bound slow answers; too low cuts the reasoning short |
| **Max output tokens** | `max_tokens` | the longest answer, thinking included | leave empty for the context's room; set it for short, predictable replies |
| **System prompt** | first `system` message | instructions kept at the head of every conversation | short and specific |
| **Stop strings** | `stop` | the answer ends before any of these strings | for formats that end with a marker |
| **Temperature** | `temperature` | randomness: 0 always takes the likeliest token | the model's value (1.0 for Qwen3.8-Flash-Next); 0.6-0.8 for steadier code |
| **Top-p** | `top_p` | samples only from the likeliest tokens that together reach this probability | the model's (0.95) |
| **Top-k** | `top_k` | samples only among the k likeliest tokens (1-64) | the model's (20) |
| **Min-p** | `min_p` | drops tokens less likely than this share of the top one | 0-0.05 |
| **Repetition penalty** | `repetition_penalty` | makes recent tokens less likely (1 = off) | 1-1.1; higher degrades code and lists |
| **Presence penalty** | `presence_penalty` | a flat penalty for any token already used (0-2) | 0; up to 1.5 if answers loop |
| **Frequency penalty** | `frequency_penalty` | a penalty that grows with each use (0-2) | 0 |
| **Penalty window** | `penalty_last_n` | how many recent tokens the penalties look at | 64 (the engine's) |
| **Seed** | `seed` | the same seed with the same settings repeats an answer | empty for varied answers |
| **Context policy** | (chat page) | what the chat does when the conversation outgrows the context: **Stop at the limit**, **Sliding window** (drops the oldest turns), **Compact** (the model summarizes the middle turns at the threshold; the system prompt, the first message and the newest two turns stay verbatim) | Compact for long sessions; Stop when every word must stay |
| **Compact at (% of context)** | (chat page) | the share of the context that starts compaction (30-100%, default 80%) | 80% |
| **Structured output** | `response_format` | Text, JSON object, or JSON schema (with a name and the schema) | for programs that parse the answer |
| **Tries after a rejected answer** | `structured_retries` | how many times an invalid JSON answer is generated again, with the error fed back (0-5, default 2) | 2 |

**Which preset a client opens with**: the one it used last, if it still exists; otherwise it follows the current
default (Machine Default, else the model's values). Settings changed by hand are kept, and the panel says which preset
is in use. **Model defaults** puts the model's own values back; **Restore Machine Default** the wizard's.

## Calibration

The Loader predicts the answer and prompt speeds of every plan. The prediction starts from a reference measurement and
scales it by the expert-cache hit rate (how many experts the plan keeps in VRAM), the verify window and the prompt
chunk. **Calibrate** in the Loader (also run automatically at the first load of a model) measures the running
configuration with three requests: an answer without reasoning (512 tokens), one with reasoning (1,024 tokens) and a
long prompt (about 8,000 tokens). The result goes to `calibration\<model>.json` in the user data folder, keyed by the
model and the settings. From then on the predictions of that model start from the calibration of exactly the same settings, else
from the model's newest one, instead of the built-in reference, and the Loader says which one it used. The wizard's measurement is the same calibration, repeated
for the drafting floors and verify windows it tries. Next to the speeds, the file keeps the RAM and commit the load
really took (see "Measured memory").

## Language

English by default; Italian on request. One choice serves the app, the web page, the scripts and the command-line
tools, in this order: `--lang it|en` (scripts and app, for that run) > the `BASALT_LANG` environment variable > the
choice saved in **Settings → Language** on the web page (or with `python -m serve.lang set it|en|auto`, kept in
`prefs.json` in the user data folder) > English. The language of Windows is not used. The Monitor and Hardware pages
and some technical server messages are in English only.

---

# Upstream Basalt README

The rest of this file is the README of [Basalt](https://github.com/jesdga95/basalt) by jesdga95, as it was when this
fork was made.

Basalt is a local inference engine for the **Qwen3.8-Flash-Next** mixture-of-experts model. It targets one thing:
Linux with an NVIDIA Blackwell GPU (sm_120). The experts that do not fit in VRAM live in system RAM (and optionally on
a second GPU used as a peer expert tier), so a large MoE model runs on a single workstation.
Ready-to-run `.basalt` packs are on [Hugging Face](https://huggingface.co/jesdga/Qwen3.8-Flash-Next-Basalt).

It is a fork of [Strata](https://github.com/Niko1221/Strata) (MIT), stripped down to that one target. See
[THIRD_PARTY_LICENSES](THIRD_PARTY_LICENSES).

<p align="center">
  <img src="docs/img/frontend.gif" alt="The Basalt web UI while a reply streams" width="720">
</p>

<p align="center"><sub>The web UI at <code>/</code> while a reply streams: live decode speed, draft accept rate, context, GPU load and the request log.</sub></p>

## What is in the repository

- `src/`, `include/`: the C++/CUDA engine (one binary, `basalt`).
- `serve/`: a Python server with an OpenAI- and Anthropic-compatible API, and a web UI (chat, monitor, settings).
- `tools/convert/`: the model-file converter (`convert.py`), its section builders and the drafter fetcher (`mtp_fetch.py`).
- `tools/`: the .basalt reader and writer (`model_file.py`), the PPL/KLD quality gate (`ppl_gate.py`), the greedy-identity
  check and the tokenizer.
- `tests/`: parity and unit tests.

## Requirements

- Linux x86-64 with AVX2 (AVX-512 is used when present).
- An NVIDIA Blackwell GPU (compute capability 12.0) and CUDA 13.
- CMake 3.24 or newer and a C++20 compiler.
- Enough system RAM to hold the experts that do not fit in VRAM.
- Python 3.10 or newer for the server: `pip install -r requirements.txt`.

## Build

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j        # bin/basalt, bin/basalt-bench, bin/basalt-vision
```

### Tests

The tests are off by default. Configure with `-DBUILD_TESTING=ON`, build, then run `ctest`:

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=ON
cmake --build build -j
ctest --test-dir build
```

The kernel parity tests need the GPU; the file-format, cache and memory unit tests do not. `expert_parity` and `pool_test`
need local expert data (`experts.bin` in the source directory, not shipped), and `platform_memory_test` needs a locked-memory
limit raised first (`ulimit -l`); without that, those three fail.

`basalt-bench` reads its pre-tokenized prompts from `data/bench` (token ids for the Qwen3.8-Flash-Next tokenizer: the
three decode prompts and a prefill filler); run it from the repository root or pass `--fixtures DIR`. `basalt-bench
--help` says how the files are made.

The build needs nothing outside this repository and fetches nothing. The parts of ggml and llama.cpp that Basalt uses
(quantization formats, CPU dot products, the prompt path's MMQ kernels) are extracted into `src/` and `include/basalt/ggml/`,
with their MIT licence in `THIRD_PARTY_LICENSES`.

## Run

```sh
bin/basalt serve --model /path/to/model.basalt --host 127.0.0.1 --port 7232 [--alias NAME]
```

The server listens on 127.0.0.1 by default; do not expose it beyond that without `--api-key`.

Its web UI is at http://127.0.0.1:7232/: **Chat** streams from the server's own `/v1/chat/completions` (the conversation stays in the browser), **Monitor** and **Hardware** show the live statistics, and **Settings** edits the main fields of the `--config` file (port, parallel, vision, max-context, kv, spec, spec-min-p, expert-cache, prefill) through `GET`/`POST /api/settings`. Values are checked with the server's and engine's own rules, every other key in the file is kept, and the old file is saved as `FILE.bak`. Saving needs the API key when one is set and works only from the PC the server runs on; the changes apply when the server is started again.

### Settings: `--config`

The engine's built-in defaults are untuned (4096 context, fp16 KV, no speculation, no expert cache), and the model
file carries no engine flags. Put every setting in one `basalt_config.json` and pass `--config`:

```json
{
  "server": {
    "model": "model.basalt",
    "alias": "Qwen 3.8 FN",
    "gpu": "0,1",
    "host": "127.0.0.1",
    "port": 7232
  },
  "engine": {
    "expert-cache": "auto",
    "prefill": "auto",
    "spec": 7,
    "spec-min-p": 0.5,
    "max-context": 262144,
    "kv": "int8",
    "resident-experts": true,
    "vram-reserve-mib": 700
  }
}
```

```sh
bin/basalt serve --config basalt_config.json                     # everything from the file
bin/basalt serve --config basalt_config.json --max-context 32768  # the command line wins
```

### Concurrent requests: `--parallel N`

`--parallel N` (1..8, also `"parallel"` in the config's `server` section) serves N requests at once in the
engine's batch slots; the default 1 is the single-stream path. Further requests wait in arrival order (a later one
may go first only to resume a slot that holds its conversation, a few times at most). A slot is freed when its
request finishes, is cancelled or its client leaves. `/metrics` reports `live.slots` (`total`, `active`,
`queue_depth`). As in llama.cpp, `--max-context` is shared: each slot holds `--max-context / N` tokens (`--parallel 4
--max-context 262144` gives four slots of 65536; ask for `--max-context 1048576` for four of 262144). The solo
path, used while one request runs alone, has the same length, so a request that fits a slot never outgrows it, and
one that does not is refused with a 400 naming that length. `/slots`, `/props` and `/v1/models` report the context
each slot has. `--slot-full-context` (also `"slot-full-context"` in the config's `server` section) gives every slot
the whole `--max-context` instead, at N times the KV cache; the startup line says so. The slots' KV cache, and one
solo session of the same length, come out of the expert cache.

While other slots are decoding, a new prompt is read beside them: the prompt path runs on its own thread, the slots'
decode windows keep running (their kernels take the higher stream priority), and the few steps that touch the draft
layer or the output run between two windows. A long prompt therefore does not stop the other requests. A prompt that
arrives while no other slot is decoding is read as in the single-stream path.

At startup the server prints one `[server] kv cache:` line per device with all the session state placed there, for
example `12.5 GiB on cuda2 (solo 262144 tokens + 2 slots x 262144 tokens + draft K/V 0.5 GiB)`. Because that memory is
taken from the expert cache, the engine warns (`basalt warning: the N batch slots' sessions ... take X GiB of CUDA0
...`) when it leaves the cache short of the expert set, or at under half of what it would hold without the slots, and
names the remedies: a smaller `--max-context`, no `--slot-full-context`, fewer `--parallel` slots. If a slot's session
does not fit at all, the engine says which slot (`WARNING: --batch N: slot K's session does not fit`) and starts with
the slots that did fit; if fewer than two fit, `--batch is off` and the server runs one request at a time.

- `server` holds the server's options (`python -m serve.server --help`), named as the flags without the dashes:
  strings or numbers, `true`/`false` for a switch (`lazy`, ...), a list for the repeatable ones
  (`allowed-host`, `cors-origin`). `model` and `tokenizer` paths are relative to the file.
- `engine` holds engine flags (`bin/basalt --help`), the same way: a string or number, `true` for a switch, `false`
  to leave it off. Any flag the engine accepts works, on the command line as well (`--kv int8`, `--resident-experts`).
  A name the engine does not accept stops the server at startup, as does one in the wrong section.
- Precedence: command line > config file > built-in defaults. `spec`, `spec-min-p`, `mtp-max-t`, `peer-exclusive` and
  `max-context` are engine flags; they are also server flags on the command line.
- The API key is command line only (`--api-key`); a config file that sets it is refused.
- The startup line `[server] engine: ...` lists the settings that came from the file or the command line; without
  `--config` it says the built-in defaults apply.

### Prompt chunks: `--prefill`

A prompt is read in chunks. `--prefill auto` (the usual setting) takes the largest chunk whose buffers the expert cache
can lend: the buffers are carved from the cache's last slots for the length of the prompt and the experts in those slots
are copied back afterwards, so a short prompt lends only what it needs. A bare `auto` reads a prompt in the fewest equal chunks of at most 12288 tokens (16384 with `--peer-exclusive`), because
a chunk's cost per token stops falling past a few thousand tokens; it is still limited to what the lend can fund, which with
`--resident-experts` is the lent slots whose experts the RAM copy holds plus a fixed amount of file reading, and without
`--resident-experts` is 8192. `--prefill auto:N` (N = 8192, 16384 or 32768) sets the ceiling yourself; a number sets a
fixed chunk. A prompt longer than one chunk is split into equal chunks (a short tail chunk costs nearly a whole pass over
the streamed experts). The startup line `prompt chunk auto: N tokens` says what was
chosen.

### Vision: `--vision`

Image input is one server option, `--vision off | on | peer | dynamic | cpu` (config: `"vision": "peer"` in the
`server` section). It is `off` by default. The encoder is Basalt's own and shares its kernels in every mode, so all
five give bitwise identical embeddings.

| mode | where the encoder runs | VRAM cost | fresh-image latency (IQ3_S, 400 W) |
| --- | --- | --- | --- |
| `off` | no image input | none | |
| `on` | sidecar `bin/basalt-vision` on the main GPU, always loaded | about 2.2 GiB of the main card's expert cache | TTFT 0.77 s at 1k image tokens, 1.4 s at 3k, 1.8 s at 4k |
| `peer` | the same sidecar on the second `--gpu` card | about 1.8 GiB of the peer card's cache, none on the main card | 0.86 s, 2.1 s, 2.9 s |
| `dynamic` | inside the engine, on the main GPU | none at rest (0.87 GiB of pinned RAM) | 0.77 s, 1.45 s, 1.8 s |
| `cpu` | sidecar on the CPU | none | about 6 s at 1k, 40 s at 3k, 70 s at 4k |

`dynamic` borrows up to 1.55 GiB of expert-cache slots per image (the prompt path's lend), uploads the weights from the
pinned copy (about 30 ms), encodes on a side stream, and refills the same slots from RAM (about 60 ms), so the cache is
identical afterwards. Other `--parallel` slots pause for the encode. An image already seen is served from the
encoder cache in tens of milliseconds in every mode. `peer` on a single GPU warns and loads the encoder on that GPU;
with two cards, `on` says so at startup and names `peer` as the way to put it on the second card.

```json
{ "server": { "gpu": "0,1", "vision": "peer" } }
```

`on` and `peer` use the sidecar, which the main build also produces (`tools/vision` also builds on its own); the
server looks for it next to `bin/basalt`. `dynamic` needs no sidecar.

The image encoder is Basalt's own, written for this model's vision tower (no llama.cpp or ggml): bf16 tensor-core
products and flash attention with fp32 accumulation on the GPU, AVX-512/AVX2 on the CPU. It replaces llama.cpp's
mtmd encoder as a drop-in, is faster than it at every image size, and stays closer to an fp32 reference: mtmd's fp16
attention drifts on large images, ours does not.

## The `.basalt` model file

A model is one file. It holds everything the engine, the server and the vision sidecar load: the weights, the
tokenizer and chat template, the MTP drafter, the expert profile, the 8-bit hyper-connection copies, the vision
projector, and the default sampling. It carries no engine flags: those come from `--config` or the command line.

- **Layout:** a 64 KiB header (magic `BASALTMF`, version, a table of up to 511 named sections, each with its offset,
  size and SHA-256), then the sections, each aligned to 2 MiB so it can be memory-mapped or read with `O_DIRECT`.
- **Sections:** `meta` (JSON: geometry, the source GGUF metadata, every tensor's type, shape and offset,
  sampling, provenance and license), `tokenizer/*`, `dense`, `experts` and `experts.layout` (the routed
  experts, laid out per layer), `gguf` (every non-expert source tensor), `mtp/*` (the drafter), `profile`, `hc8` and
  `vision/*`. The authoritative description is [include/basalt/core/model_format.hpp](include/basalt/core/model_format.hpp).
- **No requantization:** tensors are copied byte for byte from the source GGUF. The only exception is
  `--compat-bf16`, for GGUFs that store the small projections the engine reads as BF16 in another type; it converts
  them and records that in `meta`.

Build one from the source GGUF shards (give shard 1; the others are found beside it):

```sh
python3 tools/convert/convert.py --gguf MODEL-00001-of-0000N.gguf --mmproj MMPROJ.gguf --out MODEL.basalt \
    [--mtp-source DIR [--mtp-dense FILE]] [--draft-vocab FILE] [--profile FILE] [--hc8 FILE] [--compat-bf16] [--name NAME]
```

`python3 tools/convert/convert.py --help` describes every input.

## Models

Quality measured with `basalt-bench --quality`: 20 chunks of wikitext-2 (512 tokens each), scored through the decode
path against the Q8_0 reference. Lower KLD and higher top-1 agreement are closer to Q8_0. How precision is handled and how quality is measured:
[docs/precision.md](docs/precision.md).

| model | source | file | experts | PPL | KLD mean | same top-1 |
|---|---|---:|---:|---:|---:|---:|
| GSQ-RCO Q2_0 | [ISTA-DASLab](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF) | 64.9 GiB | 31.6 GiB | 2.971 | 0.486 | 80.6% |
| GSQ-RCO IQ3_XXS | [ISTA-DASLab](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF) | 74.4 GiB | 40.0 GiB | 2.535 | 0.277 | 86.8% |
| GSQ-RCO IQ3_S | [ISTA-DASLab](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF) | 81.0 GiB | 46.8 GiB | 2.387 | 0.177 | 89.4% |
| UD-Q2_K_XL | [Unsloth](https://huggingface.co/unsloth/Qwen3.8-Flash-Next-GGUF) | 76.5 GiB | 42.9 GiB | 2.755 | 0.323 | 84.3% |
| UD-IQ3_XXS | [Unsloth](https://huggingface.co/unsloth/Qwen3.8-Flash-Next-GGUF) | 79.4 GiB | 45.3 GiB | 2.564 | 0.244 | 87.0% |
| UD-Q3_K_XL | [Unsloth](https://huggingface.co/unsloth/Qwen3.8-Flash-Next-GGUF) | 86.8 GiB | 52.0 GiB | 2.457 | 0.160 | 89.3% |
| UD-Q4_K_XL | [Unsloth](https://huggingface.co/unsloth/Qwen3.8-Flash-Next-GGUF) | 106.7 GiB | 71.7 GiB | 2.321 | 0.068 | 93.0% |
| Q8_0 | [Unsloth](https://huggingface.co/unsloth/Qwen3.8-Flash-Next-GGUF) | 178.3 GiB | 119.5 GiB | 2.282 | reference | |

The IQ3_S, IQ3_XXS and Q2_0 packs carry an MTP layer fine-tuned on the model's own output (self-distillation), which
raises draft acceptance. Output is unchanged, because verify decides every token. Q4_K_XL and Q8_0 keep the original
MTP layer. `--mtp-dense FILE` embeds that tuned layer (the `mtp/dense` section's raw bytes, checked against the
source layout) and records it as self-distilled, with its SHA-256, in `meta.mtp`. The tuned layer ships with the
packs, inside the model file; the converter input is only needed to build a pack yourself.

The model files are derived from Qwen3.8-Flash-Next and are distributed under the
[Qwen Community License 1.0](https://huggingface.co/Qwen/Qwen3.8-Flash-Next/blob/main/LICENSE), not Basalt's MIT
license. The GSQ-RCO quants come from IST Austria's Deep Algorithms and Systems Lab; please cite the
[GSQ](https://arxiv.org/abs/2604.18556) and [RCO](https://arxiv.org/abs/2605.00649) papers if you use them.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT, see [LICENSE](LICENSE).
