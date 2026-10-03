# code-doc

VS Code with AI support but **without GitHub Copilot**:

- **Autocomplete / inline editing** — [Continue](https://continue.dev) extension driven by a **local Ollama** server running **Qwen2.5‑Coder 14B** (Fill‑in‑the‑Middle, tab completion).
- **Chat / prompt workflows** — Continue's chat panel (and *Cline* when you want the agent to plan and edit files for you) pointed at the same local models.
- No GitHub account, no Copilot license, no code ever leaves the machine.

Everything runs locally, so there is no per-request cost and no data sent to a cloud provider.

## Contents

| File | What it is |
| --- | --- |
| [`continue/config.yaml`](continue/config.yaml) | The Continue `config.yaml` (models, roles, context lengths). Copy to `%USERPROFILE%\.continue\config.yaml`. |
| [`MICROCONTROLLER.md`](MICROCONTROLLER.md) | PlatformIO / embedded workflow: `platformio.ini`, build & flash, debugging, rules files, model split for firmware. |
| [`LICENSE`](LICENSE) | Repository license. |

## Requirements

- Windows 10/11 (instructions below; macOS/Linux work the same for Ollama + Continue).
- VS Code.
- [Ollama](https://ollama.com/download) installed (verified here with `ollama 0.35.1`).
- A GPU with enough VRAM to keep the 14B model resident (this setup targets ~12 GB, e.g. an RTX 5070).
- ~20 GB disk for all four models.

## Install

### 1. Ollama + models

```powershell
# install Ollama from https://ollama.com/download, then:
ollama pull qwen2.5-coder:14b              # ~9 GB - main model: autocomplete, edit, apply, chat
ollama pull granite4.2:8b                  # ~5 GB - fast chat
ollama pull llama3.1:8b                    # ~5 GB - fast fallback chat
ollama pull nomic-embed-text-v2-moe:latest # ~1 GB - embeddings / semantic search
ollama list                                # verify
```

`qwen2.5-coder:14b` must come from the **official Ollama library**. The plain `:7b` tag is a non‑FIM derivative and gives noticeably worse completions; the `:14b` and `:32b` tags ship the native fill‑in‑the‑middle template.

### 2. Continue extension in VS Code

1. Open the Extensions view — click the **Extensions** icon in the activity bar, or `Ctrl`+`Shift`+`P` →
   *Extensions: View Extensions*. (`Ctrl`+`Shift`+`X` is the VS Code default but does **not** fire on this
   machine; see *Keybindings* below.)
2. Search **Continue** and install it (this setup is written against Continue `2.0.0`).
3. Copy the config into place:

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.continue" | Out-Null
Copy-Item .\continue\config.yaml "$env:USERPROFILE\.continue\config.yaml"
```

4. Reload VS Code (`Ctrl`+`Shift`+`P` → *Developer: Reload Window*).

### 3. Keep the model in VRAM (context length)

`config.yaml` pins `contextLength: 8192` per model. A machine‑wide `OLLAMA_CONTEXT_LENGTH` would be
requested instead and can push the model into CPU offload, which makes tab completion crawl. It is
currently unset on this machine, but check it after any Ollama tinkering:

```powershell
# check whether one is set (empty output = not set)
[Environment]::GetEnvironmentVariable("OLLAMA_CONTEXT_LENGTH", "User")

# remove it - config.yaml sets contextLength per model instead
[Environment]::SetEnvironmentVariable("OLLAMA_CONTEXT_LENGTH", $null, "User")
```

With `contextLength: 8192`, the ~9 GB model stays fully in VRAM on a 12 GB GPU.

### 4. Verify

```powershell
ollama ps     # after a request: the model should be listed as 100% GPU
```

Type a few lines in a `.py`/`.ts` file and press `Tab` — Continue should complete it inline.

## Autocomplete (Qwen2.5‑Coder 14B)

The inline completion is served by `qwen2.5-coder:14b`, which Continue talks to over Ollama's
`/api/generate` endpoint with a Fill‑in‑the‑Middle prompt.

### Keybindings on *this* machine

**Keymap policy: IntelliJ only.** Two competing keymap extensions were installed at some point. The
Notepad++ one has been **uninstalled** because it re-mapped `Ctrl`+`L` to *Delete Line*, `Ctrl`+`B` to
*Jump to Bracket* and `Ctrl`+`Y` to *Redo*, colliding with Continue and IntelliJ. Do not install another
keymap extension — each one silently overrides the IntelliJ mappings, and there is no `"keymap"` setting
that shows which are active.

| Source | Bindings | Note |
| --- | --- | --- |
| `k--kato.intellij-idea-keybindings` | 220 | IntelliJ mappings, contributed via `package.json` — **not** via a `"keymap"` setting, so nothing in `settings.json` hints at them |
| `ms-vscode.notepadplusplus-keybindings` | 47 | **Uninstalled.** `.obsolete` lists it as `true` and `extensions.json` no longer registers it, so its bindings are gone — the leftover folder is deleted on the next full VS Code restart |
| `continue` | 21 | |
| `%APPDATA%\Code\User\keybindings.json` | 5 | Yours — two entries *remove* Continue defaults, three *rebind* them |

Verified result per action (re-scanned **after** the Notepad++ removal):

| Action | Chord | Status |
| --- | --- | --- |
| Accept the full suggestion | `Tab` | Works — no extension shadows `inlineSuggest.commit` |
| Reject the suggestion | `Esc` | Works (`continue.exitEditMode`, `continue.rejectJump`) |
| Accept word-by-word | `Ctrl`+`→` | Works |
| Force a suggestion now | `Ctrl`+`Alt`+`Space` | Works — unclaimed by every installed extension |
| Toggle tab autocomplete | `Ctrl`+`K` `Ctrl`+`A` | Works — unclaimed |
| Toggle next-edit suggestions | `Ctrl`+`K` `Ctrl`+`N` | Works — unclaimed |
| Open Continue in its own window | `Ctrl`+`K` `Ctrl`+`M` | Works — unclaimed |
| **Inline edit a selection** | **`Shift`+`Alt`+`E`** | Your custom binding. `Ctrl`+`I` **does not work** — you unbound `continue.focusEdit` and IntelliJ maps `Ctrl`+`I` to suggest / code action |
| **Focus chat input** | **`Ctrl`+`L`** | ✅ Works now that the Notepad++ keymap is gone — `Ctrl`+`L` is claimed by Continue alone |
| Focus chat input **without** clearing | **`Shift`+`Alt`+`C`** | Your custom binding. `Ctrl`+`Shift`+`L` is unbound by you and taken by IntelliJ (`selectHighlights`) |
| Accept a chat diff | `Shift`+`Ctrl`+`Enter` | ⚠️ **Still conflicting** — IntelliJ maps it to *Insert Line Below*. Rebind with the snippet below, or click *Apply* in the diff view |
| Reject a chat diff | `Ctrl`+`Z` | ⚠️ **Conflict** with *Undo*. Use *Reject* in the diff view |
| Command Palette | `Ctrl`+`Shift`+`P` or `Ctrl`+`Shift`+`A` | Both work; `Ctrl`+`Shift`+`A` is the IntelliJ *Find Action* |
| Extensions view | `Ctrl`+`Shift`+`X` | ⚠️ **Not bound by any of the 37 installed extensions** — see note below |

`Shift`+`Alt`+`E` collides with `PowerShell.ExpandAlias` inside `.ps1` files only. `Shift`+`Alt`+`C` collides
with IntelliJ's *Copy File Path*, but only when the editor is **not** focused, so it is safe while typing.

> **On `Ctrl`+`Shift`+`X`:** I checked every registered extension's `contributes.keybindings` and none binds,
> removes or shadows `workbench.view.extensions` (Extensions) — so the IntelliJ keymap is *not* what disables
> it. Reliable alternatives: the **Extensions** icon in the activity bar, or `Ctrl`+`Shift`+`A` →
> *Extensions: View Extensions*.

Rebind anything ambiguous via `Ctrl`+`Shift`+`A` → *Preferences: Open Keyboard Shortcuts* (search the
command ID, click the pencil, press the new chord). Add a `{"key": …, "command": …}` entry to
`%APPDATA%\Code\User\keybindings.json` to make it permanent.

Only one genuine conflict is left — *Accept a chat diff*. Append this to `keybindings.json` to move it off
IntelliJ's *Insert Line Below*:

```jsonc
{ "key": "shift+alt+y", "command": "continue.acceptDiff", "when": "continue.diffVisible" },
{ "key": "shift+ctrl+enter", "command": "-continue.acceptDiff" }
```

Commands with no default binding: `continue.newSession`, `continue.viewHistory`, `continue.openConfigPage`,
`continue.viewLogs`, `continue.selectFilesAsContext`, `continue.rebuildCodebaseIndex`.

Why this model is the only one with the `autocomplete` role in [`continue/config.yaml`](continue/config.yaml):

- Ollama ships `qwen2.5-coder:14b` with a native FIM template
  (`{{- if .Suffix }}<|fim_prefix|>…`) and advertises the `insert` capability, so Continue uses the real
  FIM code path — the model gets *prefix + suffix* and returns only the middle.
- `granite4.2:8b` / `llama3.1:8b` have a bare `{{ .Prompt }}` template with no `.Suffix`. Continue falls
  back to pasting a raw `<|fim_prefix|>…<|fim_middle|>` prompt, and the model happily continues *past*
  the insertion point into unrelated prose. Fine for chat, bad for tab completion.
- `contextLength: 8192` keeps the ~9 GB model fully resident on a 12 GB GPU. Requesting the default large
  context would push it into CPU offload.

> Tip: the plain `qwen2.5-coder:7b` tag is a non‑FIM derivative and completes noticeably worse. Use
> `qwen2.5-coder:14b` (or `:32b` on a bigger GPU).

## Chat / prompt (Granite 4.2 8B)

Open the Continue sidebar and pick a chat model in the picker:

- **Granite 4.2 8B** (`granite4.2:8b`) — IBM Granite 4.2, 8.8B params, 128k native context, tool use and
  "thinking" support. Measured ~105 tok/s vs ~65 tok/s for the 14B on this machine, so it is the default
  pick for plain conversation, planning and explaining code.
- **Llama 3.1 8B** (`llama3.1:8b`) — very fast and light; fallback only.
- **Qwen2.5‑Coder 14B** — use when the answer is *code* rather than prose.

Typical prompts:

```text
Explain what this module does and where the state transitions happen.
Write a pytest suite for src/parser.py covering malformed input.
Draft a migration plan from the old REST client to the new SDK.
```

`Shift`+`Alt`+`C` focuses the chat input **without** clearing the previous question; `Shift`+`Alt`+`E` puts the
current selection into inline edit mode (`Esc` leaves it). Add files with `@` in the chat input or run
*continue.selectFilesAsContext* from the Command Palette.

## Cline (agentic chat / file editing)

Cline is the agent-style counterpart to Continue's plain chat: you describe a change, it plans, asks
permission to run commands, and applies multi-file edits with checkpoints.

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
5. **Model Id** → type the exact tag from `ollama list`, e.g. `granite4.2:8b` (fast chat) or
   `qwen2.5-coder:14b` (code/editing). Cline can also fetch the list from `http://localhost:11434/api/tags`.
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

Point Cline at the same 14B model Continue uses for autocomplete, so the code it writes matches the code it
will be completing. Granite 4.2 8B is faster for planning and explanation; the 14B is better for the actual
edits.

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
| Very slow first reply | Model not loaded yet; run `ollama run qwen2.5-coder:14b` once to warm it |
| Replies get slower as the task grows | Enable **Use Compact Prompt**; start a new task when context fills up |
| Cline ignores project files | Add the folder to Cline's approved working directory |

## Cloud models: Cline Free and ClinePass

Local Ollama is the default, but Cline can also reach hosted models. Two of those providers are worth
knowing about. **Pricing and model lineups change often — verify in the model picker before trusting a
number here.** Source: [Cline Free](https://docs.cline.bot/getting-started/free-models) /
[ClinePass](https://docs.cline.bot/getting-started/clinepass).

### Cline Free

Rotating, limited-time promotions on select models, at no cost up to a quota. Any account can use them.

- Models appear tagged **FREE** in the picker under both the **Cline** and **ClinePass** providers.
- The promotion set changes over time, so treat the list below as "currently referenced", not a promise.
- Not available through the Cline API — IDE extension and CLI only.
- Free-tier prompts may be used to improve model quality.
- When the quota runs out, requests fail; nothing silently bills you.

Currently referenced in your Cline `4.1.22` bundle:

| Model ID | Notes |
| --- | --- |
| `cline-free/deepseek-v4.1-flash` | Same weights as the ClinePass Flash tier — good default |
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
| Inline autocomplete, quick edits | `qwen2.5-coder:14b` (local, `Tab`) |
| Chat, planning, explanations | `granite4.2:8b` (local) |
| Multi-file refactor, long context | `cline-pass/glm-5.3`, or `cline-pass/qwen3.7-plus` past 256 K |
| Hardest reasoning, budget allows | `cline-pass/kimi-k3` |
| ClinePass quota spent | `cline-free/deepseek-v4.1-flash` |
| Private code that must not leave the machine | Local Ollama only |

Switching model mid-task is free on the local side, so there is no reason to burn ClinePass quota on
something a local 8 B model handles.

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

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| No completions on `Tab` | Continue not installed/enabled, or Ollama unreachable | `ollama list` must show the model; reload the window; check the Continue output view |
| Completions feel sluggish | CPU offload | `ollama ps` — if not 100% GPU, lower `contextLength` or clear `OLLAMA_CONTEXT_LENGTH` |
| Completion continues into prose | A non-FIM model holds the `autocomplete` role | Only `qwen2.5-coder:14b` supports FIM; keep `autocomplete` off Granite/Llama |
| Ollama "not enough memory" | Model + context exceeds VRAM | Close other GPU apps, or keep `contextLength: 8192` |
| Chat replies are generic | Fast chat model selected for a code question | Switch to *Qwen2.5‑Coder 14B* in the model picker |
| Semantic search returns nothing | Embed role points at the built-in embedder | Select *Nomic Embed v2 MoE*, then rebuild the index |
| Cline cannot reach Ollama | Wrong Base URL | Use `http://localhost:11434`, then `ollama serve` |

## Daily workflow

1. `ollama serve` already runs as a service — nothing to launch.
2. Code in VS Code; `Tab` accepts a Qwen2.5‑Coder completion, `Ctrl`+`Alt`+`Space` forces one.
3. `Ctrl`+`L` for chat, `Shift`+`Alt`+`E` to edit a selection. Avoid `Shift`+`Ctrl`+`Enter` —
   IntelliJ maps it to *Insert Line Below*.
4. Cline when the change spans several files or needs commands run.
5. `ollama ps` first whenever something feels slow — note it lists only *loaded* models, so an empty result
   just means nothing has been requested yet.

## FAQ

**Do I need GitHub Copilot or a cloud API key?**
No. All four models run through Ollama on localhost.

**Can I run this on CPU only?**
Yes, but the 14B model will be slow for tab completion — expect seconds instead of milliseconds.

**Why not use one model for everything?**
Granite 4.2 8B is ~1.6× faster for prose and Qwen2.5‑Coder 14B is much better at code, especially with
FIM. Splitting the roles gets both.

**How do I add a model?**
Add an entry under `models:` in `config.yaml`, run `ollama pull <tag>`, then select it in the model picker.

**Where is Cline's config stored?**
Under `%USERPROFILE%\.cline\data\` (see *Where Cline stores things* above) — but use the in-editor settings
gear, since the files are rewritten whenever a setting changes.

