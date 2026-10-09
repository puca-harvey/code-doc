# code-doc

## Table of Contents

- [Contents](#contents)
- [Requirements](#requirements)
- [Install](#install)
  - [1. Ollama + models](#1-ollama-models)
  - [2. Continue extension in VS Code](#2-continue-extension-in-vs-code)
  - [3. Verify](#3-verify)
  - [4. Cline extension in VS Code](#4-cline-extension-in-vs-code)
- [Autocomplete](#autocomplete)
- [Cline (chat / file editing)](#cline-chat-file-editing)
- [Cloud Models and BYOK](#cloud-models-and-byok)
- [Embeddings / codebase awareness](#embeddings-codebase-awareness)
- [Refreshing the config schema after a Continue upgrade](#refreshing-the-config-schema-after-a-continue-upgrade)
- [Troubleshooting](#troubleshooting)
- [Daily workflow](#daily-workflow)
- [FAQ](#faq)

VS Code with AI support but **without GitHub Copilot**:

- **Autocomplete** — [Continue](https://continue.dev) driven by a **local Ollama** server running a
  Fill-in-the-Middle code model (tab completion).
- **Chat and file editing** — [Cline](https://cline.bot), pointed at the same local models.
- No GitHub account, no Copilot license, no code ever leaves the machine.

Everything runs locally, so there is no per-request cost and no data sent to a cloud provider.

## Contents

| File                                           | What it is                                                                                                         |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [`continue/config.yaml`](continue/config.yaml) | The Continue `config.yaml` (models, roles, context lengths). Copy to `%USERPROFILE%\.continue\config.yaml`.        |
| [`12GB.md`](12GB.md)                           | Hardware profile for a 12 GB card (RTX 5070) — model set, context length, measured speeds.                         |
| [`8GB.md`](8GB.md)                             | Hardware profile for an 8 GB card (RTX 4060) — smaller model set and a reduced context length.                     |
| [`MICROCONTROLLER.md`](MICROCONTROLLER.md)     | PlatformIO / embedded workflow: `platformio.ini`, build & flash, debugging, rules files, model split for firmware. |
| [`LICENSE`](LICENSE)                           | Repository license.                                                                                                |

VRAM decides which profile applies; see the hardware‑specific files for model list and context length.

| Card             | Profile              |
| ---------------- | -------------------- |
| 12 GB (RTX 5070) | [`12GB.md`](12GB.md) |
| 8 GB (RTX 4060)  | [`8GB.md`](8GB.md)   |

## Requirements

- Windows 10/11 (instructions below; macOS/Linux work the same for Ollama + Continue).
- VS Code.
- [Ollama](https://ollama.com/download) installed (verified here with `ollama 0.35.1`).
- Enough VRAM to keep the autocomplete model resident — see the profiles above.

## Install

### 1. Ollama + models
- Install Ollama from https://ollama.com/download.
- Pull models for your GPU VRAM (see 12GB.md or 8GB.md).
- Verify with `ollama list`.
- Install the **Ollama VS Code extension** (`Ollama.ollama`) — Extensions view → search **Ollama** → Install. It adds a native Ollama panel in the activity bar and, importantly, makes the **Ollama** API provider available inside Cline's settings so you can point it at `ornith-1.5:35b`.

### 2. Continue extension in VS Code

1. Open the Extensions view — click the **Extensions** icon in the activity bar, or `Ctrl`+`Shift`+`P` →
   _Extensions: View Extensions_.
2. Search **Continue** and install it (this setup is written against Continue `2.0.0`).
3. Copy the config into place:

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.continue" | Out-Null
Copy-Item .\continue\config.yaml "$env:USERPROFILE\.continue\config.yaml"
```

4. Reload VS Code (`Ctrl`+`Shift`+`P` → _Developer: Reload Window_).

The shipped `config.yaml` targets the 12 GB profile. On a smaller card, apply the edits in
[`8GB.md`](8GB.md#edit-configyaml) before copying it.

### 3. Verify

```powershell
ollama ps     # after a request: the model should be listed as 100% GPU
```

Type a few lines in a `.py`/`.ts` file and press `Tab` — Continue should complete it inline.


### 4. Cline extension in VS Code

1. VS Code → Extensions view (activity-bar icon, or `Ctrl`+`Shift`+`P` → _Extensions: View Extensions_) →
   search **Cline** → _Install_.
   (Or download the `.vsix` from the
   [Cline marketplace page](https://marketplace.visualstudio.com/items?itemName=saoudrizwan.claude-dev)
   and run _Extensions: Install from VSIX…_.)
2. Open Cline — click the Cline icon in the activity bar, or `Ctrl`+`Shift`+`P` → _Cline: Open in New Tab_.
3. Click the **settings gear** (bottom of the Cline sidebar) → **API Provider** → **Ollama**.
4. **Base URL** defaults to `http://localhost:11434` — leave it unless you changed Ollama's port.
5. **Model Id** → type the exact tag from `ollama list`, e.g. `ornith-1.5:35b` (fast chat) or the code model from
   your [hardware profile](12GB.md). Cline can also fetch the list from `http://localhost:11434/api/tags`.
6. Enable **Use Compact Prompt** (Settings → Features) — smaller context, much faster replies on a local model.
7. Click **Done**.

Ollama already runs on port `11434` as a background service, so there is nothing else to start. Confirm with:

```powershell
curl.exe http://localhost:11434/api/tags
```
## Autocomplete

Continue's `autocomplete` role needs a model with a native fill-in-the-middle (FIM) template. Ollama ships the
Qwen2.5-Coder models with one:

```text
{{- if .Suffix }}<|fim_prefix|>{{ .Prompt }}<|fim_suffix|>{{ .Suffix }}<|fim_middle|>
```

That `{{- if .Suffix }}` branch is the whole point. Continue sends _prefix + suffix_ and gets back only the middle,
so the model completes the line you are on. A model without FIM — the chat model, `ornith-1.5:35b` — has a bare
`{{ .Prompt }}` template with no `.Suffix`, so Continue falls back to pasting a raw `<|fim_prefix|>…<|fim_middle|>` prompt
and the model continues _past_ the insertion point into unrelated prose. Fine for chat, useless for tab completion.

Both `qwen2.5-coder:14b` and `qwen2.5-coder:7b` carry this template. Which one to use depends on your VRAM; see
[`12GB.md`](12GB.md) or [`8GB.md`](8GB.md).

### Keybindings

**Division of labour: Continue is for autocomplete only, Cline is for everything conversational.** Continue's
chat-oriented chords are therefore left alone — they are not remapped, and IntelliJ keeps what it wants.

**Keymap policy: IntelliJ first.** Do not install a second keybindings extension. Keymap extensions contribute
their mappings through `package.json` and the most recently installed one silently wins, so chords quietly change
meaning. There is no `"keymap"` setting that shows which extensions are active, so a conflict is hard to spot
until a familiar shortcut stops doing what you expect.

| Source                                 | Bindings | Note                                                                                                                              |
| -------------------------------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `k--kato.intellij-idea-keybindings`    | 220      | IntelliJ mappings, contributed via `package.json` — **not** via a `"keymap"` setting, so nothing in `settings.json` hints at them |
| `continue`                             | 21       | Only the autocomplete subset is used                                                                                              |
| `%APPDATA%\Code\User\keybindings.json` | 5        | Yours — two entries _remove_ Continue defaults, three _rebind_ them                                                               |

Autocomplete chords, verified on this machine:

| Action                          | Chord                 | Status                                                 |
| ------------------------------- | --------------------- | ------------------------------------------------------ |
| Accept the full suggestion      | `Tab`                 | Works — no extension shadows `inlineSuggest.commit`    |
| Reject the suggestion           | `Esc`                 | Works (`continue.exitEditMode`, `continue.rejectJump`) |
| Accept word-by-word             | `Ctrl`+`→`            | Works                                                  |
| Force a suggestion now          | `Ctrl`+`Alt`+`Space`  | Works — unclaimed by every installed extension         |
| Toggle tab autocomplete         | `Ctrl`+`K` `Ctrl`+`A` | Works — unclaimed                                      |
| Toggle next-edit suggestions    | `Ctrl`+`K` `Ctrl`+`N` | Works — unclaimed                                      |
| Open Continue in its own window | `Ctrl`+`K` `Ctrl`+`M` | Works — unclaimed                                      |

Cline chords:

| Action                        | Chord                                                                   | Status                                                   |
| ----------------------------- | ----------------------------------------------------------------------- | -------------------------------------------------------- |
| Open Cline                    | `Ctrl`+`Shift`+`P` → _Cline: Open in New Tab_, or the activity-bar icon | Works                                                    |
| Add the selection to the chat | `Ctrl`+`'`                                                              | Works — jumps to the chat input when nothing is selected |

Editor chords that IntelliJ owns:

| Action          | Chord                                    | Status                                                      |
| --------------- | ---------------------------------------- | ----------------------------------------------------------- |
| Command Palette | `Ctrl`+`Shift`+`P` or `Ctrl`+`Shift`+`A` | Both work; `Ctrl`+`Shift`+`A` is the IntelliJ _Find Action_ |

`Shift`+`Alt`+`E` (Continue inline edit) collides with `PowerShell.ExpandAlias` inside `.ps1` files only, and
`Shift`+`Alt`+`C` with IntelliJ's _Copy File Path_, but only when the editor is **not** focused.

Rebind anything via `Ctrl`+`Shift`+`A` → _Preferences: Open Keyboard Shortcuts_ (search the command ID, click the
pencil, press the new chord). Add a `{"key": …, "command": …}` entry to `%APPDATA%\Code\User\keybindings.json` to
make it permanent. An entry whose command starts with `-` removes a default.

Continue commands with no default binding: `continue.newSession`, `continue.viewHistory`,
`continue.openConfigPage`, `continue.viewLogs`, `continue.selectFilesAsContext`, `continue.rebuildCodebaseIndex`.

## Cline (chat / file editing)

Cline is the agent-style half of this setup: you describe a change, it plans, asks permission to run commands, and
applies multi-file edits with checkpoints. Continue handles autocomplete; Cline handles everything conversational.

Installed here as `saoudrizwan.claude-dev` (Cline `4.1.22`).

### Install

1. VS Code → Extensions view (activity-bar icon, or `Ctrl`+`Shift`+`P` → _Extensions: View Extensions_) →
   search **Cline** → _Install_.
   (Or download the `.vsix` from the
   [Cline marketplace page](https://marketplace.visualstudio.com/items?itemName=saoudrizwan.claude-dev)
   and run _Extensions: Install from VSIX…_.)
2. Open Cline — click the Cline icon in the activity bar, or `Ctrl`+`Shift`+`P` → _Cline: Open in New Tab_.
3. Click the **settings gear** (bottom of the Cline sidebar) → **API Provider** → **Ollama**.
4. **Base URL** defaults to `http://localhost:11434` — leave it unless you changed Ollama's port.
5. **Model Id** → type the exact tag from `ollama list`, e.g. `ornith-1.5:35b` (fast chat) or the code model from
   your [hardware profile](12GB.md). Cline can also fetch the list from `http://localhost:11434/api/tags`.
6. Enable **Use Compact Prompt** (Settings → Features) — smaller context, much faster replies on a local model.
7. Click **Done**.

Ollama already runs on port `11434` as a background service, so there is nothing else to start. Confirm with:

```powershell
curl.exe http://localhost:11434/api/tags
```

### Use

- **New Task** — type a task in natural language, e.g. _"Refactor `parse_config()` in `src/config.py` into a
  dataclass and update all call sites"_.
- Cline replies with a plan — approve or revise it before it edits anything.
- Each file change appears as a diff; review and accept per file.
- Checkpoints (stored under `%USERPROFILE%\.cline\data\checkpoint-scratch\`) let you roll back a bad edit.
- Cline asks before running terminal commands; auto‑approve per‑command or per‑tool as you get comfortable.
- `Ctrl`+`'` adds the current selection to the chat, or jumps to the chat input when nothing is selected.

Point Cline at `ornith-1.5:35b` for everything conversational, including edits. It holds the `chat`, `edit` and
`apply` roles in `config.yaml` — the local chat model of record. For planning, a large `glm-5.3-flash:cloud` via Ollama
is a good off-card option when you'd rather not load another local model; see *Ollama Cloud Models* below.
`qwen2.5-coder:14b` keeps only `autocomplete`, where its native FIM template is the only thing that matters.

### Where Cline stores things

| Path                                                         | Contents                                                       |
| ------------------------------------------------------------ | -------------------------------------------------------------- |
| `%USERPROFILE%\.cline\data\settings\providers.json`          | Provider entries (`cline`, `ollama`, …) and `lastUsedProvider` |
| `%USERPROFILE%\.cline\data\settings\global-settings.json`    | Global toggles (auto-update, telemetry)                        |
| `%USERPROFILE%\.cline\data\settings\cline_mcp_settings.json` | MCP servers                                                    |
| `%USERPROFILE%\.cline\data\sessions\`                        | One folder per task                                            |
| `%USERPROFILE%\.cline\data\db\sessions.db`                   | Session index (SQLite)                                         |
| `%USERPROFILE%\.cline\data\logs\`                            | `hooks.jsonl` etc.                                             |

Prefer the in-editor settings UI over hand-editing these; the files are rewritten on every change.

> Note: `lastUsedProvider` in `providers.json` is the provider Cline restores on startup. It is currently
> `cline` (the hosted service) even though an `ollama` entry exists — switch to Ollama in the UI if you want
> everything local.

### Cline troubleshooting

| Symptom                              | Fix                                                                                     |
| ------------------------------------ | --------------------------------------------------------------------------------------- |
| "Could not connect to Ollama"        | `curl.exe http://localhost:11434/api/tags` — if it fails, start Ollama (`ollama serve`) |
| Empty/garbled replies                | Wrong Model Id — use the exact tag from `ollama list`                                   |
| Very slow first reply                | Model not loaded yet; warm it with `ollama run <tag>` once                              |
| Replies get slower as the task grows | Enable **Use Compact Prompt**; start a new task when context fills up                   |
| Cline ignores project files          | Add the folder to Cline's approved working directory                                    |

## Cloud Models and BYOK

Cline can utilize many providers, including cloud-based models and those run locally within Ollama. The most flexible approach is **BYOK (Bring Your Own Key)**, where you obtain an API key directly from a provider (e.g., DeepSeek, Anthropic, OpenAI) and configure it in Cline's settings. This gives you direct control over costs and model versions.

### Configuring BYOK in Cline

1. Open the Cline settings (gear icon in the editor).
2. Select the appropriate **API Provider** from the dropdown.
3. Enter your **API Key**.
4. Choose your preferred **Model** from the list.

### BYOK Examples (DeepSeek and Mistral)

| Model ID | Context Window | Best For | Cost (per 1M tokens) | Register and obtain API key |
| --- | --- | --- | --- | --- |
| `deepseek-flash` | 1M | General-purpose chat, code understanding, reasoning. Good for everyday coding questions. | Input: $0.30<br>Cached: $0.006<br>Output: $1.20<br>(Off-peak: Input $0.15, Cached $0.003, Output $0.60) | [DeepSeek Platform](https://platform.deepseek.com) |
| `deepseek-v4-pro` | 1M | Code generation, complex reasoning, agentic tasks. Optimized for coding agents. | Input: $1.32<br>Cached: $0.044<br>Output: $3.96<br>(Off-peak: Input $0.66, Cached $0.022, Output $1.98) | [DeepSeek Platform](https://platform.deepseek.com) |
| `mistral-large-4` | 1M | Complex reasoning, multilingual tasks, advanced code generation. | Input: $0.68<br>Cached: $0.07<br>Output: $2.09<br>(Off-peak pricing applies) | [Mistral Console](https://console.mistral.ai/registration/) |


### Free Alternatives

If you prefer not to use a paid API key, there are a few free options:

#### Ollama Cloud Models (Free Tier + Pay-As-You-Go)

Ollama Cloud provides a free tier with starter usage credits. Once those credits are exhausted, requests draw from purchased usage credits (pay-as-you-go). **Free accounts include 1 concurrent request.** To initialize a cloud model, run it once in your terminal:

```powershell
ollama run gemma4:cloud
ollama run glm-5.3-flash:cloud
ollama run gpt-oss:20b-cloud
ollama run gpt-oss:120b-cloud
ollama run nemotron-3-nano:30b-cloud
ollama run nemotron-3-super:cloud
ollama run nemotron-3-ultra:cloud
```

After running, the model will be available for selection in Cline (Provider: `Ollama`, Model ID: e.g., `gemma4:cloud`).

| Model ID | Context Window | Best For | Cost (per 1M tokens) — Free credits used first, then real money | How to Call |
| --- | --- | --- | --- | --- |
| `gemma4:cloud` (or `gemma3:cloud`) | 128K | General-purpose chat, multimodal (text + images), summarization, reasoning. Good default for everyday coding questions. | Input: $0.14<br>Cached: $0.05<br>Output: $0.40 | `ollama run gemma4:cloud` |
| `glm-5.3-flash:cloud` | 1M | Cheap, fast — planning, chat, small edits; 18B active MoE, natively multimodal. The off-card fallback for the local `ornith-1.5:35b` when you'd rather not load another model. | Input: $0.15<br>Cached: $0.03<br>Output: $0.50 | `ollama run glm-5.3-flash:cloud` |
| `gpt-oss:20b-cloud` | 128K | Agentic tasks, function calling, web browsing, python tool calls, structured outputs. Lower latency, good for specialized/local use-cases. Configurable reasoning effort (low/medium/high). | Input: $0.07<br>Cached: $0.035<br>Output: $0.30 | `ollama run gpt-oss:20b-cloud` |
| `gpt-oss:120b-cloud` | 128K | Complex reasoning, agentic workflows, higher quality outputs. Full chain-of-thought access for debugging. Apache 2.0 license. | Input: $0.15<br>Cached: $0.014<br>Output: $0.60 | `ollama run gpt-oss:120b-cloud` |
| `nemotron-3-nano:30b-cloud` | 1M | Efficient agentic tasks, long-context reasoning + non-reasoning unified model. Hybrid MoE (3.5B active / 30B total). Good for coding agents, IT automation. | Input: $0.06<br>Output: $0.24 | `ollama run nemotron-3-nano:30b-cloud` |
| `nemotron-3-super:cloud` | 256K | Complex multi-agent applications, collaborative agents, high-volume workloads (e.g., IT ticket automation). 12B active / 120B total MoE. Strong on SWE-Bench, LiveCodeBench. | Input: $0.015<br>Cached: $0.015<br>Output: $0.60 | `ollama run nemotron-3-super:cloud` |
| `nemotron-3-ultra:cloud` | 256K (1M effective) | Long-running agent workflows, deep research, complex enterprise workflows across hundreds of steps. 55B active / 550B total. Best for agent orchestration, coding agents. | Input: $0.10<br>Cached: $0.10<br>Output: $3.00 | `ollama run nemotron-3-ultra:cloud` |

> **Notes on Ollama Cloud pricing:**
> - **Free tier:** Starter usage credits included (resets monthly from sign-up date). Only a subset of "starter models" available on Free plan.
> - **Pro ($20/mo):** $60 usage credits/month, access to larger pro models, 3 concurrent requests.
> - **Max ($100/mo):** $300 usage credits/month, early access to newest models, 10 concurrent requests.
> - **Off-peak pricing** (outside 12:00–18:00 UTC weekdays, all day weekends) applies to some models.
> - Running models locally via Ollama is always unlimited and free.

#### Cline Free

Check the [Cline Free models documentation](https://docs.cline.bot/getting-started/free-models) for current available free tiers. Note that these offerings change frequently.

**Configuration in Cline:**

- **Provider:** `Ollama`
- **Model ID:** `gemma4:cloud` (or any of the cloud models above)

Rotating, limited-time promotions on select models, at no cost up to a quota, may still be available in Cline. Any account can use them.

- Models appear tagged **FREE** in the picker under both the **Cline** and **ClinePass** providers.
- Not available through the Cline API — IDE extension and CLI only.
- Free-tier prompts may be used to improve model quality.
- When the quota runs out, requests fail; nothing silently bills you.


### ClinePass

See [MODELS.md](MODELS.md) for the full list of available models and pricing.

## Embeddings / codebase awareness

```powershell
ollama pull nomic-embed-text-v2-moe:latest
```

- 475M params MoE (305M active), 768 dims (reducible to 256 via Matryoshka), 512-token window, ~100 languages.
- Continue embeds with `POST /api/embed {model, input: [chunks]}` and this model answers it.
- **Pick "Nomic Embed v2 MoE" for the Embed role in the model picker** — the remembered selection is
  currently _Transformers.js (Built-In)_, which needs no local server but is slower.
- Changing the embedder invalidates the stored index; rebuild it afterwards.
- Leave Continue's default chunk size alone: the 512-token window is much shorter than the old
  `nomic-embed-text` (8192) and Ollama truncates anything longer.
- Known caveat: the model card recommends `search_document: ` / `search_query: ` prefixes for best
  retrieval, which Continue does not add — retrieval quality is slightly below the card's numbers.

## Refreshing the config schema after a Continue upgrade

`config.yaml` starts with a `$schema=` line pointing at an offline copy of Continue's JSON schema. After
upgrading Continue, refresh it:

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.continue" | Out-Null
Copy-Item "$env:USERPROFILE\.vscode\extensions\continue.continue-*\config-yaml-schema.json" "$env:USERPROFILE\.continue\"
```

That gives you IntelliSense and validation of `roles:` / `contextLength:` inside VS Code.

## Troubleshooting

Hardware-specific symptoms (CPU offload, out-of-memory) are in the profile for your card — [`12GB.md`](12GB.md) or
[`8GB.md`](8GB.md).

| Symptom                         | Likely cause                                          | Fix                                                                                  |
| ------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------ |
| No completions on `Tab`         | Continue not installed/enabled, or Ollama unreachable | `ollama list` must show the model; reload the window; check the Continue output view |
| Completion continues into prose | A non-FIM model holds the `autocomplete` role         | Only a Qwen2.5-Coder tag supports FIM; keep `autocomplete` off the chat model        |
| Cline cannot reach Ollama       | Wrong Base URL                                        | Use `http://localhost:11434`, then `ollama serve`                                    |
| Chat replies are generic        | Fast chat model selected for a code question          | Switch Cline to the code model from your hardware profile                            |
| Semantic search returns nothing | Embed role points at the built-in embedder            | Select _Nomic Embed v2 MoE_, then rebuild the index                                  |

## Daily workflow

1. `ollama serve` already runs as a service — nothing to launch.
2. Code in VS Code; `Tab` accepts a completion, `Ctrl`+`Alt`+`Space` forces one.
3. Cline for anything conversational or multi-file.
4. `ollama ps` first whenever something feels slow — note it lists only _loaded_ models, so an empty result
   just means nothing has been requested yet.

## FAQ

**Do I need GitHub Copilot or a cloud API key?**
No. Every local model runs through Ollama on localhost.

**Can I run this on CPU only?**
Yes, but tab completion will be slow — expect seconds instead of milliseconds. Prefer the smaller FIM model; see
the note in [`12GB.md`](12GB.md).

**Why not use one model for everything?**
Autocomplete requires a model with a native FIM (Fill-in-the-Middle) template, so the locally installed `qwen2.5-coder:14b` is dedicated to that role via Continue. For everything else—chat, planning, `edit`, and `apply`—I use either `gemma4:cloud` via Ollama or BYOK (Bring Your Own Key) cloud models like DeepSeek.

**How do I add a model?**
Depends whether it is a local or a cloud model.
For installing a local model and running it, see the [Ollama documentation](https://ollama.com/docs/get-started).
For Ollama cloud models you have to run it once, and then it will be available to Cline.
For BYOK models in Cline, choose the appropriate provider and configure it together with the API key.

**Where is Cline's config stored?**
Under `%USERPROFILE%\.cline\data\` (see _Where Cline stores things_ above) — but use the in-editor settings
gear, since the files are rewritten whenever a setting changes.
