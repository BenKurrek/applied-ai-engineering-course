# Topic 15 — Model & Ecosystem Landscape — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This file is taught sub-chapter by sub-chapter, in order. Each sub-chapter has core
teaching content, key terms, common misconceptions, a worked example, and check
questions the tutor uses to confirm understanding before moving on. A mixed-format exam
bank is at the end; the tutor draws from it for the gated topic exam (scored out of
100, pass mark 85). Topic 15 is the lay-of-the-land topic: the frontier model families
and their strengths, the open- vs. closed-weights trade-off, the major API shapes
(Anthropic Messages vs. OpenAI Chat Completions / Responses), the supporting API
surface (SDKs, Batch, Files, embeddings), the agent/RAG frameworks, and the inference
providers and gateways that sit between you and the models. Specific model names and
prices move fast — facts here are current as of **May 2026** and you should treat exact
numbers as a snapshot, while the *concepts and trade-offs* are durable.

---

## 15.1 — Frontier model families and their strengths

### Concept

"Frontier models" are the most capable models available at a given time, produced by a
handful of well-resourced labs. As of May 2026 the landscape:

**Closed-weights frontier families:**

- **Anthropic — Claude.** Three-tier naming: **Opus** (most capable, deepest reasoning
  and agentic/coding work), **Sonnet** (balanced — strong capability at a much lower
  price, the default workhorse), **Haiku** (fastest and cheapest). May 2026 generation:
  Claude **Opus 4.7** (released April 2026), **Sonnet 4.6**, **Haiku 4.5** [1].
  Reputation: top-tier coding and agentic tool use, strong instruction-following,
  long-form writing, and a safety focus (Constitutional AI). *Benchmark standings move
  fast — verify any "leads benchmark X" claim against current results before relying on
  it.*
- **OpenAI — GPT.** May 2026 flagship is **GPT-5.5** (released April 2026; a low-latency
  **GPT-5.5 Instant** followed in May 2026) [2]; the o-series reasoning lineage is
  folded into the GPT line. Reputation: strong general-purpose reasoning, broad
  ecosystem, mature tooling, multimodal.
- **Google — Gemini.** May 2026 flagship **Gemini 3.1 Pro** (released February 2026,
  with smaller Flash variants) [3]. Reputation: best-in-class multimodal (native
  text/image/audio/video) and very long context (1M-token windows [3]), tight
  Google-ecosystem integration, strong reasoning scores.
- **xAI — Grok.** **Grok 4**-generation; competitive general models with real-time data
  integration. *(Exact current Grok version dates fast — confirm before relying on it.)*

**Open-weights frontier-class families:**

- **Meta — Llama.** The family that anchored the open-weights ecosystem; widely
  fine-tuned and self-hosted.
- **DeepSeek.** Chinese lab; **DeepSeek-V4** generation, MoE models released under
  permissive licenses, known for strong capability at low cost and for pioneering
  open reasoning models (R1).
- **Alibaba — Qwen.** Broad family of sizes and strong multilingual performance; a
  popular open-weights base.
- **Mistral.** European lab; efficient open models plus some commercial offerings.
- **Moonshot — Kimi**, **Zhipu — GLM**: large open MoE models reaching frontier-class
  capability, often agentic-focused.

The durable point for an applied engineer: **pick by fit, not hype.** Rough strengths —
Claude for coding/agentic work and instruction-following; GPT for broad general-purpose
reasoning and ecosystem; Gemini for multimodal and very long context; open models
(Llama/DeepSeek/Qwen/Mistral) when you need self-hosting, data control, cost control, or
customization. Within a family, the tier matters as much as the family: most
production traffic should run on a *mid-tier* model (Sonnet-class), with the flagship
reserved for genuinely hard requests and the small tier for high-volume simple ones —
that is model cascading (Topic 12). And critically: **abstract over the provider** so
you can switch as the frontier reorders, which it does every few months.

### Key terms

- **Frontier model** — among the most capable models available at a given time.
- **Model family / tier** — a provider's lineup (e.g. Claude) and its capability levels
  (Opus / Sonnet / Haiku).
- **Closed-weights** — model weights are not released; usable only via the provider's
  API (Claude, GPT, Gemini, Grok).
- **Open-weights** — weights are downloadable and self-hostable (Llama, DeepSeek, Qwen,
  Mistral).
- **Model cascading / routing** — sending most traffic to a cheaper tier and escalating
  hard requests to the flagship.

### Common misconceptions

- ❌ "There is one 'best' model." → ✅ Best depends on the task — coding vs. multimodal
  vs. long-context vs. cost vs. self-hosting; and the ranking churns every few months.
- ❌ "Always use the flagship for quality." → ✅ Most traffic should run on a mid-tier
  model; reserve the flagship for hard requests (cost cascading).
- ❌ "Pick a provider and hard-code it." → ✅ The frontier reorders constantly; build a
  provider abstraction so you can switch models.
- ❌ "Open models are far behind closed ones." → ✅ The gap has narrowed sharply;
  several open MoE models are frontier-class on many tasks.

### Worked example

A team building a coding agent benchmarks three options on their own eval set (not
public leaderboards, to avoid contamination — Topic 9). On their tasks Claude Opus-class
leads on multi-file agentic edits, Gemini wins when the task involves screenshots/
diagrams (multimodal), and a fine-tuned open Qwen model is "good enough" for simple
single-file fixes at a fraction of the cost. They ship a cascade: the open model for
easy fixes, Claude for hard agentic work, Gemini for the multimodal subset — chosen by
*measured fit*, behind a provider abstraction so they can re-evaluate next quarter.

### Check questions

1. A lead argues against a provider-abstraction layer: "we picked the best model after
   careful evaluation, it's not changing, and the abstraction is just extra code."
   Beyond a future model swap, name two *concrete* things this position gives up that
   matter even if they never change their primary model. — **Answer:** Two from:
   (1) *Fallbacks* — when the primary provider has an outage or rate-limits you, an
   abstraction lets you route to an alternate without an emergency rewrite; hard-coding
   means an outage is your outage. (2) *Cascading* — routing easy requests to a cheaper
   model and hard ones to the flagship needs more than one model behind a common
   interface. (Also: per-model regression evals; A/B-testing a candidate model.) Even
   with a fixed "best" model, reliability and cost optimization both require speaking to
   more than one model — and the frontier *does* reorder anyway.
2. A team runs *all* traffic — simple FAQ lookups and hard multi-step reasoning alike —
   on the flagship tier of their chosen family, reasoning "we picked the best family,
   so use its best model." What is wrong with this, and what would you change? —
   **Answer:** Getting the *tier* wrong wastes money or quality regardless of family.
   The flagship costs several× the mid-tier and is slower; running simple FAQ lookups on
   it burns cost on capability those requests never needed. The fix is model cascading:
   a mid-tier model as the default workhorse, the flagship reserved for genuinely hard
   requests, and a small fast tier for high-volume trivial ones. Tier choice is a
   cost/quality lever as decisive as the family choice.

---

## 15.2 — Open vs. closed weights — tradeoffs

### Concept

A foundational architectural decision: consume a model through a provider's API
(**closed weights**) or run a model whose weights you can download (**open weights**).
This is not "free vs. paid" — open-weights models still cost money to run — it is a
**control vs. convenience** trade-off.

**Closed-weights (API) — Claude, GPT, Gemini:**
- *Pros:* no infrastructure — the provider handles GPUs, scaling, serving, batching;
  immediate access to the most capable frontier models; the provider does safety
  tuning, updates, and uptime; you pay per token with zero capex.
- *Cons:* your data leaves your boundary (mitigated by zero-retention/enterprise tiers,
  but it is still a third party); vendor lock-in and exposure to price changes, rate
  limits, deprecations, and outages; the model can change underneath you (a silent
  behavior shift — pin versions and run regression evals); limited customization.

**Open-weights — Llama, DeepSeek, Qwen, Mistral:**
- *Pros:* **full data control** — the model can run entirely inside your own
  infrastructure (or even air-gapped), which can be decisive for regulated industries,
  sensitive data, or strict residency requirements; **no per-token API cost** (you pay
  for compute instead — favorable at high, steady volume); **deep customization** — full
  fine-tuning, LoRA adapters, quantization, custom serving; **no lock-in** and version
  stability — the weights do not change unless you change them; offline/edge deployment.
- *Cons:* **you own the infrastructure** — GPUs, a serving stack (vLLM/SGLang/TGI),
  scaling, monitoring, on-call — real and substantial engineering cost; the very top of
  the frontier is often still a closed model; **you own safety** — no provider guardrail
  layer, so you must build moderation/guardrails yourself; "open weights" usually means
  weights + license, *not* training data or training code (so not fully reproducible);
  licenses vary (truly permissive Apache/MIT vs. community licenses with restrictions —
  read them).

**The decision drivers:** data sensitivity / residency / compliance (push toward open
or zero-retention); scale and cost profile (high steady volume favors self-hosted
open; spiky or low volume favors API); need for customization (favors open); team
capability and willingness to operate GPU infrastructure (favors closed if you cannot);
need for the absolute top capability (often favors closed). Many organizations run a
**hybrid**: a closed frontier API for hard/low-volume tasks, a self-hosted open model
for high-volume or sensitive workloads. The provider abstraction (15.1) is what makes
hybrid practical.

A note worth knowing for interviews: some providers run *only* open-weights models as a
deliberate product stance (privacy-first inference services), where "we never touch a
closed model" is a core differentiator — being able to articulate the open-weights case
crisply matters.

### Key terms

- **Open weights** — downloadable, self-hostable model weights (Llama, DeepSeek, Qwen,
  Mistral).
- **Closed weights** — weights kept private; access only via the provider's API
  (Claude, GPT, Gemini).
- **Vendor lock-in** — dependence on one provider's API, pricing, availability, and
  behavior.
- **Data residency** — the requirement that data stay within a jurisdiction or
  infrastructure boundary; a major driver toward self-hosted open models.
- **Hybrid deployment** — using both closed APIs and self-hosted open models, routed by
  task.

### Common misconceptions

- ❌ "Open weights means free." → ✅ You trade per-token API fees for GPU and operations
  cost; it is only cheaper at sufficient steady volume.
- ❌ "Open weights means fully open source / reproducible." → ✅ Usually it is weights +
  a license, not the training data or code — and licenses vary in permissiveness.
- ❌ "Self-hosting an open model is just a deployment detail." → ✅ It means owning
  GPUs, a serving stack, scaling, monitoring, on-call, *and* the safety/guardrail layer
  — substantial ongoing engineering.
- ❌ "Closed-model behavior is stable." → ✅ Providers update models; pin versions and
  run regression evals or you can be silently regressed.

### Worked example

A healthcare company processes patient records and cannot send PHI to a third party
under its compliance regime. Closed APIs — even zero-retention tiers — are a hard
governance "no." They deploy an open-weights model (a Llama/Qwen-class model) on GPUs
inside their own VPC, serve it with vLLM, and fine-tune it on de-identified domain data.
They accept the cost: a platform team to run the serving stack, build their own
moderation guardrails, and handle scaling. The driver was not cost — it was data
residency and compliance, which only self-hosted open weights could satisfy.

### Check questions

1. A startup with low, spiky traffic and a tiny team is choosing open vs. closed
   weights. Which fits and why? — **Answer:** Closed (API). Low/spiky volume means
   per-token pricing beats paying for idle GPUs; a tiny team cannot run a GPU serving
   stack, scaling, on-call, and a homegrown safety layer. They get frontier capability
   with zero infrastructure.
2. A hospital and a high-traffic consumer app both consider open weights. The hospital
   chooses open weights and accepts the heavy operational burden; the consumer app's
   architect says "for us that burden would not be worth it." Explain what makes the
   trade-off resolve differently for each. — **Answer:** For the hospital the decisive
   driver is *data control / residency* — patient data legally cannot leave its
   boundary, and no closed-API tier (even zero-retention) satisfies that; open weights
   self-hosted is effectively a hard *requirement*, so the operational burden is simply
   the price of compliance. For the consumer app there is usually no such hard
   constraint: a closed API gives frontier capability with zero infrastructure, and
   self-hosting only pays off at sufficient *steady* volume. Open weights is decided by
   whether a control/residency requirement *forces* it — not by preference.

---

## 15.3 — API differences — Anthropic Messages vs. OpenAI Chat Completions / Responses

### Concept

The two dominant API shapes. They are conceptually similar — send a conversation, get a
completion — but differ in structure in ways that matter when you build, especially
when abstracting across providers.

**Anthropic Messages API.** You send a `messages` array of turns with `role`
(`user` / `assistant`) and `content`. The **system prompt is a separate top-level
`system` parameter**, not a message in the array [4]. `content` can be a string or an array
of typed **content blocks** — `text`, `image`, `tool_use`, `tool_result`,
`thinking` — which makes multimodal input, tool calls, and extended thinking uniform:
everything is a block. Tools are defined with name, description, and a JSON Schema
`input_schema`; the model returns a `tool_use` block, and you reply with a `user`
message containing a `tool_result` block. `max_tokens` is **required**. The API is
**stateless** — you resend the full conversation each turn.

**OpenAI Chat Completions API.** The long-standing standard, widely cloned — many other
providers and local servers expose an "OpenAI-compatible" endpoint, making it a de facto
lingua franca. You send a `messages` array where the **system prompt is the first
message** (`role: "system"`). Tool calls come back as `tool_calls` on the assistant
message, with arguments as a **JSON-encoded string** (you parse it); you reply with a
message of `role: "tool"`. Also **stateless** — resend the history each turn. Responses
are wrapped in a `choices` array.

**OpenAI Responses API.** OpenAI's newer primitive, recommended for new projects (Chat
Completions remains supported) [5]. Key differences from Chat Completions:
- **Output is a list of typed `items`** (messages, tool calls, reasoning items) rather
  than a `choices` array — closer in spirit to Anthropic's typed content blocks.
- **Optional server-side state.** You can pass a previous response ID and let OpenAI
  retain conversation state, instead of resending the whole history — a real departure
  from the stateless model.
- **Built-in/hosted tools** — web search, file search, code interpreter, computer use,
  and remote MCP servers are invocable directly, without you orchestrating them.
- **Better reasoning + caching** — designed for reasoning models, with improved prompt-
  cache utilization.

**What actually bites you in practice:** system-prompt placement (separate parameter
vs. first message); tool-result roles (`user`+`tool_result` block vs. a `tool` message);
tool-argument encoding (structured object vs. JSON string to parse); whether
`max_tokens` is required; response shape (`content` blocks vs. `choices` vs. `items`);
streaming event formats differ; and statefulness (all stateless except Responses' opt-in
server-side state). This is exactly why teams put a thin **provider-abstraction layer**
in front (15.1/15.2) — or use a gateway/SDK that normalizes these — so application code
is not littered with provider-specific shapes.

### Key terms

- **Messages API (Anthropic)** — `messages` array + separate `system` param; typed
  content blocks; `max_tokens` required; stateless.
- **Chat Completions (OpenAI)** — `messages` array with system as the first message;
  tool args as JSON strings; `choices` array; stateless; the de facto compatibility
  standard.
- **Responses API (OpenAI)** — newer primitive; typed `items` output; optional
  server-side state; built-in hosted tools; recommended for new projects.
- **Content block / item** — a typed unit of message content (text, image, tool_use,
  tool_result, reasoning).
- **OpenAI-compatible endpoint** — a non-OpenAI API that mimics the Chat Completions
  shape so existing tooling works.

### Common misconceptions

- ❌ "All LLM APIs are interchangeable — same JSON." → ✅ They differ in system-prompt
  placement, tool-result roles, argument encoding, response shape, and statefulness;
  abstract over them.
- ❌ "The system prompt is always the first message." → ✅ That is the Chat Completions
  convention; Anthropic uses a *separate top-level parameter*.
- ❌ "Responses API and Chat Completions are the same thing renamed." → ✅ Responses
  adds typed items, optional server-side state, and built-in hosted tools — a genuine
  architectural step beyond Chat Completions.
- ❌ "Tool-call arguments arrive as a parsed object everywhere." → ✅ OpenAI Chat
  Completions returns them as a JSON-encoded *string* you must parse (and validate).

### Worked example

A team's app code is full of `response.choices[0].message.content` and assumes the
system prompt is `messages[0]`. They want to add Claude as a fallback (Topic 12). Direct
swap breaks: Claude has no `choices` (it has `content` blocks), and its system prompt is
a separate `system` parameter. The fix is a provider-abstraction layer — an internal
`generate(messages, system, tools)` function with one adapter per provider that maps the
neutral shape onto Messages, Chat Completions, or Responses and normalizes the result
back. Application code now speaks one shape; swapping providers or running a cascade is
a config change, not a rewrite.

### Check questions

1. A developer building a provider adapter for the Anthropic Messages API copies their
   working OpenAI Chat Completions adapter and just changes the URL and key. Their tool
   calls and system prompt break. Predict the *specific* breakages they will hit and
   why an adapter — not a URL swap — is required. — **Answer:** Several concrete
   breakages: the system prompt must move from `messages[0]` to a separate top-level
   `system` parameter (Anthropic has no `"system"` role); tool results must be sent as
   a `user` message containing a `tool_result` content block, not a `role: "tool"`
   message; the response is read as typed `content` blocks, not `choices[0].message`;
   and `max_tokens` is required. (Also: tool-call arguments arrive structured, not as a
   JSON string to parse.) These are structural shape differences, so a real adapter that
   maps a neutral interface onto each provider's shape is required — a URL swap is not.
2. A team on OpenAI Chat Completions resends the entire conversation history on every
   turn and writes their own retry/backoff and SSE-parsing code. Which two parts of
   OpenAI's API surface would each *directly* reduce that work, and how? — **Answer:**
   (1) The *Responses API* offers optional server-side conversation state: pass a
   previous-response ID and let OpenAI retain history instead of resending it every
   turn — removing the resend-the-whole-history work (and improving cache utilization).
   (2) The official *SDK* bakes in retries-with-backoff and SSE/streaming parsing as
   typed events — removing the hand-rolled reliability and parsing code. Together they
   replace two chunks of bespoke code with maintained provider features.

---

## 15.4 — SDKs, Batch API, Files API, embeddings APIs

### Concept

Beyond the core text-generation endpoint, every major provider ships a supporting API
surface. Know what each is for.

**SDKs (official client libraries).** Provider-maintained libraries — `anthropic` and
`openai` for Python and TypeScript, plus community SDKs — that wrap the raw HTTP API.
They handle auth, request/response (de)serialization, **typed objects**,
**streaming** helpers (turning the SSE byte stream into an iterator of events),
**automatic retries with backoff** on transient errors, timeouts, and pagination. Use
the SDK rather than raw HTTP unless you have a specific reason — it gets reliability
behaviors (Topic 12) right by default. Caveat: SDKs are provider-specific, so a thin
abstraction still sits above them for multi-provider portability.

**Batch API.** For **asynchronous, latency-tolerant** bulk workloads. You upload a file
of many requests; the provider processes them within a window (commonly up to 24 h) and
returns a results file. The headline benefit is **~50% cost** versus synchronous calls,
plus it sidesteps real-time rate-limit choreography. The trade-off is latency — minutes
to hours, not seconds — so it is wrong for anything interactive. Ideal for: bulk
classification/extraction, generating an eval dataset, large-scale offline scoring,
backfills, synthetic-data generation. If a workload does not need a fast response, the
Batch API is usually the cheapest way to run it.

**Files API.** For uploading and referencing files (PDFs, images, datasets,
spreadsheets) by ID rather than inlining their bytes into every request. Two main uses:
(1) supplying documents/images as model input without re-encoding them per call —
upload once, reference by ID; (2) carrying the input/output files for the Batch API.
It also helps with large content that would be unwieldy or repetitive inline.

**Embeddings APIs.** A *different kind of model*: an embedding model maps text (or
images) to a fixed-length **dense vector** capturing semantic meaning, so that similar
content yields nearby vectors (cosine similarity — Topic 10). They are the backbone of
**RAG** (embed chunks and queries, retrieve by vector similarity), **semantic search**,
**clustering**, **classification**, **deduplication**, and **semantic caching** (Topic
12). They are cheap and fast relative to generation. Notably, **Anthropic does not
provide a first-party embeddings model** — Anthropic's docs instead point users to
third-party embedding providers, historically highlighting **Voyage AI** (now part of
MongoDB, following MongoDB's 2025 acquisition) and listing alternatives such as OpenAI,
Cohere, Google, or an open model [6]. The durable point is *not* which vendor a doc page
names this quarter — vendor recommendations and ownership move — but that an
Anthropic-based RAG stack must source embeddings from *somewhere else*. OpenAI and
Google do offer first-party embeddings endpoints. Key practical points: an embedding's
**dimensionality** is fixed per model; you cannot compare vectors across different
embedding models; and changing your embedding model means **re-embedding your whole
corpus**.

### Key terms

- **SDK** — official provider client library handling auth, serialization, typed
  objects, streaming, and retries.
- **Batch API** — asynchronous bulk processing (≤~24 h) at ~50% cost; for
  latency-tolerant workloads.
- **Files API** — upload files once and reference them by ID across requests / for
  batch jobs.
- **Embedding model** — a model that maps text/images to dense semantic vectors;
  powers RAG, semantic search, clustering, semantic caching.
- **Dimensionality** — the fixed length of an embedding vector; model-specific and not
  cross-comparable.

### Common misconceptions

- ❌ "The Batch API is just a faster way to send many requests." → ✅ It is *slower*
  per request (async, up to hours) but ~50% cheaper — for latency-tolerant work, not
  speed.
- ❌ "Embedding models and chat models are the same kind of model." → ✅ Embedding
  models output a vector, not text; they are a separate model class with their own
  endpoints.
- ❌ "I can switch embedding models freely." → ✅ Vectors are not comparable across
  models; switching means re-embedding the entire corpus.
- ❌ "Every major provider has its own embeddings API." → ✅ Notably Anthropic does not;
  Anthropic-based RAG stacks use a third-party embedding provider.
- ❌ "Raw HTTP is fine; SDKs are optional sugar." → ✅ SDKs bake in retries, backoff,
  streaming parsing, and typed handling — reliability behaviors you would otherwise
  reimplement and get subtly wrong.

### Worked example

A team must classify 2 million historical support tickets by topic to seed an analytics
dashboard. Their first attempt fires synchronous calls in a loop — slammed by rate
limits, and the projected cost is high. Better design: write all 2M requests to a
JSONL file, upload it via the **Files API**, submit it to the **Batch API**. The job
completes within the 24-hour window at ~50% of synchronous cost and with no rate-limit
firefighting. Separately, for the *live* dashboard search box they use an **embeddings
API** to power semantic search over the tickets — a different API for a different
(interactive) job.

### Check questions

1. A team's nightly job re-scores 800k stored records and keeps getting rate-limited;
   an engineer's instinct is "raise our rate limits." A second engineer says "this is
   the wrong fix entirely." Why is the rate-limit-raise the wrong instinct here, and
   what should they use? — **Answer:** The job is *bulk and latency-tolerant* — nothing
   is waiting on a second-level response — so it is a textbook fit for the *Batch API*:
   submit all 800k requests as one file, get results within ~24 h at ~50% lower cost,
   and the rate-limit problem disappears because batch is processed off the synchronous
   path. Raising synchronous rate limits just lets you hammer the real-time API harder
   and still pay full price; for offline bulk work, batch is both cheaper and removes
   the rate-limit choreography. (Batch would be wrong only if the work were interactive.)
2. A team wants to upgrade their RAG system to a newer embedding model and assumes it
   is a config change like swapping the chat model. Explain why it is far more invasive,
   in terms of what an embedding vector actually is. — **Answer:** An embedding vector
   is a point in a *specific model's* learned semantic space, with that model's
   dimensionality — vectors from two different models are not comparable, and cosine
   similarity between them is meaningless. So a query embedded by the new model cannot
   be matched against a corpus indexed by the old one; retrieval would return noise.
   Upgrading the embedding model therefore requires *re-embedding the entire corpus* —
   a full re-index, not a config flag. (Swapping the *chat* model has no such cost.)

---

## 15.5 — Frameworks — LangChain/LangGraph, LlamaIndex, DSPy, Pydantic AI

### Concept

Frameworks sit between your application and the provider APIs, offering abstractions for
common LLM patterns. The major options as of 2026:

- **LangChain / LangGraph.** LangChain is the broad, long-established toolkit — chains,
  prompt templates, memory, and a huge library of integrations (model providers, vector
  stores, document loaders). **LangGraph** is its lower-level successor for **agents**:
  it models an agent as an explicit **graph / state machine** with nodes, edges, and
  shared state, supporting cycles, durable execution, checkpoints, and human-in-the-loop
  interrupts. LangGraph is widely used for **stateful production agentic workflows**;
  classic LangChain is associated with fast prototyping but also with abstraction
  overhead.
- **LlamaIndex.** Specialized for **RAG and knowledge-intensive** applications. Strong
  data connectors, indexing strategies, query engines, and retrieval pipelines. The
  go-to when retrieval over your own data is the core of the system.
- **DSPy.** A different philosophy: **programming, not prompting.** You declare the
  *signature* of each step (inputs → outputs) and DSPy **compiles/optimizes the actual
  prompts** (and few-shot examples) automatically against a metric — treating prompt
  engineering as an optimization problem rather than hand-tuned strings. Best when you
  want the system to optimize its own prompts and you have a metric to optimize against.
- **Pydantic AI.** A newer, **type-safe, Python-native** agent framework from the
  Pydantic team. Built around **structured outputs and validation** (Pydantic models
  for tool inputs and results), dependency injection, and a deliberately lighter,
  less-magic surface. Favored where type safety and structured, validated outputs are
  cultural priorities.

**The view to have — when frameworks help vs. when raw API calls are better:**

Frameworks *help* when: you are prototyping fast and want batteries-included
integrations; you need a pattern the framework does well (LangGraph's stateful agent
graphs, LlamaIndex's retrieval pipelines, DSPy's prompt optimization); or a standard
abstraction genuinely speeds the team up.

Raw API calls (or a thin in-house layer) are *better* when: the task is simple (a
framework adds layers, indirection, and dependency surface for a single call);
you need precise control over the exact tokens in the context window — frameworks can
obscure what is actually sent, which matters for prompt caching, cost, and debugging;
you are hitting framework abstraction leaks and fighting the framework; or you want
minimal dependencies and full debuggability. A common mature path: **prototype with a
framework, then drop to raw API calls (or a slim internal harness) for the hot,
production-critical paths** where control and transparency matter most.

The principled stance: a framework is a *tool*, not a default. Adopt one when it
matches your dominant pattern and removes real work — but never let it stop you from
knowing exactly what prompt and messages hit the model, because that knowledge is what
Topics 1–14 are all about.

### Key terms

- **LangChain** — broad LLM toolkit: chains, templates, memory, large integration
  library; prototyping-oriented.
- **LangGraph** — graph/state-machine framework for stateful production agents; cycles,
  checkpoints, human-in-the-loop.
- **LlamaIndex** — RAG-specialized framework: data connectors, indexing, retrieval/query
  pipelines.
- **DSPy** — "programming not prompting"; declare step signatures and a metric, and it
  compiles/optimizes prompts automatically.
- **Pydantic AI** — type-safe, Python-native agent framework centered on structured
  outputs and validation.

### Common misconceptions

- ❌ "You need a framework to build an LLM app." → ✅ A loop, the provider SDK, and your
  own code are enough — and often clearer for production-critical paths.
- ❌ "LangChain and LangGraph are the same." → ✅ LangChain is the broad toolkit;
  LangGraph is a lower-level graph/state-machine framework purpose-built for stateful
  agents.
- ❌ "DSPy is just another prompt-template library." → ✅ DSPy *optimizes* prompts
  automatically against a metric — prompt engineering as compilation, a different
  paradigm.
- ❌ "Adopt the most popular framework by default." → ✅ Match the framework to your
  dominant pattern (agents → LangGraph, RAG → LlamaIndex, prompt optimization → DSPy,
  type-safe structured agents → Pydantic AI), or use raw calls — popularity is not fit.

### Worked example

A team prototypes a document-Q&A assistant with LlamaIndex — its connectors and
indexing get a working RAG pipeline up in a day. As they move to production they hit
the limits of magic: they cannot see exactly what ends up in the context window, prompt
caching is not hitting because the framework reorders content, and a subtle retrieval
bug is buried under abstraction. For the hot path they drop to raw SDK calls plus a slim
in-house retriever — fewer dependencies, full visibility into the messages sent,
cache-friendly prompt ordering they control. They keep LlamaIndex for offline indexing.
Framework for prototyping and the part it does well; raw calls for the
production-critical path.

### Check questions

1. A team's framework-built RAG feature has a prompt-caching problem: cache hit rates
   are near zero and they cannot tell why. They suspect the provider. Why is the
   *framework* a more likely culprit, and what does this say about frameworks on hot
   production paths? — **Answer:** Prompt caching depends on requests sharing an *exact
   prefix*; a framework that reorders or rebuilds the context between calls (or hides
   what it actually assembles) silently breaks prefix stability, so cache hits collapse —
   and because the framework obscures the exact tokens sent, the team cannot even see
   it. The lesson: on hot, production-critical paths where prompt caching, cost, and
   debugging depend on knowing *exactly* what hits the model, a framework's abstraction
   can be a liability — mature practice prototypes with a framework, then drops to raw
   calls (or a thin in-house layer) for those paths.
2. A team currently hand-writes and manually tunes their prompt strings against a
   quality metric, iterating by trial and error. A colleague says "DSPy would automate
   exactly this." Explain what DSPy would change about their workflow — and why it is a
   different paradigm, not just a fancier template library. — **Answer:** A template
   library only substitutes variables into prompt strings the team still writes and
   tunes by hand — it automates *string assembly*, not *quality*. DSPy changes the
   workflow itself: the team declares each step's input→output *signature* and supplies
   the *metric*, and DSPy compiles and optimizes the actual prompts and few-shot
   examples automatically against that metric. The manual trial-and-error tuning is
   replaced by optimization — "programming, not prompting" — which is a different
   paradigm, not a richer templating tool.

---

## 15.6 — Inference providers and gateways

### Concept

There is a layer between "I picked a model" and "I am calling an API" that the previous
sub-chapters skipped: **who actually runs the inference**, and **what sits in front of
it**. For an open-weights model especially, "use Llama" is not a deployment — someone
has to host it. There are three options, and a fourth thing — a gateway — that can wrap
any of them.

**1. The first-party provider.** Closed models have exactly one source: Anthropic for
Claude, OpenAI for GPT, Google for Gemini. No choice of host — you call the lab.

**2. Self-hosting** (15.2). You run the open-weights model on your own GPUs with vLLM/
SGLang/TGI. Maximum control, maximum operational burden.

**3. Third-party inference providers.** This is the gap to fill. A set of companies run
open-weights models *as a service* — you call their API, they own the GPUs and the
serving stack. You get open-model economics and choice without operating any
infrastructure. The notable players:

- **Together AI, Fireworks AI, Baseten, Replicate** — host a broad catalog of
  open-weights models (Llama, DeepSeek, Qwen, Mistral, etc.) behind an API, usually with
  an OpenAI-compatible endpoint. They compete on price, model selection, latency, and
  fine-tuning/dedicated-deployment options.
- **Groq, Cerebras, SambaNova** — inference specialists running open models on
  *custom hardware* (Groq's LPU, Cerebras's wafer-scale chip) to deliver dramatically
  higher tokens/sec than GPU serving. You go to them specifically for **low latency /
  high throughput** on supported models.
- **Cloud-platform model services — AWS Bedrock, Google Vertex AI, Azure AI Foundry.**
  The hyperscalers each offer a managed catalog that includes *both* third-party
  closed models *and* open models through one API, inside the cloud account you already
  have. The pull is **procurement and governance**: billing, IAM, VPC networking, data
  residency, and compliance all run through your existing cloud relationship — often the
  deciding factor for an enterprise, independent of the model itself. (Note Bedrock and
  Vertex also serve some first-party frontier models — e.g. Claude is available on both
  — so they blur the "first-party vs. third-party" line.)

**4. Gateways / routers — a different thing.** A **gateway** (or **LLM router**) is not
a host; it is a *single API endpoint that sits in front of many providers and models*.
**OpenRouter** is the best-known: one API key, one OpenAI-compatible interface, and
behind it hundreds of models across dozens of providers, with automatic
fallback/routing and unified billing. Gateways come in two flavors that are worth
distinguishing:

- a **hosted/proxy gateway** (OpenRouter) — a third party in your request path;
- a **self-hosted gateway library** (LiteLLM, and the gateway features inside
  observability tools) — you run the routing layer yourself.

**What a gateway changes — and what it does not.** A gateway *does* give you: one
integration instead of N; provider **fallback and load-balancing** without bespoke code
(Topic 12's reliability layer, bought rather than built); **cost-based or
latency-based routing** across providers; centralized **spend caps, keys, logging, and
caching**; and access to many models without N contracts. What it does *not* erase:
the **API-shape differences** of 15.3 still exist underneath — a good gateway normalizes
them for you (that is much of its value), but a tool result or a reasoning field can
still leak through, so you verify behavior per model. A gateway also adds a **hop**
(latency) and, if hosted, **another party in your data path** (a privacy/compliance
consideration) and **another point of failure**. And a gateway does not improve the
underlying model — it is plumbing, not capability.

**The build-vs-buy decision.** Frame it like any infrastructure choice:

- *First-party API* — the default for a closed frontier model; simplest, no middle
  layer.
- *Third-party inference provider* — when you want an open model without running GPUs,
  or want a specific provider's price/latency (e.g. Groq for speed).
- *Self-host* — when data residency, scale economics, or deep customization force it
  (15.2).
- *Cloud-platform service (Bedrock/Vertex/Azure)* — when enterprise procurement,
  governance, and existing-cloud integration dominate the decision.
- *Gateway* — when you want multi-provider reach, fallback, and unified
  cost/observability *without building that layer yourself* — i.e. it is the
  buy-rather-than-build answer to the provider-abstraction layer of 15.1/15.3. The
  trade-off is the extra hop and (for a hosted gateway) the extra party in your data
  path.

The throughline of Topic 15: you choose a model, *and* you choose how it is served, *and*
you decide whether a gateway buys you the abstraction layer or you build it yourself.

### Key terms

- **Inference provider** — a company that hosts open-weights models as an API service,
  so you get open-model economics without operating GPUs (Together, Fireworks, Baseten,
  Groq, etc.).
- **Inference-specialist hardware** — custom non-GPU silicon (Groq LPU, Cerebras
  wafer-scale) used to serve open models at very high tokens/sec.
- **Cloud-platform model service** — a hyperscaler's managed model catalog (AWS
  Bedrock, Google Vertex AI, Azure AI Foundry) accessed through your existing cloud
  account, chosen largely for procurement/governance fit.
- **Gateway / LLM router** — a single API endpoint in front of many providers/models
  (OpenRouter hosted; LiteLLM self-hosted), adding fallback, routing, and unified
  keys/billing/observability.
- **Build vs. buy (provider abstraction)** — building your own provider-abstraction
  layer (15.1/15.3) versus buying it as a gateway.

### Common misconceptions

- ❌ "Open-weights models can only be self-hosted." → ✅ Third-party inference providers
  (Together, Fireworks, Groq, …) host them as an API — open-model choice and economics
  with zero infrastructure.
- ❌ "A gateway is just another model provider." → ✅ A gateway *hosts nothing*; it is a
  routing layer in front of many providers, bought instead of building your own
  abstraction.
- ❌ "A gateway makes all models behave identically." → ✅ It normalizes much of the
  API-shape difference (15.3), but underlying model behavior still varies — verify per
  model; the gateway is plumbing, not capability.
- ❌ "Bedrock/Vertex/Azure are just reseller markups." → ✅ Their pull is procurement,
  IAM, VPC/data-residency, and compliance through your existing cloud account — often
  the deciding enterprise factor independent of the model.
- ❌ "A gateway has no downsides." → ✅ It adds a network hop, another point of failure,
  and — if hosted — another party in your data path; weigh that against the reach and
  fallback it buys.

### Worked example

A team built on the OpenAI first-party API wants three things: a cheaper open model for
high-volume simple requests, a Claude fallback for reliability, and one place to see
spend across all of it. Two routes. **Build:** stand up the provider-abstraction layer
of 15.1/15.3 themselves — adapters for each API shape, fallback logic, a cost dashboard
— full control, real engineering. **Buy:** put a **gateway** (OpenRouter hosted, or
LiteLLM self-hosted) in front — one OpenAI-compatible endpoint reaches an open model on
a third-party inference provider, GPT, and Claude; fallback and unified billing/logging
come built in. They choose the self-hosted LiteLLM gateway: it buys the abstraction and
fallback layer without a hosted third party in their data path, at the cost of running
the gateway and accepting one extra hop. The model choices are unchanged — what changed
is who built the layer in front of them.

### Check questions

1. A team needs to serve an open-weights Llama-class model in production but has no
   platform team and does not want to run GPUs. They conclude "open weights means we
   must self-host, and we can't, so we're stuck with closed models." Where is their
   reasoning wrong, and what are their actual options? — **Answer:** The false premise
   is "open weights ⇒ self-host." Third-party **inference providers** (Together,
   Fireworks, Baseten, Groq, and others) run open-weights models *as an API service* —
   the team gets the open model they want, with its economics and choice, while the
   provider owns the GPUs and serving stack. A cloud-platform service (Bedrock, Vertex,
   Azure) is another managed route, attractive if procurement/governance through their
   existing cloud matters. Self-hosting is only *one* way to consume an open model, and
   it is the one with the heaviest operational burden — not a precondition.
2. A team is about to build the provider-abstraction layer from 15.1/15.3 — adapters per
   API shape, fallback logic, unified cost tracking. An architect says "a gateway gives
   us most of that off the shelf." Explain what a gateway would and would not replace,
   and the trade-off of using one. — **Answer:** A gateway (OpenRouter hosted, or
   LiteLLM self-hosted) is exactly the *buy* answer to that *build*: it gives a single
   endpoint in front of many providers/models, normalizes most of the API-shape
   differences, and provides fallback/routing and unified keys, billing, and logging —
   so it replaces the bulk of the bespoke abstraction layer. What it does *not* replace:
   underlying model behavior still varies, so per-model regression evals (and verifying
   tool/streaming behavior) are still on the team. The trade-offs are an added network
   hop, another point of failure, and — for a *hosted* gateway — another party in the
   data path (a privacy/compliance consideration), which a self-hosted gateway avoids.
   The gateway is plumbing bought instead of built, not a capability upgrade.

---

## Topic 15 — Exam Question Bank

> These questions test *durable concepts* — provider abstraction, the open/closed
> trade-off, API-shape families, the supporting surface, frameworks, and inference
> providers/gateways — deliberately *without* depending on any specific model name,
> version, or price. The May-2026 model snapshot in 15.1 is illustrative only and is
> never the subject of an exam answer; if a model is named here it is as a stand-in for
> a *role* (a closed frontier model, an open-weights model), not a tested fact.

### True / False

1. There is one clearly best frontier model for all tasks. — **Answer:** False. The
   best model depends on the task (coding, multimodal, long-context, cost,
   self-hosting), and the ranking changes every few months — which is why model choice
   belongs behind a provider abstraction and is decided by your own evals, not a
   snapshot.
2. "Open weights" means the model is free to run. — **Answer:** False. Open weights
   means downloadable/self-hostable weights; you still pay for GPUs and operations —
   cheaper only at sufficient steady volume.
3. In the Anthropic Messages API, the system prompt is the first message in the
   `messages` array. — **Answer:** False. It is a separate top-level `system`
   parameter; the first-message convention is OpenAI Chat Completions.
4. The Batch API trades latency for roughly 50% lower cost. — **Answer:** True. It
   processes bulk requests asynchronously (up to ~24 h) at about half the synchronous
   price.
5. Embedding vectors from different embedding models can be compared directly. —
   **Answer:** False. Each model produces vectors in its own space; switching models
   requires re-embedding the whole corpus.
6. OpenAI's Responses API can optionally retain conversation state server-side. —
   **Answer:** True. You can pass a previous response ID instead of resending the full
   history — a departure from the stateless Chat Completions model.
7. You need a framework like LangChain to build a production LLM application. —
   **Answer:** False. A loop, the provider SDK, and your own code suffice; raw calls
   are often clearer for production-critical paths.
8. Anthropic provides its own first-party embeddings model. — **Answer:** False.
   Anthropic does not; Anthropic-centric RAG stacks use a third-party embedding
   provider.
9. An open-weights model can only be run by self-hosting it on your own GPUs. —
   **Answer:** False. Third-party inference providers host open-weights models as an
   API service; cloud-platform model services do too. Self-hosting is one route, not a
   precondition.
10. A gateway (e.g. OpenRouter, LiteLLM) hosts models itself. — **Answer:** False. A
    gateway hosts nothing — it is a routing layer in front of many providers/models,
    adding fallback, unified billing, and one interface; it is the "buy" answer to
    building your own provider abstraction.

### Multiple Choice

1. A team picks a frontier model by reading this month's public benchmark leaderboard
   and hard-codes the winner. Six months later a competitor model leads. What is the
   durable lesson their approach missed? A) They should always pick the cheapest model
   B) Public leaderboards are contaminated and useless C) The frontier reorders every
   few months, so model choice should sit behind a provider abstraction and be driven
   by *your own* eval set, not a public snapshot D) They should have picked the
   open-weights option — **Answer:** C. The specific "best" model is perishable;
   the durable practice is to abstract over the provider so swapping is a config change,
   and to choose by measured fit on your own (uncontaminated) evals. Picking by a public
   snapshot guarantees you are wrong within months.
2. The strongest reason to choose open-weights models despite the operational burden
   is: A) They are always more capable B) Full data control / residency — the model
   runs in your own infrastructure C) They never need GPUs D) They are easier to
   operate — **Answer:** B.
3. An app built on OpenAI Chat Completions accesses the model output as
   `response.choices[0].message.content` and sets the system prompt as `messages[0]`.
   A developer adds Claude as a drop-in fallback by just changing the API key. What
   breaks, and what does this reveal? A) Nothing — the APIs are identical B) Claude
   returns typed `content` blocks (no `choices` array) and takes the system prompt as a
   separate top-level parameter, so both access patterns fail — revealing the need for a
   provider-abstraction layer C) Only the model name needs changing D) Claude does not
   support tool calls — **Answer:** B. The two APIs differ in response shape and
   system-prompt placement (among other things), so provider-specific shapes leaking
   into app code make swapping impossible without an abstraction layer.
4. A pipeline sends the same 80-page PDF to the model in 200 separate requests, each
   re-encoding the full PDF bytes inline. Which part of the supporting API surface
   directly fixes this, and how? A) The Batch API — it makes the requests cheaper
   B) The Files API — upload the PDF once and reference it by ID across all 200
   requests, instead of re-inlining the bytes each time C) An embeddings API — embed
   the PDF D) The SDK — it compresses requests — **Answer:** B. The Files API exists
   precisely to upload a file once and reference it by ID, avoiding repeated inline
   re-encoding of the same large content. (Batch would help only if the work were
   also offline; the inefficiency here is the repeated re-encoding.)
5. A team has a measurable quality metric and wants the *system* to improve its own
   prompts rather than hand-tuning prompt strings. Which framework's philosophy matches
   this, and how does it differ from a prompt-template library? A) LlamaIndex — it
   indexes data B) DSPy — you declare each step's input→output signature and a metric,
   and it compiles/optimizes the actual prompts and few-shot examples automatically;
   a template library only substitutes variables into strings you still hand-write
   C) LangGraph — it draws the agent as a graph D) Pydantic AI — it validates outputs —
   **Answer:** B. DSPy treats prompt engineering as an optimization/compilation problem
   ("programming, not prompting"); a template library does no optimization at all.
6. A team's core problem is high-quality retrieval over a large internal document
   corpus — chunking, indexing, query pipelines. Which framework is the best *fit*, and
   what is the reasoning principle? A) DSPy, because it optimizes prompts B) LlamaIndex,
   because it is RAG/knowledge-specialized — match the framework to your *dominant
   pattern* C) LangGraph, because it is the most powerful D) Whichever has the most
   GitHub stars — **Answer:** B. LlamaIndex specializes in retrieval/indexing pipelines.
   The principle is fit-to-dominant-pattern: agents → LangGraph, RAG → LlamaIndex,
   prompt optimization → DSPy, type-safe structured agents → Pydantic AI — popularity is
   not fit.
7. A team uses the official `openai` and `anthropic` SDKs and assumes that means their
   code is now provider-portable. Why is that assumption wrong, and what is still
   needed? A) It is correct — SDKs make code portable B) SDKs handle reliability
   (retries, streaming parsing, typed objects) but are each provider-specific, so a
   thin provider-abstraction layer is still needed above them for portability C) SDKs
   are slower than raw HTTP D) Only one SDK can be used per app — **Answer:** B. Use
   the SDK over raw HTTP for built-in reliability behavior — but the SDK does not erase
   provider API differences; multi-provider portability still requires your own
   abstraction layer on top.
8. Switching your RAG system's embedding model requires: A) Nothing B) Re-embedding the
   entire corpus C) Only a config change D) Upgrading the chat model — **Answer:** B.
   Vectors are not comparable across models.
9. A team wants to add an open-weights model for high-volume requests but has no
   platform team and will not run GPUs. The cleanest option that gives them the open
   model without operating infrastructure is: A) They cannot use open models at all
   B) A third-party inference provider (or a cloud-platform model service) that hosts
   open-weights models as an API C) Build their own GPU cluster D) Switch entirely to
   the flagship closed model — **Answer:** B. Inference providers (Together, Fireworks,
   Groq, …) and cloud-platform services (Bedrock/Vertex/Azure) host open-weights models
   behind an API — open-model economics and choice with zero infrastructure;
   self-hosting is not a precondition for using an open model.
10. The most accurate description of what an LLM gateway (OpenRouter / LiteLLM) buys you
    is: A) A more capable model B) A single endpoint in front of many providers/models
    with fallback, routing, and unified billing/observability — i.e. the buy-rather-than-
    build answer to a provider-abstraction layer C) Free inference D) Elimination of all
    per-model behavior differences — **Answer:** B. A gateway is plumbing, not
    capability: it consolidates many providers behind one interface with fallback and
    unified cost/logging. It normalizes most API-shape differences but underlying model
    behavior still varies, and it adds a hop (and, if hosted, another party in the data
    path).

### Short Answer

1. "Within a family, the tier matters as much as the family." Explain what this means
   and why getting the *tier* wrong wastes money or quality regardless of which family
   you chose. — **Model answer:** Every major family ships capability tiers (e.g. a
   flagship, a balanced mid-tier, a small fast tier) that differ several-fold in price
   and substantially in latency. Most production traffic should run on the mid-tier;
   the flagship is reserved for genuinely hard requests and the small tier for
   high-volume simple ones (model cascading). If you default everything to the flagship
   you burn cost on requests that did not need it; if you default everything to the
   small tier you lose quality on the hard ones. So the tier decision drives cost and
   quality independently of — and as much as — the family-vs-family choice.
2. Two startups both adopt open-weights models. Startup A does it because "open weights
   are free." Startup B does it because they have a strict data-residency requirement.
   Which startup's reasoning is sound, and why is the other's a likely mistake? —
   **Model answer:** B's reasoning is sound — data residency/control is the strongest
   driver toward open weights, and it is a requirement no closed API tier can satisfy.
   A's reasoning is flawed: open weights are not "free" — you trade per-token API fees
   for GPUs, a serving stack, scaling, on-call, and a homegrown safety layer. Self-hosting
   is only cheaper at sufficient *steady* volume; for a small or spiky-traffic startup it
   is usually *more* expensive and operationally heavier than an API. The choice is
   control vs. convenience, not free vs. paid.
3. Both the Anthropic Messages API and OpenAI Chat Completions are *stateless* — you
   resend the full conversation each turn. OpenAI's Responses API adds an *optional*
   server-side-state mode. Explain what changes for the developer with server-side
   state, and why "optional" matters. — **Model answer:** With server-side state, the
   developer passes a previous-response ID and lets the provider retain the
   conversation, instead of re-sending the entire history every turn — less data over
   the wire and simpler client code for multi-turn flows. It is a genuine departure
   from the stateless model, not a rename. "Optional" matters because stateless calls
   remain valid and are simpler/portable for lightweight flows; the developer chooses
   per use case rather than being forced into server-managed state.
4. What is the Batch API for and what is its trade-off? — **Model answer:**
   Asynchronous bulk processing of latency-tolerant workloads (offline classification,
   eval-dataset generation, backfills) within ~24 h at ~50% cost. Trade-off: results
   take minutes-to-hours, so it is unsuitable for interactive use.
5. A team's RAG system runs on embedding model X. A newer embedding model Y benchmarks
   better, so they switch their *query* embedding to Y while leaving the existing corpus
   vectors (built with X) in place. Retrieval quality collapses. Explain why — and what
   they actually had to do. — **Model answer:** Each embedding model produces vectors in
   its *own* space — different geometry, often different dimensionality — so a vector
   from model Y is not comparable to a vector from model X; cosine similarity between
   them is meaningless. Querying with Y against an X-built index returns essentially
   noise. To adopt Y they must **re-embed the entire corpus** with Y so queries and
   corpus live in the same space. Embedding-model choice is effectively locked in once a
   corpus is indexed; switching is a full re-index, not a config change.
6. When are raw API calls better than a framework? — **Model answer:** For simple tasks
   where a framework only adds indirection, and when you need precise control and
   visibility over the exact tokens in the context window (prompt caching, cost,
   debugging). Common practice: prototype with a framework, drop to raw calls for hot
   production paths.
7. When a team self-hosts an open-weights model, they often budget for GPUs and a
   serving stack but forget one whole category of work that a closed API was silently
   providing. What is it, and why does it matter? — **Model answer:** The safety /
   guardrail layer. A closed provider does safety tuning and ships moderation/guardrail
   infrastructure; self-hosting an open model means *you* own moderation, injection/
   jailbreak detection, and content filtering — there is no provider safety net. It
   matters because a team that budgets only for compute will ship an open-model system
   with no guardrails at all. ("Open weights" also typically excludes training data/code
   and provider-managed updates — but the forgotten *operational* cost is owning safety.)
8. Distinguish a third-party *inference provider* from an LLM *gateway*, and say when a
   team would reach for each. — **Model answer:** An *inference provider* (Together,
   Fireworks, Baseten, Groq, …) actually *hosts and runs* open-weights models — it owns
   the GPUs/serving stack and exposes the model behind an API; you reach for one when
   you want an open model without operating infrastructure, or want a specific
   provider's price/latency (e.g. specialist hardware for high tokens/sec). A *gateway*
   (OpenRouter hosted, LiteLLM self-hosted) *hosts nothing* — it is a routing layer that
   sits in front of many providers/models, giving one endpoint, fallback/routing, and
   unified keys/billing/observability; you reach for one when you want multi-provider
   reach and the provider-abstraction/reliability layer *bought rather than built*. One
   runs models; the other routes among runners. (Cloud-platform services like
   Bedrock/Vertex/Azure are a third thing again — managed catalogs chosen mainly for
   procurement and governance fit.)

### Long Answer

1. Lay out the open- vs. closed-weights decision framework with the drivers. — **Model
   answer / rubric:** Frame it as control vs. convenience, not free vs. paid. Closed
   (API): zero infrastructure, top frontier capability, provider-managed safety/uptime;
   but data leaves your boundary, vendor lock-in, price/availability/behavior exposure,
   limited customization. Open: full data control/residency, deep customization
   (fine-tune, quantize, custom serving), no lock-in, offline/edge; but you own GPUs,
   the serving stack, scaling, on-call, and the safety layer, the top of the frontier is
   often still closed, and "open weights" usually excludes training data/code. Drivers:
   data sensitivity/residency/compliance, scale and cost profile (steady high volume vs.
   spiky/low), customization need, team capability to operate GPU infra, need for
   absolute top capability. Many run a hybrid, enabled by a provider abstraction.
2. Compare the Anthropic Messages API, OpenAI Chat Completions, and OpenAI Responses
   API. — **Model answer / rubric:** Messages (Anthropic): `messages` array + separate
   `system` parameter; typed content blocks (text/image/tool_use/tool_result/thinking);
   tool results as a `user`+`tool_result` block; `max_tokens` required; stateless.
   Chat Completions (OpenAI): `messages` array with system as the first message; tool
   calls as `tool_calls` with JSON-string arguments; `tool`-role tool results; `choices`
   array; stateless; the de facto OpenAI-compatible standard. Responses (OpenAI):
   typed `items` output, optional server-side state via a previous-response ID, built-in
   hosted tools (web/file search, code interpreter, computer use, remote MCP), better
   reasoning support and prompt-cache utilization; recommended for new projects, Chat
   Completions still supported. Conclusion: differences in system-prompt placement,
   tool-result roles, argument encoding, response shape, and statefulness justify a
   provider-abstraction layer.
3. When do frameworks help and when are raw API calls better? Give a reasoned position.
   — **Model answer / rubric:** Frameworks help for fast prototyping with
   batteries-included integrations and when your dominant pattern matches what a
   framework does well — LangGraph for stateful agent graphs, LlamaIndex for RAG
   pipelines, DSPy for automatic prompt optimization, Pydantic AI for type-safe
   structured agents. Raw calls (or a thin in-house layer) are better for simple tasks
   (frameworks add indirection and dependencies), when you need precise control and
   visibility into the exact context window (prompt caching, cost, debugging), and when
   you are fighting abstraction leaks. A framework is a tool matched to a pattern, not a
   default; a mature path prototypes with a framework and drops to raw calls for hot,
   production-critical paths — but always knowing exactly what hits the model.
4. Explain the supporting API surface — SDKs, Batch, Files, embeddings — and when each
   is used. — **Model answer / rubric:** SDKs: provider client libraries handling auth,
   serialization, typed objects, streaming parsing, retries/backoff — use over raw HTTP
   for built-in reliability; still provider-specific. Batch API: asynchronous bulk
   processing (≤~24 h) at ~50% cost for latency-tolerant work — bulk classification,
   eval-dataset generation, backfills, synthetic data; wrong for interactive use. Files
   API: upload files once and reference by ID — model-input documents/images and batch
   input/output files. Embeddings APIs: a separate model class mapping text/images to
   dense semantic vectors — powering RAG, semantic search, clustering, semantic caching;
   note Anthropic has no first-party embeddings model, and switching embedding models
   means re-embedding the corpus.
5. Lay out the ways an LLM can be *served* and the build-vs-buy decision. — **Model
   answer / rubric:** Four ways to consume a model: (a) the *first-party provider* — the
   only source for a closed frontier model, simplest, no middle layer; (b) *self-hosting*
   an open-weights model on your own GPUs (vLLM/SGLang/TGI) — maximum control and the
   heaviest operational burden, chosen when residency/scale/customization force it;
   (c) a *third-party inference provider* (Together, Fireworks, Baseten, Groq, …) that
   hosts open-weights models as an API — open-model economics and choice with zero
   infrastructure, with inference specialists (Groq, Cerebras) offering custom hardware
   for very high tokens/sec; (d) a *cloud-platform model service* (AWS Bedrock, Google
   Vertex, Azure AI Foundry) — a managed catalog of closed and open models accessed
   through your existing cloud account, chosen mainly for procurement, IAM, VPC/data-
   residency, and compliance fit. Separately, a *gateway* (OpenRouter hosted, LiteLLM
   self-hosted) is not a host but a routing layer in front of many providers — the
   buy-rather-than-build answer to the provider-abstraction layer of 15.1/15.3, adding
   fallback, routing, and unified billing/observability at the cost of an extra hop and
   (if hosted) another party in the data path. The senior signal is recognizing that
   model choice and serving choice are *separate* decisions, and that a gateway lets you
   buy the abstraction rather than build it.

### Applied Scenario

1. Your app hard-codes the OpenAI Chat Completions API throughout. Leadership wants a
   Claude fallback for reliability and a cheaper open model for high-volume simple
   requests. What do you build? — **Model answer / rubric:** A provider-abstraction
   layer. Define a neutral internal interface — `generate(system, messages, tools,
   params)` — and one adapter per provider that maps it onto Messages, Chat Completions
   / Responses, and the open-model server, normalizing the differing system-prompt
   placement, tool-result roles, argument encoding, response shapes, and streaming
   formats back to a common shape. On top of that, add routing/cascading (cheap open
   model for simple requests, escalate hard ones), fallback (circuit breaker → alternate
   provider), and per-model regression evals so a swap is a config change and quality is
   verified. Application code now speaks one shape.
2. A regulated financial firm cannot send customer data to any third-party API but
   wants an LLM-powered internal assistant. Walk through the model and infrastructure
   decision. — **Model answer / rubric:** The compliance constraint rules out closed
   APIs even with zero-retention — choose an open-weights model (Llama/Qwen/Mistral/
   DeepSeek-class) self-hosted entirely within the firm's own infrastructure or VPC,
   possibly air-gapped. Plan the cost honestly: GPUs, a serving stack with continuous
   batching (vLLM/SGLang/TGI), autoscaling, monitoring/observability, on-call, and a
   homegrown safety/guardrail layer (no provider moderation). Choose model size/tier by
   measured task fit and available hardware; consider LoRA/QLoRA fine-tuning on
   de-identified domain data and quantization for serving efficiency. The driver is data
   residency/compliance, which only self-hosted open weights satisfy — cost is secondary.
3. A research team is building an LLM feature and has three distinct jobs: (a) generate
   a 30,000-example labeled eval dataset once, offline, as cheaply as possible;
   (b) at request time, find the most relevant prior research notes to ground each
   answer; (c) handle the interactive user-facing answer generation itself. Map each
   job to the right part of the API surface and justify, then name the one decision in
   (b) that is expensive to reverse later. — **Model answer / rubric:** (a) Eval-dataset
   generation is bulk and offline → write requests to a JSONL file, upload via the
   Files API, run via the Batch API (~50% cost, ~24 h window, no rate-limit
   firefighting); latency is irrelevant. (b) Finding relevant notes at request time is
   semantic retrieval → an embeddings API: embed the notes into a vector store once,
   embed each query live, retrieve by cosine similarity. (c) Interactive answer
   generation → standard synchronous generation calls (Messages / Chat Completions /
   Responses), streamed. The expensive-to-reverse decision in (b) is the *choice of
   embedding model*: vectors are not comparable across models, so switching later forces
   re-embedding the entire notes corpus. Three jobs, three parts of the surface — the
   senior signal is not forcing one API to do all of it.
4. A startup is choosing a framework to build a customer-support agent and is leaning
   toward "whatever is most popular." Advise them. — **Model answer / rubric:**
   Popularity is not fit. Identify the dominant pattern: a support agent is a stateful
   agentic workflow with tools, retries, and human-in-the-loop — LangGraph (graph/state
   machine, checkpoints, interrupts) fits that well; LlamaIndex fits if retrieval over a
   knowledge base is the core; Pydantic AI fits if type-safe structured tool I/O is a
   priority; DSPy fits if they want automatic prompt optimization against a metric.
   Recommend prototyping with the framework that matches the dominant pattern, but
   keeping the hot, production-critical paths thin and controllable — possibly raw SDK
   calls plus a slim harness — so prompt caching, cost, and debugging stay transparent.
   Choose by pattern fit and the ability to see exactly what hits the model, not by
   GitHub stars.

---

## Sources

[1] Anthropic — Pricing (current Claude models: Opus 4.7, Sonnet 4.6, Haiku 4.5) — https://platform.claude.com/docs/en/about-claude/pricing
[2] OpenAI — Introducing GPT-5.5 — https://openai.com/index/introducing-gpt-5-5/
[3] Google DeepMind — Gemini 3.1 Pro model card — https://deepmind.google/models/model-cards/gemini-3-1-pro/
[4] Anthropic — Messages API reference — https://platform.claude.com/docs/en/api/messages
[5] OpenAI — Migrate to the Responses API — https://developers.openai.com/api/docs/guides/migrate-to-responses
[6] Anthropic — Embeddings (Claude API docs; points to third-party embedding providers) — https://platform.claude.com/docs/en/build-with-claude/embeddings
