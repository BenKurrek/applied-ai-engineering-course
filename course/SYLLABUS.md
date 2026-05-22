# Applied AI Engineering — Course Syllabus

A mastery-based course covering the practical knowledge required to build, evaluate,
and ship production systems on top of large language models.

This syllabus is the **static, shared definition** of the course. It contains no
progress or personal data — a student's progress is tracked separately in `PROGRESS.md`.

## Who this course is for

Software engineers who work with LLMs — building agents, evaluation pipelines, and
AI-powered features — and who want rigorous, interview-grade command of the
fundamentals rather than surface familiarity.

## How the course works

The course is taught in a mastery-based, tutored format. For every topic:

1. **Teach** — concepts are taught in small chunks, fundamental → advanced, with
   concrete worked examples.
2. **Check** — short comprehension checks during teaching surface misunderstandings
   early, before they compound.
3. **Notes** — a running cheat sheet is built for the topic as it is taught.
4. **Quiz** — a formative quiz follows the teaching. Missed material is re-taught and
   re-quizzed until it is solid. Quizzes do not count toward the grade.
5. **Exam** — a formal, closed-book exam concludes the topic.

Topics are taken **in order**; each one assumes the topics before it. A topic is
complete only once its exam is passed.

## Assessment

Each topic ends with an exam scored out of 100. **The pass mark is 85.**

Exams are **mixed-format**, combining:

- True/false (often with a "why?" follow-up)
- Multiple choice
- Short answer — define a term, explain a mechanism
- Long answer — explain a system, compare approaches, reason through a trade-off
- Applied scenario — "here is a situation: what is happening, and what would you do?"

Free-form answers are graded on the quality of reasoning, with partial credit — not on
keyword matching. An exam scoring below 85 is followed by targeted review of the weak
areas and a retake.

## Course completion

The course is complete when all sixteen topic exams have been passed at 85 or above. A
graduate can explain Applied AI Engineering from first principles — how models work,
how to prompt and constrain them, how to build agents, how to evaluate them, and how to
run them safely and economically in production.

---

# Curriculum

Five phases, sixteen topics.

## Phase 1 — Core LLM Mechanics

How large language models actually work, from the ground up. Everything in later phases
assumes this material.

### Topic 1 — LLM Fundamentals
- 1.1 What an LLM is — next-token prediction as the whole job
- 1.2 Training vs. inference; why weights are frozen when you use the model
- 1.3 The transformer at a block level — embeddings → attention → FFN → unembedding
- 1.4 Self-attention — queries/keys/values; why cost is O(n²) in sequence length
- 1.5 Autoregressive generation; prefill vs. decode and why the distinction matters
- 1.6 Logits → softmax → probability distribution over the vocabulary
- 1.7 Parameters vs. context; statelessness of the API
- 1.8 Inference-time quantization and speculative decoding
- **Topic 1 Exam**

### Topic 2 — Tokenization
- 2.1 What a token is; tokens ≠ words; tokens-per-word ratios
- 2.2 Byte-Pair Encoding (BPE) and byte-level BPE
- 2.3 WordPiece vs. Unigram / SentencePiece
- 2.4 Vocabulary size tradeoffs
- 2.5 Special tokens — BOS/EOS, padding, role/turn delimiters, tool tokens
- 2.6 Tokenization bugs — number splitting, whitespace, glitch tokens, multilingual cost
- 2.7 How tokenization interacts with prompt caching
- **Topic 2 Exam**

### Topic 3 — Inference & Sampling
- 3.1 Temperature and its effect on the logit distribution
- 3.2 top-p (nucleus) and top-k sampling
- 3.3 Greedy decoding vs. sampling; why beam search is rarely used
- 3.4 Repetition / frequency / presence penalties
- 3.5 logprobs — what they are and using them for confidence & classification
- 3.6 max_tokens, stop sequences
- 3.7 Determinism caveats; seeds and their limits
- **Topic 3 Exam**

### Topic 4 — Context Window & the LLM Turn
- 4.1 The turn — messages array; roles (system/user/assistant/tool)
- 4.2 How multi-turn conversations are constructed (append & resend)
- 4.3 Context window size vs. effective context
- 4.4 "Lost in the middle" / context rot
- 4.5 Context engineering — ordering, compaction, summarization, eviction
- 4.6 Input vs. output token accounting
- 4.7 Context management strategies — sliding window, summarization, retrieval-on-demand
- **Topic 4 Exam**

## Phase 2 — Prompting and Context Control

Getting the most out of a fixed model by controlling what goes into the context window.

### Topic 5 — Prompt Engineering
- 5.1 The system prompt — role and best practices
- 5.2 Zero-shot / few-shot / many-shot; when examples help vs. hurt
- 5.3 Chain-of-thought; how reasoning models changed it
- 5.4 Structured prompting — XML tags, delimiters, sections
- 5.5 Prefilling the assistant turn to constrain output
- 5.6 Instruction placement, recency effects, specificity
- 5.7 Failure modes — instruction conflict, distraction, sycophancy
- 5.8 Prompts as code — versioning and testing
- 5.9 Prompt optimization and meta-prompting
- 5.10 Decomposition, prompt chaining, and self-consistency
- **Topic 5 Exam**

### Topic 6 — Prompt Caching
- 6.1 What it is — reusing the KV cache for a repeated prompt prefix
- 6.2 The exact-prefix-match requirement
- 6.3 Prompt-structure implications — static content first, variable content last
- 6.4 Cache TTL
- 6.5 Pricing math — cache writes vs. cache reads vs. base input
- 6.6 When caching pays off and when it doesn't
- 6.7 Latency impact (TTFT)
- **Topic 6 Exam**

## Phase 3 — Agents and Tool Use

Turning a single model call into a system that can take actions and run a loop.

### Topic 7 — Tool Use / Function Calling
- 7.1 Tool definitions and JSON Schema
- 7.2 The flow — tool_use block → your code executes → tool_result → continue
- 7.3 Parallel tool calls
- 7.4 The model only requests; you execute (and the safety implication)
- 7.5 Tool-result formatting and error handling
- 7.6 Tool-choice controls — auto / required / specific / none
- 7.7 Writing good tool descriptions — the description is a prompt
- 7.8 MCP (Model Context Protocol) — what it is, why it exists, client/server model
- 7.9 Code-execution orchestration ("code mode")
- **Topic 7 Exam**

### Topic 8 — Agents & Harnesses
- 8.1 What makes something an agent vs. a single LLM call
- 8.2 The agentic loop — think → act → observe → repeat
- 8.3 ReAct (reason + act)
- 8.4 Planning — upfront vs. interleaved; decomposition; subagents
- 8.5 Memory — short-term (context) vs. long-term (external store)
- 8.6 Anatomy of the harness — loop, dispatch, context mgmt, retries, guardrails, logging
- 8.7 Multi-agent systems — orchestrator/worker, handoffs, when it helps vs. hurts
- 8.8 Failure modes — loops, context overflow, error cascades, goal drift
- 8.9 Termination and budget limits
- 8.10 Human-in-the-loop checkpoints
- 8.11 Evaluating agents — trajectory eval vs. final-outcome eval
- 8.12 The cost model of a growing trajectory
- 8.13 Context compaction mechanics
- 8.14 Observability — trace and span structure
- **Topic 8 Exam**

## Phase 4 — Evaluation, Retrieval and Output Control

Measuring quality, grounding models in external knowledge, and constraining their output.

### Topic 9 — Evaluations
- 9.1 Why evals exist; offline vs. online
- 9.2 Golden / eval datasets — how to build them
- 9.3 Regression testing for prompts and models
- 9.4 Metric types — exact match, F1, pass@k, BLEU/ROUGE and their weaknesses
- 9.5 LLM-as-judge — single-output scoring vs. pairwise comparison
- 9.6 Rubric-based evaluation — decomposing quality into scored dimensions
- 9.7 Judge-model ensembles / panels
- 9.8 Judge biases — position, verbosity, self-preference — and mitigations
- 9.9 Human evaluation and inter-annotator agreement
- 9.10 Eval calibration — validating the judge against humans
- 9.11 Hyperparameter / prompt search — grid vs. random
- 9.12 A/B testing LLM features in production
- 9.13 Benchmarks (MMLU, GPQA, SWE-bench, etc.) and benchmark contamination
- **Topic 9 Exam**

### Topic 10 — RAG (Retrieval-Augmented Generation)
- 10.1 Why RAG exists — knowledge cutoff, private data, grounding, citations
- 10.2 Embeddings — dense vectors, semantic similarity, cosine similarity
- 10.3 Chunking strategies and tradeoffs
- 10.4 Vector databases and ANN search (HNSW, IVF)
- 10.5 Hybrid search — dense + sparse (BM25)
- 10.6 Reranking — cross-encoder rerankers
- 10.7 RAG failure modes
- 10.8 Agentic RAG
- 10.9 RAG vs. long-context vs. fine-tuning — the decision framework
- **Topic 10 Exam**

### Topic 11 — Structured Outputs
- 11.1 JSON mode vs. constrained decoding vs. tool-use-for-structure
- 11.2 How constrained decoding actually works — schema, grammar, and the token FSM
- 11.3 Strict schema guarantees and their limits
- 11.4 How structured output can degrade reasoning quality; workarounds
- 11.5 Parsing and validation (Pydantic / Zod); graceful retries
- 11.6 The refusal vs. strict-schema conflict
- **Topic 11 Exam**

## Phase 5 — Production, Safety and Frontier

Running AI systems in the real world, keeping them safe, and understanding the wider
training and model landscape.

### Topic 12 — Production Concerns
- 12.1 Latency metrics — TTFT, TPOT/ITL, tokens/sec
- 12.2 Streaming — SSE and partial-parsing challenges
- 12.3 Cost modeling — input/output/cached pricing; $/request, $/user
- 12.4 Throughput and batching — continuous batching
- 12.5 Speculative decoding — a latency lever
- 12.6 Rate limits — RPM/TPM, backoff, queueing
- 12.7 Reliability — timeouts, fallback models, circuit breakers, idempotency
- 12.8 Observability — tracing, logging, token/cost dashboards
- 12.9 Caching layers — exact-match, semantic, prompt cache
- 12.10 Security — secrets, PII, data retention
- **Topic 12 Exam**

### Topic 13 — Safety, Guardrails & Adversarial
- 13.1 Prompt injection — direct and indirect
- 13.2 Jailbreaks
- 13.3 Defense-in-depth; enforcing limits below the model
- 13.4 Least-privilege tool design, sandboxing, allowlists
- 13.5 Hallucination — causes, grounding, citation-forcing, abstention
- 13.6 The lethal trifecta
- 13.7 Content moderation / classifier guardrails
- 13.8 PII redaction
- 13.9 Memory poisoning and model-supply-chain attacks
- **Topic 13 Exam**

### Topic 14 — Model Training Landscape
- 14.1 Pretraining vs. post-training
- 14.2 SFT, RLHF, RLAIF, DPO, Constitutional AI
- 14.3 LoRA / QLoRA — parameter-efficient fine-tuning
- 14.4 Distillation and quantization
- 14.5 When to fine-tune vs. prompt vs. RAG
- 14.6 Reasoning models / extended thinking — RL on verifiable rewards, test-time compute
- 14.7 Mixture of Experts (MoE)
- 14.8 Context-length extension — RoPE scaling
- **Topic 14 Exam**

### Topic 15 — Model & Ecosystem Landscape
- 15.1 Frontier model families and their strengths
- 15.2 Open vs. closed weights — tradeoffs
- 15.3 API differences — Anthropic Messages vs. OpenAI Chat Completions / Responses
- 15.4 SDKs, Batch API, Files API, embeddings APIs
- 15.5 Frameworks — LangChain/LangGraph, LlamaIndex, DSPy, Pydantic AI
- 15.6 Inference providers and gateways
- **Topic 15 Exam**

### Topic 16 — System Design for AI
- 16.1 The AI system design framework — a repeatable structure
- 16.2 Design drill — customer-support agent
- 16.3 Design drill — coding agent
- 16.4 Design drill — eval pipeline
- 16.5 Design drill — RAG knowledge assistant
- 16.6 Design drill — high-throughput batch classification
- 16.7 Trade-off articulation — latency/quality, cost/capability, mono vs. multi-agent
- **Topic 16 Exam**
