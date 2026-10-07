# code-doc

VS Code with AI support but **without GitHub Copilot**:

- **Autocomplete** — [Continue](https://continue.dev) driven by a **local Ollama** server running a
  Fill-in-the-Middle code model (tab completion).
- **Chat and file editing** — [Cline](https://cline.bot), pointed at the same local models.
- No GitHub account, no Copilot license, no code ever leaves the machine.

Everything runs locally, so there is no per-request cost and no data sent to a cloud provider.

## Contents

| File | What it is |
| --- | --- |
| [`continue/config.yaml`](continue/config.yaml) | The Continue `config.yaml` (models, roles, context lengths). Copy to `%USERPROFILE%\.continue\config.yaml`. |
| [`12GB.md`](12GB.md) | Hardware profile for a 12 GB card (RTX 5070) — model set, context length, measured speeds. |
| [`8GB.md`](8GB.md) | Hardware profile for an 8 GB card (RTX 4060) — smaller model set and a reduced context length. |
| [`MICROCONTROLLER.md`](MICROCONTROLLER.md) | PlatformIO / embedded workflow: `platformio.ini`, build & flash, debugging, rules files, model split for firmware. |
| [`LICENSE`](LICENSE) | Repository license. |

**VRAM decides which profile applies.** The general setup below is hardware-independent; the model list and context
length are not. Follow the profile for your card and skip the model-specific parts here.

| Card | Profile |
| --- | --- |
| 12 GB (RTX 5070) | [`12GB.md`](12GB.md) |
| 8 GB (RTX 4060) | [`8GB.md`](8GB.md) |

## Requirements

- Windows 10/11 (instructions below; macOS/Linux work the same for Ollama + Continue).
- VS Code.
- [Ollama](https://ollama.com/download) installed (verified here with `ollama 0.35.1`).
- Enough VRAM to keep the autocomplete model resident — see the profiles above.

## Install

### 1. Ollama + models

```powershell
# install Ollama from https://ollama.com/download, then follow the profile for your card:
#   12GB.md  -> ollama pull qwen2.5-coder:14b ; ollama pull ornith-1.5:9b ; ...
#   8GB.md   -> ollama pull qwen2.5-coder:7b  ; ollama pull ornith-1.5:9b ; ...
ollama list                                # verify
```

Model choice, sizes and context length are hardware-specific and live in the profiles.

### 2. Continue extension in VS Code

1. Open the Extensions view — click the **Extensions** icon in the activity bar, or `Ctrl`+`Shift`+`P` →
   *Extensions: View Extensions*.
2. Search **Continue** and install it (this setup is written against Continue `2.0.0`).
3. Copy the config into place:

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.continue" | Out-Null
Copy-Item .\continue\config.yaml "$env:USERPROFILE\.continue\config.yaml"
```

4. Reload VS Code (`Ctrl`+`Shift`+`P` → *Developer: Reload Window*).

The shipped `config.yaml` targets the 12 GB profile. On a smaller card, apply the edits in
[`8GB.md`](8GB.md#edit-configyaml) before copying it.

### 3. Verify

```powershell
ollama ps     # after a request: the model should be listed as 100% GPU
```

Type a few lines in a `.py`/`.ts` file and press `Tab` — Continue should complete it inline.

## Autocomplete

Continue's `autocomplete` role needs a model with a native fill-in-the-middle (FIM) template. Ollama ships the
Qwen2.5-Coder models with one:

```text
{{- if .Suffix }}<|fim_prefix|>{{ .Prompt }}<|fim_suffix|>{{ .Suffix }}<|fim_middle|>
```

That `{{- if .Suffix }}` branch is the whole point. Continue sends *prefix + suffix* and gets back only the middle,
so the model completes the line you are on. A model without FIM — the chat model, `ornith-1.5:9b` — has a bare
`{{ .Prompt }}` template with no `.Suffix`, so Continue falls back to pasting a raw `<|fim_prefix|>…<|fim_middle|>` prompt
and the model continues *past* the insertion point into unrelated prose. Fine for chat, useless for tab completion.

Both `qwen2.5-coder:14b` and `qwen2.5-coder:7b` carry this template. Which one to use depends on your VRAM; see
[`12GB.md`](12GB.md) or [`8GB.md`](8GB.md).

### Keybindings

**Division of labour: Continue is for autocomplete only, Cline is for everything conversational.** Continue's
chat-oriented chords are therefore left alone — they are not remapped, and IntelliJ keeps what it wants.

**Keymap policy: IntelliJ first.** Do not install a second keybindings extension. Keymap extensions contribute
their mappings through `package.json` and the most recently installed one silently wins, so chords quietly change
meaning. There is no `"keymap"` setting that shows which extensions are active, so a conflict is hard to spot
until a familiar shortcut stops doing what you expect.

| Source | Bindings | Note |
| --- | --- | --- |
| `k--kato.intellij-idea-keybindings` | 220 | IntelliJ mappings, contributed via `package.json` — **not** via a `"keymap"` setting, so nothing in `settings.json` hints at them |
| `continue` | 21 | Only the autocomplete subset is used |
| `%APPDATA%\Code\User\keybindings.json` | 5 | Yours — two entries *remove* Continue defaults, three *rebind* them |

Autocomplete chords, verified on this machine:

| Action | Chord | Status |
| --- | --- | --- |
| Accept the full suggestion | `Tab` | Works — no extension shadows `inlineSuggest.commit` |
| Reject the suggestion | `Esc` | Works (`continue.exitEditMode`, `continue.rejectJump`) |
| Accept word-by-word | `Ctrl`+`→` | Works |
| Force a suggestion now | `Ctrl`+`Alt`+`Space` | Works — unclaimed by every installed extension |
| Toggle tab autocomplete | `Ctrl`+`K` `Ctrl`+`A` | Works — unclaimed |
| Toggle next-edit suggestions | `Ctrl`+`K` `Ctrl`+`N` | Works — unclaimed |
| Open Continue in its own window | `Ctrl`+`K` `Ctrl`+`M` | Works — unclaimed |

Cline chords:

| Action | Chord | Status |
| --- | --- | --- |
| Open Cline | `Ctrl`+`Shift`+`P` → *Cline: Open in New Tab*, or the activity-bar icon | Works |
| Add the selection to the chat | `Ctrl`+`'` | Works — jumps to the chat input when nothing is selected |

Editor chords that IntelliJ owns:

| Action | Chord | Status |
| --- | --- | --- |
| Command Palette | `Ctrl`+`Shift`+`P` or `Ctrl`+`Shift`+`A` | Both work; `Ctrl`+`Shift`+`A` is the IntelliJ *Find Action* |

`Shift`+`Alt`+`E` (Continue inline edit) collides with `PowerShell.ExpandAlias` inside `.ps1` files only, and
`Shift`+`Alt`+`C` with IntelliJ's *Copy File Path*, but only when the editor is **not** focused.

Rebind anything via `Ctrl`+`Shift`+`A` → *Preferences: Open Keyboard Shortcuts* (search the command ID, click the
pencil, press the new chord). Add a `{"key": …, "command": …}` entry to `%APPDATA%\Code\User\keybindings.json` to
make it permanent. An entry whose command starts with `-` removes a default.

Continue commands with no default binding: `continue.newSession`, `continue.viewHistory`,
`continue.openConfigPage`, `continue.viewLogs`, `continue.selectFilesAsContext`, `continue.rebuildCodebaseIndex`.

## Cline (chat / file editing)

Cline is the agent-style half of this setup: you describe a change, it plans, asks permission to run commands, and
applies multi-file edits with checkpoints. Continue handles autocomplete; Cline handles everything conversational.

Installed here as `saoudrizwan.claude-dev` (Cline `4.1.22`).

### Install

1. VS Code → Extensions view (activity-bar icon, or `Ctrl`+`Shift`+`P` → *Extensions: View Extensions*) →
   search **Cline** → *Install*.
   (Or download the `.vsix` from the
   [Cline marketplace page](https://marketplace.visualstudio.com/items?itemName=saoudrizwan.claude-dev)
   and run *Extensions: Install from VSIX…*.)
2. Open Cline — click the Cline icon in the activity bar, or `Ctrl`+`Shift`+`P` → *Cline: Open in New Tab*.
3. Click the **settings gear** (bottom of the Cline sidebar) → **API Provider** → **Ollama**.
4. **Base URL** defaults to `http://localhost:11434` — leave it unless you changed Ollama's port.
5. **Model Id** → type the exact tag from `ollama list`, e.g. `ornith-1.5:9b` (fast chat) or the code model from
   your [hardware profile](12GB.md). Cline can also fetch the list from `http://localhost:11434/api/tags`.
6. Enable **Use Compact Prompt** (Settings → Features) — smaller context, much faster replies on a local model.
7. Click **Done**.

Ollama already runs on port `11434` as a background service, so there is nothing else to start. Confirm with:

```powershell
curl.exe http://localhost:11434/api/tags
```

### Use

- **New Task** — type a task in natural language, e.g. *"Refactor `parse_config()` in `src/config.py` into a
  dataclass and update all call sites"*.
- Cline replies with a plan — approve or revise it before it edits anything.
- Each file change appears as a diff; review and accept per file.
- Checkpoints (stored under `%USERPROFILE%\.cline\data\checkpoint-scratch\`) let you roll back a bad edit.
- Cline asks before running terminal commands; auto‑approve per‑command or per‑tool as you get comfortable.
- `Ctrl`+`'` adds the current selection to the chat, or jumps to the chat input when nothing is selected.

Point Cline at `ornith-1.5:9b` for everything conversational, including edits. It holds the `chat`, `edit` and
`apply` roles in `config.yaml`, because it is ~1.6× faster than the code model. `qwen2.5-coder:14b` keeps only
`autocomplete`, where its native FIM template is the only thing that matters. If inline edits feel weaker than when the
14 B held them, move `edit` and `apply` back in `config.yaml` — it is a two-line change.

### Where Cline stores things

| Path | Contents |
| --- | --- |
| `%USERPROFILE%\.cline\data\settings\providers.json` | Provider entries (`cline`, `ollama`, …) and `lastUsedProvider` |
| `%USERPROFILE%\.cline\data\settings\global-settings.json` | Global toggles (auto-update, telemetry) |
| `%USERPROFILE%\.cline\data\settings\cline_mcp_settings.json` | MCP servers |
| `%USERPROFILE%\.cline\data\sessions\` | One folder per task |
| `%USERPROFILE%\.cline\data\db\sessions.db` | Session index (SQLite) |
| `%USERPROFILE%\.cline\data\logs\` | `hooks.jsonl` etc. |

Prefer the in-editor settings UI over hand-editing these; the files are rewritten on every change.

> Note: `lastUsedProvider` in `providers.json` is the provider Cline restores on startup. It is currently
> `cline` (the hosted service) even though an `ollama` entry exists — switch to Ollama in the UI if you want
> everything local.

### Cline troubleshooting

| Symptom | Fix |
| --- | --- |
| "Could not connect to Ollama" | `curl.exe http://localhost:11434/api/tags` — if it fails, start Ollama (`ollama serve`) |
| Empty/garbled replies | Wrong Model Id — use the exact tag from `ollama list` |
| Very slow first reply | Model not loaded yet; warm it with `ollama run <tag>` once |
| Replies get slower as the task grows | Enable **Use Compact Prompt**; start a new task when context fills up |
| Cline ignores project files | Add the folder to Cline's approved working directory |

## Cloud models: Cline Free and ClinePass

Local Ollama is the default, but Cline can also reach hosted models. Two of those providers are worth
knowing about. **Pricing and model lineups change often — verify in the model picker before trusting a
number here.** Source: [Cline Free](https://docs.cline.bot/getting-started/free-models) /
[ClinePass](https://docs.cline.bot/getting-started/clinepass).

### Cline Free / Ollama Cloud
\n\nSince `deepseek-v4.1-flash` is no longer available in the Cline free tier, the best stable alternative is using free tier models via **Ollama Cloud**.

**Prerequisites:**
1. Sign up at [ollama.com](https://ollama.com).
2. Initialize the cloud model by running once in your terminal:
   ```powershell
   ollama run gemma4:cloud
   ```

**Configuration in Cline:**
- **Provider:** `Ollama`
- **Model ID:** `gemma4:cloud`

Rotating, limited-time promotions on select models, at no cost up to a quota, may still be available. Any account can use them.

- Models appear tagged **FREE** in the picker under both the **Cline** and **ClinePass** providers.
- The promotion set changes over time, so treat the list below as "currently referenced", not a promise.
- Not available through the Cline API — IDE extension and CLI only.
- Free-tier prompts may be used to improve model quality.
- When the quota runs out, requests fail; nothing silently bills you.

Currently referenced in your Cline `4.1.22` bundle:

| Model ID | Notes |
| --- | --- |
| `ollama/gemma4:cloud` | Stable free tier via Ollama Cloud — best general-purpose free bet |
| `cline-free/mimo-v2.6-flash` | Very cheap tier; fine for planning and chat |
| `cline-free/muse-spark-1.3-contributor` | Smallest/cheapest; best for cheap mechanical edits |

**Use them for:** exploring a model before paying, throwaway refactors, and as a safety net when the
ClinePass quota is exhausted mid-task.

### ClinePass

Flat **$9.99/month**, giving **2–5× the usage** of standard API rates on a curated set of open coding
models. It exists because standard rate limits throttle exactly the workloads Cline performs — many turns
of file reads, command runs and edits.

It is a **separate provider** from Cline (usage-billing); you can hold both.

| Model ID | Input / Output per 1M | Best for |
| --- | --- | --- |
| `cline-pass/glm-5.3` | $1.40 / $4.40 | Strong all-rounder; good default for refactors |
| `cline-pass/glm-5.3-flash` | $0.15 / $0.50 | Cheap, fast — planning, chat, small edits |
| `cline-pass/kimi-k3` | $3.00 / $15.00 | Most expensive here; reserve for hard tasks |
| `cline-pass/deepseek-v4-pro` | $1.32 / $3.96 peak, $0.66 / $1.98 off-peak | Peak/off-peak pricing — schedule big runs off-peak |
| `cline-pass/deepseek-v4.1-flash` | $0.30 / $1.20 | Cheap workhorse |
| `cline-pass/mimo-v2.5` | $0.14 / $0.28 | Cheapest non-flash tier |
| `cline-pass/mimo-v2.5-pro` | $1.74 / $3.48 | |
| `cline-pass/minimax-m3` | $0.30 / $1.20 | |
| `cline-pass/muse-spark-1.3-contributor` | $0.10 / $0.20 | Cheapest overall |
| `cline-pass/qwen3.8-max` | $2.00 / $6.00 | High quality on code |
| `cline-pass/qwen3.7-max` | $2.50 / $7.50 | |
| `cline-pass/qwen3.7-plus` | $0.40 / $1.60 up to 256 K, then $1.20 / $4.80 | **Large context** — the pick for big refactors |

Those are *reference* prices showing how usage is measured against your quota; you are not billed per token.

Three limits apply: a **5-hour rolling window**, **weekly**, and **monthly**. Check the dashboard at
[app.cline.bot](https://app.cline.bot/dashboard/subscription?personal=true).

> **Deprecated and gone:** GLM-5.2, Kimi K2.6, Kimi K2.7 Code, DeepSeek V4 Flash. Your local Cline bundle
> still references `cline-pass/glm-5.2` and `cline-pass/mimo-v2.6-flash` / `-pro`, so it is newer than some
> docs — trust the picker over both.

**Recommended split across the whole setup:**

| Situation | Model |
| --- | --- |
| Inline autocomplete (`Tab`) | The FIM code model from your [hardware profile](12GB.md) — the only role it holds |
| Chat, planning, edits, explanations | `ornith-1.5:9b` (local) |
| Multi-file refactor, long context | `cline-pass/glm-5.3`, or `cline-pass/qwen3.7-plus` past 256 K |
| Hardest reasoning, budget allows | `cline-pass/kimi-k3` |
| ClinePass quota spent | `ollama-cloud/gemma4` |
| Private code that must not leave the machine | Local Ollama only |

Switching model mid-task is free on the local side, so there is no reason to burn ClinePass quota on
something a local 9 B model handles.

ClinePass models are also usable outside Cline via the [Cline API](https://docs.cline.bot/api/overview)
(OpenAI-compatible Chat Completions, same `cline-pass/...` slug in the `model` field).

## Embeddings / codebase awareness

```powershell
ollama pull nomic-embed-text-v2-moe:latest
```

- 475M params MoE (305M active), 768 dims (reducible to 256 via Matryoshka), 512-token window, ~100 languages.
- Continue embeds with `POST /api/embed {model, input: [chunks]}` and this model answers it.
- **Pick "Nomic Embed v2 MoE" for the Embed role in the model picker** — the remembered selection is
  currently *Transformers.js (Built-In)*, which needs no local server but is slower.
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

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| No completions on `Tab` | Continue not installed/enabled, or Ollama unreachable | `ollama list` must show the model; reload the window; check the Continue output view |
| Completion continues into prose | A non-FIM model holds the `autocomplete` role | Only a Qwen2.5-Coder tag supports FIM; keep `autocomplete` off the chat model |
| Cline cannot reach Ollama | Wrong Base URL | Use `http://localhost:11434`, then `ollama serve` |
| Chat replies are generic | Fast chat model selected for a code question | Switch Cline to the code model from your hardware profile |
| Semantic search returns nothing | Embed role points at the built-in embedder | Select *Nomic Embed v2 MoE*, then rebuild the index |

## Daily workflow

1. `ollama serve` already runs as a service — nothing to launch.
2. Code in VS Code; `Tab` accepts a completion, `Ctrl`+`Alt`+`Space` forces one.
3. Cline for anything conversational or multi-file.
4. `ollama ps` first whenever something feels slow — note it lists only *loaded* models, so an empty result
   just means nothing has been requested yet.

## FAQ

**Do I need GitHub Copilot or a cloud API key?**
No. Every local model runs through Ollama on localhost.

**Can I run this on CPU only?**
Yes, but tab completion will be slow — expect seconds instead of milliseconds. Prefer the smaller FIM model; see
the note in [`12GB.md`](12GB.md).

**Why not use one model for everything?**
The code model is the only one with a native FIM template, so `autocomplete` has no alternative. Everything else —
chat, planning, `edit`, `apply` — goes to `ornith-1.5:9b`, which is ~1.6× faster. That is a speed-over-code-quality
trade; give the roles back to the code model if its edits read better to you.

**How do I add a model?**
Add an entry under `models:` in `config.yaml`, run `ollama pull <tag>`, then select it in the model picker.

**Where is Cline's config stored?**
Under `%USERPROFILE%\.cline\data\` (see *Where Cline stores things* above) — but use the in-editor settings
gear, since the files are rewritten whenever a setting changes.

