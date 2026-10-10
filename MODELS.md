### ClinePass

Flat **$9.99/month**, giving **2–5× the usage** of standard API rates on a curated set of open coding

models. It exists because standard rate limits throttle exactly the workloads Cline performs — many turns

of file reads, command runs and edits.

It is a **separate provider** from Cline (usage-billing); you can hold both. It is the flat-payment option for

**heavy** usage — for light use, free + pay-per-use is usually cheaper (see _Which model for which task_ below).

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

| Situation | Best layer | Why |
| --- | --- | --- |
| Inline autocomplete (Tab) | Local qwen2.5-coder:14b (free) | The only local FIM model; the only role that needs a native FIM template |
| Everyday chat, planning, edits, explanations | Local ornith-1.5:35b (free) | Free, unlimited, private; on the card and 65K. Good quality per cent |
| Everyday chat when local is slow or cold | nemotron-3-super:cloud via Ollama | Same quality band as local 35 B, much faster to respond; runs under free credits for me |
| Long input (whole codebases, big specs) | glm-5.3-flash:cloud via Ollama | 1M context at near-chat cost; 35B alone hits its size limit there |
| Hard reasoning, deep debugging | DeepSeek V4 Pro | Best price/performance for pure reasoning; reach for it when local 35B shows its size |
| Visual/theme/UX work (screenshot in) | Ollama multimodal cloud (glm-5.3-flash:cloud, gemma4:cloud) | Natively multimodal; 35B sees no images |
| Private code that must not leave the machine | Local Ollama only | Nothing leaves the box |
| Heavy, quota-burning usage | ClinePass | See below — the flat tier for sustained, high-turn workloads |
**The priority order (quality over speed, cost into account).** Start every task on the **free local** models.

There is nothing to decide and nothing to spend, so use free as long as it is good enough. Graduate only when a job genuinely outgrows 35 B:

1. **Local, free first.** ornith-1.5:35b (chat / edit / apply) and qwen2.5-coder:14b (autocomplete) cover the bulk of any work — including most documentation, maker and web tasks — at zero cost. Switching mid-task is free on this side, so there is no reason to burn quota on what 35 B handles.

2. **Ollama cloud — for big input, or just to be faster.** Big input: when the input is too large for the 65 K local context, glm-5.3-flash:cloud (1 M context) is the cheapest high-capacity option. But big input is not the only reason to leave the local models — for speed, and at least the local quality without burning pay-per-use, nemotron-3-super:cloud is the faster drop-in (see the note below). Spend your ollama.com credits here and on that fallback.

> **Free credits.** Ollama's Free plan gives you a starter amount of usage for a limited set of "starter" models; buying usage credits unlocks every model. Which models are free-eligible for your account isn't public — it depends on your account; when a cloud model isn't free-eligible, Ollama bills you. A practical rule of thumb: confirm the model runs under free credits once, then reuse it freely.

> **Speed vs. quality, the real question.** `ornith-1.5:35b` is the quality reference locally, but it is slow and cold-start-sensitive — if it is not already loaded, Cline can time out and fall back to the last model used, and even `ollama run ornith-1.5:35b` to prewarm it takes a while. So the question is which free-eligible cloud model matches its quality at a fraction of the wait. For me, **`nemotron-3-super:cloud` (12B active / 120B MoE, ~256K context)** is the answer: strong on SWE-Bench and LiveCodeBench, and it delivers at least the quality of local 35 B while being noticeably faster.

> Don't waste free credits on the pricey ones. Off the cheaper general-purpose options (`glm-5.3-flash:cloud`, `gpt-oss:20b-cloud`, `gemma4:cloud`) and `nemotron-3-nano:30b-cloud`, reach for the heavier models (`nemotron-3-ultra:cloud`, output $3.00/1M) only when you actually need that much capability.

3. **DeepSeek for hard reasoning.** When a problem needs stronger reasoning than 35 B offers and you want the best price/performance — DeepSeek V4 Pro (and its cheaper off-peak Flash tier). Note: off-peak hours vary between BYOK providers like DeepSeek/Mistral and Ollama cloud models; check each provider's documentation for exact timing.

4. **ClinePass only for heavy usage.** It is a flat 9.99 US$/month, gives 2–5× the usage of standard rates, and its limits (5-hour rolling window, weekly, monthly) only bite under sustained high volume. For light use, free + pay-per-use is almost always cheaper; hold it as the tier you activate when you are doing long, quota-heavy runs and the quota window, not the pay-per-token, is what you care about.

**Per-task guidance (the three you work with).**

| Task | Default | When to graduate |
| --- | --- | --- |
| **1. Technical documentation** (specs, datasheets, this kind of work) | Local ornith-1.5:35b — free, good style match for prose; slow or cold-start → nemotron-3-super:cloud, same quality band but much faster | A datasheet or spec too large for 65 K -> glm-5.3-flash:cloud (1 M); exact citations that must be verifiable -> DeepSeek V4 Pro |
| **2. Maker — Arduino / ESP32 + peripherals** | Local ornith-1.5:35b + qwen2.5-coder:14b (firmware FIM); when 35 B is slow to warm → nemotron-3-super:cloud for the same quality at a fraction of the wait | A register- or timing-level bug that needs deeper reasoning -> DeepSeek V4 Pro, or glm-5.3-flash:cloud reading a datasheet. Local 35 B can lose coherence across very large diffs even at 65 K — that is the one case that warrants offloading |
| **3. WordPress site** (theme/plugin, e.g. a sports club) | Local ornith-1.5:35b — free, multi-file PHP/CSS/HTML; local slow or cold → nemotron-3-super:cloud for same-quality output at faster speed | A larger feature -> glm-5.3-flash:cloud (1 M to hold the site + its structure); the visual/theme side -> a multimodal cloud model (screenshot in); heavy WooCommerce logic -> DeepSeek V4 Pro, or cline-pass/kimi-k3 if you want max quality and have quota |
Note on BYOK models: Mistral Large 4 (1 M context) is another strong option for complex reasoning and multilingual tasks, available via DeepSeek-compatible APIs or directly from Mistral. Its off-peak pricing also applies (see README.md for details).

ClinePass models are also usable outside Cline via the [Cline API](https://docs.cline.bot/api/overview)

(OpenAI-compatible Chat Completions, same `cline-pass/...` slug in the `model` field).