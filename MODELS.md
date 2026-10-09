### ClinePass

Flat **$9.99/month**, giving **2–5× the usage** of standard API rates on a curated set of open coding
models. It exists because standard rate limits throttle exactly the workloads Cline performs — many turns
of file reads, command runs and edits.

It is a **separate provider** from Cline (usage-billing); you can hold both.

| Model ID                                | Input / Output per 1M                         | Best for                                           |
| --------------------------------------- | --------------------------------------------- | -------------------------------------------------- |
| `cline-pass/glm-5.3`                    | $1.40 / $4.40                                 | Strong all-rounder; good default for refactors     |
| `cline-pass/glm-5.3-flash`              | $0.15 / $0.50                                 | Cheap, fast — planning, chat, small edits          |
| `cline-pass/kimi-k3`                    | $3.00 / $15.00                                | Most expensive here; reserve for hard tasks        |
| `cline-pass/deepseek-v4-pro`            | $1.32 / $3.96 peak, $0.66 / $1.98 off-peak    | Peak/off-peak pricing — schedule big runs off-peak |
| `cline-pass/deepseek-v4.1-flash`        | $0.30 / $1.20                                 | Cheap workhorse                                    |
| `cline-pass/mimo-v2.5`                  | $0.14 / $0.28                                 | Cheapest non-flash tier                            |
| `cline-pass/mimo-v2.5-pro`              | $1.74 / $3.48                                 |                                                    |
| `cline-pass/minimax-m3`                 | $0.30 / $1.20                                 |                                                    |
| `cline-pass/muse-spark-1.3-contributor` | $0.10 / $0.20                                 | Cheapest overall                                   |
| `cline-pass/qwen3.7-max`                | $2.50 / $7.50                                 |                                                    |
| `cline-pass/qwen3.7-plus`               | $0.40 / $1.60 up to 256 K, then $1.20 / $4.80 | **Large context** — the pick for big refactors     |

Those are _reference_ prices showing how usage is measured against your quota; you are not billed per token.

Three limits apply: a **5-hour rolling window**, **weekly**, and **monthly**. Check the dashboard at
[app.cline.bot](https://app.cline.bot/dashboard/subscription?personal=true).

> **Deprecated and gone:** GLM-5.2, Kimi K2.6, Kimi K2.7 Code, DeepSeek V4 Flash. Your local Cline bundle
> still references `cline-pass/glm-5.2` and `cline-pass/mimo-v2.6-flash` / `-pro`, so it is newer than some
> docs — trust the picker over both.

**Recommended split across the whole setup:**

| Situation                                    | Model                                                                             |
| -------------------------------------------- | --------------------------------------------------------------------------------- |
| Inline autocomplete (`Tab`)                  | The FIM code model from your [hardware profile](12GB.md) — the only role it holds |
| Chat, planning, edits, explanations          | `ornith-1.5:35b` (local)                                                          |
| Multi-file refactor, long context            | `cline-pass/glm-5.3`, or `cline-pass/qwen3.7-plus` past 256 K                     |
| Hardest reasoning, budget allows             | `cline-pass/kimi-k3`                                                              |
| ClinePass quota spent                        | `ollama-cloud/gemma4`                                                             |
| Private code that must not leave the machine | Local Ollama only                                                                 |

Switching model mid-task is free on the local side, so there is no reason to burn ClinePass quota on
something a local 9 B model handles.

ClinePass models are also usable outside Cline via the [Cline API](https://docs.cline.bot/api/overview)
(OpenAI-compatible Chat Completions, same `cline-pass/...` slug in the `model` field).