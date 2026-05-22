# Topic 06 — Prompt Caching — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This file is taught sub-chapter by sub-chapter, in order. Each sub-chapter has core
teaching content, key terms, common misconceptions, a worked example, and check
questions the tutor uses to confirm understanding before moving on. The exam question
bank at the end is the pool the tutor draws from for the gated topic exam. Work through
the sub-chapters sequentially. This topic assumes Topic 1 (prefill vs. decode, the KV
cache) and Topic 5 (prompt structure) — refer back to them if anything here is shaky.

---

## 6.1 — What it is — reusing the KV cache for a repeated prompt prefix

### Concept

To understand prompt caching you must first recall the **KV cache** from Topic 1.
Generating with a transformer has two phases. **Prefill** processes the entire input
prompt in parallel: for every token, at every layer, the model computes key (K) and
value (V) vectors used by attention. These K/V vectors are stored in the **KV cache** so
that during **decode** — generating output tokens one at a time — each new token can
attend to all prior tokens without recomputing their K/V from scratch. Prefill is
compute-bound and its cost scales with the number of input tokens.

Here is the key observation that motivates prompt caching. The K/V vectors for a given
token depend *only on that token and the tokens before it* — never on what comes after.
So if two API requests begin with the *exact same sequence of tokens* (the same
prefix), the K/V vectors for that prefix are *identical* in both requests. Computing
them the second time is pure waste.

**Prompt caching** exploits this: the provider stores the KV cache produced for a prompt
prefix and, on a later request with the same prefix, *reuses* the stored K/V vectors
instead of recomputing them in prefill. The model still runs decode normally, and still
prefills any *new* tokens after the cached prefix — but the expensive prefill of the
shared prefix is skipped.

This is a real, common pattern. The same long system prompt, the same tool definitions,
the same few-shot examples, the same large reference document, the same conversation
history-so-far — these are sent on call after call. Without caching you pay full prefill
for them every single time. With caching you pay to compute them once, then pay a
fraction to reuse them.

Two things to be precise about. First, caching is a **cost and latency optimization that
does not change what the model is asked to do** — the model sees the same tokens either
way. A cache hit reuses K/V vectors that are *equivalent up to normal floating-point
nondeterminism*: floating-point matrix multiplication is not associative, so a recomputed
prefix and a cached one can differ in the last bits, exactly as two uncached runs of the
same prompt already can. Providers do not promise bitwise reproducibility either way.
What caching does *not* introduce is a *new* or *systematic* quality difference — so
A/B-testing answer quality with caching on vs. off is not a meaningful experiment (more
on this below). Second, it is **prefix caching, not response caching.** It does not store "question → answer" pairs and skip the model. The
model still runs on every request; only the redundant prefill work is saved. (A separate
mechanism — an exact-match or semantic *response* cache — does skip the model, and is
covered in Topic 12; do not confuse the two.)

Providers expose this differently. Anthropic's Claude API requires you to opt in by
marking cache breakpoints with `cache_control` — but note Anthropic now offers *two*
modes: an **automatic** mode where you add a single top-level `cache_control` field and
the system manages breakpoints for you, and an **explicit breakpoints** mode where you
place `cache_control` on individual content blocks for fine-grained control.[1] OpenAI
and Google use **automatic / implicit** caching: the provider detects repeated prefixes
with no code change at all.[2] Either way the underlying mechanism is the same — reuse
of the prefix KV cache.

### Key terms

- **KV cache** — stored key/value attention vectors for tokens already processed, so
  decode doesn't recompute them.
- **Prefill** — the parallel phase that processes the input prompt and produces its KV
  cache; compute-bound.
- **Prompt (prefix) caching** — storing the KV cache for a prompt prefix and reusing it
  on later requests that share that prefix.
- **Cache hit / cache miss** — a request whose prefix is found in the cache (prefill
  reused) vs. not found (full prefill).
- **Explicit vs. implicit caching** — caching you opt into via `cache_control`
  (Anthropic — either a single top-level field with system-managed breakpoints, or
  explicit per-block breakpoints) vs. fully automatic caching the provider applies with
  no code change at all (OpenAI, Google).[1][2]

### Common misconceptions

- ❌ Prompt caching stores answers, so a repeated question skips the model. → ✅ It
  caches the prefix's KV vectors only; the model still runs every request. Answer-level
  caching is a different mechanism (response cache).
- ❌ A cache hit can systematically change or degrade the model's output. → ✅ A cached
  prefix is equivalent to a recomputed one up to normal floating-point nondeterminism —
  the same last-bit variation two uncached runs already show. Caching introduces no new
  or systematic quality difference; there is nothing to A/B-test.
- ❌ Caching speeds up output generation (decode). → ✅ It speeds up prefill of the
  shared prefix; decode of new output tokens is unaffected.

### Worked example

A customer-support bot sends a 4,000-token system prompt (policies, tone, tool
descriptions) on every request, followed by a short user question.

- **Without caching:** every request prefills all 4,000 system tokens — recomputing
  identical K/V vectors thousands of times a day.
- **With caching:** the first request computes and stores the KV cache for the 4,000
  system tokens. Every subsequent request within the TTL finds that prefix in the cache,
  reuses the K/V vectors, and only prefills the short, new user question. The 4,000-token
  prefill — the bulk of the input cost and a chunk of the latency — is paid once, not
  thousands of times.

### Check questions

1. Two requests share the first 2,000 tokens, then differ. Request A's shared prefix is
   followed by 50 unique tokens; request B's by 6,000 unique tokens. The cached prefix
   K/V is reused for *both*. Explain why "what comes after the prefix" is irrelevant to
   whether the prefix's cache is valid — and tie it to the directional property of how
   K/V vectors are computed. — **Answer:** A token's K/V vectors depend only on that
   token and the tokens *before* it — never on later tokens (attention at position N
   looks back, not forward). So the first 2,000 tokens compute identical K/V regardless
   of whether 50 or 6,000 tokens follow. The cache stores exactly those
   position-0-through-N-dependent vectors, which is why differing *suffixes* never
   invalidate a shared *prefix*.
2. A skeptic worries that turning on caching might subtly change model outputs and wants
   to A/B test answer quality with caching on vs. off. Explain why that A/B test is
   pointless, grounding the answer in what is actually stored and reused. — **Answer:**
   The test is pointless because a cache hit reuses K/V vectors the model would otherwise
   have recomputed — caching changes *where* those vectors come from (loaded vs.
   recomputed), not what they represent. They are equivalent up to normal floating-point
   nondeterminism: float matmul isn't associative, so a recomputed prefix can already
   differ in the last bits run-to-run regardless of caching. Caching adds no *new* or
   *systematic* quality difference — only cost and latency change — so there is no
   quality dimension that an A/B test could isolate.
3. A teammate proposes "we'll cache our FAQ bot so when two users ask the same question
   we skip the model and return the stored answer instantly." Identify which mechanism
   they're actually describing, how it differs from prompt caching, and whether prompt
   caching would still help their FAQ bot. — **Answer:** They're describing a *response
   cache* (exact-match or semantic answer caching) — a different mechanism, covered in
   Topic 12, that does skip the model. Prompt caching never skips the model: it reuses
   the prefix's KV cache but still runs decode every request. Prompt caching would still
   help the FAQ bot — its stable system prompt / instructions / FAQ context prefix is
   reused on every call — but it cuts prefill cost and latency, it does not eliminate the
   model call.

---

## 6.2 — The exact-prefix-match requirement

### Concept

Prompt caching works on a **prefix** — a contiguous run of tokens from the *very start*
of the prompt up to some point. The match must be **exact**: byte-for-byte (really,
token-for-token) identical from token zero. This is a direct consequence of the
mechanism. Token *N*'s K/V vectors depend on tokens 0…N. If token 5 differs between two
requests, then token 5's K/V vectors differ — and so do token 6's, token 7's, and every
token after, because each one attends to the changed token. The cache entry stored for
the old prefix is simply *wrong* for the new one past the divergence point.

The two consequences engineers must internalize:

1. **It is a prefix, not a substring.** The cache cannot match content in the *middle*
   of your prompt. If you insert a fresh timestamp at the top, every cached token after
   it is invalidated. The cacheable region is always anchored at the start.
2. **One differing token early busts everything after it.** A single changed
   character — a different date, a re-ordered word, an extra space, a trailing newline,
   a non-deterministically serialized JSON key order — at position *k* means tokens *k*
   onward are a cache miss. The prefix *up to* the divergence may still hit; everything
   after is recomputed.

This is why caching is unforgiving about determinism. Anything that varies between
otherwise-identical calls and sits early in the prompt destroys the cache: timestamps,
random IDs, request UUIDs, "today's date" if it changes, user names interpolated into
the system prompt, or unstable serialization (Python dict ordering, JSON key ordering,
floating-point formatting). The fix is to make the prefix **byte-stable** — identical
tokens, in identical order, every call.

A second subtlety, from Topic 2: caching matches on **tokens**, not characters. The
boundary of the cacheable region must fall on a clean token boundary. In practice the
provider handles this, but it explains why providers cache in fixed increments (OpenAI
extends the matched prefix in 128-token steps) and why very short content may not be
cacheable at all (6.4 covers minimum lengths).

How providers express the match point:

- **Anthropic** — you place `cache_control` breakpoints. Everything from the start of
  the prompt up to and including a breakpoint is a cache segment. You can define up to
  **4 cache breakpoints**; the system's **lookback window is 20 blocks** — per
  breakpoint it checks at most 20 positions for a usable earlier cache entry.[1]
- **OpenAI / Google** — automatic. The provider hashes the prefix and finds the longest
  previously-computed matching prefix itself (OpenAI matches in **128-token increments**
  above the 1,024-token minimum).[2]

Either way the rule is identical: **the cache is reused only up to the first point of
divergence.**

### Key terms

- **Prefix** — a contiguous token run anchored at the start of the prompt.
- **Exact match** — token-for-token identical; any difference ends the match there.
- **Cache bust / invalidation** — losing a cache hit because a token in the prefix
  changed.
- **Byte-stable prefix** — a prefix whose tokens are identical on every call, so it
  reliably hits.
- **Cache breakpoint** — (Anthropic) an explicit `cache_control` marker defining the end
  of a cacheable segment.

### Common misconceptions

- ❌ Caching can match a repeated chunk anywhere in the prompt. → ✅ It only matches a
  prefix anchored at the start; mid-prompt content cannot be cached on its own.
- ❌ A small change like an extra space or different JSON key order is harmless. → ✅
  Any token-level difference busts the cache from that point on; the prefix must be
  byte-stable.
- ❌ A late change in the prompt invalidates the whole cache. → ✅ The prefix *before*
  the change still hits; only tokens from the divergence point onward are recomputed.

### Worked example

Two system prompts that look "the same":

```
Request A: "You are a support agent. Today is 2026-05-20. Follow the policy below..."
Request B: "You are a support agent. Today is 2026-05-21. Follow the policy below..."
```

The prefix matches up to `Today is 2026-05-2`, then diverges at `0` vs `1`. Every token
from there on — the entire multi-thousand-token policy — is a cache miss on request B.

Fix: move the volatile token out of the prefix. Put the date *after* the cached static
content, near the user turn:

```
[ cached static prefix: role + full policy, no date ]
[ uncached tail: "Today is 2026-05-21." + user question ]
```

Now the date changes daily without ever busting the cache on the policy.

### Check questions

1. A prompt's first 4,000 tokens are byte-identical to the previous call; token 4,001
   differs; tokens 4,002–10,000 are again byte-identical to the previous call. How much
   of this prompt can hit the cache, and why can't the identical tail (4,002–10,000) be
   reused even though it matches? — **Answer:** Only tokens 1–4,000 can hit; everything
   from 4,001 onward is recomputed. The tail can't be reused even though its *text*
   matches because each of its tokens' K/V vectors attend back through token 4,001 —
   which changed — so their K/V values are now different. Caching matches a contiguous
   prefix from the start; a matching span in the middle is not independently cacheable.
2. An engineer assembles a prompt by concatenating retrieved document chunks, and the
   retrieval layer returns them in a different order on each call (it iterates a hash
   set). The chunk *text* is identical across calls — same documents every time. Why
   does caching still fail, and what's the one-line fix? — **Answer:** Caching requires
   a *byte-stable* prefix — identical tokens in identical *order*. If the same chunks
   arrive in a different order, the token sequence differs from call to call, so the
   prefix diverges early and the cache busts. Fix: sort the chunks deterministically
   (e.g. by document ID) before concatenating, so the assembled prefix is identical
   every call.
3. Two designs put a per-request value in the system prompt. Design A interpolates a
   request UUID into line 2 (near the top). Design B appends "Request ID: <uuid>" as the
   final line, right before the user turn. Both include the UUID — so why does B keep the
   cache working while A destroys it? — **Answer:** Caching reuses the longest *exact
   prefix* from the start. In A the UUID sits early, so every token after line 2 — the
   whole policy — diverges and misses on every call; almost nothing caches. In B the
   stable policy comes first and is byte-identical every call, so it caches; only the
   short volatile tail (UUID + question) is uncached. Same data, but position relative
   to the stable content is everything.

---

## 6.3 — Prompt-structure implications — static content first, variable content last

### Concept

Because caching matches a *prefix* and requires *exact* matching (6.2), it imposes a
single, dominant design rule on how you lay out a prompt:

> **Order content from most stable to least stable. Put everything static at the front;
> put everything variable at the end.**

This is the inverse of how engineers naturally write prompts (often "here's the user's
question... and here's a document to help"). To cache well, you must restructure.

A cache-optimal prompt, front to back:

1. **Tool definitions** — usually identical across all calls in an app. Most stable.
2. **System prompt** — role, rules, policies. Stable.
3. **Large static context** — a knowledge base, a long document, a code file, a style
   guide that doesn't change per request. Stable.
4. **Few-shot examples** — fixed example set. Stable.
5. **Conversation history** — grows over a session but the *earlier* turns are stable;
   each completed turn becomes a stable prefix for the next request.
6. **The current user turn / query** — different every call. Least stable. Goes last.

The payoff: the long, expensive, static front (often the overwhelming majority of input
tokens) is cached, and only the short, variable tail is prefilled fresh.

Practical rules that follow from this:

- **Never interleave volatile data into the static region.** A timestamp, user ID, or
  per-request value placed early forfeits the cache on everything after it. Hoist all
  volatile data to the tail.
- **Watch ordering stability.** If you assemble the prompt by iterating a dict or
  concatenating retrieved chunks in nondeterministic order, the "static" region isn't
  actually byte-stable. Sort deterministically.
- **Conversation history caches incrementally.** After turn 1 completes, turns
  [system + user₁ + assistant₁] are a stable prefix; turn 2's request reuses that cache
  and only prefills user₂. Each turn extends the cached prefix. This makes multi-turn
  chat naturally cache-friendly *if* you keep the prefix stable and don't rewrite
  history.
- **Don't rewrite or summarize earlier context mid-session if you can avoid it** — any
  edit to earlier turns busts the cache from the edit point. (When you *must* compact,
  accept the one-time re-cache cost.)
- **Anthropic breakpoints:** place `cache_control` at the end of each stable segment —
  e.g. one after tools, one after the system prompt, one after the conversation prefix.
  Multiple breakpoints let different-length stable prefixes all be reused.

This sub-chapter is the single most actionable idea in the topic: **cache-aware prompt
design is just "stable stuff first, variable stuff last," applied rigorously.**

### Key terms

- **Static / stable content** — prompt content identical across requests (tools, system
  prompt, fixed docs, examples).
- **Variable content** — prompt content that changes per request (the user query,
  timestamps, per-request data).
- **Cache-aware prompt design** — ordering a prompt stable-first / variable-last to
  maximize the cacheable prefix.
- **Incremental caching** — each completed conversation turn extending the stable
  cached prefix for the next request.

### Common misconceptions

- ❌ The order of content in a prompt is just stylistic. → ✅ With caching, order
  determines how much of the prompt is cacheable; stable-first is a hard requirement.
- ❌ It's fine to put a timestamp at the top "for context." → ✅ A timestamp at the top
  busts the entire cache after it; volatile values go at the end.
- ❌ Multi-turn chat can't be cached because the messages array keeps changing. → ✅ The
  earlier turns are a stable prefix; each turn extends the cache incrementally, as long
  as history isn't rewritten.

### Worked example

Cache-hostile layout (variable content first):

```
User question: "How do I export a report?"   <-- changes every call
[ 6,000-token product manual ]                <-- static, but now uncacheable
[ system prompt ]
```

Nothing before the user question can be cached, because the user question is at
position zero and changes every call. The whole 6,000-token manual is re-prefilled every
request.

Cache-optimal layout (static content first):

```
[ tool definitions ]            \
[ system prompt ]                |  static prefix — cached
[ 6,000-token product manual ]  /   <-- cache_control breakpoint here
User question: "How do I export a report?"   <-- variable tail, prefilled fresh
```

Now ~6,000+ tokens hit the cache on every call after the first; only the ~10-token
question is prefilled new.

### Check questions

1. An app's prompt contains, in this order: the user's question, a fixed 5,000-token
   style guide, the system prompt, and tool definitions. The team is told to "reorder
   for caching." Give the corrected order and justify the placement of the *first* and
   *last* element specifically. — **Answer:** Corrected order: tool definitions → system
   prompt → 5,000-token style guide → user question. First element (tool definitions):
   they're identical across every call in the app — the most stable content — so they
   anchor the longest reusable prefix when placed first. Last element (user question):
   it changes every call — the least stable content — so it must go last; if it sat
   anywhere earlier, everything after it (the style guide, etc.) would be uncacheable.
   The rule is most-stable-first, most-variable-last.
2. In a multi-turn chat, turn 5's request includes turns 1–4. Explain why turn 5 mostly
   hits the cache "for free" — *and* what single operation, if the app does it before
   turn 5, would destroy that cache hit. — **Answer:** After each turn completes, the
   messages up to that point form a fixed prefix; turn 5's request reuses the cached
   prefix built up through turn 4 and only prefills the new user turn — incremental
   caching. The operation that destroys it: rewriting or summarizing the earlier turns
   (compaction) — any edit to turns 1–4 changes the prefix from the edit point onward, so
   turn 5 can no longer reuse it and pays a fresh write.
3. A developer puts a `<current_time>` block at the top of the prompt "so the model
   always knows the time," and separately a teammate puts the *same* block as the last
   line before the user question. Both prompts give the model the time. Explain the
   caching consequence of each choice and why they diverge so sharply. — **Answer:** The
   top placement busts the cache for everything after it on every call — the time changes
   constantly, so the whole prompt below it is re-prefilled (and re-written) each
   request; caching effectively does nothing. The bottom placement leaves the entire
   static region above it byte-stable and cacheable; only the short volatile tail (time
   + question) is uncached. Identical information, opposite caching outcome — because
   caching keys on an exact prefix from the start, volatile content must live in the
   tail.

---

## 6.4 — Cache TTL

### Concept

A cached prefix does not live forever. Each cache entry has a **time-to-live (TTL)** —
an idle window after which it is evicted. If the next request with the matching prefix
arrives within the TTL, it's a cache hit. If the prefix sits idle longer than the TTL,
the entry is evicted and the next request is a cache miss that pays full prefill (and,
where applicable, a fresh cache write).

**Anthropic (Claude API).** Two TTL options, selected in `cache_control`:[1]

- **5-minute TTL** — the default (`{"type": "ephemeral"}`). Minimum lifetime ~5 minutes.
- **1-hour TTL** — opt-in (`{"type": "ephemeral", "ttl": "1h"}`), at a higher write
  price (see 6.5).

Crucially, **the TTL is a sliding window refreshed on every hit.** Each time the cached
prefix is used, its TTL clock resets — Anthropic's docs state the cache "is refreshed
for no additional cost each time the cached content is used."[1] So a prefix that is hit
at least once every 5 minutes stays warm indefinitely on the 5-minute tier — you do not
re-pay the write. The TTL only matters during *idle* gaps. Anthropic describes entries
as having a "minimum lifetime" and being "promptly, though not immediately, deleted"
after it elapses — an entry lives *at least* that long after its last use.[1]

There are also **minimum cacheable lengths** — content shorter than the threshold simply
isn't cached (no error; both `cache_creation_input_tokens` and
`cache_read_input_tokens` come back 0). The key facts to internalize are that the
threshold *exists*, that it is *model-specific*, and that it does *not* simply track the
model tier. As one illustrative pair on the Claude API: Claude Sonnet 4.6 has a
**1,024-token** minimum, while Claude Opus 4.7 has a **4,096-token** minimum — so the
same ~3,000-token prefix is cacheable on Sonnet but silently uncached on Opus.[1] Other
models sit at other values; always look up the current threshold for the specific model
you target rather than memorizing a table.

**OpenAI.** Automatic caching with no TTL knob. Cached prefixes generally remain active
for **5–10 minutes of inactivity**, and always within **about one hour** of last use.[2]
Minimum prompt length to be eligible is **1,024 tokens**, matched in 128-token
increments.[2]

**Google (Gemini).** Implicit caching (automatic, default-on for 2.5+ models) plus
explicit caching where you create a cache object with a developer-set TTL and pay a
storage cost for the duration.

The engineering takeaways: (1) caching helps only when reuse is *frequent enough* to
land within the TTL — bursty or steady traffic benefits, sparse traffic doesn't; (2) on
Anthropic, choose the 1-hour TTL when reuse gaps exceed 5 minutes but a re-write every
gap would be wasteful, and weigh its higher write cost; (3) you can deliberately keep a
cache warm with a cheap periodic "keep-alive" request if the economics justify it.

### Key terms

- **TTL (time-to-live)** — the idle window a cache entry survives before eviction.
- **Sliding TTL / refresh-on-hit** — each cache hit resets the TTL clock, so frequently
  used prefixes stay warm without re-writing.
- **5-minute vs. 1-hour cache (Anthropic)** — the default short TTL vs. the opt-in
  longer TTL at a higher write price.
- **Minimum cacheable length** — the token threshold below which content is not cached
  at all.
- **Cache eviction** — removal of a cache entry after its TTL elapses (or under capacity
  pressure).

### Common misconceptions

- ❌ The TTL counts from when the cache was written, so it always expires 5 minutes
  later. → ✅ It's a sliding window refreshed on every hit; a prefix used regularly
  stays cached indefinitely without re-paying the write.
- ❌ Any prompt can be cached. → ✅ Content below the model's minimum cacheable length
  (e.g. 4,096 tokens for Opus 4.x) is silently not cached.
- ❌ Refreshing the TTL on a hit costs another cache-write fee. → ✅ Refresh on hit is
  free; you only pay the read price on a hit.

### Worked example

A bot has a 5,000-token cached system prefix on Anthropic's 5-minute tier.

- Requests arrive every ~30 seconds all afternoon → every request is within the TTL,
  each hit slides the window, the prefix stays warm for hours, the write is paid once.
- Then traffic stops for 12 minutes. The entry is evicted after ~5 idle minutes. The
  next request is a cache miss: full prefill of the 5,000 tokens plus a fresh cache
  write.
- If idle gaps of ~10–15 minutes are common, switching that prefix to the 1-hour TTL
  keeps it warm across the gaps — at a 2× write price, paid only when a write actually
  happens. Whether that's worth it is the pricing math of 6.5/6.6.

### Check questions

1. A prefix on Anthropic's 5-minute TTL is hit at 0:00, 0:04, and 0:09 (minutes). A
   colleague claims it must have been evicted and re-written at least once because "5
   minutes passed." Is the colleague right? Explain using how the TTL clock behaves. —
   **Answer:** No. The 5-minute TTL is a *sliding window refreshed on every hit*, not a
   fixed countdown from creation. The 0:04 hit was within 5 minutes of 0:00 and reset the
   clock; the 0:09 hit was within 5 minutes of 0:04 and reset it again. No idle gap ever
   exceeded 5 minutes, so the entry stayed warm throughout — one write, two free reads,
   no re-write.
2. The same 3,000-token system prompt is used by an app on Claude Opus 4.7 and by another
   app on Claude Sonnet 4.6. One app sees caching savings and the other sees none, with
   no error in either. Explain. — **Answer:** Minimum cacheable length is model-specific.
   On Sonnet 4.6 the threshold is 1,024 tokens, so a 3,000-token prefix is cacheable —
   savings appear. On Opus 4.7 the threshold is 4,096 tokens, so the same 3,000-token
   prefix is below the minimum and silently not cached — no savings, no error, and
   `cache_creation`/`cache_read` both report 0. Same prompt, different model, different
   outcome.
3. An agent's requests arrive in a regular rhythm: a burst of activity, then a ~12-minute
   idle gap, repeatedly. The team is choosing between the 5-minute and 1-hour TTL. Reason
   through which to pick and what the deciding cost factor is. — **Answer:** On the
   5-minute TTL the 12-minute idle gaps exceed the window, so the cache evicts every gap
   and each post-gap request pays a fresh write — the write penalty is incurred
   repeatedly. The 1-hour TTL survives the 12-minute gaps, so the prefix stays warm and
   most calls become cheap reads. The deciding factor is the 1-hour tier's higher write
   price (2× vs 1.25× base): it's worth it here because it converts repeated full writes
   into a single write plus many reads — but you should still do the write-penalty vs.
   read-savings math at the observed reuse rate to confirm.

---

## 6.5 — Pricing math — cache writes vs. cache reads vs. base input

### Concept

Prompt caching splits input tokens into three priced categories. You must be able to do
this math cold.

1. **Base input tokens** — normal, uncached input. Price = 1× the model's input rate.
2. **Cache write (cache-creation) tokens** — tokens written into the cache on a miss.
   They cost *more* than base input, because the provider both processes and stores
   them. On Anthropic: **1.25×** base input for the 5-minute TTL, **2×** base input for
   the 1-hour TTL.[3]
3. **Cache read (cache-hit) tokens** — tokens served from the cache on a hit. They are
   *much cheaper*: on Anthropic, **0.1×** base input — a **90% discount**.[1] (OpenAI's
   cache-read discount is provider- and model-specific and has changed over time: the
   original GPT-4o-era automatic caching gave a **50% discount** (~0.5× base), but
   OpenAI's current flagship models — e.g. GPT-5.5 and GPT-5.4 — price cached input at
   **0.1× base, a 90% discount**;[4] Google's explicit cache reads are ~0.1× / 90% off
   for 2.5+ models, plus a storage fee. Always verify against the provider's current
   pricing page.)

Output tokens are unaffected by caching — they're always billed at the output rate.

Concrete numbers, Claude Opus 4.7 (base input $5 / million tokens, output $25 / MTok):[3]

| Token category        | Multiplier | Price / Mtok |
|-----------------------|------------|--------------|
| Base input            | 1×         | $5.00        |
| 5-min cache write     | 1.25×      | $6.25        |
| 1-hour cache write    | 2×         | $10.00       |
| Cache read            | 0.1×       | $0.50        |
| Output                | —          | $25.00       |

**The break-even logic.** A cached prefix costs you 1.25× on the *write* (first call)
and 0.1× on every *read* (subsequent calls). Compare against the no-cache cost of 1×
every call.

- First call (write): you pay 1.25× instead of 1× — a 0.25× *penalty*.
- Each later call (read): you pay 0.1× instead of 1× — a 0.9× *saving*.

So after the **first cache read**, the 0.9× saving already exceeds the 0.25× write
penalty: **on the 5-minute TTL, caching pays off after just one reuse.** For the 1-hour
TTL the write is 2× (a 1× penalty), so each read saves 0.9× — you need **two reads** to
break even.

**Worked cost comparison.** A 10,000-token static prefix, Opus 4.7, 5-minute TTL, prefix
reused across 100 requests:

- *No caching:* 100 × 10,000 tokens × $5/Mtok = 100 × $0.05 = **$5.00** on the prefix.
- *With caching:* 1 write (10,000 × $6.25/Mtok = $0.0625) + 99 reads (99 × 10,000 ×
  $0.50/Mtok = 99 × $0.005 = $0.495) = $0.0625 + $0.495 = **$0.5575**.
- Savings ≈ **89%** on the prefix's input cost. The more reads, the closer the average
  cost per call approaches the 0.1× read price.

Reading the usage object: on Anthropic, the response reports
`cache_creation_input_tokens` (tokens written to the cache — you paid the write
multiplier), `cache_read_input_tokens` (tokens retrieved from the cache — you paid the
read multiplier), and `input_tokens` — which on Anthropic counts *only* the tokens after
the last cache breakpoint, i.e. the uncached tail at 1×. So
`total_input = cache_creation_input_tokens + cache_read_input_tokens + input_tokens`.[1]
A healthy cached workload shows `cache_read` dominating and `cache_creation` near zero
after warm-up.

### Key terms

- **Base input price** — the 1× per-token input rate; the reference for all multipliers.
- **Cache write / cache-creation tokens** — tokens stored into the cache, priced above
  base (1.25× at 5-min, 2× at 1-hour on Anthropic).
- **Cache read / cache-hit tokens** — tokens served from cache, priced far below base
  (0.1× on Anthropic).
- **Break-even** — the number of reuses at which cumulative caching cost falls below the
  no-cache cost (1 read for 5-min TTL, 2 for 1-hour TTL on Anthropic).

### Common misconceptions

- ❌ Cached input is free. → ✅ Cache reads cost ~0.1× base (Anthropic) — cheap, not
  free — and the write costs *more* than base.
- ❌ Caching is always cheaper. → ✅ If the prefix is written but rarely or never re-read
  before eviction, you pay the write penalty for nothing and lose money.
- ❌ Caching reduces output token cost too. → ✅ Caching only affects input-token
  pricing; output is billed at the normal rate regardless.

### Worked example

Should you cache a 20,000-token document prefix on Sonnet 4.6 (base input $3/Mtok) if
it's reused 5 times within 5 minutes?

- 5-min write: 1.25 × $3 = $3.75/Mtok; read: 0.1 × $3 = $0.30/Mtok.
- No cache: 5 × 20,000 × $3/Mtok = 5 × $0.06 = **$0.30**.
- With cache: 1 write ($0.075) + 4 reads (4 × 20,000 × $0.30/Mtok = 4 × $0.006 =
  $0.024) = **$0.099**.
- Savings ≈ **67%** even at just 5 reuses. Caching wins clearly.

### Check questions

1. A 12,000-token prefix on Sonnet 4.6 (base input $3/MTok) is written once on the
   5-minute TTL and then read exactly twice. Compute the total prefix input cost with
   caching, compare it to the no-cache cost over the same 3 calls, and state whether
   caching won. — **Answer:** Per call the prefix is 12,000 tokens = 0.012 MTok.
   No-cache: 3 calls × 0.012 MTok × $3 = $0.108. With caching: 1 write at 1.25× ($3.75
   /MTok) = 0.012 × $3.75 = $0.045; 2 reads at 0.1× ($0.30/MTok) = 2 × 0.012 × $0.30 =
   $0.0072; total = $0.0522. Caching won — $0.0522 vs $0.108, roughly a 52% saving even
   at just two reads.
2. The break-even point differs between the two Anthropic TTL tiers — one read for the
   5-minute tier, two reads for the 1-hour tier. Derive *why* from the multipliers,
   rather than restating the rule. — **Answer:** Break-even is where cumulative read
   savings cover the write penalty. Write penalty = (write multiplier − 1). Read saving
   per read = (1 − read multiplier) = 1 − 0.1 = 0.9×. 5-minute write is 1.25×, penalty
   0.25×; one read saves 0.9× > 0.25×, so break-even at the first read. 1-hour write is
   2×, penalty 1.0×; one read saves only 0.9× (< 1.0×), so it takes a second read
   (1.8× total saved > 1.0×). The tier difference is entirely driven by the larger write
   penalty of the 1-hour tier.
3. A workload's usage objects show `cache_creation_input_tokens` high on nearly every
   request and `cache_read_input_tokens` near zero, well past warm-up. Give the two most
   likely root causes, and explain how this pattern means caching is *costing* money
   rather than saving it. — **Answer:** Likely causes: (a) the prefix isn't byte-stable —
   some volatile token early in it busts the cache every call; (b) a TTL/traffic mismatch
   — reuse gaps exceed the TTL, so the entry evicts before the next request. Either way
   every call writes (at 1.25–2× base) and almost none reads (at 0.1×). Since writes cost
   *more* than plain base input and the cheap reads never materialize, the workload pays
   a write penalty on every call with no offsetting saving — caching is a net loss until
   the prefix or the TTL is fixed.

---

## 6.6 — When caching pays off and when it doesn't

### Concept

Prompt caching is not free upside; it's a trade with a clear profile. It pays off when
the gains (cheap, fast reads) outweigh the costs (write penalty, complexity), and it
loses when they don't.

**Caching pays off when:**

- **The prefix is large.** The savings scale with the number of cached tokens. A
  10k-token cached system prompt + tools + docs saves real money; a 200-token one saves
  nothing (and may be below the minimum cacheable length anyway).
- **The prefix is reused frequently.** High request volume against the *same* static
  content — a support bot, a coding assistant, a doc-Q&A service — means many reads per
  write. Multi-turn conversations qualify: the history is reused every turn.
- **Reuse happens within the TTL.** Requests must land close enough together (5 min, or
  1 hour on the extended tier) to hit a warm cache. Steady or bursty traffic is ideal.
- **The prefix is genuinely stable.** It must be byte-identical across calls — no
  interleaved timestamps, IDs, or nondeterministic ordering.

**Caching does NOT pay off when:**

- **One-off or low-volume calls.** A prefix used once is a pure write penalty (1.25–2×)
  with no read to amortize it. A unique prompt per request can't be cached at all.
- **The "static" content actually changes.** If volatile data is mixed into the prefix
  (6.2/6.3), each call is a fresh write — you pay the penalty every time and never
  read. This is the most common way teams *lose* money on caching.
- **Reuse gaps exceed the TTL.** Sparse traffic — a request every 20 minutes on a
  5-minute cache — evicts before reuse; every call is a write. Either move to the
  1-hour TTL, add keep-alives, or don't cache.
- **The prefix is short.** Below the minimum cacheable length nothing is cached;
  marginally above it the savings may not justify the effort.

**The decision in practice:** estimate (cached prefix size) × (expected reads before
eviction). High product → cache, structure the prompt stable-first, pick the TTL that
covers your reuse gaps. Low product → don't bother, or restructure the workload to raise
reuse first. Then *measure*: watch `cache_read` vs `cache_creation` in the usage data
and your cache hit rate. A low hit rate or persistently high `cache_creation` means the
cache isn't working — usually an unstable prefix or a TTL/traffic mismatch — and you
should fix the structure rather than assume caching is helping.

One more nuance: with Anthropic's **explicit** caching you can over-cache — spending
write penalties on prefixes that don't get reused. With OpenAI/Google **automatic**
caching there's no write penalty to mismanage (reads are simply discounted when they
happen), so the failure mode is just "didn't get hits," not "paid to write for nothing."

### Key terms

- **Reuse factor** — how many times a cached prefix is read before eviction; the primary
  driver of whether caching pays.
- **Write penalty** — the extra cost of a cache write over base input, wasted if no read
  follows.
- **Cache hit rate** — fraction of requests (or prefix tokens) served from cache; the
  health metric for a cached workload.
- **Keep-alive request** — a cheap periodic request issued to refresh a cache's TTL and
  keep it warm.

### Common misconceptions

- ❌ Turning on caching always lowers cost. → ✅ For one-off calls or unstable prefixes
  it raises cost via unredeemed write penalties.
- ❌ If you enabled caching, it's working. → ✅ You must verify hit rate / `cache_read`
  vs `cache_creation`; an unstable prefix can mean you're only ever writing.
- ❌ Caching is worth it for any repeated prompt. → ✅ The prefix must be large enough
  (above the minimum length, ideally thousands of tokens) for the savings to matter.

### Worked example

Two services, both "use caching."

- **Service A — doc-Q&A bot:** 30,000-token knowledge base as a static prefix, 50
  requests/minute, prefix byte-stable. Reuse factor is huge, gaps well under the TTL.
  Caching turns ~30k input tokens/request into a 0.1× read after warm-up — a ~90%
  input-cost cut. *Cache: yes.*
- **Service B — nightly batch summarizer:** processes 500 unique documents once each,
  every night, each as its own prompt. No prefix is ever reused (different document
  every call), and the runs are 24 hours apart anyway. Enabling explicit caching just
  adds a 1.25× write penalty to every call with zero reads. *Cache: no* — or cache only
  the shared system prompt / instructions prefix if it clears the minimum length, and
  leave the per-document content uncached.

### Check questions

1. Rank these three workloads by how much they'll benefit from caching, and justify the
   ranking with the size × reuse product: (a) a 300-token prompt reused 100,000×/day;
   (b) a 50,000-token prefix reused 10,000×/day; (c) a 60,000-token prompt that is unique
   per request. — **Answer:** Order, best to worst: (b), then (a), then (c). (b) has a
   large prefix *and* high reuse — the product is huge, and 50k tokens is well above any
   minimum cacheable length. (a) has enormous reuse but a tiny 300-token prefix —
   likely below the minimum cacheable length, and even if cacheable the per-call saving
   is negligible because so few tokens are involved. (c) has a large prefix but *zero*
   reuse — a unique prompt can't be cached at all, so it gets no benefit (and explicit
   caching would only add write penalties). The decision driver is prefix size ×
   expected reads, with a minimum-length floor.
2. A team enables caching and their monthly bill goes *up* by a few percent. They
   protest "caching is supposed to save money." Construct the specific scenario in which
   this is the expected, correct outcome — not a bug. — **Answer:** It's expected when
   the workload is mostly cache *writes* with few or no reads to amortize them — e.g.
   one-off / low-volume calls, or a prefix that isn't byte-stable so every call re-writes
   it, or reuse gaps that exceed the TTL so the entry evicts before reuse. Cache writes
   cost *more* than base input (1.25–2×), so a write-dominated workload pays a penalty on
   every call with no offsetting cheap read — the bill correctly rises. Caching is a
   trade, not free upside; this is the losing side of the trade.
3. Caching is "on" but the bill didn't drop. What single piece of evidence do you check
   first, and what does each possible reading of it tell you? — **Answer:** Check the
   usage data — specifically `cache_read_input_tokens` vs `cache_creation_input_tokens`
   (and the resulting hit rate). If `cache_read` dominates and `cache_creation` is near
   zero, caching *is* working and the savings should show — look elsewhere (e.g. output
   cost). If `cache_creation` stays high call after call, the cache is being written but
   not read — an unstable prefix or a TTL/traffic mismatch — and you fix the prompt
   structure or TTL rather than assume caching helped.

---

## 6.7 — Latency impact (TTFT)

### Concept

Prompt caching is a cost optimization *and* a latency optimization — and the latency win
is often the more visible one to users.

Recall the latency breakdown from Topic 1 / Topic 12. **Time to first token (TTFT)** is
how long the user waits before the response starts streaming. TTFT is dominated by
**prefill** — the model must process the entire input prompt before it can emit the
first output token. Prefill time scales with input length: a long system prompt + tools
+ documents can be tens of thousands of tokens, and prefilling them is the bulk of TTFT.
**Decode** (generating each subsequent token) drives inter-token latency / TPOT and is
unaffected by caching.

On a cache **hit**, the prefix's prefill is *skipped* — the K/V vectors are loaded from
the cache (a fast memory operation) instead of recomputed (an expensive matrix
operation). Only the short uncached tail is prefilled. So **TTFT drops sharply** — in
practice often by a large fraction for prompts with a big static prefix; OpenAI states
prompt caching "can reduce latency by up to 80%" for cache-heavy prompts.[2] The longer
the cached prefix relative to the variable tail, the bigger the TTFT win. (Exact
reductions are workload-dependent; treat any single percentage as illustrative.)

Key points to be precise about:

- **The win is on TTFT (and total latency), not on TPOT.** Caching does nothing for the
  per-token generation speed once streaming starts; it removes the *upfront* wait.
- **A cache miss has the normal TTFT.** First-of-its-kind requests, post-eviction
  requests, and unstable-prefix requests pay full prefill. Caching makes the *common*
  case fast, not every case.
- **The write itself isn't slower for the user.** A cache-creation request prefills
  normally and writes to cache as a side effect; it's a normal-latency request, just one
  that populates the cache for next time.
- **This compounds with the cost win.** The same mechanism — skipping redundant
  prefill — both saves money (6.5) and saves time. For latency-sensitive products
  (interactive chat, coding assistants, voice), a cache-warm path can be the difference
  between a snappy and a sluggish feel.

A useful framing: caching shifts a workload's cost from "every request pays full
prefill" to "one request pays full prefill, the rest pay a cheap fast read." For a
high-traffic app with a large stable prefix, that's simultaneously the cheapest and the
fastest design — which is why cache-aware prompt structure (6.3) is considered table
stakes for production LLM systems.

### Key terms

- **TTFT (time to first token)** — latency from request to the first streamed output
  token; dominated by prefill.
- **TPOT / inter-token latency** — time per generated token during decode; *not*
  improved by caching.
- **Prefill skip** — on a cache hit, loading the prefix's K/V from cache instead of
  recomputing it, which cuts TTFT.
- **Warm path / cold path** — requests that hit the cache (fast TTFT) vs. miss it
  (normal TTFT).

### Common misconceptions

- ❌ Caching makes the model generate tokens faster. → ✅ It speeds up prefill (TTFT
  only); decode speed / TPOT is unchanged.
- ❌ Every request gets the latency win. → ✅ Only cache hits do; misses (first request,
  post-eviction, unstable prefix) pay full prefill.
- ❌ The latency benefit is small. → ✅ For prompts with a large static prefix, skipping
  that prefill commonly cuts TTFT by a large fraction — often the most user-visible
  benefit of caching.

### Worked example

A coding assistant sends a 25,000-token prefix (system prompt + repo context + tool
defs) plus a ~300-token user instruction.

- **Cache miss (cold):** prefill must process all ~25,300 input tokens before the first
  output token — TTFT might be, say, ~3–4 seconds.
- **Cache hit (warm):** the 25,000-token prefix is loaded from cache; only ~300 new
  tokens are prefilled — TTFT might drop to well under a second.
- Decode is identical in both cases: once tokens start streaming, they stream at the
  same rate. The user experiences caching purely as "the answer starts much sooner."

### Check questions

1. A user complains a chat app "feels slow." Two sub-symptoms: (i) a long pause before
   any text appears, and (ii) text then streams out slowly token by token. Which symptom
   can prompt caching help, which can't, and why does the distinction fall exactly along
   the prefill/decode line? — **Answer:** Caching helps (i) — the pause before text is
   TTFT, dominated by prefilling the input; a cache hit skips the prefix's prefill, so
   the pause shrinks. It cannot help (ii) — slow token-by-token streaming is TPOT /
   decode speed, and decode is never cached (it generates new tokens that can't have been
   precomputed). The split is exactly prefill (cacheable, drives TTFT) vs. decode (not
   cacheable, drives TPOT).
2. Two prompts both have a cache hit. Prompt A is a 24,000-token static prefix + a
   200-token tail; prompt B is a 1,000-token static prefix + a 200-token tail. Both hit
   the cache fully on the prefix — yet A's TTFT improvement from caching is far larger
   than B's. Explain why the *ratio* of prefix to tail, not just "got a hit," determines
   the size of the latency win. — **Answer:** TTFT is the prefill time, which scales with
   how many tokens must be prefilled. Caching removes the *prefix* prefill but the tail
   is still prefilled fresh. For A, caching eliminates prefill of 24,000 of 24,200
   tokens — almost all of it — so TTFT drops dramatically. For B, it eliminates only
   1,000 of 1,200 — the prefill was already small, so the absolute time saved is minor.
   The win is proportional to how much of the total prefill the cached prefix represents.
3. Does the first (cache-writing) request to a new prefix get the latency benefit, and
   does it pay a *latency* penalty for doing the write? — **Answer:** No benefit and no
   extra penalty. The first request pays full prefill at normal TTFT — it can't benefit
   because there's nothing cached yet. But writing the cache is a side effect of that
   normal prefill, not an added slow step, so it's a normal-latency request that simply
   populates the cache so *later* requests get the fast warm path.

---

## Topic 06 — Exam Question Bank

A pool the tutor draws from for the gated exam, mixed-format, scored out of 100, 85 to
pass.

### True / False

1. If a workload sends the same question text twice, prompt caching lets the second
   request return the stored answer without running the model. — **Answer:** False. That
   describes a *response* cache. Prompt caching reuses the prefix's KV vectors; the model
   still runs decode (and prefills any new tokens) on every request.
2. Because a cache hit loads stored vectors instead of recomputing them, a heavily cached
   response is systematically lower quality than an uncached one. — **Answer:** False. A
   cached prefix is equivalent to a recomputed one up to normal floating-point
   nondeterminism — the same last-bit variation two uncached runs of the same prompt
   already exhibit, since float matmul is not associative. Caching introduces no new or
   systematic quality difference; it changes cost and latency only.
3. A prompt's first 1,000 tokens differ from the previous call but tokens 1,001–9,000 are
   identical; therefore tokens 1,001–9,000 can be served from cache. — **Answer:** False.
   Caching matches a contiguous prefix from token zero. A divergence at the start means
   the cache breaks immediately; an identical span in the middle cannot be reused on its
   own.
4. On Anthropic's 5-minute TTL, a prefix that goes idle for 7 minutes and is then
   requested again will still be a cache hit, because the TTL refreshes on use. —
   **Answer:** False. Refresh-on-hit only resets the clock *while the prefix is being
   used*. A 7-minute idle gap exceeds the 5-minute window, so the entry is evicted and
   the next request is a miss (full prefill + fresh write).
5. Moving a workload's cached prefix from the 5-minute TTL to the 1-hour TTL makes cache
   *reads* cheaper. — **Answer:** False. The TTL tier changes the *write* multiplier
   (1.25× vs 2×); cache reads are 0.1× base on both tiers. The 1-hour tier costs more to
   write, not less to read.
6. A team runs a job of 5,000 one-off requests, each with a unique 20k-token prompt, and
   enables explicit caching expecting savings. Their cost will go down. — **Answer:**
   False. Unique prompts share no prefix, so every call is a write with no read to
   amortize it; explicit caching adds the write penalty (1.25–2×) and raises cost.
7. Prompt caching speeds up the streaming of output tokens once a response has started.
   — **Answer:** False. It cuts TTFT by skipping the prefix's prefill; per-token decode
   speed (TPOT) is unaffected, since decode is never cached.
8. The same 3,000-token system prompt is cacheable on Claude Sonnet 4.6 but not on Claude
   Opus 4.7. — **Answer:** True. Minimum cacheable length is model-specific: 1,024 tokens
   for Sonnet 4.6 (3,000 qualifies) but 4,096 tokens for Opus 4.7 (3,000 is below the
   floor and is silently not cached).

### Multiple Choice

1. Prompt caching avoids redundant work specifically in which phase of generation, and
   what makes that phase's output reusable?
   A) decode — because output tokens repeat  B) prefill — because a prefix's K/V vectors
   depend only on it and earlier tokens, so they're identical across requests sharing
   that prefix  C) decode — because the KV cache stores the answer  D) prefill — because
   the embedding matrix is shared — **Answer:** B. Caching reuses prefill's K/V vectors,
   reusable because they don't depend on anything after the prefix.
2. A single changed token at position 10 of a 5,000-token prompt causes:
   A) the whole cache including position 0–9 to be lost  B) no cache effect at all
   C) a cache miss for tokens 10 onward; 0–9 may still hit  D) an API error —
   **Answer:** C.
3. An app's prompt has a 12k-token knowledge base and a per-call user question. Putting
   the user question *first* (before the knowledge base) is bad for caching because:
   A) the question is too short to cache  B) the question changes every call, so placing
   it at position zero makes everything after it — including the 12k base — uncacheable
   C) questions can never be cached  D) the knowledge base must be split into 128-token
   chunks — **Answer:** B. The variable element at the front forfeits the cacheable
   prefix.
4. A 10,000-token prefix on Claude Opus 4.7 (base input $5/MTok) is written once on the
   5-minute TTL and read 9 times. The total prefix input cost is closest to:
   A) $0.50  B) $0.11  C) $0.0625  D) $0.05 — **Answer:** B. Write: 0.01 MTok × $6.25 =
   $0.0625; 9 reads: 9 × 0.01 × $0.50 = $0.045; total ≈ $0.1075.
5. A workload's reuse pattern has a request roughly every 8 minutes. On Anthropic, the
   sensible caching choice is:
   A) the 5-minute TTL, since it's the cheaper write  B) the 1-hour TTL, because the
   8-minute gaps would evict a 5-minute cache and force a fresh write every call
   C) no caching is ever possible with gaps over 5 minutes  D) split the prompt into
   30-second segments — **Answer:** B. The 1-hour TTL survives the gaps; its higher write
   price is offset by converting repeated writes into reads.
6. The first request that writes a new prefix to the cache:
   A) gets the full latency benefit  B) pays the cache-read price  C) pays full prefill
   at normal TTFT and populates the cache for later requests  D) is rejected —
   **Answer:** C.
7. Anthropic's 1-hour TTL needs two cache reads to break even (vs. one for the 5-minute
   TTL) because:
   A) the 1-hour read price is double the 5-minute read price  B) the 1-hour write
   multiplier is 2× base — a 1× penalty — and a single read's 0.9× saving can't cover it
   C) the 1-hour TTL caps reads at two  D) 1-hour caches are smaller — **Answer:** B. The
   tier difference is driven entirely by the larger write penalty.
8. The most common way teams *lose* money on explicit caching is:
   A) the TTL being too long  B) interleaving volatile data into the "static" prefix so
   every call is a write  C) reads being too cheap  D) using too few tools —
   **Answer:** B.

### Short Answer

1. The exact-prefix-match requirement is not an arbitrary API rule — it follows from how
   K/V vectors are computed. Derive the requirement from the mechanism, and use that
   derivation to explain why a matching paragraph in the *middle* of a prompt still
   cannot be cached on its own. — **Model answer:** A token's K/V vectors depend on that
   token and every token before it. So any difference at position k changes the K/V of
   token k and of every later token (each attends back through k). A stored cache is
   therefore only valid as a contiguous run from token zero up to the first divergence —
   that's why the match must be exact and prefix-anchored. A middle paragraph can't be
   cached alone because its tokens' K/V depend on everything before it; unless that
   entire preceding context is also identical and matched from token zero, the
   paragraph's vectors differ.
2. A teammate proposes "cache everything aggressively — writes and reads are both just
   discounted input, so more caching is always cheaper." Identify the false assumption
   and correct it with the actual pricing relationships. — **Model answer:** The false
   assumption is that a cache write is *discounted*. It is not — a cache write costs
   *more* than base input (Anthropic: 1.25× on the 5-minute tier, 2× on the 1-hour
   tier), because the provider processes *and* stores the tokens. Only cache *reads* are
   discounted (0.1× base). So caching a prefix that won't be re-read loses money. "More
   caching is always cheaper" is wrong; "caching frequently-reused large prefixes is
   cheaper" is right.
3. Two engineers disagree: one says a 5-minute-TTL cache "expires 5 minutes after you
   create it," the other says "5 minutes after you last use it." Who is right, and what
   practical consequence follows for a steadily-trafficked prefix? — **Model answer:**
   The second engineer is right — the TTL is a sliding window refreshed on every hit, so
   it counts from *last use*, not creation. Practical consequence: a prefix hit at least
   once every 5 minutes stays warm indefinitely on a single write — you never re-pay the
   write fee for steady traffic. The TTL only bites during idle gaps longer than the
   window.
4. A latency-obsessed engineer asks: "If I cache aggressively, will my tokens-per-second
   streaming speed improve?" Answer the question and explain which part of the
   request/response pipeline caching does and doesn't touch. — **Model answer:** No —
   streaming speed (TPOT / inter-token latency) will not improve. Caching skips prefill
   of the cached prefix, which cuts TTFT (the wait before the first token). Streaming
   speed is governed by decode — generating each new output token — and decode is never
   cached, because output tokens can't be precomputed. Caching touches the input-prefill
   side, not the output-decode side.
5. You're advising on three workloads: (a) a chatbot at 100 requests/minute against a
   fixed 20k-token system prefix; (b) a tool that runs once a day on a unique document;
   (c) an agent with a stable 30k prefix but requests ~25 minutes apart. For each, say
   whether to cache and why — and where you'd hesitate, name the deciding variable. —
   **Model answer:** (a) Cache — large prefix, very high reuse, gaps far under the TTL;
   clear win. (b) Don't cache the document — one-off, no reuse, the write penalty has
   nothing to amortize it; only a shared instruction prefix (if any, and above the
   minimum length) is worth caching. (c) Caching is possible but the 25-minute gaps
   exceed the 5-minute TTL — deciding variable is the TTL: use the 1-hour TTL (or
   keep-alives), and do the write-penalty-vs-read-savings math at the observed reuse
   rate; if reuse is too sparse even for the 1-hour tier, don't cache.
6. Your bill didn't drop after enabling caching. Walk through how you'd use the usage
   object to distinguish "caching is working, savings are elsewhere" from "caching is
   silently failing," and what the failing case implies about the prompt. — **Model
   answer:** Inspect `cache_read_input_tokens` vs `cache_creation_input_tokens` across
   requests after warm-up. If `cache_read` dominates and `cache_creation` is near zero,
   caching *is* working — the unchanged bill is from something else (e.g. output tokens,
   or the prefix being a small share of total cost). If `cache_creation` stays high every
   call, caching is failing: the prefix is being written but not read, which implies it
   isn't byte-stable (a volatile token early in it) or the TTL keeps expiring before
   reuse. The fix is structural — stabilize the prefix or change the TTL.
7. Multi-turn chat is described as "naturally cache-friendly." Explain the mechanism that
   makes it so, and then describe one common app behavior that silently throws that
   benefit away. — **Model answer:** Mechanism: once a turn completes, the messages up
   to that point are a fixed prefix; the next request reuses that cached prefix and only
   prefills the new user turn — the cache extends incrementally turn by turn. The common
   behavior that throws it away: rewriting or summarizing earlier turns mid-session
   (history compaction). Any edit to earlier turns changes the prefix from the edit point
   on, so subsequent requests can no longer reuse the built-up cache and pay a fresh
   write.

### Long Answer

1. Explain the mechanism of prompt caching from first principles, starting from the KV
   cache, and why it follows that caching requires an exact prefix match. — **Model
   answer / rubric:** Should cover: prefill computes K/V vectors per token per layer,
   stored in the KV cache so decode needn't recompute them. A token's K/V depends only
   on it and prior tokens. Therefore two requests with an identical prefix produce
   identical K/V for that prefix — recomputing is waste. Prompt caching stores and
   reuses that prefix KV cache. Because every token attends to all earlier ones, one
   differing token at position k changes all K/V from k on, so reuse is valid only up to
   the first divergence — hence the exact, prefix-anchored match requirement. The output
   is unchanged; only redundant prefill is saved.
2. You're designing the prompt for a high-traffic customer-support assistant. Lay out a
   cache-optimal prompt structure and justify each placement decision. — **Model answer
   / rubric:** Order, front to back: tool definitions → system prompt (role, policy) →
   large static context (knowledge base / docs) → few-shot examples → conversation
   history → current user query last. Justification: caching matches a prefix anchored
   at the start and needs byte-stability, so the most-stable content goes first to
   maximize the cacheable region; the variable per-request query goes last so it doesn't
   bust the prefix. Hoist all volatile data (timestamps, IDs) to the tail. Place
   Anthropic `cache_control` breakpoints at the end of each stable segment. Conversation
   history caches incrementally — don't rewrite earlier turns. Net effect: a large
   static prefix is written once and read cheaply/fast thereafter.
3. Walk through the pricing math and break-even analysis for Anthropic prompt caching,
   for both TTL tiers. — **Model answer / rubric:** Should give: base input = 1×; 5-min
   write = 1.25×; 1-hour write = 2×; read = 0.1×; output unaffected. No-cache cost is 1×
   per call. With caching: first call pays the write multiplier, later calls pay 0.1×.
   5-min: write penalty over base = 0.25×; each read saves 0.9× — so after one read the
   saving exceeds the penalty → break-even at 1 reuse. 1-hour: write penalty = 1×; each
   read saves 0.9× — so two reads (1.8× saved) are needed → break-even at 2 reuses.
   Should include a numeric example (e.g. a 10k-token prefix over N calls) showing the
   average per-call cost trending toward the 0.1× read price as reuse grows.
4. Compare Anthropic's explicit prompt caching with OpenAI's automatic caching — the
   developer model, pricing, TTL, and failure modes. — **Model answer / rubric:**
   Anthropic: explicit — developer sets `cache_control` breakpoints (up to 4); 5-min
   default or opt-in 1-hour TTL; writes cost 1.25×/2×, reads 0.1×; minimum cacheable
   length is model-specific (~4,096 tokens for Opus 4.x); failure mode is over-caching —
   paying write penalties for prefixes that aren't reused. OpenAI: automatic/implicit —
   no code change, provider hashes and matches the longest prior prefix in 128-token
   increments; eligible above ~1,024 tokens; no separate write penalty; cache-read
   discount is model-specific (50% on the older GPT-4o generation, but 90% / 0.1× base
   on current flagship models such as GPT-5.5 and GPT-5.4 — check OpenAI's current
   pricing); TTL not configurable, ~5–10 min idle, max ~1 hour; failure mode is just
   "no hits," not paid-for-nothing writes. Both rest on the same prefix-KV-reuse
   mechanism and reward stable-prefix prompt design.

### Applied Scenario

1. A team enabled Anthropic prompt caching on a bot with a large, genuinely static
   8,000-token system prompt — no timestamps, no IDs, byte-identical text every call.
   Their bill still didn't drop, and the usage object shows `cache_creation_input_tokens`
   high on every request. The prompt is built by serializing a Python dict of
   policy sections into the system prompt. Diagnose the non-obvious cause and give the
   fix. — **Model answer / rubric:** The cause is *ordering instability*, not volatile
   *content*. The policy text is the same, but serializing a dict (or concatenating
   retrieved sections) without a deterministic order can emit the sections in a different
   sequence from call to call. Different token *order* means the prefix diverges early
   even though every section's text is identical — so the cache busts and every call is a
   fresh write. Fix: serialize with a deterministic order (e.g. sort keys, or use an
   explicit ordered structure) so the assembled prefix is byte-identical every call.
   Verify via `cache_read` rising and `cache_creation` falling. Key teaching point:
   "byte-stable" means identical tokens *in identical order*, not just identical content.
2. A doc-Q&A service sends a 40,000-token reference document with each user question.
   Traffic is steady at ~20 requests/minute. The document is the same for all users.
   Should they cache, with what structure and TTL, and what's the rough payoff? —
   **Model answer / rubric:** Yes — large prefix, high reuse, gaps far under any TTL.
   Structure: tool defs + system prompt + the 40k-token document as a static prefix
   (cache breakpoint after it), user question last. 5-minute TTL suffices given steady
   sub-minute traffic (no need to pay the 2× 1-hour write). Payoff: after warm-up the
   ~40k input tokens are billed at 0.1× instead of 1× — roughly a 90% cut in input cost
   for that prefix — plus a large TTFT reduction since the document's prefill is skipped.
3. A nightly batch job summarizes 1,000 distinct documents, one prompt per document,
   once each. An engineer wants to "turn on caching to save money." What do you advise?
   — **Model answer / rubric:** Caching the per-document content won't help — each
   document is unique, so there's no prefix reuse; explicit caching would just add a
   1.25× write penalty per call for zero reads, raising cost. Advice: don't cache the
   document content. If there's a shared instructions/system-prompt prefix common to all
   1,000 calls and it clears the model's minimum cacheable length, cache *that* prefix
   only (it's reused 1,000 times within the run). Also note the Batch API as the right
   cost lever for this workload, and that runs 24h apart wouldn't share a cache anyway.
4. Your interactive coding assistant feels sluggish — users wait ~4 seconds before any
   response. The prompt is a 30,000-token repo-context prefix plus a short instruction.
   Explain the cause and how caching helps, and what it won't fix. — **Model answer /
   rubric:** The ~4s wait is TTFT, dominated by prefilling the 30,000-token prefix.
   Prompt caching: structure the prompt with the stable repo context as a prefix; after
   the first (cold) request, subsequent requests load the prefix's K/V from cache and
   prefill only the short instruction, cutting TTFT dramatically (warm path well under a
   second). It also cuts input cost (~0.1× reads). What it won't fix: the cold/first
   request still pays full prefill; TPOT / token streaming speed is unchanged; and if
   the repo context changes between requests the prefix won't be stable, so keep volatile
   parts out of the cached region.
5. An Anthropic-based agent runs long tool-using sessions where a request arrives every
   ~8 minutes on average. They use the default 5-minute TTL and see lots of
   `cache_creation_input_tokens`. What's happening and what are the options? — **Model
   answer / rubric:** The ~8-minute gaps exceed the 5-minute TTL, so the cache evicts
   between requests and each call re-writes the prefix (high `cache_creation`) instead
   of reading it — paying the write penalty repeatedly with few reads. Options: (1)
   switch the prefix to the 1-hour TTL — its 2× write costs more per write but it
   survives the 8-minute gaps, turning most calls into 0.1× reads; do the math on write
   penalty vs. read savings at the observed reuse rate. (2) Issue cheap keep-alive
   requests inside the 5-minute window to keep the cache warm. (3) If sessions are truly
   sparse and reuse is low, accept that caching may not pay and don't cache. Decide by
   measuring hit rate and `cache_read` vs `cache_creation` under each option.

---

## Sources

[1] Anthropic — Prompt caching (Claude API: `cache_control`, breakpoints, 20-block
lookback, TTL, minimum cacheable lengths, usage fields) —
https://platform.claude.com/docs/en/build-with-claude/prompt-caching

[2] OpenAI — Prompt Caching in the API (automatic caching, 1,024-token minimum,
128-token increments, inactivity window, latency reduction) —
https://openai.com/index/api-prompt-caching/

[3] Anthropic — Pricing (base input/output rates and prompt-caching multipliers:
1.25× / 2× cache writes, 0.1× cache reads) —
https://platform.claude.com/docs/en/about-claude/pricing

[4] OpenAI — API Pricing (current per-model rates; cached input at 0.1× of standard
input for GPT-5.5 / GPT-5.4) — https://developers.openai.com/api/docs/pricing
