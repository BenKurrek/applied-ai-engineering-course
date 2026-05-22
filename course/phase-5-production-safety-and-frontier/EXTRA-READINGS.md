# Phase 5 — Extra Readings

> Curated further reading for Phase 5 (Production, Safety & Frontier). These go deeper
> than the course material. All URLs verified May 2026. Topics 14–15 are time-sensitive
> — model/vendor specifics in those readings will age; the underlying ideas will not.

---

## Topic 12 — Production Concerns

- **Efficient Memory Management for LLM Serving with PagedAttention** — Kwon et al.
  (arXiv:2309.06180) — https://arxiv.org/abs/2309.06180
  The original vLLM/PagedAttention paper; the definitive primary source on the KV-cache
  memory management that makes continuous batching practical.

- **Exponential Backoff And Jitter** — Marc Brooker, AWS Architecture Blog —
  https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
  The canonical short article on why fixed-interval retry causes thundering herds and
  how full/equal/decorrelated jitter compare — with simulation data.

- **Anthropic — Prompt caching** (Claude API docs) —
  https://platform.claude.com/docs/en/build-with-claude/prompt-caching
  Primary source on how provider-side prompt (KV) caching works, cache-write vs.
  cache-read pricing multipliers, breakpoints, and TTL behavior.

- **Anthropic — Streaming Messages** (Claude API docs) —
  https://platform.claude.com/docs/en/build-with-claude/streaming
  The exact SSE event types (`message_start`, `content_block_delta`,
  `input_json_delta`, `message_delta`, errors) — grounds the partial-parsing material.

- **OpenAI — Rate limits** (API docs) —
  https://developers.openai.com/api/docs/guides/rate-limits
  Vendor reference on RPM/TPM/RPD limits, 429 handling, and the cookbook's
  backoff-with-jitter examples — a second provider's perspective on 12.5.

- **Release It! (2nd ed.), Michael Nygard — chapters on stability patterns** —
  https://pragprog.com/titles/mnee2/release-it-second-edition/
  The standard reference for circuit breakers, bulkheads, timeouts, and fail-fast — the
  reliability patterns 12.6 applies to LLM dependencies.

---

## Topic 13 — Safety, Guardrails & Adversarial

- **OWASP Top 10 for LLM Applications (2025)** — OWASP —
  https://owasp.org/www-project-top-10-for-large-language-model-applications/
  The industry-standard taxonomy of LLM risks; LLM01 (prompt injection) and LLM02
  (sensitive information disclosure) map directly onto this topic.

- **The lethal trifecta for AI agents** — Simon Willison —
  https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
  The original framing of the threat model in 13.6, with real-world exfiltration
  examples; Willison's `lethal-trifecta` tag collects ongoing case studies.

- **Constitutional Classifiers: Defending against universal jailbreaks** — Anthropic —
  https://www.anthropic.com/research/constitutional-classifiers
  Primary source on classifier-based guardrails as a separate layer; includes
  red-teaming results and the input/output classifier architecture.

- **Many-shot jailbreaking** — Anthropic —
  https://www.anthropic.com/research/many-shot-jailbreaking
  The research that named the long-context attack in 13.2 and showed attack success
  scaling with the number of in-context shots.

- **Universal and Transferable Adversarial Attacks on Aligned Language Models** —
  Zou et al. (arXiv:2307.15043) — https://arxiv.org/abs/2307.15043
  The GCG paper behind "adversarial suffixes" — gradient-optimized strings that
  transfer across models.

- **Prompt Shields in Azure AI Content Safety** — Microsoft Learn —
  https://learn.microsoft.com/en-us/azure/ai-services/content-safety/concepts/jailbreak-detection
  A concrete vendor implementation of injection/jailbreak detection covering both user
  prompt attacks and document (indirect) attacks — a worked example of a mitigation
  layer.

---

## Topic 14 — Model Training Landscape

- **Constitutional AI: Harmlessness from AI Feedback** — Bai et al. (arXiv:2212.08073) —
  https://arxiv.org/abs/2212.08073
  The primary source for CAI and RLAIF-style harmlessness training.

- **Direct Preference Optimization** — Rafailov et al. (arXiv:2305.18290) —
  https://arxiv.org/abs/2305.18290
  The DPO paper; explains precisely how the reward model and RL loop are folded into a
  single classification-style loss.

- **LoRA: Low-Rank Adaptation of Large Language Models** — Hu et al. (arXiv:2106.09685) —
  https://arxiv.org/abs/2106.09685
  The original low-rank-adaptation insight and evidence that fine-tune updates are
  low-rank.

- **QLoRA: Efficient Finetuning of Quantized LLMs** — Dettmers et al.
  (arXiv:2305.14314) — https://arxiv.org/abs/2305.14314
  NF4, double quantization, and paged optimizers — the techniques behind single-GPU
  fine-tuning of 65B-class models.

- **DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via RL** — DeepSeek-AI
  (arXiv:2501.12948) — https://arxiv.org/abs/2501.12948
  The open reasoning-model paper; shows RL on verifiable rewards (GRPO) producing
  reasoning behavior with minimal SFT.

- **YaRN: Efficient Context Window Extension** — Peng et al. (arXiv:2309.00071) —
  https://arxiv.org/abs/2309.00071
  The primary source for non-uniform RoPE-frequency scaling and context extension.

- **Mixtral of Experts** — Jiang et al. (arXiv:2401.04088) —
  https://arxiv.org/abs/2401.04088
  A clear, well-documented open MoE model paper — concretely illustrates total vs.
  active parameters and top-k routing.

---

## Topic 15 — Model & Ecosystem Landscape

- **Anthropic — Pricing** (Claude API docs) —
  https://platform.claude.com/docs/en/about-claude/pricing
  Authoritative, frequently updated pricing for every Claude model — the right place to
  re-check the time-sensitive numbers in 12.3 and 15.x.

- **Anthropic — Messages API reference** —
  https://platform.claude.com/docs/en/api/messages
  Primary spec for the `system` parameter, content blocks, tool use, and `max_tokens`.

- **OpenAI — Migrate to the Responses API** —
  https://developers.openai.com/api/docs/guides/migrate-to-responses
  Side-by-side Chat Completions vs. Responses; covers typed items, server-side state,
  and built-in tools.

- **OpenAI — Introducing GPT-5.5** —
  https://openai.com/index/introducing-gpt-5-5/
  Vendor announcement for the May-2026 OpenAI flagship — a snapshot of capability
  claims to compare against the material.

- **Google DeepMind — Gemini 3.1 Pro model card** —
  https://deepmind.google/models/model-cards/gemini-3-1-pro/
  Official model card with context-window, multimodality, and benchmark claims.

- **Open LLM Leaderboard** — Hugging Face —
  https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard
  A live view of the open-weights frontier; useful for sanity-checking the
  "open models have narrowed the gap" claim against current data.

---

## Topic 16 — System Design for AI

- **Building effective agents** — Anthropic —
  https://www.anthropic.com/research/building-effective-agents
  The most-cited practical guide on agent design patterns, when to use an agent vs. a
  workflow, and why simpler designs win — directly supports 16.3 and 16.5.

- **Firecracker — Secure and fast microVMs** —
  https://firecracker-microvm.github.io/
  The primary source for the sandboxing/microVM control that is the headline safety
  mechanism for the coding-agent drill (16.3).

- **NIST AI Risk Management Framework (AI RMF 1.0)** — NIST —
  https://www.nist.gov/itl/ai-risk-management-framework
  The govern/map/measure/manage framework — a standards-body lens on the
  requirements-and-safety-first structure of 16.1.

- **Anthropic — Claude Code / agent harness engineering posts** —
  https://www.anthropic.com/engineering
  Engineering write-ups on context management, tool design, and agent harness
  decisions — concrete grounding for the coding-agent and trade-off drills.

- **AI Engineering** — Chip Huyen (O'Reilly, 2024) —
  https://www.oreilly.com/library/view/ai-engineering/9781098166298/
  A book-length treatment of evaluating, deploying, and operating LLM systems — the
  closest single reference to the whole-phase synthesis Topic 16 asks for.
