# Topic 12 — Production Concerns — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This file is taught sub-chapter by sub-chapter, in order. Each sub-chapter has core
teaching content, key terms, common misconceptions, a worked example, and check
questions the tutor uses to confirm understanding before moving on. A mixed-format exam
bank is at the end; the tutor draws from it for the gated topic exam (scored out of
100, pass mark 85). Topic 12 is about everything *around* the model call — the
engineering that turns a working prototype into a service that is fast, cheap,
reliable, observable, and secure under real load.

---

## 12.1 — Latency metrics — TTFT, TPOT/ITL, tokens/sec

### Concept

A single LLM request is not one latency number — it is a *curve*. Because models
generate text autoregressively (one token at a time), the user experiences a
*beginning* and a *streaming tail*. You must measure both separately or you will
optimize the wrong thing.

The canonical metrics:

- **TTFT (Time To First Token)** — wall-clock time from sending the request to
  receiving the first output token. TTFT is dominated by the **prefill** phase: the
  model must process every input token before it can emit anything. TTFT therefore
  scales with *prompt length* (and with queue wait time at the provider). A 200-token
  prompt might give 300 ms TTFT; a 100k-token prompt might give 4–8 s. Prompt caching
  attacks TTFT directly.
- **TPOT (Time Per Output Token)**, also called **ITL (Inter-Token Latency)** — the
  average gap between consecutive output tokens during the **decode** phase. This is
  governed by memory bandwidth (each token requires reading the full model weights and
  KV cache), and is roughly constant per model regardless of prompt length. Typical
  frontier-model TPOT is 10–40 ms/token.
- **Tokens/sec** — the reciprocal of TPOT, the more marketing-friendly framing
  (e.g. "50 tok/s"). For a single stream, throughput ≈ 1 / TPOT.
- **Total latency** — `TTFT + (output_tokens − 1) × TPOT`. A 500-token completion at
  20 ms TPOT spends ~10 s in decode *alone*, so for long outputs total latency is
  dominated by output length, not prompt length.

Why the split matters: TTFT and TPOT have *different* root causes and *different*
fixes. Slow TTFT → shorten/cache the prompt, or you are queued (provider capacity).
Slow TPOT → smaller/faster model, or a less loaded endpoint. If you only track "total
latency" you cannot tell which lever to pull. The user-perceived experience also
splits: TTFT is "did it hang?", TPOT is "is it readable as it streams?". Humans read
at ~5–8 words/sec, so any model above ~15 tok/s feels responsive once streaming
starts — which is why TTFT is usually the metric users complain about.

Always report **percentiles**, never means. p50 tells you the typical case; p99 tells
you what your worst-served users get. LLM latency distributions are heavy-tailed
(queueing, retries, long outputs), so a 600 ms p50 can hide a 12 s p99.

### Key terms

- **TTFT** — time to first token; prefill-bound; scales with prompt length.
- **TPOT / ITL** — time per output token / inter-token latency; decode-bound; roughly
  constant per model.
- **Prefill** — parallel processing of the input prompt; compute-bound.
- **Decode** — sequential generation of output tokens; memory-bandwidth-bound.
- **Tail latency** — p95/p99 latency; what your unluckiest users experience.

### Common misconceptions

- ❌ "Latency is one number." → ✅ It is at least two independent numbers (TTFT, TPOT)
  with different causes.
- ❌ "A longer prompt makes every token slower." → ✅ A longer prompt raises TTFT
  (more prefill) but barely changes TPOT; a longer *KV cache* slightly slows decode.
- ❌ "Average latency is fine for SLAs." → ✅ Always use percentiles; means hide tails.

### Worked example

A single request lays out on a time axis as a *beginning* and a *streaming tail* —
TTFT spans the queue wait plus prefill, and every decode tick after the first token is
one TPOT:

```
 t=0                                                          total latency
  │                                                                  │
  ├──[ queue wait ]──[────────── prefill ──────────]│ tok tok tok ... │
  │                                                 │  ↑   ↑   ↑      │
  │◀───────────────── TTFT ───────────────────────▶ │  └─ TPOT ─┘ ... │
  │   (provider capacity)   (scales w/ prompt len)   │  each gap = TPOT│
  │                                       first token┘                │
  │◀──────────────────────────── total latency ──────────────────────▶│
        total = TTFT + (output_tokens − 1) × TPOT
```

A summarization endpoint: 8k-token input, 400-token output. Measured TTFT = 1.2 s,
TPOT = 25 ms. Total ≈ 1.2 + 399 × 0.025 ≈ 11.2 s. The product team wants it under 6 s.
Shortening the prompt to 4k tokens cuts TTFT to ~0.7 s — saves only 0.5 s. The real
win is the output: capping the summary at 150 tokens cuts decode from ~10 s to ~3.7 s,
landing total at ~4.4 s. Lesson: for long outputs, *output length is the latency
budget*, and TTFT optimization is a rounding error.

### Check questions

1. Your TTFT jumped from 0.4 s to 3 s but TPOT is unchanged. Prompt length is
   unchanged. What is the likely cause? — **Answer:** Provider-side queueing /
   capacity pressure. Unchanged prompt rules out more prefill work; unchanged TPOT
   rules out a model swap; the only thing left is wait time before prefill starts.
2. A teammate proposes "if we cut our prompt from 20k tokens to 4k, the answer should
   stream out about 5× faster." Is this right? Explain what actually changes and what
   does not. — **Answer:** No. Shortening the prompt cuts *prefill* work, so TTFT drops
   and the answer starts sooner. But the streaming *rate* (TPOT) is decode-bound and
   memory-bandwidth-limited: each token costs essentially a fixed pass over the model
   weights regardless of prompt size (a longer KV cache adds only a mild effect). So a
   500-token answer still streams at roughly the same tokens/sec — only the time-to-
   start improved, not the per-token speed.

---

## 12.2 — Streaming — SSE and partial-parsing challenges

### Concept

**Streaming** delivers output tokens to the client as they are generated rather than
waiting for the full completion. The dominant transport is **Server-Sent Events
(SSE)**: a long-lived HTTP response with `Content-Type: text/event-stream`, where the
server pushes `data:` lines as chunks arrive. The client reads the stream
incrementally. (WebSockets are also used, especially for bidirectional voice; SSE is
simpler and one-directional, which fits text generation.)

Why stream:

- **Perceived latency.** The user sees output after TTFT (~1 s) instead of after total
  latency (~10 s). The work is identical; the *experience* is transformed.
- **Early cancellation.** If the user navigates away or the answer is clearly wrong,
  you can abort the request and stop paying for decode tokens.
- **Progress signal.** A streaming UI proves the system is alive, reducing "is it
  broken?" abandonment.

The hard part is **partial parsing**. A stream gives you *token fragments*, not
semantically complete units. Concrete problems:

- **Structured output mid-stream.** If the model is emitting JSON, at any instant you
  hold a *prefix* like `{"name": "Ali` — not valid JSON. You cannot `JSON.parse` it.
  Options: (a) buffer everything and parse only at the end (loses streaming benefit for
  structured data); (b) use a streaming/partial JSON parser that yields best-effort
  partial objects; (c) stream a non-JSON surface (markdown) and reserve JSON for
  non-streamed calls.
- **Token-boundary splits.** A multibyte UTF-8 character or an emoji can be split
  across two chunks. Decode the byte stream incrementally with a stateful decoder, not
  chunk-by-chunk, or you get mojibake.
- **Tool calls.** Providers stream tool-call arguments as fragments too; you must
  accumulate `input_json_delta` (Anthropic) / function-argument deltas (OpenAI) until
  the block is complete before executing.
- **Errors after 200 OK.** Streaming sends HTTP 200 immediately, then the body. An
  error mid-stream (rate limit, model failure) arrives as an *error event in the
  stream*, not an HTTP status. Your client must handle a stream that ends abnormally.
- **Stop reasons.** The final event carries why generation stopped (`end_turn`,
  `max_tokens`, `tool_use`, `stop_sequence`). Always inspect it — a `max_tokens` stop
  means the output is truncated and probably invalid.

### Key terms

- **SSE (Server-Sent Events)** — unidirectional server→client streaming over HTTP;
  the standard transport for LLM token streams.
- **Chunk / delta** — an incremental piece of the response (text, tool-arg fragment,
  metadata).
- **Partial parsing** — extracting meaning from an incomplete output prefix.
- **Stop reason / finish reason** — metadata explaining why generation ended.

### Common misconceptions

- ❌ "Streaming makes the response faster." → ✅ It makes it *feel* faster; total token
  count and total decode time are unchanged.
- ❌ "I can JSON.parse each chunk." → ✅ Each chunk is a fragment; mid-stream JSON is an
  invalid prefix. Buffer or use a partial parser.
- ❌ "If HTTP returns 200, the request succeeded." → ✅ With streaming, errors arrive
  inside the stream body after the 200; you must parse the event types.

### Worked example

The SSE exchange is one HTTP request that opens a long-lived response and then pushes a
sequence of events — note the 200 OK arrives *before* any content, so an error event can
still arrive afterward:

```
   CLIENT                                   SERVER
     │                                         │
     │──── POST /messages (stream: true) ──────▶│
     │                                         │  open stream
     │◀──── HTTP 200 OK  text/event-stream ─────│  (status sent BEFORE content)
     │                                         │
     │◀──── data: {chunk}  (text delta) ────────│ ┐
     │◀──── data: {chunk}  (text delta) ────────│ │ repeated as
     │◀──── data: {chunk}  (tool-arg delta) ────│ │ tokens decode
     │◀──── data: {chunk}  ... ─────────────────│ ┘
     │                                         │
     │◀──── event: error  (rate limit / fail) ──│  ← CAN arrive here,
     │              ── or ──                    │    after the 200
     │◀──── final event: stop_reason ───────────│  end_turn / max_tokens / ...
     │                                         │
```

A chat UI streams markdown — works great. The team then adds a "structured profile"
feature that asks the model for JSON and tries to render fields live. They call
`JSON.parse` on the accumulated buffer every chunk and it throws constantly. Fix: they
adopt a partial-JSON parser that returns `{name: "Ali", age: undefined}` for the prefix
`{"name":"Ali","ag`, rendering known fields and leaving the rest blank until complete.
Alternatively, since the profile is small, they stop streaming that one call and parse
once at the end — simpler, and the 800 ms wait is acceptable for a non-chat surface.

### Check questions

1. A product surface streams a long markdown explanation token-by-token and it works
   well. The team wants to add a second feature that returns a strict JSON object the
   UI parses live as it arrives. They copy the same streaming code. Why will the JSON
   feature misbehave where the markdown one did not — and what are the two viable
   fixes? — **Answer:** Markdown is *renderable as a prefix* — a partial markdown string
   is still valid markdown. A partial JSON string is *not* valid JSON (unclosed
   strings/braces), so `JSON.parse` on the accumulating buffer throws on almost every
   chunk. Fixes: (a) use a partial/streaming JSON parser that yields best-effort partial
   objects, or (b) stop streaming that call and parse once at completion (acceptable if
   the payload is small and the surface is non-chat).
2. A streamed request returned HTTP 200, yet the user sees a truncated answer and your
   error rate metric shows nothing. Explain why the error metric missed it and where
   the real signal is. — **Answer:** With streaming, the 200 is sent the instant the
   stream *opens*, before any content — so HTTP status cannot reflect a mid-stream
   problem and a status-based error metric will not flag it. The real signal is the
   final stream event's stop reason (`max_tokens` → truncated at the output cap) or an
   in-stream error event (provider/model failure mid-generation). The client must treat
   both as failures even though the HTTP status was 200.

---

## 12.3 — Cost modeling — input/output/cached pricing; $/request, $/user

### Concept

LLM cost is **metered per token**, and the meter has multiple rates. To run a
sustainable product you must be able to compute $/request and $/user *before* you ship,
and watch them after.

The rate structure (Anthropic published API pricing, May 2026, per **million**
tokens) [1]:

| Model            | Input | Output | Cache write (5m) | Cache read |
|------------------|-------|--------|------------------|------------|
| Claude Haiku 4.5 | $1.00 | $5.00  | $1.25            | $0.10      |
| Claude Sonnet 4.6| $3.00 | $15.00 | $3.75            | $0.30      |
| Claude Opus 4.7  | $5.00 | $25.00 | $6.25            | $0.50      |

(These exact numbers date fast — treat the table as a May-2026 snapshot and verify
against the provider's pricing page; the *structure* below is the durable lesson.)

Three structural facts to internalize:

1. **Output tokens cost several × input tokens.** For all three current Claude models
   the ratio is exactly 5× [1]; OpenAI's and Google's flagship models show a similar
   input-cheaper-than-output pattern — but the exact multiplier is provider- and
   model-specific, not a fixed law. It follows from the mechanics: decode is the
   expensive, memory-bandwidth-bound, sequential phase. Verbose outputs are where
   budgets die. Reasoning/extended-thinking models bill their hidden thinking tokens as
   output, so a "short" answer can have a large invisible output cost.
2. **Cached input is ~10% of base input; a 5-minute cache write is 25% above base
   input.** (Anthropic prices a 5-minute cache write at 1.25× base input and a cache
   read at 0.1× base input; a 1-hour cache write is 2× base input [1].) So caching pays
   off when a prefix is reused more than ~once or twice (see 12.9 / Topic 6).
3. **Batch API is 50% off** both input and output for asynchronous (≤24 h) workloads —
   the cheapest lever if latency does not matter [1].

**The cost model.** For one request:

```
cost = (input_tokens / 1e6) × input_rate
     + (output_tokens / 1e6) × output_rate
     + cache adjustments
```

Then roll up: **$/request** × requests/user/day × 30 = **$/user/month**. Compare that
to revenue/user. A consumer chat app charging $20/month that lets users burn
500k Opus output tokens/day is losing money badly — at $25/M output that is
0.5 × $25 × 30 = **$375/user/month**, ~19× the subscription price.

Practical levers, cheapest first: pick the smallest model that passes evals (a Haiku
call is 5× cheaper input / 5× cheaper output than Opus); cap `max_tokens`; cache stable
prefixes; trim the prompt; route easy requests to a small model and hard ones to a
large model (model cascading); use the Batch API for anything async.

### Key terms

- **$/request** — fully-loaded token cost of a single API call.
- **$/user** — $/request rolled up by usage frequency; the number that must be below
  revenue/user.
- **Model cascade / routing** — sending easy requests to a cheap model, escalating
  only hard ones.
- **Blended rate** — effective $/token after accounting for caching and the in/out mix.

### Common misconceptions

- ❌ "Input and output tokens cost the same." → ✅ Output costs several × input (5× on
  current Claude models [1]); verbose answers dominate cost.
- ❌ "Reasoning models are cheap because the answer is short." → ✅ Hidden thinking
  tokens bill as output and can dwarf the visible answer.
- ❌ "We'll worry about cost after launch." → ✅ Model $/user before launch; a 100×
  miss vs. revenue is an existential bug, not a tuning task.

### Worked example

A support assistant on Sonnet 4.6: each request averages 6,000 input tokens (system
prompt + history + KB snippet) and 350 output tokens. Cost =
(6000/1e6 × $3) + (350/1e6 × $15) = $0.018 + $0.00525 = **$0.0233/request**. Users
average 8 requests/day → $0.186/day → **$5.60/user/month**.

Because output is billed at 5× input on current Claude models, it is clearer to compare
designs in **input-equivalent tokens** — weight each output token as 5 input tokens.
Three candidate designs for the same Sonnet feature:

| Design | Prompt (input) tokens | Output tokens | Input-equivalent total (input + 5 × output) |
|--------|----------------------:|--------------:|--------------------------------------------:|
| A — long prompt, short answer  | 5,000 |   200 | 5,000 + 1,000 = **6,000**  |
| B — short prompt, long answer  | 2,000 | 1,000 | 2,000 + 5,000 = **7,000**  |
| C — this support assistant     | 6,000 |   350 | 6,000 + 1,750 = **7,750**  |

The raw token counts mislead — Design B looks "smaller" at 3,000 raw tokens but is the
most expensive of the three once output is weighted at 5×. A verbose answer, not a long
prompt, is usually where the budget goes.
 Now cache the 4,500-token
static prefix: cached reads cost ~$0.30/M, so input becomes
(4500/1e6 × $0.30) + (1500/1e6 × $3) = $0.00135 + $0.0045 = $0.006, plus output
$0.00525 → **$0.0112/request**, roughly halving the bill to ~$2.70/user/month.

### Check questions

1. A team is comparing two prompt designs for the same Sonnet feature. Design A: a
   2,000-token prompt that yields a 1,000-token answer. Design B: a 5,000-token prompt
   that yields a 200-token answer. Both feel similar to users. Which is cheaper per
   request, and what is the principle? — **Answer:** Design B is cheaper. Output is
   billed at 5× the input rate, so weight each in input-equivalent tokens: A ≈ 2,000 +
   (1,000 × 5) = 7,000; B ≈ 5,000 + (200 × 5) = 6,000. Design A's larger answer costs
   more than its smaller prompt saves. Principle: a token of output is worth ~5 tokens
   of input, so verbose answers — not long prompts — are usually where the budget goes;
   compare designs in input-equivalent terms, not raw token counts.
2. Your $/user is $4 and revenue/user is $10. The CEO wants Opus for "quality." What is
   the risk? — **Answer:** Opus input/output is ~1.7× Sonnet's; $/user could jump to
   ~$7, eroding margin. Cascade: keep Sonnet/Haiku as default, escalate only flagged-
   hard requests to Opus.

---

## 12.4 — Throughput and batching — continuous batching

### Concept

**Throughput** is aggregate tokens/sec across *all* concurrent requests a system
serves — distinct from single-stream latency. A GPU serving one request wastes most of
its parallel compute; the economics of inference depend on packing many requests
together.

The naive approach, **static batching**, groups N requests, runs them in lockstep, and
returns all N when the *slowest* finishes. Problem: completions vary wildly in length
(10 tokens vs. 2,000). With static batching the whole batch waits for the longest
generation, and finished slots sit idle — terrible utilization.

**Continuous batching** (a.k.a. in-flight / dynamic batching, popularized by vLLM and
now standard in TGI, TensorRT-LLM, SGLang) [2] operates at the *token-step* granularity
instead of the request granularity. At every decode step the scheduler can:

- evict requests that just finished (free their slots and KV cache immediately), and
- admit newly arrived requests into the freed slots.

So the batch composition changes every step. A request that needs 2,000 tokens does
not block a 10-token request — the short one leaves, a new one joins. This is the
single biggest lever for inference throughput; it can multiply tokens/sec several-fold
over static batching at the same hardware.

Why you should care as an *API consumer* even though the provider does the batching:

- It explains the **latency/throughput trade-off you observe.** Under heavy provider
  load, your individual TTFT and TPOT rise because your tokens share GPU steps with
  more co-tenants. There is genuine tension: providers tune batch size for throughput
  (cost) vs. per-user latency.
- It explains **rate limits** (12.6): your TPM/RPM caps are how the provider keeps
  batches from overflowing.

(A related provider-internal mechanism, **PagedAttention**, pages the KV cache into
non-contiguous fixed-size blocks to cut memory fragmentation so more requests fit per
GPU — you never touch it as an API consumer, but it is part of why continuous batching
scales [2].)

For self-hosted models, choosing a server with continuous batching (vLLM, SGLang, TGI)
versus a naive loop is often a 5–20× cost difference at the same quality.

### Key terms

- **Throughput** — total tokens/sec across all concurrent requests.
- **Static batching** — fixed request groups; returns when the slowest finishes; poor
  utilization.
- **Continuous batching** — per-token-step scheduling; finished requests leave and new
  ones join mid-batch.
- **PagedAttention** — provider-internal paged KV-cache memory management that cuts
  fragmentation so more requests fit per GPU; an API consumer never touches it.

### Common misconceptions

- ❌ "Higher throughput means lower latency." → ✅ They trade off; bigger batches raise
  throughput (and lower cost/token) but raise per-user TPOT.
- ❌ "Batching is the provider's problem, irrelevant to me." → ✅ It explains your
  rate limits and load-dependent latency, and it is *your* problem the moment you
  self-host.
- ❌ "Continuous batching just means a bigger batch." → ✅ It means a *dynamically
  recomposed* batch updated every decode step.

### Worked example

Picture the batch as a grid — rows are batch slots, columns are decode steps. Under
static batching a finished request leaves its slot **idle** until the slowest one ends;
under continuous batching a freed slot is **refilled immediately** with a new request:

```
 STATIC BATCHING (slot freed at step → idle till slowest finishes)
            step→  s1  s2  s3  s4  s5  s6
   slot 1 [ R1 ]   ██  ██  ▒▒  ▒▒  ▒▒  ▒▒    R1 done @s2 → ▒ idle
   slot 2 [ R2 ]   ██  ██  ██  ██  ██  ██    R2 long, blocks the batch
   slot 3 [ R3 ]   ██  ▒▒  ▒▒  ▒▒  ▒▒  ▒▒    R3 done @s1 → ▒ idle
   slot 4 [ R4 ]   ██  ██  ██  ▒▒  ▒▒  ▒▒    R4 done @s3 → ▒ idle
                       ▒ = wasted GPU; batch returns only at s6

 CONTINUOUS BATCHING (freed slot refilled the very next step)
            step→  s1  s2  s3  s4  s5  s6
   slot 1   R1  R1 │R5  R5  R5  R5│         R1 leaves @s2, R5 admitted
   slot 2   R2  R2  R2  R2  R2  R2          R2 still running, no block
   slot 3   R3 │R6  R6  R6 │R8  R8│         R3 leaves @s1, R6 then R8
   slot 4   R4  R4  R4 │R7  R7  R7│         R4 leaves @s3, R7 admitted
                       no idle cells — GPU stays saturated
```

A self-hosted Llama endpoint with static batching size 16: a batch finishes only when
its longest completion (say 1,500 tokens) is done, so 15 slots sit idle for most of the
run. Measured aggregate throughput: ~900 tok/s. Switching to vLLM with continuous
batching, finished requests free their slots instantly and the scheduler keeps the GPU
saturated — aggregate throughput rises to ~4,800 tok/s on the same A100, cutting cost
per million tokens by ~5× with no quality change.

### Check questions

1. A self-hosted endpoint uses static batching of 16. In one batch, 15 requests finish
   in ~50 tokens each and one needs 1,800 tokens. Walk through what the GPU is doing
   for most of that batch, and what continuous batching would do differently. —
   **Answer:** With static batching the whole batch returns only when the *longest*
   completion (1,800 tokens) finishes. The 15 short requests complete early, but their
   slots sit idle for the rest of the run — so for most of the batch the GPU is doing
   ~1/16th of the work it could. Continuous batching schedules per token-step: the
   moment a short request finishes, its slot is evicted and a newly arrived request is
   admitted, so the GPU stays saturated and the slow request never blocks the others.
2. Your provider TPOT rises every weekday afternoon. Continuous-batching explanation?
   — **Answer:** Peak load means more co-tenant requests share each decode step, so
   your tokens get a smaller slice of GPU time per step — higher TPOT despite no
   change on your side.

---

## 12.5 — Speculative decoding — a latency lever

> **Tiered sub-chapter — overview required, deep-dive optional.** This chapter has
> two parts: a short **Overview** that everyone needs (taught and tested normally),
> and a longer **Deep dive** with the mechanism details (skip-by-default, available
> on request). After the Overview and its check questions, the tutor will ask
> whether to take the deep dive now, skip it for later, or advance to §12.6. The deep
> dive is interview-bait and worth doing eventually, but it is not required to
> advance, and its content is not in the topic exam.

### Overview — Concept

**Speculative decoding** is a decode-acceleration technique that an LLM provider may
already be running for you. The scheme uses two models:

- A small, fast **draft model** cheaply *proposes* the next few tokens (say 4) one at a
  time — it is cheap precisely because it is small.
- The large **target model** then verifies all 4 drafted tokens in a *single* forward
  pass. For each position it checks whether the token the big model would itself have
  chosen matches the draft. It **accepts** the longest correct prefix and **rejects**
  the rest, then generates the next token itself.

Crucially this is **lossless**: the output distribution is identical to what the target
model would have produced alone — speculative decoding is an acceleration, not an
approximation, so it does not change quality [4]. Typical speedups are ~1.5–3× on TPOT,
depending on how often the draft is accepted.

The **acceptance rate** is everything. If the draft model agrees with the target most
of the time (easy, predictable text — boilerplate, common phrasing), most drafted
tokens are accepted and the speedup is large. On hard, surprising text the draft is
often wrong, few tokens are accepted, and you have paid for draft work that was thrown
away — the speedup shrinks toward zero (and can even be slightly negative). The draft
model must be *well-matched* to the target: a good predictor of the same distribution.

Why an *API consumer* should care even though the provider implements it: it is one
reason an endpoint's TPOT can vary with *content* (predictable outputs stream faster
than novel ones), and it is a real lever to enable when you self-host — a ~2× decode
speedup at zero quality cost. It belongs in the same mental bucket as continuous
batching: a provider-side mechanism that explains the latency you observe and a knob you
own the moment you run your own serving stack.

### Overview — Key terms

- **Speculative decoding** — accelerating decode by having a small draft model propose
  several tokens that the large target model verifies in one forward pass; lossless.
- **Draft model** — the small, fast model that proposes candidate tokens.
- **Target model** — the large model whose output distribution is preserved; it
  verifies and accepts/rejects drafted tokens.
- **Acceptance rate** — the fraction of drafted tokens the target accepts; governs the
  achieved speedup.

### Overview — Common misconceptions

- ❌ "Speculative decoding trades quality for speed." → ✅ It is *lossless* — the output
  distribution matches the target model run alone; only speed changes.
- ❌ "It always gives a big speedup." → ✅ The speedup tracks the acceptance rate;
  predictable text accelerates well, surprising text barely at all.

### Overview — Worked example

A self-hosted 70B model serves two workloads. A summarization endpoint produces output
that heavily echoes the source document — its text is highly predictable, so the draft
is accepted ~80% of the time and TPOT drops ~2.5×. A creative-writing endpoint produces
novel, surprising prose — the draft is accepted far less often, and speculative decoding
yields only ~1.2×. Same technique, same models; the *content's predictability* set the
acceptance rate and therefore the speedup. Quality is unchanged on both, because the
scheme is lossless.

### Overview — Check questions

1. A teammate worries that turning on speculative decoding will subtly change the
   model's answers and wants to re-run the full eval suite to check for quality
   regressions. Are they right to worry? — **Answer:** No. Speculative decoding is
   *lossless*: the target model verifies every drafted token against what it would
   itself have chosen and only accepts matches, so the final output distribution is
   identical to running the target model alone. It changes *latency*, not *output* — so
   it does not need a quality re-eval (a latency benchmark is the right check). It is an
   acceleration, not an approximation — unlike, say, quantization or a smaller model.
2. Two endpoints run the same speculative-decoding setup; one gets a 2.5× speedup and
   the other barely 1.1×. Neither team changed the models. What property of the
   *traffic* explains the gap? — **Answer:** The **acceptance rate** — how often the
   draft model's proposed tokens match what the target model would have chosen. The
   fast endpoint serves predictable text (boilerplate, output that echoes the prompt),
   so most drafted tokens are accepted and many tokens are emitted per target forward
   pass. The slow endpoint serves novel/surprising text, so the draft is usually wrong,
   few tokens are accepted, and the drafting work is mostly wasted. The technique is
   identical; the predictability of the output distribution sets the speedup.

---

### Deep dive — Concept *(optional)*

The Overview told you speculative decoding is lossless and that its speedup tracks the
acceptance rate. This section unpacks *why* the mechanism is a win in the first place
(it comes straight from decode being memory-bandwidth-bound), and surveys the variants
you may hear named on senior interviews. It is **not required** to advance and is not in
the topic exam.

**Why verifying N tokens ≈ generating 1 — the memory-bandwidth framing.** Decode is the
slow phase (12.1): each output token requires a full pass over the model weights, and
that is **memory-bandwidth-bound** — the GPU spends most of its time moving weights, not
computing. A subtle consequence: generating *one* token and verifying *several* candidate
tokens in a single forward pass cost almost the same, because both are dominated by the
one weight read. The extra arithmetic to score N positions instead of 1 is essentially
free against the bandwidth bill. That is *the* mechanical reason speculative decoding
works at all: the win is throughput per forward pass — when the draft model guesses
well, the target model emits several tokens for the price of one weight read instead of
one.

**Variants you may hear named.** Speculative decoding does not require a separate draft
model:

- **Separate small draft model** — the classic form: a smaller LM of the same family,
  cheap enough that the per-token drafting cost is negligible.
- **Self-speculation / Medusa** — extra lightweight prediction heads bolted on the
  target model itself do the drafting. No separate model to host; the target model
  drafts for itself.
- **N-gram / prompt-lookup decoding** — "drafts" by copying spans straight from the
  prompt. Very effective when the output quotes the input (summarization, code editing,
  RAG answers that echo sources) — no learned draft model needed at all.

### Deep dive — Key terms

- **Memory-bandwidth-bound** — the per-step bottleneck of decode: time is dominated by
  streaming the model's weights from memory, not by arithmetic. The reason scoring N
  candidate tokens in one forward pass costs about the same as scoring 1.
- **Self-speculation / Medusa** — drafting via extra prediction heads on the target
  model itself, eliminating the separate draft model.
- **N-gram / prompt-lookup drafting** — drafting by copying spans from the prompt;
  best when the output echoes the input.

### Deep dive — Common misconceptions

- ❌ "It works because two models are faster than one." → ✅ It works because verifying
  N tokens costs about the same as generating one — decode is memory-bandwidth-bound, so
  the extra verification compute is nearly free.
- ❌ "Speculative decoding always needs a separate draft model." → ✅ Self-speculation
  (Medusa heads) and n-gram/prompt-lookup drafting both avoid a separate model — the
  target drafts for itself, or you copy spans from the prompt.

### Deep dive — Worked example

The draft model proposes a short run of tokens; the target model verifies all of them
in a *single* forward pass, accepts the longest matching prefix, and emits its own
correction for the first mismatch:

```
  DRAFT model proposes 4 tokens:        [ t1 ][ t2 ][ t3 ][ t4 ]
                                           │     │     │     │
  TARGET model verifies all 4 in ONE      ▼     ▼     ▼     ▼
  forward pass — per-position check:      ✓     ✓     ✓     ✗
                                          └──────┬──────┘     │
                          accept prefix t1 t2 t3 ◀┘           │
                          reject t4, TARGET emits correction ◀┘
   → 4 tokens advanced (3 drafted + 1 target) for the price of 1 target pass
```

**The cost framing — why this is a win.** Decode is memory-bandwidth-bound, so a target
forward pass is dominated by the one weight read; verifying N candidate tokens in that
pass costs *about the same* as generating 1:

```
  WITHOUT spec. decoding:  generate 1 token   = 1 target forward pass
  WITH spec. decoding:     verify 4 tokens    ≈ 1 target forward pass
                           → up to 4 tokens advanced per pass instead of 1
```

So the verification work is nearly free; the only question is how many of the drafted
tokens survive (the acceptance rate). Concretely, if the summarization workload above
drafts 4 tokens per round and 80% are accepted on average, each target forward pass
advances ~3.2 drafted tokens plus 1 target-generated correction — roughly 4× the token
output of vanilla decode for ~1× the target-pass cost, which is where the ~2.5× TPOT
gain comes from (the rest goes to the draft model's own work and overhead). At 30%
acceptance the same arithmetic yields ~1.2 drafted + 1 = ~2.2 tokens per pass — barely
better than vanilla, sometimes worse once you pay for the wasted draft work.

### Deep dive — Check questions

1. A teammate says "speculative decoding is faster because the draft model does most of
   the work and the big model just rubber-stamps it." Where is that wrong, and what is
   the actual mechanical reason the scheme wins? — **Answer:** The big model is *not*
   rubber-stamping — it runs a full forward pass to score every drafted position, and
   that pass costs as much as one normal decode step. The win is not that the target
   does less work; it is that *one* target forward pass can verify N tokens for almost
   the same cost as generating 1, because decode is memory-bandwidth-bound and the pass
   is dominated by the one weight read. The drafted tokens that survive verification are
   "free" tokens piggy-backing on a forward pass you were going to pay for anyway.
2. A team can't host a separate draft model alongside their target. Name two ways to
   still get speculative-decoding-style speedups *without* a second model, and the kind
   of workload each one is best at. — **Answer:** (a) **Self-speculation / Medusa** —
   extra lightweight prediction heads bolted onto the target model itself do the
   drafting; works generally because the target drafts for itself. (b) **N-gram /
   prompt-lookup drafting** — propose spans copied straight from the prompt; best when
   the output echoes the input (summarization, code editing, RAG answers that quote
   sources), because the acceptance rate on copied spans is high.

---

## 12.6 — Rate limits — RPM/TPM, backoff, queueing

### Concept

Providers cap how fast you can consume capacity, typically along two axes:

- **RPM — Requests Per Minute**: number of API calls.
- **TPM — Tokens Per Minute**: total tokens (input + often output) processed.

There are frequently more: **ITPM/OTPM** (separate input/output token limits),
**concurrent request** caps, and **daily** caps. Limits are per-org/per-key and scale
with usage tier/spend. You hit a limit and get **HTTP 429 (Too Many Requests)**, often
with a `Retry-After` header.

Two failure shapes: a **burst** (you briefly exceed RPM) and **sustained overload**
(your steady demand exceeds TPM). They need different responses.

**Backoff with jitter.** On a 429 (or 5xx), do not retry immediately or on a fixed
schedule — that creates a *thundering herd* where every client retries in sync and
re-overloads the provider. Use **exponential backoff**: wait `base × 2^attempt`, and
add **random jitter** so retries spread out [3]. Respect `Retry-After` if present (it
overrides your computed delay). Cap the number of retries and the total delay.

```
delay = min(cap, base * 2**attempt) * (0.5 + random())   # full jitter
```

**Queueing.** Backoff handles bursts; it does not fix sustained overload — retrying a
demand that *structurally* exceeds TPM just moves the failure later. For that you need
a **client-side queue with a rate limiter** (e.g. a token-bucket throttle) that paces
outbound requests to stay under TPM/RPM, smoothing bursts into a steady stream. Add a
**concurrency limiter** (max in-flight requests) and a **bounded queue** so that under
true overload you shed load deterministically (reject or degrade) instead of growing an
unbounded backlog and blowing memory/latency.

**Pre-emptive token estimation.** Because TPM counts tokens, a good limiter estimates a
request's token cost *before* sending and only admits it if the budget allows —
otherwise a single huge prompt silently blows the minute's budget.

**Idempotency** (12.7) matters here: retries can cause duplicate side effects unless
calls carry an idempotency key.

### Key terms

- **RPM / TPM** — requests- and tokens-per-minute caps.
- **429** — HTTP "Too Many Requests"; the rate-limit signal, often with `Retry-After`.
- **Exponential backoff with jitter** — retry delay that grows geometrically plus
  randomness, to avoid synchronized retry storms.
- **Token bucket** — a rate-limiter algorithm: tokens refill at a fixed rate; a request
  spends tokens; empty bucket → wait.
- **Thundering herd** — many clients retrying in lockstep and re-triggering the
  overload.

### Common misconceptions

- ❌ "Just retry on 429 until it works." → ✅ Without backoff+jitter you cause a
  thundering herd; without a queue you cannot fix sustained overload.
- ❌ "Fixed-interval retry is fine." → ✅ Fixed intervals keep clients synchronized;
  jitter is what de-synchronizes them.
- ❌ "RPM is the only limit." → ✅ TPM (and sometimes separate input/output token
  limits and concurrency caps) bind first for long-prompt workloads.

### Worked example

A nightly job fires 5,000 classification calls in a tight loop. RPM is 1,000, so ~80%
return 429. The naive fix — retry immediately — produces a synchronized storm and the
job still fails. Correct fix: a token-bucket limiter set to ~950 RPM (headroom under
the cap) plus a concurrency cap of 50; requests are paced, the bucket smooths bursts,
and the job completes in ~5.5 minutes with zero 429s. Even better for a nightly job:
submit it to the **Batch API** — 50% cheaper and no rate-limit choreography at all.

### Check questions

1. A team already uses exponential backoff (`base × 2^attempt`) — no fixed intervals —
   yet during an outage their provider still sees sharp synchronized retry spikes that
   keep re-overloading it. Their backoff has no jitter. Explain why pure exponential
   backoff still produces synchronized spikes, and what jitter changes. — **Answer:**
   Exponential backoff sets *how long* each client waits, but if many clients failed at
   the same instant they all compute the *same* delay and therefore retry at the same
   instant — attempt 1 at +1 s, attempt 2 at +2 s, and so on, in lockstep. The growth is
   geometric but the clients stay phase-aligned, so each retry wave is still a
   thundering herd. Jitter adds randomness to each delay, scattering the retries across
   a window so the waves flatten out.
2. You retry-with-backoff but the job still fails after an hour. What is wrong? —
   **Answer:** Your *sustained* demand exceeds TPM/RPM. Backoff only absorbs bursts;
   you need a rate-limiting queue to pace demand under the cap (or the Batch API).

---

## 12.7 — Reliability — timeouts, fallback models, circuit breakers, idempotency

### Concept

LLM APIs are remote dependencies with non-trivial failure rates: transient 5xx, 429s,
slow tails, regional outages, occasional total provider outage. A production system
needs explicit reliability engineering or it inherits every wobble of its upstream.

**Timeouts.** Never make an LLM call without a timeout. But "one timeout" is wrong
because of streaming: set a **connect timeout** (short, e.g. 5 s — fail fast if the
provider is unreachable), and for streaming use an **inactivity/idle timeout** (e.g. "no
token for 20 s → abort") rather than a single overall deadline, because a legitimate
long completion can run for minutes. A flat 30 s overall timeout would kill valid long
generations *and* hang for 30 s on a dead connection.

**Retries.** Retry *transient* failures (429, 500, 502, 503, 504, connection resets)
with exponential backoff + jitter. Do **not** retry deterministic failures (400 bad
request, 401 auth, 422 schema) — they will fail identically. Cap retry count.

**Fallback models.** If the primary model/provider fails or times out, fall back to an
alternative — a different model, a different region, or a different provider entirely.
This requires either a provider-agnostic abstraction or two SDK paths. Trade-off: a
fallback model may have different behavior, schema quirks, or quality, so eval the
fallback path too — a silent fallback that produces worse answers is a stealth
regression.

**Circuit breakers.** When a dependency is *consistently* failing, retrying every
request just piles latency on a doomed call and can worsen the upstream outage. A
circuit breaker tracks the recent failure rate; when it crosses a threshold the breaker
**opens** and calls fail fast (or go straight to the fallback) without even attempting
the dead provider. After a cooldown it goes **half-open**, lets a probe request
through, and **closes** again if that succeeds. States: closed (normal) → open (fail
fast) → half-open (probing).

**Idempotency.** Retries and fallbacks mean the *same logical request can execute more
than once*. If the LLM call triggers side effects (a tool that sends an email, charges
a card, writes a row), duplicates are a real bug. Attach an **idempotency key** — a
stable unique ID for the logical operation — so the server (or your tool layer)
deduplicates. Many LLM providers accept an idempotency key on the request; your own
tools should honor one too. For pure text generation idempotency is less critical, but
agentic systems with real-world tools must have it.

### Key terms

- **Connect timeout vs. idle timeout** — fail-fast on unreachable provider vs.
  abort-on-stall during a long stream.
- **Transient vs. deterministic failure** — retry the former (429/5xx), never the
  latter (400/401/422).
- **Fallback model** — an alternative model/provider used when the primary fails.
- **Circuit breaker** — closed → open → half-open state machine that stops hammering a
  dead dependency.
- **Idempotency key** — a stable ID making a repeated request execute its side effects
  only once.

### Common misconceptions

- ❌ "One 30 s timeout covers everything." → ✅ Streaming needs an idle timeout; a flat
  deadline kills valid long outputs and hangs on dead connections.
- ❌ "Always retry on failure." → ✅ Retrying a 400/401 is pointless; only transient
  errors are retryable.
- ❌ "A circuit breaker is just retry logic." → ✅ Retry keeps *trying*; a breaker
  *stops* trying when a dependency is clearly down, failing fast instead.
- ❌ "Idempotency doesn't matter for LLM calls." → ✅ It matters the moment a call has
  side effects (tools that send/charge/write).

### Worked example

The circuit breaker is a three-state machine — it stops hammering a dead dependency by
moving from CLOSED to OPEN, then probes recovery via HALF-OPEN:

```
        failure rate exceeds threshold
        (e.g. 5 consecutive failures)
   ┌──────────────────────────────────────────────┐
   │                                              ▼
┌──────────┐                                  ┌────────┐
│  CLOSED  │                                  │  OPEN  │
│  normal: │                                  │  fail  │
│  calls   │                                  │  fast / │
│  pass    │                                  │ fallback│
│  through │                                  └────┬───┘
└──────────┘                                       │ cooldown
   ▲     ▲                                          │ elapsed
   │     │                                          ▼
   │     │  probe fails              ┌──────────────────┐
   │     └──────────────────────────│    HALF-OPEN      │
   │                                 │ let ONE probe     │
   │   probe succeeds                │ request through   │
   └─────────────────────────────────└──────────────────┘
```

A coding agent calls the LLM, which sometimes triggers a `deploy()` tool. The provider
has a 30 s blip; the client retries the LLM call, the model again returns the same
`deploy` tool_use, and the deploy fires twice — two releases. Fix: (1) the `deploy`
tool requires an idempotency key derived from the plan hash, so the second invocation
is a no-op; (2) a circuit breaker on the provider opens after 5 consecutive failures
and routes to a fallback model in another region; (3) the LLM call uses a 5 s connect
timeout + 25 s idle timeout. The next blip degrades gracefully instead of double-
deploying.

### Check questions

1. A provider has a full 20-minute regional outage. System A retries every failed call
   with exponential backoff (5 attempts). System B has a circuit breaker that opens
   after 5 consecutive failures. Describe what a *user* of each system experiences
   during the outage, and why. — **Answer:** System A: every request still runs the
   full retry sequence — 5 attempts with growing backoff — so each user waits many
   seconds (or tens of seconds) only to get a failure; the system also keeps hammering
   the down provider, slowing its recovery. System B: after the first few failures the
   breaker opens and subsequent calls fail fast (or jump straight to a fallback) without
   attempting the dead provider — users get an immediate degraded response instead of a
   long hang. Retry keeps *trying*; a breaker *stops* trying once the dependency is
   clearly down.
2. Two LLM features both retry on timeout. Feature X summarizes a document and returns
   text. Feature Y is an agent whose tool can issue a refund. After a provider blip, one
   team gets duplicate-charge complaints and the other gets nothing. Which is which, and
   what single mechanism would have prevented the bug? — **Answer:** Feature Y produced
   the duplicate charges. A retry (or fallback) can re-execute the *same logical
   request*; for pure text generation (X) a second run just wastes tokens — no
   real-world effect — but for a call with side effects (Y's refund tool) the effect
   fires twice. The fix is an idempotency key: a stable unique ID for the logical
   operation so the tool/server deduplicates a repeated invocation into a no-op.
   Idempotency matters precisely when a call can cause side effects, not for plain
   generation.

---

## 12.8 — Observability — tracing, logging, token/cost dashboards

### Concept

You cannot debug, cost-control, or improve an LLM system you cannot see. Observability
for AI has the same three pillars as any distributed system — **logs, metrics,
traces** — plus LLM-specific content.

**Tracing.** A single user interaction can fan out into many model and tool calls
(especially in agents). A **trace** is the tree of all spans for one logical request; a
**span** is one unit of work (one LLM call, one tool execution, one retrieval). Each
LLM span should record: model, prompt/messages, completion, token counts (in/out/
cached), latency split (TTFT, total), temperature and other params, stop reason, cost,
and any error. Tracing is what lets you answer "why did this agent run cost $0.40 and
take 40 s?" — you see the 12 spans, spot the one tool that retried 5×, and the
3,000-token system prompt re-sent each loop.

**Logging prompts and completions.** Log the *actual* inputs and outputs, not just
metadata — this is the raw material for evals, regression datasets, and incident
forensics. But this is exactly where security (12.10) bites: prompts/outputs routinely
contain PII and secrets. Redact before logging, or store full content in a restricted-
access store with retention limits, and keep redacted logs in the general system.

**Metrics / dashboards.** Track over time and per route/feature: request volume, error
rate by type (429 vs. 5xx vs. timeout), latency percentiles (p50/p95/p99 of TTFT and
total), token usage, **and cost**. A cost dashboard broken down by feature/model/user
catches the runaway prompt or the looping agent *before* the monthly bill does.
**Quality** signals belong here too — eval scores, thumbs-up/down rates, guardrail trip
rates, fallback-path usage — so a quality regression is visible, not just a latency
one.

**Tooling.** LLM-specific observability platforms — **Langfuse**, **LangSmith**,
**Braintrust**, **Arize Phoenix**, **Helicone** — give trace trees, prompt/completion
capture, token/cost rollups, eval integration, and dataset curation out of the box.
Many speak **OpenTelemetry**, so LLM spans sit in the same trace as your HTTP/DB spans.
The deciding question is usually whether they tie observability to evals (12 → 9).

### Key terms

- **Trace** — the full span tree for one logical request.
- **Span** — one unit of work (LLM call, tool call, retrieval) with timing/metadata.
- **OpenTelemetry (OTel)** — the open standard for traces/metrics/logs; LLM tools
  increasingly emit OTel.
- **Cost dashboard** — token/$ usage broken down by model, feature, and user over time.
- **LLM observability platform** — Langfuse / LangSmith / Braintrust / Phoenix /
  Helicone; trace + log + cost + eval tooling for LLM apps.

### Common misconceptions

- ❌ "Logging the final answer is enough observability." → ✅ For agents you need the
  whole span tree; the cost/latency problem is usually a middle step.
- ❌ "Just log every prompt verbatim." → ✅ Prompts carry PII/secrets; redact or
  access-restrict, or you create a compliance breach.
- ❌ "Observability is only latency and errors." → ✅ For LLMs you must also watch cost
  and quality (eval scores, guardrail trips) or regressions go silent.

### Worked example

A support agent's average cost triples overnight with no deploy. The cost dashboard,
sliced by feature, isolates it to the "summarize ticket history" path. Opening a trace
for one slow request shows 9 LLM spans instead of the usual 2 — the agent is looping
because a tool started returning an error the model keeps retrying. Without tracing the
team sees only "cost up, latency up"; with it they see the exact looping span and the
failing tool, and fix it in an hour.

### Check questions

1. One agent run cost $0.60 and took 45 seconds; the final answer looks completely
   normal. You have only the logged final response. Why can you not diagnose the cost,
   and what would a trace show that the logged response cannot? — **Answer:** The cost
   and latency were spent in *intermediate* steps the final response never reveals — an
   agent fans out into many LLM and tool calls, and a normal-looking final answer says
   nothing about how it was reached. A trace is the span tree of the whole run: it would
   show, e.g., a tool that errored and was retried 6×, a 4,000-token system prompt
   re-sent on every loop, or an agent that looped. You debug cost/latency by inspecting
   the spans, not the final output.
2. A team's dashboard tracks request volume, latency percentiles, error rate, and token
   cost — all flat and healthy. Yet users are quietly getting worse answers after a
   prompt change, and an injection-detection guardrail started tripping far more often.
   Why did the dashboard miss both, and what must it also track? — **Answer:** The
   dashboard tracks only operational and cost signals. A quality regression (worse
   answers) and a safety signal (rising guardrail trips) can both move while volume,
   latency, errors, and cost stay flat — they are orthogonal dimensions. The dashboard
   must also track *quality* signals (eval scores, thumbs-up/down rates) and *safety*
   signals (guardrail trip rates, fallback-path usage) per route/feature, or those
   regressions stay invisible until users complain.

---

## 12.9 — Caching layers — exact-match, semantic, prompt cache

### Concept

There are three distinct caches in an LLM stack. They sit at different layers, hit on
different conditions, and stack — confusing them is a common interview tell.

**1. Exact-match response cache.** A key-value store keyed on a *hash of the full
request* (model + messages + params); the value is the full completion. On an identical
repeat request you return the stored answer with zero model call — latency near 0,
cost 0. Hit condition: byte-identical input. Best for deterministic, repeated requests:
the same FAQ question, the same classification of the same document, idempotent
batch re-runs. Useless for anything with variable phrasing or `temperature > 0` (you
would cache one sample and serve it forever). TTL/invalidation must account for stale
answers and model upgrades.

**2. Semantic cache.** Instead of exact match, embed the incoming query, search a
vector store of past queries, and if the nearest neighbor is within a similarity
threshold, return *its* cached answer. So "What's your refund policy?" and "How do I
get a refund?" hit the same entry. Powerful for natural-language repetition, but
**dangerous**: too loose a threshold returns a confidently wrong answer for a question
that was only *similar*, not equivalent (e.g. "refund policy" vs. "refund policy for
digital goods"). Needs careful threshold tuning, eval coverage, and usually a
restriction to stable, non-personalized answers.

**3. Prompt cache (KV cache).** This is the provider-side cache of Topic 6: it does not
cache the *answer*, it caches the model's internal **KV cache** for a repeated prompt
*prefix*, so prefill is skipped for the cached portion. It hits whenever a new request
shares an exact prefix with a recent one (system prompt, tool defs, few-shot examples,
a long document) — even when the final user query is different and the answer is new.
It cuts TTFT and input cost (~90% off cached input) but the model still runs and still
generates fresh output. TTL is short (~5 min, refreshed on hit).

**How they stack** (cheapest/fastest first): exact-match → semantic → prompt cache →
model call. Exact-match needs identical input and returns instantly; semantic needs
similar input; prompt cache needs a shared prefix and still runs the model. Layering
all three is common: exact-match for hot repeats, semantic for paraphrase clusters,
prompt cache for everything that still reaches the model.

### Key terms

- **Exact-match cache** — keyed on the request hash; returns the stored completion;
  zero model call; needs byte-identical input.
- **Semantic cache** — embedding-similarity lookup; hits paraphrases; risks wrong-but-
  similar answers; threshold-sensitive.
- **Prompt cache / KV cache** — provider-side reuse of prefill state for a shared
  prefix; the model still runs and still generates new output.
- **Cache invalidation / TTL** — expiry and staleness control; mandatory across all
  three.

### Common misconceptions

- ❌ "Prompt caching and response caching are the same thing." → ✅ Prompt cache reuses
  *prefill state* (model still runs); response cache returns the *stored answer* (model
  does not run).
- ❌ "Semantic caching is just a smarter exact-match cache." → ✅ It can return an
  answer to a *different* question that was merely similar — a correctness hazard
  exact-match never has.
- ❌ "Caching is always a win." → ✅ Stale answers, personalized content, model
  upgrades, and `temperature > 0` all make naive caching wrong; each layer needs
  invalidation and eval coverage.

### Worked example

The three caches form a fall-through stack — a request drops to the next layer only on
a miss, and each layer it survives costs more than the last:

```
  request
     │
     ▼
  ┌─────────────────────────┐   hit ──▶ return stored answer
  │ 1. EXACT-MATCH cache    │           cost ≈ 0, latency ≈ 0
  │    (request hash)       │           (no model call)
  └───────────┬─────────────┘
              │ miss
              ▼
  ┌─────────────────────────┐   hit ──▶ return neighbor's answer
  │ 2. SEMANTIC cache       │           cost ≈ 0 (one embedding lookup)
  │    (embedding similarity)│          ⚠ wrong-but-similar risk
  └───────────┬─────────────┘
              │ miss
              ▼
  ┌─────────────────────────┐   hit ──▶ skip PREFILL for shared prefix
  │ 3. PROMPT cache (KV)    │           ~90% off cached input, lower TTFT
  │    (shared prefix)      │           — model STILL decodes a fresh answer
  └───────────┬─────────────┘
              │ (always)
              ▼
  ┌─────────────────────────┐           full prefill + full decode
  │ 4. MODEL CALL           │           — most expensive: full input + output
  └─────────────────────────┘
```

A docs Q&A bot. Telemetry: 15% of queries are *byte-identical* repeats (popular
questions copied from a help page) and another ~25% are paraphrases of a small cluster
of common questions. They add an exact-match cache → 15% of traffic served instantly at
zero cost. They add a semantic cache at cosine 0.92 → another ~20% served, but eval
catches that 0.88–0.92 sometimes confuses "reset password" with "reset 2FA," so they
raise the threshold to 0.95 and restrict semantic hits to non-account-specific topics.
The remaining ~65% still hits the model, where prompt caching of the 3k-token system
prompt + retrieved docs cuts TTFT and input cost. Three layers, three different jobs.

### Check questions

1. A docs bot adds a semantic cache. A user asks "how do I cancel my annual plan?" and
   gets back a confident, fluent answer — that was actually cached for "how do I cancel
   my monthly plan?". Exact-match caches never produce a failure like this. Explain the
   difference in failure modes between the two caches, and what knob caused this. —
   **Answer:** An exact-match cache keys on a hash of the *identical* request, so its
   only possible failure is *staleness* (a once-correct answer that is now out of date)
   — it can never return an answer to a *different* question. A semantic cache hits on
   embedding *similarity*: "annual" and "monthly" cancellation are near-neighbors, so a
   too-loose similarity threshold served the wrong-but-similar entry as a confident
   answer. The dangerous knob is the threshold; semantic caches must be tuned
   conservatively and restricted to stable, non-personalized topics.
2. A request shares its 3k-token system prompt with a recent call but has a brand-new
   user question. Which caches can hit? — **Answer:** Only the prompt cache (shared
   prefix → skipped prefill). Exact-match misses (input differs) and semantic misses
   unless the question paraphrases a cached one; the model still runs and generates a
   fresh answer.

---

## 12.10 — Security — secrets, PII, data retention

### Concept

An LLM feature is an attack surface and a data-handling system. Topic 13 covers
adversarial threats (prompt injection, jailbreaks); 12.10 covers *operational* security
— secrets, sensitive data, and retention.

**Secrets.** API keys are bearer credentials: anyone holding the key spends your
money and reads your data. Rules: never hard-code keys or commit them (`.env` +
`.gitignore`, or a secrets manager — Vault, AWS Secrets Manager, Doppler); never expose
a provider key to a browser or mobile client (the key would ship in the bundle — proxy
all calls through your backend); scope keys narrowly and rotate them; use separate keys
per environment so a leaked dev key cannot touch prod. Watch the inverse leak too: the
*model* must never be handed your secrets in a prompt or tool output, because anything
in context can be coaxed back out (Topic 13).

**PII and sensitive data.** Prompts and completions routinely contain names, emails,
addresses, health and financial data. Principles:
- **Data minimization** — send the model only what the task needs; do not paste an
  entire customer record when one field suffices.
- **Redaction** — strip or tokenize PII before it leaves your boundary where feasible
  (see 13.8); always redact before *logging* (12.8).
- **Know your provider's data policy** — whether inputs are used for training and how
  long they are retained. Enterprise/zero-retention tiers exist; consumer endpoints
  often differ. This must match your contractual and regulatory commitments
  (GDPR, HIPAA, SOC 2). Anthropic's API, for instance, does not train on API inputs by
  default — but you must *verify* this per provider and per tier, not assume it.

**Data retention.** Decide and document how long *you* keep prompts, completions, and
traces, and enforce it with automated deletion. Long retention helps evals and
forensics but increases breach blast radius and can violate "right to be forgotten."
Common pattern: short retention (e.g. 30 days) for full-content logs, longer for
redacted/aggregate metrics. Also track the provider's retention — your effective
retention is the *max* of yours and theirs.

**Other operational controls:** TLS in transit; encryption at rest for logs/caches/
vector stores (a semantic cache or RAG index is a copy of your sensitive data);
access controls and audit logs on prompt/trace stores; tenant isolation in multi-tenant
apps so one customer's data cannot leak into another's context or cache.

### Key terms

- **Secrets management** — keeping API keys out of code/clients via env vars or a
  vault; rotation and per-environment scoping.
- **Data minimization** — sending the model only the data the task requires.
- **Zero-retention / no-train tier** — a provider mode where inputs are not stored or
  used for training; must be verified per provider.
- **Data retention policy** — documented, automated limit on how long prompts/
  completions/traces are kept.
- **Tenant isolation** — ensuring one customer's data cannot surface in another's
  context, cache, or RAG index.

### Common misconceptions

- ❌ "It's fine to call the provider directly from the mobile app." → ✅ That ships your
  API key in the client bundle; always proxy through a backend.
- ❌ "All providers train on whatever you send." → ✅ Policies vary by provider and
  tier; many API tiers are no-train/zero-retention — but you must verify, not assume.
- ❌ "Retention is just the provider's setting." → ✅ Your effective retention is the
  max of yours and theirs; your own logs/caches/vector store are sensitive-data copies
  you must govern.
- ❌ "Redaction only matters for the model call." → ✅ It matters just as much for logs,
  traces, and the semantic cache and RAG index — all are durable copies.

### Worked example

A health-adjacent app pastes the full patient record into the prompt and logs every
prompt verbatim to a 2-year log store for "debugging." Three failures: (1) it sends far
more PII than the task needs — fix with data minimization, send only the relevant
fields; (2) the verbose log store is a 2-year-deep PII breach waiting to happen — fix
with PII redaction before logging plus 30-day retention on full-content logs; (3) the
provider tier was the default consumer tier — fix by moving to the enterprise zero-
retention tier and signing the BAA. Net: less data sent, less data kept, no training on
inputs.

### Check questions

1. A mobile team ships the provider API key inside the app, "encrypted and obfuscated
   in the bundle," and argues that is safe enough. A week after launch, an unknown party
   is burning the company's token quota. Explain why obfuscation did not protect the
   key and what the correct architecture is. — **Answer:** The key is a *bearer*
   credential — whoever holds it can use it. Obfuscation only hides it; anyone can
   inspect the shipped app bundle or intercept its network traffic and extract the key,
   because the app must hold the real key to make calls. Once extracted, the attacker
   spends your quota and reads your data. The correct architecture is a thin backend
   proxy that holds the key server-side (in a vault/env, per-environment, rotated); the
   client authenticates to *your* backend, which makes the provider call. A secret that
   ships to the client is not a secret.
2. Your provider has 30-day retention and you keep redacted logs for 1 year. What is
   your effective exposure window for raw PII? — **Answer:** ~30 days — the provider's
   raw-input window — since your own long-term logs are redacted. Effective retention
   of raw PII is the max of the *raw* copies, which here is the provider's 30 days.

---

## Topic 12 — Exam Question Bank

### True / False

1. TTFT scales with prompt length while TPOT is roughly constant per model. —
   **Answer:** True. TTFT is prefill-bound (more input tokens to process); TPOT is
   memory-bandwidth-bound per decode step and largely independent of prompt size.
2. Streaming reduces the total number of tokens generated and therefore the cost. —
   **Answer:** False. Streaming changes delivery, not work; token count, decode time,
   and cost are unchanged — only *perceived* latency improves.
3. Output tokens are billed at roughly the same rate as input tokens. — **Answer:**
   False. Output costs several × input — exactly 5× on current Claude models, and a
   similar input-cheaper-than-output pattern elsewhere — because decode is the
   expensive, sequential, memory-bandwidth-bound phase. (The *exact* multiplier is
   provider/model-specific, not a universal constant.)
4. Continuous batching schedules at the token-step level, letting finished requests
   leave and new ones join mid-batch. — **Answer:** True. That is precisely what
   distinguishes it from static batching.
5. Exponential backoff without jitter is sufficient to avoid retry storms. —
   **Answer:** False. Without jitter, clients that failed together retry together
   (thundering herd); jitter de-synchronizes them.
6. A circuit breaker and a retry policy do the same thing. — **Answer:** False. Retry
   keeps attempting; a breaker *stops* attempting a clearly-dead dependency and fails
   fast (or routes to a fallback).
7. A prompt cache hit means the model does not run at all. — **Answer:** False. A
   prompt cache skips *prefill* for a shared prefix; the model still decodes and
   produces a fresh answer. Only an exact-match *response* cache skips the model.
8. Putting a provider API key in a mobile app is acceptable if the app is obfuscated.
   — **Answer:** False. Obfuscation is not security; bearer keys must stay server-side
   behind a proxy.
9. Speculative decoding speeds up generation at the cost of a small, bounded quality
   loss. — **Answer:** False. It is *lossless* — the target model verifies every
   drafted token, so the output distribution is identical to running the target alone;
   only latency changes.

### Multiple Choice

1. A summarizer's *total* latency doubled this week. TTFT is unchanged, but TPOT rose
   from 18 ms to 36 ms. Prompt length and output length are unchanged. Most likely
   cause? A) The provider's prefill queue is congested B) Your prompts got longer
   C) The endpoint is now serving a larger/slower model, or is more heavily co-tenanted
   D) Your `max_tokens` cap was raised — **Answer:** C. Unchanged TTFT rules out a
   prefill/queue change; unchanged output length rules out a `max_tokens` change. A
   doubled *per-token* time points at the decode phase — either a bigger model on that
   endpoint or heavier batch co-tenancy stealing GPU time per decode step.
2. A team has four workloads. Which one should they move to the Batch API to cut cost,
   *without* hurting the user experience? A) The live chat endpoint, because it is the
   biggest line item B) Generating a 20,000-example regression eval dataset overnight
   C) The autocomplete feature, because each call is tiny D) A "summarize this page"
   button users click and wait on — **Answer:** B. The Batch API trades latency
   (minutes-to-hours) for ~50% cost; the only safe candidate is the one no user is
   waiting on. A, C, and D are all interactive — moving any of them to Batch would make
   users wait hours. Eval-dataset generation is offline and latency-tolerant.
3. Which cache can return a *wrong* answer to a question that was only similar to a
   past one? A) Exact-match cache B) Prompt cache C) Semantic cache D) None —
   **Answer:** C. Semantic caching hits on embedding similarity; a loose threshold
   serves the cached answer to a non-equivalent question.
4. Which HTTP status should you generally *not* retry? A) 429 B) 503 C) 400 D) 502 —
   **Answer:** C. 400 is a deterministic bad-request error; retrying reproduces it.
   429/502/503 are transient.
5. A request has a 1 s TTFT and produces a 500-token answer at 20 ms/token (~11 s
   total). The team needs to cut total latency in half. Which single change moves the
   needle most? A) Halve the prompt length B) Cache the system prompt C) Cap the answer
   at ~250 tokens / prompt for brevity D) Add streaming — **Answer:** C. Decode
   (~10 s) dominates total latency here; only reducing output length materially cuts
   it. A and B attack TTFT, which is just ~1 s of an 11 s total. D improves *perceived*
   latency but not total. For long outputs, output length is the latency budget.
6. A self-hosted endpoint enables speculative decoding. Two endpoints run the identical
   setup but one sees a 2.5× decode speedup and the other only 1.1×. What best explains
   the difference, and is either endpoint's output quality affected? A) The slow
   endpoint quantized its model — quality dropped B) The fast endpoint serves more
   predictable text, so the draft model's acceptance rate is high; neither endpoint's
   quality changes because speculative decoding is lossless C) The slow endpoint has a
   bigger context window D) Speculative decoding randomly helps some requests —
   **Answer:** B. The speedup tracks the draft model's *acceptance rate* — predictable
   output (boilerplate, text echoing the prompt) is drafted accurately and accelerates
   well, surprising text does not. Quality is unaffected on both: the target model
   verifies every drafted token, so the output distribution is identical to running it
   alone.
7. For a long streaming completion, the right timeout design is: A) One 30 s overall
   timeout B) No timeout C) A short connect timeout plus an inactivity/idle timeout
   D) A 10-minute overall timeout — **Answer:** C. Connect timeout fails fast on a dead
   provider; idle timeout aborts a stalled stream without killing valid long outputs.
8. Which signal would a *cost-only* dashboard miss? A) Token usage B) Request volume
   C) Guardrail trip rate D) Error rate — **Answer:** C. Guardrail trips are a
   safety/quality signal; cost dashboards track tokens and dollars, not quality/safety.

### Short Answer

1. Two teams report "our latency is bad." Team A's users complain the answer "takes
   forever to start"; Team B's users say it starts fast but "crawls as it types."
   Which metric is each team's problem, and what *different* fix does each point to? —
   **Model answer:** Team A has a TTFT problem (prefill-bound, scales with prompt
   length and provider queue wait) — fixes: shorten or prompt-cache the prompt, or
   reduce queueing. Team B has a TPOT problem (decode-bound, memory-bandwidth-limited,
   roughly constant per model) — fixes: a smaller/faster model or a less loaded
   endpoint. The point of the question: a single "total latency" number cannot tell
   the two apart, so it cannot tell you which lever to pull.
2. Why is partial parsing hard when streaming structured output? — **Model answer:**
   Each chunk is a token fragment, so the accumulated buffer is an incomplete JSON
   prefix (unclosed strings/braces) and `JSON.parse` fails. You must buffer to
   completion or use a partial/streaming JSON parser.
3. Explain the difference between a burst and sustained overload, and the right
   response to each. — **Model answer:** A burst is a brief spike over RPM/TPM — handle
   with backoff+jitter retries. Sustained overload is steady demand exceeding the cap —
   backoff cannot fix it; you need a rate-limiting queue (or the Batch API).
4. What is an idempotency key and when is it essential? — **Model answer:** A stable
   unique ID for a logical operation so the server/tool deduplicates retries. Essential
   when an LLM call has side effects (tools that send email, charge cards, write data),
   because retries/fallbacks can re-execute the request.
5. Why report latency percentiles instead of the mean? — **Model answer:** LLM latency
   is heavy-tailed (queueing, retries, long outputs); the mean hides the tail. p50 is
   the typical case, p99 is what the worst-served users get — SLAs live on percentiles.
6. A docs assistant uses a 3k-token system prompt + retrieved docs, then a user
   question. Walk through what each of the three caches does for these two requests:
   (a) the *exact same* question asked twice in a row; (b) a *brand-new* question that
   happens to share the same 3k-token system prefix. — **Model answer:** (a) The
   exact-match response cache hits — byte-identical request → the stored completion is
   returned with no model call at all. (b) Exact-match misses (input differs) and
   semantic likely misses (new question); only the *prompt cache* hits — the shared 3k
   prefix skips prefill, cutting TTFT and input cost, but the model still decodes a
   fresh answer. The discriminator: a response cache hit means the model does not run;
   a prompt cache hit means it still runs but starts faster.
7. Name three operational security risks specific to logging LLM prompts. — **Model
   answer:** Prompts often contain PII and secrets, so verbose logs become a breach
   surface; long retention widens blast radius and can violate right-to-be-forgotten;
   and a logged secret may also have been leaked into context where the model could
   re-emit it.

### Long Answer

1. Walk through every latency lever for a system with 30k-token inputs and 600-token
   outputs, and say which matter most. — **Model answer / rubric:** Identify the split:
   TTFT is large (30k-token prefill — seconds) and decode is ~600 × TPOT (~12 s at
   20 ms). Levers: prompt caching (big TTFT and input-cost win on the stable 30k
   prefix); shrinking the prompt via retrieval/compaction (less prefill); a faster/
   smaller model (lower TPOT); capping `max_tokens` and prompting for brevity (decode
   dominates total latency for 600-token outputs); streaming (perceived latency); on a
   self-hosted stack, speculative decoding for a ~2× lossless TPOT win (especially when
   the output echoes the input). Conclusion: prompt caching for TTFT, output-length
   control for total latency — those two dominate. Bonus: percentiles, fallback for
   tail latency.
2. Design the reliability layer around a third-party LLM API. — **Model answer /
   rubric:** Timeouts — short connect timeout + idle/inactivity timeout for streaming.
   Retries — exponential backoff + jitter on transient errors only (429/5xx), capped;
   never on 400/401/422; respect `Retry-After`. Circuit breaker per provider — closed/
   open/half-open to fail fast during sustained outage. Fallback model/region/provider
   when primary fails — but eval the fallback path so it is not a stealth regression.
   Idempotency keys for any call with side effects. Rate-limiting queue to stay under
   TPM/RPM. Observability — tracing, error-type metrics, alerting on breaker state and
   fallback usage.
3. Compare the three caching layers — mechanism, hit condition, savings, risk — and
   describe how to combine them. — **Model answer / rubric:** Exact-match: hash the
   full request, return stored completion; hits on byte-identical input; saves the
   entire call (cost 0, latency ~0); risk is staleness/model upgrades; needs TTL.
   Semantic: embed query, vector-search past queries, return neighbor's answer if
   within threshold; hits on paraphrases; saves the call; risk is wrong-but-similar
   answers — threshold-sensitive, restrict to stable non-personalized content.
   Prompt cache: provider-side KV-cache reuse for a shared prefix; hits on exact prefix
   match; saves prefill (TTFT + ~90% input cost) but the model still runs; ~5 min TTL.
   Combine in order exact → semantic → prompt → model; each handles a different traffic
   pattern (identical repeats, paraphrase clusters, shared-prefix-but-new-query).
4. Explain continuous batching and why it matters to an API *consumer*, not just a
   self-hoster. — **Model answer / rubric:** Define static batching and its waste
   (batch returns at the slowest completion; finished slots idle). Define continuous
   batching: per-token-step scheduling, finished requests evicted and new ones admitted
   every step, sustained GPU saturation, multi-× throughput gain (PagedAttention, a
   provider-internal KV-cache memory manager, helps fit larger batches). Consumer
   relevance: it explains load-dependent latency (more co-tenants per step → higher TPOT
   at peak), explains why RPM/TPM limits exist, and surfaces the genuine
   latency-vs-throughput trade-off providers tune. For self-hosters it is a 5–20× cost
   lever.
5. [deep-dive] Explain speculative decoding: the mechanism, why it is lossless, and what governs
   how much it helps. — **Model answer / rubric:** Decode is memory-bandwidth-bound, and
   verifying several tokens in one forward pass costs about the same as generating one.
   A small *draft model* proposes the next few tokens; the large *target model* verifies
   them in a single pass, accepting the longest prefix that matches what it would itself
   have chosen and rejecting the rest. It is *lossless* because only target-approved
   tokens survive — the output distribution is identical to running the target alone, so
   no quality re-eval is needed. The speedup tracks the *acceptance rate*: predictable
   text (boilerplate, output echoing the prompt) is drafted well and accelerates ~2–3×;
   novel text barely benefits. Variants: a separate draft model, self-speculation
   (Medusa heads), n-gram/prompt-lookup drafting. Consumer relevance: it is one reason
   TPOT varies with content, and a real lever to enable when self-hosting.

### Applied Scenario

1. Your LLM feature's monthly bill is 4× the forecast. Latency and error rate look
   normal. How do you find and fix the cause? — **Model answer / rubric:** Go to the
   cost dashboard and slice by feature/model/user to localize the spend. Open traces
   for expensive requests — look for agent loops (extra LLM spans), oversized prompts
   re-sent each turn, uncapped `max_tokens`, reasoning-token blowup, or an unintended
   switch to a pricier model. Fixes: cap output, cache stable prefixes, route easy
   requests to a cheaper model, fix the looping tool, trim the prompt. Add a per-feature
   cost alert so the next regression pages instead of arriving on the invoice.
2. A nightly job of 50k LLM calls keeps failing partway with 429s; the on-call engineer
   added immediate retries and it got worse. What is happening and what do you do? —
   **Model answer / rubric:** Demand sustainably exceeds RPM/TPM; immediate retries
   create a thundering herd that amplifies the overload. Replace with a rate-limiting
   queue (token bucket pacing requests under the cap with headroom) plus a concurrency
   cap and exponential backoff + jitter on residual 429s. Better still, since the job
   is async, submit it to the Batch API — no rate-limit choreography and 50% cheaper.
3. A streaming chat feature intermittently shows users a frozen half-answer; HTTP
   returned 200. Diagnose and fix. — **Model answer / rubric:** A 200 only means the
   stream opened; an error or truncation occurs *inside* the stream. Inspect the final
   event's stop reason and error events: `max_tokens` → output truncated, raise the cap
   or prompt shorter; an in-stream error event → provider/model failure mid-generation,
   surface a clean error and retry/fallback; the stream may also have stalled — add an
   idle timeout to abort and retry. Ensure the client treats abnormal stream
   termination as a failure, not a completed response.
4. Your team wants to call the LLM provider directly from the React frontend "to cut a
   network hop." What do you say, and what is the correct architecture? — **Model
   answer / rubric:** Reject it. The provider key is a bearer credential; shipping it in
   the browser bundle exposes it to every user — quota theft and data access. Correct
   architecture: a thin backend proxy holds the key (in a vault/env, per-environment,
   rotated), terminates auth, enforces rate limits and per-user quotas, redacts PII
   before logging, applies guardrails, and streams the response back to the client.
   The "extra hop" is also where caching, observability, and reliability logic live —
   it is essential infrastructure, not overhead.

---

## Sources

[1] Anthropic — Pricing (Claude API docs) — https://platform.claude.com/docs/en/about-claude/pricing
[2] vLLM — PagedAttention and continuous batching (vLLM documentation) — https://docs.vllm.ai/en/latest/
[3] AWS Architecture Blog — Exponential Backoff And Jitter — https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/
[4] Leviathan, Kalman & Matias — Fast Inference from Transformers via Speculative Decoding (arXiv:2211.17192) — https://arxiv.org/abs/2211.17192
