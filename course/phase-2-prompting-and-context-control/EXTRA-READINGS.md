# Phase 2 — Further Reading

Curated, more in-depth resources for **Phase 2: Prompting and Context Control**. These go
deeper than the course material. URLs verified May 2026; provider docs are living pages,
so treat version- and price-specific details as point-in-time.

---

## Topic 05 — Prompt Engineering

- **Prompting best practices (Claude)** — Anthropic.
  https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/use-xml-tags
  Anthropic's single living reference for prompting current Claude models — clarity,
  examples, XML structuring, thinking, and model-specific tuning notes for Opus 4.7.

- **The Instruction Hierarchy: Training LLMs to Prioritize Privileged Instructions** —
  Wallace et al. (OpenAI), arXiv 2404.13208.
  https://arxiv.org/abs/2404.13208
  The primary source for *why* the system prompt outranks user instructions — explains
  the System > Developer > User > Tool priority training and its limits.

- **Chain-of-Thought Prompting Elicits Reasoning in Large Language Models** —
  Wei et al., arXiv 2201.11903.
  https://arxiv.org/abs/2201.11903
  The foundational CoT paper; grounds the mechanism the material teaches in 5.3 and the
  pre-reasoning-model era it describes.

- **Many-Shot In-Context Learning** — Agarwal et al. (Google DeepMind), arXiv 2404.11018.
  https://arxiv.org/abs/2404.11018
  Extends 5.2's few-shot discussion to the many-shot regime — when hundreds of examples
  override pretraining bias and approach fine-tuning quality.

- **Towards Understanding Sycophancy in Language Models** — Anthropic (Sharma et al.).
  https://www.anthropic.com/research/towards-understanding-sycophancy-in-language-models
  The evidence behind 5.7's sycophancy section — shows sycophancy is a general property
  of RLHF-trained assistants and ties it to human/preference-model judgments.

- **Lost in the Middle: How Language Models Use Long Contexts** — Liu et al.,
  arXiv 2307.03172 (TACL 2024).
  https://arxiv.org/abs/2307.03172
  The empirical basis for the primacy/recency/"lost in the middle" placement advice in
  5.6.

- **Effort (Claude `effort` parameter)** — Anthropic.
  https://platform.claude.com/docs/en/build-with-claude/effort
  Deep detail on the reasoning-effort knob from 5.3 — effort levels, adaptive thinking,
  and how effort replaced fixed `budget_tokens` on the newest models.

- **Migration guide (Claude Opus 4.7 / 4.6)** — Anthropic.
  https://platform.claude.com/docs/en/about-claude/models/migration-guide
  Documents the breaking changes most relevant to 5.5 — assistant-prefill removal and
  the structured-outputs / system-prompt replacements.

---

## Topic 06 — Prompt Caching

- **Prompt caching (Claude API)** — Anthropic.
  https://platform.claude.com/docs/en/build-with-claude/prompt-caching
  The authoritative reference for everything in Topic 6 on the Anthropic side:
  `cache_control`, breakpoints, the 20-block lookback, TTLs, minimum cacheable lengths,
  and how the usage fields are reported.

- **Prompt Caching in the API** — OpenAI.
  https://openai.com/index/api-prompt-caching/
  OpenAI's announcement and explainer for automatic caching — the 1,024-token minimum,
  128-token increment matching, and the contrasting "no breakpoints" developer model.

- **Pricing** — Anthropic.
  https://platform.claude.com/docs/en/about-claude/pricing
  Current per-model rates and the exact prompt-caching multipliers (1.25× / 2× writes,
  0.1× reads) underpinning the pricing math in 6.5.

- **API Pricing** — OpenAI.
  https://developers.openai.com/api/docs/pricing
  Current OpenAI per-model rates — important because the cached-input discount has moved
  from 50% (GPT-4o era) to 90% on current flagship models.

- **Effective context engineering for AI agents** — Anthropic Engineering.
  https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
  Frames caching and prompt structure inside the larger discipline of context
  engineering — compaction, memory, retrieval — useful context for 6.3's "stable-first"
  layout rule.

- **Large Language Models Can Be Easily Distracted by Irrelevant Context** —
  Shi et al., arXiv 2302.00093 (ICML 2023).
  https://arxiv.org/abs/2302.00093
  Reinforces why a *large* cached prefix is not automatically a *good* prefix — extra
  context can degrade quality, a constraint to weigh against caching's cost incentives.
