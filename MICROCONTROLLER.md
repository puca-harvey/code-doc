# Embedded / PlatformIO workflow

Guideline for firmware work in this repo: PlatformIO projects, building, flashing and debugging, plus which
model to use for which job.

> PlatformIO **Core 6.2.0** is installed under `%USERPROFILE%\.platformio\`, and the VS Code extension is
> `platformio.platformio-ide` 3.3.4. Platforms already fetched: `atmelavr`, `espressif32`.

## Contents

- [Running PlatformIO](#running-platformio)
- [Project layout](#project-layout)
- [platformio.ini](#platformioini)
- [Build, flash, monitor](#build-flash-monitor)
- [Debugging](#debugging)
- [Library dependencies](#library-dependencies)
- [Which model for which task](#which-model-for-which-task)
- [Rules files: teaching the models your board](#rules-files-teaching-the-models-your-board)
- [Gotchas](#gotchas)

## Running PlatformIO

There are two ways to drive PlatformIO, and both work out of the box — pick whichever suits the moment.

**From the IDE — no setup at all.** The PlatformIO task bar at the bottom of VS Code (→ build, ✓ upload, 🔌 monitor)
and the *PlatformIO: …* entries in the Command Palette drive the same Core install the extension manages itself.
This is the path to use for normal work.

**From a terminal — one line per shell.** `pio` lives at a fixed, known location:

```text
%USERPROFILE%\.platformio\penv\Scripts\pio.exe
```

Core is installed as a self-contained Python virtual environment, so that folder is not on `PATH` and `Get-Command
pio` will not find it. Nothing is missing — call it by full path, or add the folder for the current shell and use
the short name from then on:

```powershell
# call it by full path
& "$env:USERPROFILE\.platformio\penv\Scripts\pio.exe" --version
```

```powershell
# or add the folder to this shell, then use the short name
$env:Path += ";$env:USERPROFILE\.platformio\penv\Scripts"
pio --version    # PlatformIO Core, version 6.2.0
```

The `$env:Path +=` form lasts until the shell closes. Add the same folder under *Environment Variables → Path* to
make it permanent across new terminals.

> The first build in a fresh project downloads the platform toolchain (100s of MB). `atmelavr` and `espressif32`
> are already fetched; any other platform costs that one-time download.

## Project layout

PlatformIO expects this structure — do not relocate `src/`:

```text
my-project/
├─ platformio.ini      # the only file you hand-write
├─ src/                # firmware sources (.c/.cpp/.ino/.py)
├─ lib/                # private libraries, one folder each
├─ include/            # project-wide headers
├─ test/               # unit tests (Unity + AUnit)
└─ .pio/               # build output — never commit this
```

`src/` is mandatory. PlatformIO compiles only what it finds there plus `lib/`; code outside is ignored
unless declared under `build_src_filter`.

## platformio.ini

```ini
[platformio]
default_envs = esp32dev, uno

[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
monitor_speed = 115200
monitor_filters = esp32_exception_decoder, time      ; readable ESP32 stack traces
upload_speed = 921600
lib_deps =
    bblanchon/ArduinoJson@^7.0.4
    adafruit/Adafruit GFX Library@^1.11.0
build_flags =
    -DCORE_DEBUG_LEVEL=3
    -Wall
build_unflags = -std=gnu++11
build_type = release

[env:uno]
platform = atmelavr
board = uno
framework = arduino
monitor_speed = 9600
upload_speed = 115200

[env:nodemcuv2]        ; ESP8266 — needs its own platform, not installed yet
platform = espressif8266
board = nodemcuv2
framework = arduino
```

| Key | Why it matters |
| --- | --- |
| `default_envs` | Build all of these on one `pio run`; the IDE status bar and Cline both honour it |
| `monitor_speed` | Wrong value = garbage serial output. 9600 for ATmega328p, 115200 for ESP32 |
| `lib_deps` | Pinned with `@^x.y.z`. **Never** rely on a floating version in firmware |
| `build_flags` | `-Wall` and `CORE_DEBUG_LEVEL=3` catch most ESP32 runtime bugs early |
| `build_type` | `release` compiles smaller/faster than the default `debug`; switch locally for debugging |

Two envs, one config file — you can flash an ESP32 and an Uno from the same repo.

## Build, flash, monitor

From a terminal, with `pio` reachable as described in [Running PlatformIO](#running-platformio):

```powershell
pio run                       # build every env in default_envs
pio run -e esp32dev           # build one env
pio run -t upload             # build + flash
pio run -t upload -e esp32dev
pio run -t clean              # drop build output
pio run -t fullclean          # also drop .pio entirely
pio device monitor            # serial monitor
pio device list               # find COM/tty ports
```

From the IDE, use the PlatformIO **task bar** at the bottom (→ / ✓ / 🔌) or `Ctrl`+`Shift`+`P` →
*PlatformIO: …*. Every task is a Cline tool as well, so the agent can build and flash for you.

> ⚠️ Firmware is **hardware‑adjacent**. Let Cline run `pio run`, but require explicit approval for
> `pio run -t upload` and anything touching `pio device monitor`. See *Rules files* below.

## Debugging

| Situation | Approach |
| --- | --- |
| ESP32 | JTAG via `debug_tool = esp-prog` / `esp-bridge`; set `build_type = debug` for `-Og -g3` |
| AVR / Uno | Atmel-ICE or AVR Dragon over ISP; `-Og` only, AVR has 1 KB usable RAM for variables |
| Hard fault | `monitor_filters = esp32_exception_decoder` prints the register dump and decoded backtrace |
| Brownout | ESP32 brownout detector usually means a 5 V pin fed 3.3 V, or an undersupplied rail |

For the simplest cases, `Serial.printf` + the monitor beats a debugger. Reach for JTAG when you need to
inspect a struct or watch a variable change across an ISR.

## Library dependencies

```powershell
pio pkg install                 # install everything in lib_deps
pio pkg update                  # bump within the pinned ranges
pio pkg search adafruit         # find a library
pio pkg list                    # what is installed, with versions
```


## Which model for which task

The rule is the same as for desktop work, but firmware has an extra constraint: **the model must not
invent API calls.** Everything must compile against the real library on disk.

| Task | Model | Why |
| --- | --- | --- |
| Plan a task split, choose libraries | `ornith-1.5:9b` (local) | Fast; planning is prose, not code |
| Write a driver or fix a compile error | `ornith-1.5:9b` (local) | Holds `edit`/`apply`; ~1.6× faster than the code model |
| Explain a register or a crash dump | `ornith-1.5:9b` (local) | Explanation, not generation |
| Inline edit of one line while typing | `qwen2.5-coder:14b` (local) | The only FIM model; `Tab` completion |
| Long multi-file refactor | ClinePass — `cline-pass/glm-5.3` | Local 9B loses coherence across a big diff |
| Large-context reading (datasheets, header trees) | ClinePass — `cline-pass/qwen3.7-plus` | >256 K context tier; local 8 K is far too small |
| When the quota is gone | `cline-free/*` | See [`README.md`](README.md) for the full model table |

Practical split: **local for day-to-day edits, ClinePass when a task outgrows 8 K of context or spans many
files.** Local models are free and private, which matters when you paste proprietary driver code.

## Rules files: teaching the models your board

Continue reads `.continue/rules/` (or `.clinerules/` for Cline) as plain markdown that is always in context.
This is the highest-leverage thing you can add — it fixes the "hallucinated API" failure mode.

```text
.clinerules
──────────
# Firmware rules

## Board
- Target: ESP32 Dev Module (esp32dev), framework: Arduino, PlatformIO Core 6.2.0.
- Second env: Arduino Uno (atmelaVR) — keep sources portable to AVR: no
  dynamic allocation, no `String` in hot paths, `Serial` not `SerialUSB`.

## Always verify before writing
- Confirm a function exists in the installed library before calling it. The
  authoritative source is `.platformio/libdeps/<env>/`, not memory.
- Never invent a register name, macro or library method.
- After any change, run `pio run -e <env>` and read the real errors.

## Never
- Never run `pio run -t upload` without explicit approval.
- Never change `lib_deps` versions unless asked.
- Never write to `.pio/` — that is build output.
- Never assume a delay; use `millis()`/`micros()` timing, not `delay()` in a loop.
```

Then in Cline: **Settings → Custom Instructions** → point it at `.clinerules`, or select the file in the
chat with `@`. Continue picks up `.continue/rules/*.md` automatically.

## Gotchas

| Symptom | Cause | Fix |
| --- | --- | --- |
| `pio` not recognised in a terminal | Core's `Scripts` folder is not on `PATH` — expected, see [Running PlatformIO](#running-platformio) | Prepend `%USERPROFILE%\.platformio\penv\Scripts`, or call `pio.exe` by full path |
| Code in `src/` is not compiled | File extension or location | Only `.c/.cpp/.ino/.S` under `src/` is built |
| Garbled serial output | `monitor_speed` mismatch | 115200 for ESP32, 9600 for Uno |
| `exit status 1` with no useful error | Crashed toolchain | `pio run -t fullclean`, then rebuild |
| Build works, upload fails | Wrong port or a bootloader is running | `pio device list`; hold BOOT if the ESP32 is stuck |
| Random `undefined reference` | `#include` order / header not in `include/` | Put project headers in `include/`, not next to sources |
| Build suddenly too big | `build_type` reset to `debug` | Set `build_type = release` in the env |
| `.pio/` keeps growing | Stale build dirs per env | `pio run -t fullclean`; add `.pio/` to `.gitignore` |

Add `.pio/`, `.vscode/cpptools/`, and `.cache/` to `.gitignore` — never commit build output.

## See also

- [`README.md`](README.md) — the VS Code + Ollama + Continue + Cline setup, keybindings, and the model tables.
- [`continue/config.yaml`](continue/config.yaml) — model roles and context lengths.

