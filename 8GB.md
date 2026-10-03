# Hardware profile: 8 GB VRAM

Reference card: **NVIDIA GeForce RTX 4060** (8 GB, laptop). This page is **derived, not measured** — it was written
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
| `granite4.2:8b` | 5.3 GB | Yes, but not alongside the 7 B | Chat |
| `llama3.1:8b` | 4.9 GB | Yes, but not alongside the 7 B | Fallback chat |
| `nomic-embed-text-v2-moe:latest` | 957 MB | Yes | Embeddings |

7.62B parameters, Q4_K_M quantization, Apache 2.0.

## Model set

```powershell
ollama pull qwen2.5-coder:7b               # ~4.7 GB - autocomplete
ollama pull granite4.2:8b                  # ~5.3 GB - chat
ollama pull nomic-embed-text-v2-moe:latest # ~1 GB - embeddings
ollama list                                # verify
```

Budget **~11 GB of disk**. `llama3.1:8b` is optional on this card — it is a fallback for chat, and `granite4.2:8b`
already covers that. Add it only if you want the second option.

Ollama keeps one model loaded at a time by default and swaps as needed. A 4.7 GB autocomplete model plus a 5.3 GB
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

Do not leave `contextLength` unset on this card. Ollama would then request a large default context on top of the
weights, and the KV cache is what tips a marginal fit into offload.

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
  `ollama stop granite4.2:8b` and check again.

`ollama ps` lists only *loaded* models, so an empty result just means nothing has been requested yet. Type a few
lines in a `.py`/`.ts` file and press `Tab`.

## Cline

Point Cline at `granite4.2:8b` for planning and explanation, and at `qwen2.5-coder:7b` for the actual edits. The
14 B is the better editing model, but it will not load here — see the top of this page.

Keep **Use Compact Prompt** enabled. On a card this size, context growth is the first thing to push a request into
offload.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Ollama "not enough memory" on startup | The 14 B model is configured | Switch `autocomplete` to `qwen2.5-coder:7b` |
| Completions feel sluggish | CPU offload | `ollama ps` shows a CPU split — lower `contextLength` to `2048` |
| Chat is slow to start | Model swap, not a fault | 4.7 GB + 5.3 GB cannot stay resident together; this is expected |
| Completion continues into prose | A non-FIM model holds `autocomplete` | Keep `autocomplete` on `qwen2.5-coder:7b`, not Granite or Llama |

## Related

- [`12GB.md`](12GB.md) — the same setup with a card that fits the 14 B model.
- [`README.md`](README.md) — the general, hardware-independent setup.