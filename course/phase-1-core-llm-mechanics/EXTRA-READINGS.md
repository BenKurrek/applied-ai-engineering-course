# Phase 1 — Core LLM Mechanics — Extra Readings

A curated "further reading" list for Phase 1. These resources go deeper than the course
material. They are grouped by topic. Every URL was verified to resolve at the time of
writing (May 2026); primary sources (papers, official docs, vendor pages) are preferred
over secondary commentary.

---

## Topic 1 — LLM Fundamentals (transformers, training/inference, attention, generation)

- **Attention Is All You Need** — Vaswani et al., 2017 (arXiv) —
  https://arxiv.org/abs/1706.03762
  The original transformer paper. Read it for the encoder/decoder architecture,
  scaled dot-product attention, and multi-head attention as first defined.

- **The Illustrated Transformer** — Jay Alammar, 2018 —
  https://jalammar.github.io/illustrated-transformer/
  The most widely used visual walkthrough of the transformer data flow; an excellent
  companion to sub-chapter 1.3's "whiteboard the data flow" framing.

- **Language Models are Few-Shot Learners (GPT-3)** — Brown et al., 2020 (arXiv) —
  https://arxiv.org/abs/2005.14165
  The scaling paper that made next-token prediction's emergent capabilities concrete;
  grounds sub-chapter 1.1's "emergent capability" claim in primary evidence.

- **FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness** —
  Dao et al., 2022 (arXiv) — https://arxiv.org/abs/2205.14135
  Explains precisely how exact attention is made faster and O(n) in memory without
  changing the O(n²) arithmetic — the nuance behind sub-chapter 1.4.

- **flash-attention (reference implementation)** — Dao-AILab (GitHub) —
  https://github.com/dao-ailab/flash-attention
  The maintained code; useful if you want to see how prefill/decode kernels are
  actually structured.

- **A Survey on Efficient Inference for Large Language Models** — Zhou et al., 2024
  (arXiv) — https://arxiv.org/abs/2404.14294
  A broad map of prefill/decode, KV caching, batching, and quantization — the systems
  context behind sub-chapter 1.5.

---

## Topic 2 — Tokenization (BPE, vocabularies, special tokens, tokenizer bugs)

- **Let's build the GPT Tokenizer** — Andrej Karpathy, 2024 (YouTube, ~2h13m) —
  https://www.youtube.com/watch?v=zduSFxRajkE
  Builds a byte-level BPE tokenizer from scratch and demonstrates number splitting,
  whitespace quirks, and glitch tokens live — the best single resource for Topic 2.

- **tiktoken** — OpenAI (GitHub) — https://github.com/openai/tiktoken
  The fast BPE tokenizer for OpenAI models; run it to see real token IDs and confirm
  the "tokens ≠ words" and whitespace-folding claims yourself.

- **Neural Machine Translation of Rare Words with Subword Units** — Sennrich et al.,
  2016 (arXiv) — https://arxiv.org/abs/1508.07909
  The paper that introduced BPE to NLP; the primary source for sub-chapter 2.2.

- **SentencePiece: A simple and language independent subword tokenizer** — Kudo &
  Richardson, 2018 (arXiv) — https://arxiv.org/abs/1808.06226
  Primary source for sub-chapter 2.3 — explains the library, the `▁` meta-symbol, and
  why reversibility/language-agnosticism matter.

- **Subword Regularization (the Unigram LM tokenizer)** — Kudo, 2018 (arXiv) —
  https://arxiv.org/abs/1804.10959
  The Unigram language-model approach to tokenization and probabilistic segmentation
  referenced in sub-chapter 2.3.

- **SolidGoldMagikarp (plus, prompt generation)** — Rumbelow & Watkins, 2023
  (LessWrong / AI Alignment Forum) —
  https://www.alignmentforum.org/posts/aPeJE8bSo6rAFoLqg/solidgoldmagikarp-plus-prompt-generation
  The original write-up of glitch tokens — the primary source behind sub-chapter 2.6.

- **Introducing Meta Llama 3** — Meta AI, 2024 —
  https://ai.meta.com/blog/meta-llama-3/
  Vendor source for the move to a 128K-token byte-level BPE tokenizer; useful context
  for vocabulary-size trade-offs (2.4).

---

## Topic 3 — Inference & Sampling (temperature, top-p/top-k, decoding, logprobs)

- **The Curious Case of Neural Text Degeneration** — Holtzman et al., 2020 (arXiv) —
  https://arxiv.org/abs/1904.09751
  Introduces nucleus (top-p) sampling and explains why the highest-probability
  sequence is bland — the primary source for sub-chapters 3.2 and 3.3.

- **Anthropic Messages API reference** — Anthropic (Claude API docs) —
  https://platform.claude.com/docs/en/api/messages
  The authoritative list of which sampling knobs Anthropic exposes (`temperature` 0–1,
  `top_p`, `top_k`; no penalties; the temperature/top_p mutual-exclusion rule).

- **OpenAI — Create chat completion API reference** — OpenAI —
  https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create
  The counterpart for OpenAI's knobs (`temperature` 0–2, `top_p`, frequency/presence
  penalties −2 to 2); read both to see how provider-specific the sampling surface is.

- **Reproducible outputs with the seed parameter** — OpenAI Cookbook —
  https://developers.openai.com/cookbook/examples/reproducible_outputs_with_the_seed_parameter
  Shows the `seed` parameter and `system_fingerprint` in practice and states plainly
  that determinism is best-effort — the backbone of sub-chapter 3.7.

- **Defeating Nondeterminism in LLM Inference** — Thinking Machines Lab, 2025 —
  https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/
  A deep, well-regarded analysis of why even `temperature=0` LLM inference is not
  bit-exact (batch-invariance, reduction order) — extends sub-chapter 3.7 considerably.

---

## Topic 4 — Context Window & the LLM Turn (context engineering, cost, memory)

- **Lost in the Middle: How Language Models Use Long Contexts** — Liu et al., 2023
  (TACL; arXiv) — https://arxiv.org/abs/2307.03172
  The primary source for the U-shaped "lost in the middle" curve in sub-chapter 4.4.

- **Effective context engineering for AI agents** — Anthropic Applied AI team, 2025 —
  https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
  Anthropic's own treatment of context as a finite resource, "context rot," and
  curation strategies — directly parallels sub-chapters 4.3–4.5.

- **Effective harnesses for long-running agents** — Anthropic, 2026 —
  https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
  How real long-running agents bridge context windows with summaries and artifacts —
  the practical endgame of sub-chapter 4.7.

- **Context windows** — Anthropic (Claude API docs) —
  https://platform.claude.com/docs/en/build-with-claude/context-windows
  The authoritative reference on current Claude context windows, context awareness,
  extended-thinking token accounting, and overflow behavior.

- **Pricing** — Anthropic (Claude API docs) —
  https://platform.claude.com/docs/en/about-claude/pricing
  The source of truth for input/output rates and prompt-caching multipliers; pair it
  with sub-chapter 4.6's token-accounting math.

- **Prompt caching** — Anthropic (Claude API docs) —
  https://platform.claude.com/docs/en/build-with-claude/prompt-caching
  The exact-prefix-match rules, cache TTLs, and what invalidates a cache — the bridge
  from sub-chapter 2.7 into Topic 6.
