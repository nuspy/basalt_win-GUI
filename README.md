<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/basalt-lockup-dark.png">
    <img src="assets/basalt-lockup-light.png" alt="Basalt" width="480">
  </picture>
</p>

# Basalt for Windows: this fork

This repository is a fork of **[Basalt](https://github.com/jesdga95/basalt)** by jesdga95 (MIT license), which is itself
a fork of **[Strata](https://github.com/Niko1221/Strata)** by Niko1221 (MIT license). The engine, the `.basalt` model
format, the converter and the server come from Basalt; the copyright notices of both projects and of the code they
include (ggml, llama.cpp and others) are kept in [LICENSE](LICENSE) and [THIRD_PARTY_LICENSES](THIRD_PARTY_LICENSES).
The model files are derived from Qwen3.8-Flash-Next and are under the
[Qwen Community License 1.0](https://huggingface.co/Qwen/Qwen3.8-Flash-Next/blob/main/LICENSE), not under MIT.

What this fork adds: a native **Windows 10/11** port of the engine (the Linux code paths are unchanged), and the
pieces that make it usable on a desktop: a web UI with a Loader that predicts memory and speed before loading, load
profiles, a model library that finds, verifies and converts models, start/stop scripts, and a small native app with a
tray icon. The upstream README follows below this section, unchanged.

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
- **Web UI** (`/`, Italian or English): Chat, Monitor, Hardware, **Carica modello** (the Loader) and Settings.
  - The Loader edits every engine setting, shows the VRAM, RAM and commit the plan needs next to what is free now
    (15% of the RAM and 15% of the commit limit always stay free; the page file must never grow) and the predicted
    speed; it reloads the engine inside the running server. Calibration measures the real speeds per model and setting.
  - **Load profiles** (engine settings, port, model) and **inference presets** (chat settings), shared by the UI, the
    scripts and the app; "Valori sicuri" is a built-in profile **computed from this PC and the chosen model**.
  - Chat with **context policies** (stop at the limit, sliding window, or compaction by the model itself) and
    **structured output** (`response_format` JSON / JSON Schema) validated and generated again with the error fed back
    (`structured_retries`, default 2).
  - **Model library**: one or more models folders, a header-only scan that marks every model *Pronto* (ready),
    *Da convertire* (a supported GGUF) or *Non compatibile* (with the reason), conversion and verification in the
    background, and the default server model.
- **Server**: default port 7232, `--listen-first` (the page answers while the model loads), `/api/settings`,
  `/api/loader/*`, `/api/profiles`, `/api/models/*`.
- **Scripts** `Avvia-Basalt.ps1` / `Ferma-Basalt.ps1` with `--model`, `--config "<profile>"`, `--port`,
  `-SoloServer` for agents, attach to a running server instead of starting a second one.
- **BasaltPannello.exe**: a native start panel (model, profile, VRAM to leave free, memory check), the web UI in a
  WebView2 window and a tray icon.

## Requirements

- An **NVIDIA Blackwell GPU** (compute capability 10.x or 12.x: GeForce RTX 50 series, RTX PRO 6000, B200). The
  engine's kernels are built only for Blackwell; on any other GPU the Loader, the start script and the app say so
  before starting, and nothing is loaded.
- **Windows 10 or 11 x64** (WDDM driver) or **Linux x86-64** for the engine, the server and the web UI. The scripts and
  the app are Windows only.
- Enough RAM for the experts that do not fit in VRAM (about 50 GiB for the 3-bit files, with resident experts).
- Python 3.10 or newer with the packages of `requirements.txt`, in a virtual environment `.venv` in the repository
  folder (`python -m venv .venv` then `.venv\Scripts\pip install -r requirements.txt`).
- For the app: the .NET 8 Desktop Runtime and Microsoft Edge WebView2 Runtime (without WebView2 the app opens the
  web UI in the default browser).
- Building: Windows in [docs/WINDOWS.md](docs/WINDOWS.md) (Visual Studio v143 tools, CUDA 13.1, CMake, Ninja), Linux
  in [Build](#build) below. The app: `dotnet publish tools\windows\desktop -c Release -o tools\windows\desktop\publish`.

## Starting Basalt

### As a server (scripts, agents)

```powershell
.\Avvia-Basalt.ps1                                             # default profile, the model chosen as below
.\Avvia-Basalt.ps1 --model "D:\Models\Qwen3.8-Flash-Next-IQ3_XXS.basalt" --port 7240
.\Avvia-Basalt.ps1 --config "Valori sicuri" -SoloServer       # detached, no browser: for agents
.\Avvia-Basalt.ps1 -Prova                                      # every check, nothing started, nothing written
.\Ferma-Basalt.ps1 --port 7240                                 # stop only that server (its process tree)
```

| option | meaning |
|---|---|
| `--model "<file .basalt>"` (`-Model`) | the model to serve, as a full path |
| `--config "<profile>"` (`-Config`) | the load profile; without it, the default profile chosen in the Loader |
| `--port N` (`-Port`) | the port; wins over the profile's (default 7232) |
| `--vram-reserve-mib N` | the VRAM to leave free instead of the profile's |
| `-Converti` (`--converti`) | when the model is a GGUF to convert: show the plan, ask, convert, then serve the new file |
| `-Sfondo`, `-SoloServer` (`--solo-server`) | detached (PID in `basalt.pid`); `-SoloServer` also opens no browser and returns when `/health` answers |
| `-Prova` | dry run on a copy of the config file |
| `-File FILE` | the config file (default `basalt_config.json` beside the script, created from the profile) |

**Which model starts.** The first of these that is set: `--model`; the model saved in the load profile
(`--config`); the **default server model** (chosen in the Loader or the app with "Usa come modello predefinito per
il server", kept per user); the model already in the config file. When none is set, the script stops with exit code 1
and says how to choose one. A GGUF that is *Da convertire* is not started: the script says how to convert it, or
converts it with `-Converti` after a confirmation that states the size of the new file and the disk it goes to. A
file that is *Non compatibile* is refused with the reason.

**One server per installation.** If a Basalt server of this folder is already running, on any port, the script
starts nothing: it prints that server's address, model and profile and exits 0. A `--model` that differs from the
model it serves is not loaded, and the script says so (stop it with `Ferma-Basalt.ps1 --port N` first). The memory
check uses the Loader's own plan; a start that would need the page file to grow is refused, as is a GPU the engine
does not support (exit code 1).

Directly, on Windows or Linux: `bin\basalt.exe serve --config basalt_config.json [--port N]` (`bin/basalt serve` on
Linux; every option is in [Settings](#settings---config) below and in `python -m serve.server --help`).

For agents: the API is OpenAI-compatible at `http://127.0.0.1:<port>/v1` (`/v1/chat/completions`, `/v1/messages`,
`/v1/responses`). `GET /health` answers `{"service": "basalt", "loaded": true, ...}` once the model is in;
`basalt.pid` holds the PID, port and profile of a detached server.

### With the app or the browser

`tools\windows\desktop\publish\BasaltPannello.exe` (the name differs from the engine's `basalt.exe` on purpose). The
first time, it asks for the models folders (a folder picker, with the usual places found on this PC offered). Its
start panel then has the **model** (the one used last time preselected, with its status), the load profile, the
**VRAM to leave free**, the port and the memory check; **Avvia** starts the server in the app's own job (it ends with
the app) and shows the web UI, **Avvia solo server** starts a detached server for agents. Closing the window hides
it to the tray icon (Apri, Ricarica con altri parametri, Spegni). When a server is already running the app offers
**Collegati** (use it) or **Cambia modello** (stop only that server's process tree and load the chosen model). With
the server running, the same UI is at `http://127.0.0.1:7232/` in any browser.

### Models: folders, conversion, verification

The Loader's model section (and the app's first run) asks for one or more **models folders**, for example the folder
of your `.basalt` files and LM Studio's download folder. Every `.basalt` and `.gguf` under them is described from its
header alone and cached; nothing reads the weights. Each model gets a status:

- **Pronto**: a `.basalt` file the engine can open. **Verifica** checks every section's SHA-256 in the background.
- **Da convertire**: a GGUF set of Qwen3.8-Flash-Next with every shard present and expert formats the GPU kernels
  cover (for example IQ3_XXS, IQ3_S, Q2_0, Q8_0 and the Unsloth UD quants). **Converti** first shows a dialog with the
  size of the new file, the disk and its free space; it runs only after that confirmation, with enough free space and
  the commit below 75% of its limit. The vision projector (`mmproj`) in the same folder goes into the file; the MTP
  drafter comes from a drafter folder already on disk or is fetched from the Qwen checkpoint (`mtp_fetch.py`).
- **Non compatibile**: anything else, with the reason (another architecture, expert formats without kernels such as
  static Q2_K experts or Q4_0 expert downs, a missing shard, an unfinished download). Projector and drafter files are
  listed as attachments of their model.

The same is available from a terminal: `python -m serve.models list | json | dirs | set-dirs DIR ... | default --set
FILE | resolve [--model FILE] [--profile NAME] | convert --model GGUF [--out DIR]`.

### VRAM to leave free

"VRAM da lasciare libera" (in the Loader, in Settings and in the app; `engine.vram-reserve-mib`, saved in profiles)
is the VRAM the engine leaves untouched for the desktop, the browser, the driver and other programs. With the expert
cache on `auto`, everything else holds experts. Moving the control recomputes, live: the VRAM left for experts, how
many experts fit (for example "13.066 di 24.576 esperti in VRAM (53%)"), the expected expert-cache hit rate, the
predicted answer and prompt speeds, and the commit the load needs.

- **The trade-off**: more VRAM left free means fewer experts on the GPU (a little slower, since the misses run on the
  CPU) and, on Windows, a smaller commit (VRAM counts toward it). Less VRAM left free is faster until the desktop or
  another program runs short.
- **How to choose**: 1 GB is the least for the desktop and the driver, 2 GB with other programs on the GPU (the page
  names them). The computed safe value is 3/16 of the card, between 1 and 8 GiB (6 GiB on a 32 GB card).
  Leave more while you game or edit video on the same GPU.

### "Valori sicuri": computed, not fixed

The built-in profile is computed each time from the hardware found and the chosen model, with the same rules as
every plan: 15% of the RAM and 15% of the commit limit stay free; the VRAM reserve is 3/16 of the card; the KV cache
is q4_0 and the context the longest that fits (up to 262,144); the drafter is on (spec 7) only when the file has one,
image input only when it has a projector; resident experts and the PLE table in RAM only when they fit; the CPU
workers are the performance cores plus half the efficiency cores, less one (on a processor without efficiency cores,
the physical cores less one). Each value comes with its reason in the Loader. As an example, on an RTX 5090 (32 GB)
with 128 GB of RAM and a processor with 8 performance and 16 efficiency cores, the 3-bit file gives: 262,144 tokens,
q4_0, spec 7 at min-p 0.5, expert cache auto, resident experts, 6 GiB left free, 15 CPU workers, image input dynamic.

### Where the settings live

Per user, never in the repository: `%USERPROFILE%\.cache\basalt\` on Windows and `~/.cache/basalt/` on Linux (or
`BASALT_CACHE`) hold `profiles.json` (load profiles, inference presets, the default profile), `models.json` (the
models folders, the default server model, the last output folder), `models-scan.json` (the header cache), the
per-model calibrations and the logs of conversion and verification jobs. The app keeps the last model and its own
state in `%APPDATA%\BasaltPannello\desktop.json`. `basalt_config.json` beside the scripts is written by them from the
chosen profile and is not tracked.

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
