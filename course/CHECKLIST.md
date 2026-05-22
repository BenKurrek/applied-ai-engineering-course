# Applied AI Engineering — Concept Checklist

A flat, one-page checklist of the core concepts in the course. Each bullet is
something you should be able to **explain out loud in ~60 seconds without notes** —
use it as a quick self-test and revision aid. The full lessons live in each topic's
`material.md`; the complete outline is in `SYLLABUS.md`.

---

## 1. LLM Fundamentals
- [ ] What a transformer is at a block level: embeddings → attention → FFN → unembedding. Explain *why* attention is the core mechanism (no backprop derivation needed).
- [ ] **Self-attention** — queries/keys/values, why it's O(n²) in sequence length, what that means for context cost.
- [ ] **Autoregressive generation** — models predict one token at a time; generation is sequential, prefill is parallel.
- [ ] **Prefill vs. decode** — prefill (process the prompt, parallelizable, compute-bound) vs. decode (generate tokens one-by-one, memory-bandwidth-bound). This distinction explains nearly every latency/cost behavior.
- [ ] **Logits → probabilities** — the softmax over the vocabulary at each step.
- [ ] **Parameters vs. context** — weights are frozen at inference; all "memory" within a request is the context window.
- [ ] Why LLMs are **stateless** between API calls — you resend the full conversation every turn.
- [ ] **Inference-time quantization** and **speculative decoding** — what they are and why they speed up inference.

## 2. Tokenization
- [ ] What a token is; tokens ≠ words (~0.75 words/token in English, worse for code/other languages).
- [ ] **BPE (Byte-Pair Encoding)** — how the merge algorithm builds a vocab; **byte-level BPE** and why it guarantees no out-of-vocab.
- [ ] **WordPiece** vs. BPE vs. **Unigram/SentencePiece** — differences at a high level.
- [ ] Vocabulary size tradeoffs — larger vocab = shorter sequences but bigger embedding matrix.
- [ ] **Special tokens** — BOS/EOS, padding, role/turn delimiters, tool-call tokens.
- [ ] Why tokenization causes bugs: number splitting (arithmetic failures), whitespace sensitivity, the "SolidGoldMagikarp" glitch-token phenomenon, multilingual token inefficiency.
- [ ] Token boundaries and prompt caching interact.

## 3. Inference & Sampling
- [ ] **Temperature** — what it does to the logit distribution (0 = deterministic/greedy, higher = flatter).
- [ ] **top-p (nucleus)** and **top-k** sampling — definitions and when to use each.
- [ ] **Greedy decoding** vs. sampling; why beam search is largely unused for open-ended LLM generation.
- [ ] **Repetition/frequency/presence penalties.**
- [ ] **logprobs** — what they are and how to use them for confidence scoring, classification, and evals.
- [ ] **max_tokens**, stop sequences.
- [ ] Determinism caveats — even temperature 0 isn't perfectly deterministic (floating-point, batching, MoE routing).
- [ ] **Seed** parameters and their limits.

## 4. The Context Window & the "LLM Turn"
- [ ] What a **turn** is — the messages array, roles: `system`, `user`, `assistant`, `tool`.
- [ ] How multi-turn conversations are constructed — you append and resend the whole history.
- [ ] **Context window size** vs. **effective context** — models degrade before the hard limit.
- [ ] **"Lost in the middle"** / context rot — retrieval accuracy drops for info in the middle of long contexts.
- [ ] **Context engineering** — deliberate management of what goes in the window: ordering, compaction, summarization, eviction.
- [ ] Input vs. output token accounting and why output tokens cost more.
- [ ] **Context window management strategies** — sliding window, summarization, retrieval-on-demand.

## 5. Prompt Engineering
- [ ] **System prompt** role and best practices.
- [ ] **Zero-shot / few-shot / many-shot** — and when few-shot examples help vs. hurt.
- [ ] **Chain-of-thought** — and why reasoning models changed this.
- [ ] Structured prompting — XML tags, delimiters, sections.
- [ ] **Prefilling the assistant turn** to constrain output.
- [ ] Instruction placement, recency effects, the "be specific" principle.
- [ ] Common failure modes — instruction conflict, distraction, sycophancy.
- [ ] Prompt versioning and why prompts should be treated as code.
- [ ] **Prompt optimization and meta-prompting** — using models to improve prompts.
- [ ] **Decomposition, prompt chaining, and self-consistency** — splitting a task across calls and sampling multiple paths.

## 6. Prompt Caching
- [ ] **What it is** — caching the model's internal state (KV cache) for a repeated prompt **prefix**.
- [ ] **Prefix-match requirement** — caching only works for an *exact* prefix match; one differing token early in the prompt busts the cache for everything after.
- [ ] **Implication for prompt structure** — put static content (system prompt, tool defs, few-shot examples, large docs) at the *front*; put variable content (user query) at the *end*.
- [ ] **Cache TTL** — Anthropic's is ~5 minutes (refreshed on hit); idle time past TTL = cache miss.
- [ ] **Pricing** — cache writes cost slightly more than base input; cache reads are ~10% of input cost. Be able to do the cost math.
- [ ] When caching does and doesn't pay off (high reuse, long stable prefixes vs. one-off calls).
- [ ] How it interacts with latency — big TTFT win on cache hits.

## 7. Tool Use / Function Calling
- [ ] **Tool definition** — name, description, JSON Schema for parameters.
- [ ] The flow: model emits a `tool_use` block → your code executes → you return a `tool_result` → model continues.
- [ ] **Parallel tool calls** — model requesting multiple tools in one turn.
- [ ] Why the model never executes anything — *you* execute; the model only requests.
- [ ] Tool-result formatting, error handling, returning failures gracefully back to the model.
- [ ] **Tool-choice** controls (auto / required / specific tool / none).
- [ ] Designing good tool descriptions — the description *is* the prompt the model sees.
- [ ] Token cost of tool definitions and why you cache them.
- [ ] **MCP (Model Context Protocol)** — what it is, why it exists (standardized tool/context interface), client/server model.
- [ ] **Code-execution orchestration ("code mode")** — letting the model write code that calls tools instead of emitting tool calls directly.

## 8. Agents & Harnesses
- [ ] What distinguishes an **agent** from a single LLM call — a loop with tools, observations, and a termination condition.
- [ ] The **agentic loop** — think → act (tool call) → observe → repeat until done.
- [ ] **ReAct** (reason + act) pattern.
- [ ] **Planning** approaches — upfront plan vs. interleaved; decomposition; subagents.
- [ ] **Memory** — short-term (context) vs. long-term (external store); when and how to summarize.
- [ ] **The harness** — everything around the model: the loop, tool dispatch, context management, retries, guardrails, logging, the termination logic. Be able to whiteboard one.
- [ ] **Multi-agent systems** — orchestrator/worker, handoffs, when multi-agent helps vs. adds failure modes.
- [ ] Failure modes — infinite loops, context overflow, error cascades, goal drift, tool misuse.
- [ ] **Termination/stopping conditions** and step/budget limits.
- [ ] Human-in-the-loop checkpoints.
- [ ] Evaluating agents — trajectory eval vs. final-outcome eval.
- [ ] **The cost model of a growing trajectory** — why each step gets more expensive as context accumulates.
- [ ] **Context compaction** mechanics, and agent **observability** — trace and span structure.

## 9. Evaluations
- [ ] **Offline vs. online** evals; **golden/eval datasets** and how to build them.
- [ ] **Regression testing** for prompts/models — why every prompt change needs an eval.
- [ ] Metric types — exact match, F1, **pass@k**, BLEU/ROUGE (and their weaknesses), task-specific metrics.
- [ ] **LLM-as-judge** — single-output scoring vs. **pairwise comparison**; why pairwise is more reliable.
- [ ] **Rubric-based evaluation** — decomposing quality into scored dimensions.
- [ ] **Judge-model ensembles / panels** — why multiple judges reduce bias.
- [ ] Known **LLM-judge biases** — position bias, verbosity bias, self-preference bias, and mitigations (randomize order, normalize length).
- [ ] **Human eval** — when it's ground truth; inter-annotator agreement.
- [ ] Eval **calibration** — validating that your judge correlates with human judgment.
- [ ] Hyperparameter/prompt search — grid vs. random search.
- [ ] **A/B testing** LLM features in production.
- [ ] Benchmarks worth knowing — MMLU, GPQA, SWE-bench, HumanEval, GSM8K, AIME, and **benchmark contamination**.
- [ ] Eval-driven development as a workflow.

## 10. RAG (Retrieval-Augmented Generation)
- [ ] Why RAG exists — knowledge cutoff, private data, grounding, citations.
- [ ] **Embeddings** — dense vectors, semantic similarity, cosine similarity.
- [ ] **Chunking** strategies — fixed-size, semantic, overlap, and the tradeoffs.
- [ ] **Vector databases** — what they do (ANN search: HNSW, IVF); name a few (pgvector, Pinecone, etc.).
- [ ] **Hybrid search** — dense + sparse (BM25) and why combining beats either.
- [ ] **Reranking** — cross-encoder rerankers after initial retrieval.
- [ ] RAG failure modes — bad chunking, retrieval miss, irrelevant context distracting the model.
- [ ] **Agentic RAG** — model decides when and what to retrieve, vs. always-retrieve.
- [ ] RAG vs. long-context vs. fine-tuning — when each is the right tool.

## 11. Structured Outputs
- [ ] **JSON mode** vs. **constrained/guided decoding** (grammar/schema-enforced) vs. tool-use-for-structure.
- [ ] How constrained decoding actually works — schema, grammar, and the token-level FSM.
- [ ] **Strict schema** guarantees and their limits.
- [ ] Why structured output can degrade reasoning quality, and the prefill/two-step workarounds.
- [ ] Parsing and validation (Pydantic / Zod) and graceful retries.
- [ ] The **refusal vs. strict-schema conflict** — what happens when a model must refuse but the schema has no room for it.

## 12. Production Concerns
- [ ] **Latency metrics** — **TTFT** (time to first token), **TPOT/ITL** (inter-token latency), tokens/sec, total latency.
- [ ] **Streaming** — SSE, why you stream, partial-parsing challenges.
- [ ] **Cost modeling** — input vs. output vs. cached token pricing; estimating $/request and $/user.
- [ ] **Throughput & batching** — continuous batching, why providers batch.
- [ ] **Speculative decoding** as a latency lever.
- [ ] **Rate limits** — RPM/TPM, backoff, retries, queueing.
- [ ] **Reliability** — timeouts, fallback models, circuit breakers, idempotency.
- [ ] **Observability** — tracing (spans per LLM/tool call), logging prompts+completions, token/cost dashboards; tools like LangSmith/Langfuse/Braintrust.
- [ ] **Caching** layers — exact-match response cache, semantic cache, prompt cache.
- [ ] **Security** — secrets, PII handling, data retention, not logging sensitive prompts.

## 13. Safety, Guardrails & Adversarial
- [ ] **Prompt injection** — direct and **indirect** (via retrieved/tool content); why it's unsolved.
- [ ] **Jailbreaks** — and why instruction-tuning doesn't fully prevent them.
- [ ] **Defense-in-depth** — input filtering, output filtering, and *enforcing limits below the model* (tool-layer gates) so a compromised model still can't do damage.
- [ ] **Least-privilege tool design**, sandboxing, allowlists.
- [ ] **Hallucination** — causes, grounding, citation-forcing, abstention.
- [ ] **The lethal trifecta** — private data + untrusted content + exfiltration channel.
- [ ] Content moderation / classifier guardrails.
- [ ] PII redaction.
- [ ] **Memory poisoning** and **model-supply-chain attacks** — what they are and how to defend.

## 14. Model Training Landscape
- [ ] **Pretraining** vs. **post-training**.
- [ ] **SFT** (supervised fine-tuning), **RLHF**, **RLAIF**, **DPO**, **Constitutional AI** — one sentence each.
- [ ] **LoRA / QLoRA** — parameter-efficient fine-tuning.
- [ ] **Distillation** and **quantization** (INT8/INT4) at a conceptual level.
- [ ] **When to fine-tune vs. prompt vs. RAG** — the decision framework.
- [ ] **Reasoning models / extended thinking** — RL on verifiable rewards, test-time compute, why they changed prompting.
- [ ] **MoE (Mixture of Experts)** — sparse activation, why it affects determinism.
- [ ] **Context-length extension** — RoPE scaling, at a high level.

## 15. Model & Ecosystem Landscape
- [ ] Current frontier families and rough strengths — Claude (Opus/Sonnet/Haiku), GPT, Gemini, Llama, DeepSeek, Qwen, Mistral.
- [ ] **Open vs. closed weights** — tradeoffs.
- [ ] API differences — Anthropic Messages API vs. OpenAI Chat Completions / Responses API.
- [ ] SDKs, the **Batch API**, the **Files API**, **embeddings APIs**.
- [ ] Frameworks — LangChain/LangGraph, LlamaIndex, DSPy, Pydantic AI — and a *view* on when frameworks help vs. when raw API calls are better.
- [ ] **Inference providers and gateways** — what they are and when to route through one.

## 16. System Design for AI
Be ready to reason through an end-to-end design:
- [ ] Design drills — a customer-support agent, a coding agent, an eval pipeline, a RAG knowledge assistant, a high-throughput batch classifier.
- [ ] Cover: prompt + context strategy, tool design, the harness loop, retrieval, caching, eval/regression strategy, guardrails, latency/cost budget, observability, failure handling, rollout (A/B, canary).
- [ ] Trade-off articulation — latency vs. quality, cost vs. capability, one big agent vs. multi-agent, RAG vs. long-context.
