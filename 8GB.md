# Hardware profile: 8 GB VRAM

Reference card: **NVIDIA GeForce RTX 4060** (8 GB). This page is **derived, not measured** — it was written
from VRAM arithmetic and model sizes published by Ollama, on a machine that has a 12 GB card. Treat the layout
below as a starting point and confirm it with `ollama ps` as described in *Verify*.

The general setup — installing Ollama, Continue and Cline, and the keybindings — lives in
[`README.md`](README.md). This file only covers what depends on how much VRAM you have.

## The one change that matters

**`qwen2.5-coder:14b` does not fit.** It is 9.0 GB on disk, which already exceeds the whole card before any KV
cache is allocated. Ollama will either refuse to load it or split it across system RAM, and in the second case tab
completion crawls.

Everything else follows from that: drop to the 7 B model for autocomplete and let it keep the FIM template.

| Model | Size on disk | Fits in 8 GB? | Notes |
| --- | --- | --- | --- |
| `qwen2.5-coder:14b` | 9.0 GB | **No** | Exceeds the card on its own |
| `qwen2.5-coder:7b` | 4.7 GB | Yes | The `autocomplete` model here |
| `ornith-1.5:9b` | 6.6 GB | Yes, but not alongside the 7 B | Chat; tightest fit on this card |
| `nomic-embed-text-v2-moe:latest` | 957 MB | Yes | Embeddings |

7.62B parameters, Q4_K_M quantization, Apache 2.0.

## Model set

```powershell
ollama pull qwen2.5-coder:7b               # ~4.7 GB - autocomplete
ollama pull ornith-1.5:9b                  # ~6.6 GB - chat
ollama pull nomic-embed-text-v2-moe:latest # ~1 GB - embeddings
ollama list                                # verify
```

Budget **~12 GB of disk**. `ornith-1.5:9b` is the one to drop first if space is tight — Cline falls back to the 7 B for
chat without any other change.

`ornith-1.5:9b` at 6.6 GB is the largest model on this card that still fits, and 8 GB of VRAM is the limit that
matters: the weights alone leave roughly 1.4 GB for the KV cache. Give the chat model a small `contextLength` for that
reason — see *Context length* below.

Ollama keeps one model loaded at a time by default and swaps as needed. A 4.7 GB autocomplete model plus a 6.6 GB
chat model does not fit in 8 GB simultaneously, so expect the first token of a chat reply to pause while the chat
model loads. That pause is normal on this card and is not a misconfiguration.

## Edit `config.yaml`

[`continue/config.yaml`](continue/config.yaml) is written for the 12 GB card. Two edits are required:

1. Point the `autocomplete` role at `qwen2.5-coder:7b`.
2. Drop `contextLength` to `4096`, leaving more VRAM for the weights.

```yaml
  - name: Qwen2.5-Coder 7B
    provider: ollama
    model: qwen2.5-coder:7b
    contextLength: 4096
    roles:
      - autocomplete
```

The `ornith-1.5:9b` entry carries `contextLength: 8192` from the 12 GB config. Lower it to `4096` here too — at 6.6 GB
that model is the marginal fit on this card, and the KV cache is what tips it into offload.

Do not leave `contextLength` unset on this card. Ollama would then request a large default context on top of the
weights, and the KV cache is what tips a marginal fit into offload. That matters more than usual here: `ornith-1.5:9b`
declares a 262144 native context, so an unset length can ask for a window no 8 GB card could ever hold.

### The 7 B model keeps FIM

This matters, because the 7 B tag is a genuine fill-in-the-middle model, not a cut-down derivative. Ollama ships
`qwen2.5-coder:7b` with the same native template as the 14 B:

```text
{{- if .Suffix }}<|fim_prefix|>{{ .Prompt }}<|fim_suffix|>{{ .Suffix }}<|fim_middle|>
```

So `autocomplete` keeps working exactly as it does on the larger card — you get real *prefix + suffix* in, and only
the middle out, instead of a raw-prompt fallback that runs past the insertion point into prose. Autocomplete quality
is lower than the 14 B, but the failure mode of a non-FIM model is avoided entirely.

## Context length

Clear any machine-wide override, for the same reason as on the larger card:

```powershell
# check whether one is set (empty output = not set)
[Environment]::GetEnvironmentVariable("OLLAMA_CONTEXT_LENGTH", "User")

# remove it - config.yaml sets contextLength per model instead
[Environment]::SetEnvironmentVariable("OLLAMA_CONTEXT_LENGTH", $null, "User")
```

## Verify

This is the step that tells you whether the layout above is right, because it is the only one that reports the real
split:

```powershell
ollama ps
```

- **100% GPU** — good, tab completion will be fast.
- **A CPU percentage beside the model** — too tight. Drop `contextLength` to `2048`, or unload the chat model with
  `ollama stop ornith-1.5:9b` and check again.

`ollama ps` lists only *loaded* models, so an empty result just means nothing has been requested yet. Type a few
lines in a `.py`/`.ts` file and press `Tab`.

## Cline

Point Cline at `ornith-1.5:9b` for planning, explanation and edits. The 14 B is the better code model, but it will
not load here — see the top of this page — and it does nothing but autocomplete anyway.

Keep **Use Compact Prompt** enabled. On a card this size, context growth is the first thing to push a request into
offload.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Ollama "not enough memory" on startup | The 14 B model is configured | Switch `autocomplete` to `qwen2.5-coder:7b` |
| Completions feel sluggish | CPU offload | `ollama ps` shows a CPU split — lower `contextLength` to `2048` |
| Chat is slow to start | Model swap, not a fault | 4.7 GB + 6.6 GB cannot stay resident together; this is expected |
| Completion continues into prose | A non-FIM model holds `autocomplete` | Keep `autocomplete` on `qwen2.5-coder:7b`, not the chat model |

## Related

- [`12GB.md`](12GB.md) — the same setup with a card that fits the 14 B model.
- [`README.md`](README.md) — the general, hardware-independent setup.