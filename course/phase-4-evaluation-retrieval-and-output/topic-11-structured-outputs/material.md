# Topic 11 — Structured Outputs — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This topic is taught sub-chapter by sub-chapter, in order. Each sub-chapter has a
**Concept** section (the teaching content), **Key terms**, **Common misconceptions**, a
**Worked example**, and **Check questions** to confirm understanding before moving on.
The **Exam Question Bank** at the end is the pool the tutor draws from for the gated,
closed-book exam (scored out of 100, pass mark 85). Teach each sub-chapter fully, run its
check questions, then advance. Structured outputs are how you turn a free-text model into
a reliable component of a software system — the goal is to know the mechanisms, their
guarantees, their limits, and how to build a robust integration around them.

---

## 11.1 — JSON mode vs. constrained decoding vs. tool-use-for-structure

### Concept

An LLM emits free-form text. But software needs **machine-readable structure** — a JSON
object you can parse, validate, and pass to the next function. The naive approach — "in
the prompt, please ask for JSON" — works *most* of the time and fails the rest: the model
wraps the JSON in prose ("Here's the JSON you asked for:"), wraps it in a markdown code
fence, adds a trailing comma, omits a closing brace, hallucinates an extra field, or
emits a different type than expected. At scale, a 1% malformed-output rate is a constant
stream of production errors. **Structured outputs** are the family of mechanisms that
make the structure *reliable*. There are three, with very different guarantees.

**1. JSON mode.** A flag (e.g., OpenAI's `response_format: {"type": "json_object"}`) that
instructs the model and decoder to produce *syntactically valid JSON* — it will parse. [1]
What it does **not** guarantee: that the JSON matches *your* schema. The model can return
valid JSON with the wrong field names, missing required fields, extra fields, or wrong
value types. JSON mode solves "is it parseable JSON?" but not "is it the *right* JSON?"
You still need to validate and handle schema mismatches. It is the weakest of the three.

**2. Constrained decoding (guided / grammar-constrained decoding).** This works at the
**token-sampling level**. Recall (Topic 1/3) that generation picks the next token from a
probability distribution over the whole vocabulary. Constrained decoding **masks out
every token that would violate the schema** before sampling — at each step, only tokens
that keep the output a valid prefix of a schema-conforming document are allowed. The
schema (a JSON Schema, or a formal grammar) is compiled into a state machine that, given
"what has been generated so far," computes the set of legal next tokens. Because an
illegal token is *impossible to sample*, the output is **guaranteed** to conform to the
schema's structure — not "usually," but always, by construction. This is the strongest
guarantee. It is what OpenAI's **Structured Outputs** (`strict: true`) [2] and Anthropic's
native **Structured Outputs** use under the hood. [3] (Provider-specific note: Anthropic's
Structured Outputs is GA across recent Claude models — Sonnet 4.5, Opus 4.5, Haiku 4.5
and newer — and the JSON-output parameter is `output_config.format` with
`type: "json_schema"`; the earlier `output_format` parameter and its beta header still
work for a transition period. Always check the current provider docs, as these
parameter names and the model list evolve.) [3]

**3. Tool-use-for-structure.** Define a "tool" (function) whose input JSON Schema *is*
the structure you want, and force the model to "call" it (`tool_choice` set to that
tool). The model's tool-call arguments are your structured object — you never execute a
real function. Before native structured outputs existed, this was the standard trick to
get schema-shaped output, because tool-calling pipelines already validate arguments
against a schema. Modern providers also apply strict/constrained decoding to tool inputs
("strict tool use"). It works, but it is a workaround: it overloads a feature meant for
*actions* to do *formatting*, and it can be more brittle and harder to reason about than
a dedicated structured-output mode.

**How to choose (2026).** Prefer **native structured outputs / constrained decoding** for
a structured *final answer* — it gives the guarantee with the least ceremony. Use
**tool-use** when structure is genuinely part of an agentic flow (the model is choosing
among real tools and one of them produces structured data), or on older models/providers
without native structured outputs. Use plain **JSON mode** only when you need loose JSON
and will validate yourself, or as a fallback. And note: the model's *attention to your
prompt instructions* still matters — constrained decoding guarantees the *shape*, not the
*content*; you should describe the schema in the prompt too.

### Key terms

- **Structured output** — model output reliably conforming to a machine-readable structure (usually JSON).
- **JSON mode** — a mode guaranteeing syntactically valid JSON but not schema conformance.
- **Constrained / guided decoding** — masking illegal tokens at sampling time so output is guaranteed to conform to a schema/grammar.
- **Token masking** — zeroing the probability of tokens that would violate the schema before sampling the next token.
- **Grammar / state machine** — the compiled representation of the schema that determines legal next tokens.
- **Tool-use-for-structure** — defining a tool whose argument schema is the desired structure and forcing a call to it.
- **strict mode** — a provider flag (OpenAI's `strict: true`; Anthropic's `output_config.format`) enabling constrained decoding against a schema. Exact parameter names are provider-specific.

### Common misconceptions

- ❌ "JSON mode guarantees the output matches my schema." → ✅ JSON mode only guarantees *valid JSON*; it can return valid JSON with wrong fields, missing required keys, or wrong types. Schema conformance needs constrained decoding (strict mode).
- ❌ "Constrained decoding is just better prompting." → ✅ It operates at the token-sampling level, masking illegal tokens so a violating token is impossible to emit — a hard mechanism, not a stronger request.
- ❌ "Tool-use-for-structure and structured outputs are the same thing." → ✅ Tool-use is a workaround that repurposes the actions feature for formatting; native structured outputs are a dedicated mode. Prefer the latter for a structured final answer.

### Worked example

You want a sentiment label and a confidence score.

- **Prompt only:** `"Return JSON."` → the model sometimes returns
  ` ```json\n{"sentiment": "positive", "confidence": 0.9}\n``` ` (fenced — needs
  stripping), sometimes `Here is the result: {...}` (prose prefix — `JSON.parse` throws),
  sometimes `{"sentiment": "positive",}` (trailing comma — invalid JSON).
- **JSON mode:** always parseable JSON — but the model might return
  `{"label": "positive", "score": 0.9}` (wrong field names) or
  `{"sentiment": "positive"}` (missing `confidence`). Parses, still breaks your code.
- **Constrained / strict structured output** with the schema
  `{sentiment: enum["positive","neutral","negative"], confidence: number}` and both
  fields required → the model **cannot** emit anything but exactly those two fields with
  those types; `sentiment` cannot even be a value outside the enum. Guaranteed shape,
  every call.

### Check questions

1. A service in JSON mode expects `{"category": "...", "priority": 1}`. It receives `{"type": "billing", "urgency": "high"}` — which parses fine but breaks downstream code. Is this a JSON-mode bug, and what would have prevented it? — **Answer:** It is not a JSON-mode bug — JSON mode guarantees only *syntactically valid JSON*, which `{"type": "billing", "urgency": "high"}` is. JSON mode never promised the JSON matches *your* schema, so wrong field names and wrong types are exactly the failure it leaves open. Prevention: constrained/strict decoding against your schema, which masks any token that would produce a non-conforming field name or type.
2. Constrained decoding is sometimes described as "really strong prompting." Explain why that description is wrong, referencing what happens at the token level. — **Answer:** Prompting only changes the *probability distribution* the model samples from — a non-conforming token still has nonzero probability and can be sampled. Constrained decoding operates at the sampling step itself: it *masks* (zeros the probability of) every token that would break schema conformance, so an illegal token is literally impossible to emit. It is a hard mechanism that makes conformance a guarantee by construction, not a stronger request that the model can still violate.
3. A team gets schema-shaped output by defining a "tool" whose argument schema is their desired structure and forcing the model to call it — and it works. Why does the material still call this a workaround rather than the preferred approach? — **Answer:** Because it repurposes the tool/function-calling feature — built for the model to request *actions* it intends to execute — to do pure output *formatting*, where no function is ever run. That overloading can be more brittle and harder to reason about than a dedicated native structured-output mode. It works, and is still appropriate inside genuine agentic flows or on providers lacking native structured outputs, but for a structured *final answer* the native mode is preferred.

---

## 11.2 — How constrained decoding actually works — schema, grammar, and the token FSM

### Concept

11.1 said constrained decoding "masks illegal tokens." That is the *what*. This
sub-chapter is the *how* — and the how matters, because the mechanism has a sharp
practical consequence (a latency cost) and one genuinely hard problem (tokenizer
alignment) that every engineer using structured outputs should understand.

**Step 1 — schema → grammar.** A JSON Schema is a *declarative* description ("an object
with these fields, this one an enum, this one an integer"). It cannot be used directly to
constrain generation. So the first step compiles the schema into a **formal grammar** —
a precise, generative description of *exactly which strings are valid*. A grammar is the
same idea a programming-language parser uses; the common notation in this space is
**BNF / EBNF**, and llama.cpp's well-known format is called **GBNF** (GGML BNF). A
JSON-Schema-to-grammar compiler turns `{"type":"object","required":["x"],
"properties":{"x":{"type":"integer"}}}` into grammar rules that say, in effect: a `{`,
then the literal key `"x"`, then `:`, then a sequence of digits, then `}` — with
whitespace allowed in the legal places. JSON mode itself is just the special case where
the grammar is "any valid JSON document."

**Step 2 — grammar → finite-state machine (FSM) / automaton.** A grammar still describes
*strings*; generation happens *token by token*. So the grammar is compiled into a
**state machine**: a graph of states where each state knows "given everything generated
so far, which continuations keep the output a valid *prefix* of some schema-conforming
document." For the regular parts of a schema this is a finite-state automaton; JSON's
nesting (arbitrarily deep objects/arrays) actually needs a **pushdown automaton** (an FSM
plus a stack to track nesting depth) — but "token FSM" is the usual loose name. The key
property: at every generation step the machine is in some state, and that state defines a
**set of allowed next characters/tokens**.

**Step 3 — per-step token masking.** Now the decoding loop. At each step the model
produces logits over the *entire vocabulary* (tens or hundreds of thousands of tokens).
Constrained decoding intersects "what the FSM allows" with "what tokens exist," builds a
**mask** that sets the logit of every disallowed token to −∞, and only then samples. The
sampled token is guaranteed schema-legal; the FSM advances to its next state; repeat.
Because an illegal token has −∞ logit, it has zero probability — conformance is a
guarantee *by construction*, exactly as 11.1 claimed. (This is also why field *order* is
fixed and why a `reasoning` field must come first, 11.4: the FSM walks the schema in
order.)

**The tokenizer-alignment problem — the genuinely hard part.** Here is the subtlety that
makes this more than a parsing exercise. The grammar/FSM thinks in **characters** (or
bytes): "the next character must be a digit, or a `}`." But the model does not generate
characters — it generates **tokens**, and a token is an arbitrary multi-character chunk
chosen by the tokenizer (Topic 2). A single token might be ` {"`, or `123`, or `_name":`
— spanning a grammar boundary. So **grammar boundaries and token boundaries do not line
up.** The constraint engine cannot simply ask "is this character legal"; for *every*
token in the vocabulary it must determine "could this whole multi-character token be a
legal continuation from the current FSM state — possibly advancing the FSM through
several states?" A token that is *partly* legal and partly not must be masked out. A
token that completes one grammar element and starts the next is fine. This token-vs-FSM
reconciliation is the core engineering difficulty of constrained decoding, and doing it
*naively* — walking the FSM character-by-character against all ~100k+ vocabulary tokens
at every single step — is what makes constrained decoding slow.

**The latency cost.** Constrained decoding is **not free**. Two costs:

1. **Compilation cost (one-time per schema).** Turning a schema into a grammar and the
   grammar into an FSM/automaton takes time. A complex schema can take noticeable time to
   compile; this is paid once and is cacheable, but a brand-new schema on a latency-
   sensitive first request can stall.
2. **Per-token mask cost (every decoding step).** Computing the allowed-token mask at each
   step adds work *inside the hot decoding loop*. Done naively this can dominate — the
   masking step rivaling or exceeding the model forward pass. This is exactly the problem
   the named systems below were built to solve, by **precomputing** as much of the
   token↔FSM mapping as possible so the per-step cost becomes a cheap lookup.

The modern fast libraries get the steady-state per-token overhead very low — often near
negligible relative to the forward pass — but the principle stands: someone is paying a
compilation cost and a per-step masking cost, and on a poorly-optimized stack or a
pathological schema it is visible. "Structured output is free latency-wise" is a
misconception.

**Named systems to know.** You will encounter these by name:

- **Outlines** — a widely-used Python library that compiles JSON Schema / regex into FSMs
  for guided generation; influential for popularizing the FSM-indexing approach. [6]
- **XGrammar** — a high-performance constrained-decoding engine focused on driving the
  per-token overhead toward zero (precomputed token masks, context-aware caching); used
  in several serving stacks. [7]
- **llguidance** — a fast grammar-constraint engine (used by Microsoft's Guidance and
  others) designed for low per-token latency.
- **GBNF** — llama.cpp's BNF-style grammar format; the way you hand-write a grammar
  constraint for a locally-served model.
- **Guidance** — a higher-level library for interleaving generation and constraints.

You rarely call these directly — a provider's "Structured Outputs" / `strict: true` mode
or a serving framework (vLLM, TGI, SGLang) uses one of them under the hood. But knowing
the layer exists explains *why* a provider supports only a subset of JSON Schema (11.3):
the schema must be compilable into a grammar their engine can execute efficiently, and
features like unbounded recursion or arbitrary `pattern` regexes are hard or expensive to
turn into a fast FSM.

### Key terms

- **Grammar (BNF / EBNF / GBNF)** — a formal, generative description of exactly which strings are valid; the compiled form of a schema for constraining generation.
- **Finite-state machine / automaton** — the per-step machine compiled from the grammar; each state defines the set of legal next tokens. JSON nesting needs a pushdown automaton (FSM + stack).
- **Token mask** — the per-step vector that sets disallowed tokens' logits to −∞ before sampling.
- **Tokenizer-alignment problem** — the mismatch between character-level grammar boundaries and multi-character token boundaries, which the constraint engine must reconcile.
- **Compilation cost** — the one-time cost of turning a schema into a grammar and FSM.
- **Per-token mask cost** — the recurring per-step cost of computing the allowed-token mask inside the decoding loop.
- **Outlines / XGrammar / llguidance / Guidance** — named constrained-decoding engines/libraries that providers and serving stacks use under the hood.

### Common misconceptions

- ❌ "Constrained decoding just checks the JSON after the model writes it." → ✅ It works *during* generation, token by token: the schema is compiled to a grammar then an FSM, and each step masks illegal tokens before sampling — so an invalid document can never be produced in the first place.
- ❌ "The grammar can constrain the model character by character directly." → ✅ The model emits multi-character *tokens* that don't align with grammar boundaries; the engine must, for every vocabulary token, decide whether the whole token is a legal FSM continuation — the tokenizer-alignment problem.
- ❌ "Constrained decoding adds no latency." → ✅ It has a one-time schema-compilation cost and a per-token masking cost; fast engines (XGrammar, llguidance) make the steady-state overhead small, but it is not zero, and a naive implementation can be slow.
- ❌ "Providers limit JSON Schema support arbitrarily." → ✅ The limits trace to this mechanism: a feature must be compilable into a grammar/FSM the engine can execute efficiently, so unbounded recursion and arbitrary regex `pattern`s are restricted.

### Worked example

Constraining the output to `{"in_stock": <boolean>}`.

1. **Schema → grammar.** The compiler emits rules roughly: `root ::= "{" ws "\"in_stock\""
   ws ":" ws boolean ws "}"` and `boolean ::= "true" | "false"` (with `ws` = optional
   whitespace).
2. **Grammar → FSM.** This becomes a small automaton. State 0 expects `{`. After `{` and
   the literal key and `:`, the machine reaches a state whose *only* legal continuations
   are the start of `true` or the start of `false`. After the value, only `}` is legal,
   then the document is complete.
3. **Token masking in the value state.** Suppose generation has reached the
   value-expecting state. The model's vocabulary contains tokens like `true`, `false`,
   `tru`, `t`, `maybe`, ` "yes"`, `123`. The engine masks everything except tokens that
   are a legal prefix of `true` or `false` — so `maybe`, ` "yes"`, `123` get −∞ logits.
   Note the **alignment** issue: if the vocabulary has the single token `true` the FSM
   advances straight to the post-value state; if it only has `t`, `r`, `u`, `e`
   separately, the FSM must advance one character-state per token. The engine handles
   both — that reconciliation is precisely the tokenizer-alignment work.
4. **Result.** Whatever the model "wanted" to say, the only documents it can produce are
   `{"in_stock": true}` or `{"in_stock": false}` — guaranteed, because every other
   continuation was unsampleable.

If instead the schema were a 4-level-deep nested object with several enums, step 1–2
would take meaningfully longer to compile (paid once, cached), and step 3 would run a
larger mask computation every token — the latency cost the fast engines exist to minimize.

### Check questions

1. A teammate says "constrained decoding works by generating the text normally and then validating it against the JSON Schema, retrying if it's wrong." Identify what is wrong with this description and give the correct mechanism. — **Answer:** That describes generate-then-validate-with-retry, which is a *different*, weaker pattern. Constrained decoding works *during* generation: the schema is compiled into a grammar and then a finite-state machine; at every decoding step the FSM defines the set of legal next tokens, the engine masks (sets to −∞) the logits of all illegal tokens, and only then samples. An invalid document is never produced — conformance is guaranteed by construction, with no retry needed for *structural* validity.
2. Explain the tokenizer-alignment problem: why can the constraint engine not simply allow or forbid the next *character*? — **Answer:** The grammar/FSM reasons in characters ("next must be a digit or `}`"), but the model does not emit characters — it emits *tokens*, arbitrary multi-character chunks chosen by the tokenizer that do not align with grammar boundaries (one token might be `{"` or `_name":`, spanning grammar elements). So the engine cannot check a single character; for *every* token in the ~100k+ vocabulary it must determine whether that whole multi-character token is a legal continuation from the current FSM state, possibly advancing the FSM through several states. Reconciling token boundaries with character-level grammar boundaries is the core difficulty, and doing it naively is what makes constrained decoding slow.
3. A team measures higher TTFT/latency the first time each new schema is used, and a smaller but constant per-token overhead thereafter. Explain both observations in terms of the mechanism. — **Answer:** The first-use spike is the **compilation cost**: turning a new schema into a grammar and then an FSM/automaton is one-time work, done when the schema is first seen (and then cacheable) — so a brand-new schema stalls the first request. The constant per-token overhead is the **per-token mask cost**: at every decoding step the engine intersects the FSM's legal set with the vocabulary and builds the −∞ mask before sampling, which is recurring work inside the hot loop. Fast engines (XGrammar, llguidance) precompute token↔FSM mappings to shrink this, but it is never exactly zero — "structured output is free" is wrong.

---

## 11.3 — Strict schema guarantees and their limits

### Concept

Constrained decoding with a strict schema is powerful, but an engineer must know
**precisely what it guarantees and what it does not** — over-trusting it causes subtle
production bugs.

**What strict mode genuinely guarantees (the structural envelope):**
- The output parses as valid JSON.
- Every required field defined in the schema is present.
- No field outside the schema appears (when `additionalProperties: false`).
- Each value has the declared *type* (string, number, boolean, array, object).
- `enum` values are restricted to the allowed set — the model literally cannot emit an
  out-of-enum string.
- Arrays/objects nest as the schema specifies.

This is the **shape** of the data. It eliminates an entire class of parsing and
schema-mismatch errors.

**What strict mode does NOT guarantee (the semantics):**
- **Correctness of values.** The schema says `confidence` is a number; it does not say
  it's the *right* number. The model can still output a confident-but-wrong value, a
  hallucinated name, a fabricated date.
- **Semantic / cross-field validity.** A schema can enforce `start_date` and `end_date`
  are strings; it generally cannot enforce `end_date > start_date`. Business rules,
  invariants, and relationships between fields are beyond what most schema constraints
  express.
- **String content.** A field typed `string` accepts *any* string. Unless you add a
  pattern/format constraint, "email" can be `"not an email"`. Even formats like
  `date-time` may be only loosely enforced depending on the provider.
- **Meaningful enum choice.** Strict mode forces `sentiment` into the enum, but it does
  not stop the model from picking the *wrong* enum member for the text.
- **Truthfulness / hallucination.** A perfectly schema-valid object can be entirely
  fabricated. Structure is not grounding.

**Practical limits and footguns:**
- **Schema feature support is partial.** Providers support a *subset* of JSON Schema.
  Common rules with `strict: true` (OpenAI-style): every property must be listed in
  `required`; `additionalProperties` must be `false`; "optional" fields are expressed as
  a union with `null`, not by omission; nesting depth and total schema size are capped
  (OpenAI, for example, currently allows up to ~100 object properties, up to 5 levels of
  nesting, and does not support recursive schemas); some keywords (`minLength`, complex
  `anyOf`, recursion, certain `format`s) may be unsupported or ignored. These exact
  limits are provider-specific and change — read the provider's structured-output docs,
  as a schema valid in the JSON Schema spec may be rejected or partially honored. [4]
- **Overly rigid schemas can hurt.** Forcing the model down a constrained path can
  *degrade* output quality and reasoning (the subject of 11.4); the constraint should be
  as tight as the structure needs and no tighter.
- **A constrained model can still "fail."** It may emit a valid-but-useless object — fill
  required fields with `""`, `0`, or `"unknown"` — when it doesn't actually know. Schema
  validity is not task success.

**The mental model:** strict schema is a **structural contract**, not a **semantic
contract**. It guarantees the *container* is the right shape; it says nothing about
whether the *contents* are correct or sensible. Therefore strict mode does **not remove
the need for validation** — it removes the *parsing* and *shape* class of errors so your
validation layer (11.5) can focus on semantics, ranges, and business rules.

### Key terms

- **Structural contract** — guarantees about the shape of the output (types, required fields, enums).
- **Semantic contract** — guarantees about meaning/correctness of values — which strict mode does *not* provide.
- **additionalProperties: false** — schema rule forbidding fields not declared in the schema.
- **Cross-field invariant** — a rule relating multiple fields (e.g., `end > start`) generally not enforceable by schema.
- **Schema feature subset** — the limited portion of JSON Schema a provider's strict mode actually supports.
- **Valid-but-useless output** — schema-conforming output whose values are empty, placeholder, or wrong.

### Common misconceptions

- ❌ "Strict mode means the output is correct." → ✅ It guarantees the *shape* (types, required fields, enums), not the *correctness* of values — a schema-valid object can be entirely hallucinated.
- ❌ "I can enforce any JSON Schema with strict mode." → ✅ Providers support only a subset; rules like all-properties-required, `additionalProperties:false`, optional-as-null-union, and depth/size caps apply, and some keywords are ignored.
- ❌ "With strict mode I no longer need a validation layer." → ✅ Strict mode removes parsing/shape errors but not semantic errors, ranges, or business invariants — you still validate, just at a higher level.

### Worked example

A strict schema for an extracted invoice:

```
{ "type":"object", "additionalProperties":false,
  "required":["invoice_number","total","currency","due_date"],
  "properties":{
    "invoice_number":{"type":"string"},
    "total":{"type":"number"},
    "currency":{"type":"string","enum":["USD","EUR","GBP"]},
    "due_date":{"type":"string"} } }
```

Strict mode **guarantees**: all four fields present, `total` is a number, `currency` is
one of the three codes, no stray fields, valid JSON.

Strict mode does **not** guarantee: `total` is the *actual* invoice total (the model may
misread `1,200.00` as `1200.00` or `120000`); `due_date` is a real, correctly-formatted
date — `"sometime next month"` is a valid string; `currency` is the *correct* currency
for this invoice (the enum just bounds the choices). Those checks belong to your
validation layer: parse `due_date`, range-check `total`, and ideally cross-check the
total against line items.

### Check questions

1. A schema for an event has `start_time` and `end_time` both typed `string`, and a `status` field with `enum: ["scheduled","cancelled"]`. For each of these three potential errors, say whether strict mode catches it: (a) `end_time` is before `start_time`; (b) `status` is `"pending"`; (c) `status` is `"cancelled"` for an event that actually happened. — **Answer:** (a) Not caught — `end > start` is a cross-field invariant; strict mode enforces the *type* (string) but not relationships between fields. (b) Caught — `"pending"` is outside the enum, and strict mode makes an out-of-enum value impossible to emit. (c) Not caught — `"cancelled"` is a valid enum member; strict mode bounds the *choice set* but cannot tell whether the model picked the *correct* member for reality. The discrimination: strict mode polices the structural envelope (types, enums, required fields), not semantics or correctness.
2. An engineer says "we use strict structured outputs now, so we deleted our validation layer." Describe a concrete production bug this will let through, and what the validation layer should still do. — **Answer:** Strict mode removes parsing and shape errors but not *semantic* errors. Concrete bug: a field typed `number` for `discount_percent` receives `150` — schema-valid, but a 150% discount is nonsense; or a `string` `email` field receives `"none"`. Strict mode passes both. The validation layer must still enforce ranges (`0 ≤ discount ≤ 100`), formats (parseable email/date), cross-field invariants, and business rules — semantics the schema cannot express. Strict mode lets the validation layer *focus* on semantics; it does not eliminate it.
3. A strict-mode extraction always returns a schema-valid object, yet a downstream report is full of `""` and `"unknown"` values. The schema is conforming on every call. Explain how a "successful" structured output can still be a failure. — **Answer:** This is valid-but-useless output: when the model doesn't actually know a required field's value, strict mode still forces it to emit *something* of the right type — so it fills required fields with placeholders like `""`, `0`, or `"unknown"`. The object conforms perfectly to the schema, so schema validation passes, but the task failed — schema validity is a structural property, not task success. The fix belongs to semantic validation (reject placeholder/empty values) and the prompt/abstention design, not the schema.

---

## 11.4 — How structured output can degrade reasoning quality; workarounds

### Concept

A non-obvious but important effect: **forcing structured output can make the model
reason worse.** Engineers reach for strict JSON to make integration clean, then are
surprised that answer *quality* drops. Understanding why is the point of this
sub-chapter.

**Why structured output can hurt reasoning:**

1. **It suppresses chain-of-thought.** LLMs reason by *generating tokens* — "thinking out
   loud" in the output is how they work through a problem (Topic 5). A tight schema like
   `{"answer": number}` forces the model to emit the answer **immediately**, with no room
   to reason first. You have amputated its scratchpad. The model must produce the final
   token before it has "thought," so accuracy on anything requiring multi-step reasoning
   (math, logic, careful classification) can fall.

2. **Token-masking can fight the model's natural distribution.** Constrained decoding
   forces the next token to be schema-legal. If the model's most probable continuation
   was a reasoning word but the schema demands a `{` now, the model is pushed onto a path
   it didn't "want." The schema's *field order* even matters — if `answer` is the first
   key, the model commits to the answer before any field that could hold reasoning.

3. **Format becomes a competing objective.** The model is now juggling two goals — solve
   the problem *and* satisfy the format. Attention and capacity spent on getting the JSON
   right is not spent on the task.

The empirical picture (2024–2026 research and practice) is nuanced: for **simple
extraction/classification**, structured output is roughly as good as free text and far
more convenient. For **hard reasoning** tasks, naively forcing a terse schema can cause a
measurable accuracy drop. The effect is real but **avoidable** with the right design.
(The research here is genuinely contested — some studies report a notable drop from
format restriction, others find little or no effect once a reasoning field is allowed;
treat "terse schema can hurt hard reasoning" as a well-supported caution, not a
universal law.) [5]

**Workarounds — let the model reason, then structure:**

- **Put a reasoning field first in the schema.** Add a `reasoning` (or `chain_of_thought`,
  `scratchpad`) string field *before* the answer fields. The model fills the reasoning
  field first — its scratchpad is back — and only then the structured answer. Schema is
  still strict; reasoning happens *inside* it. This is the simplest and most effective
  fix. (Order matters: reasoning field must come before answer fields.)
- **Two-step / two-call pipeline.** Call 1: unconstrained — let the model reason freely
  in plain text and produce the answer. Call 2: a cheap, constrained call that *only*
  reformats the call-1 output into the strict schema (an extraction call). The hard
  thinking happens with no format constraint; structuring is a separate, easy step.
- **Use the provider's reasoning/thinking mode alongside structured output.** Reasoning
  models can think in a dedicated thinking channel and *then* emit the structured final
  answer — the thinking is not constrained by the schema. Modern providers explicitly
  support combining extended thinking with structured outputs.
- **Loosen the schema where you can.** If a field can be a free-text `string`, the model
  has room there; reserve tight constraints (enums, numbers) for fields that truly need
  them.
- **Prefill / tool-use** with a reasoning step preceding the structured part.

**The principle:** *separate the thinking from the formatting.* Never force the model to
produce a hard answer as its very first token. Give it a place to reason — a leading
field, a separate call, or a thinking channel — and apply the structural constraint to
the final answer only.

### Key terms

- **Reasoning degradation** — loss of answer quality caused by forcing terse structured output.
- **Scratchpad suppression** — a tight schema preventing the model from generating reasoning tokens before the answer.
- **Reasoning field** — a string field placed first in the schema so the model reasons before emitting the answer.
- **Two-step pipeline** — call 1 reasons unconstrained, call 2 reformats into the strict schema.
- **Thinking channel / extended thinking** — a model's dedicated reasoning output, separate from and unconstrained by the answer schema.
- **Separation of thinking and formatting** — the design principle of keeping reasoning unconstrained and structuring it afterward.

### Common misconceptions

- ❌ "Structured output is free — it only changes the format, not the answer." → ✅ A terse schema can suppress chain-of-thought and measurably degrade reasoning on hard tasks; format and reasoning interact.
- ❌ "If I need reasoning, I can't use strict structured output." → ✅ You can — add a `reasoning` field before the answer fields, or use a two-step pipeline or a thinking channel; you separate thinking from formatting.
- ❌ "Schema field order is cosmetic." → ✅ With autoregressive generation under constrained decoding, fields are produced in order; a `reasoning` field only helps if it comes *before* the answer fields.

### Worked example

Task: a word problem — *"A train leaves at 2:15 and arrives at 4:48. A second train
takes 40% longer. How long is the second train's trip, in minutes?"*

- **Terse schema** `{"minutes": integer}`: the model must emit the integer as its first
  content token, no reasoning. It often blurts a wrong number (e.g., 153 — forgetting the
  40%, or mis-converting hours). Accuracy is poor.
- **Schema with a leading reasoning field:**
  ```
  {"reasoning": "string", "minutes": "integer"}
  ```
  The model first fills `reasoning`: *"2:15→4:48 is 2h33m = 153 min. 40% longer = 153 ×
  1.4 = 214.2 → 214 min."* Then it emits `minutes: 214`. The scratchpad restored the
  accuracy, and the output is still strict JSON you can parse.
- **Two-step alternative:** call 1 solves it in free text; call 2, constrained, just
  extracts `{"minutes": 214}` from call 1's text.

Same strict-output requirement, correct answer — because thinking was separated from
formatting.

### Check questions

1. A team adds strict output `{"answer": integer}` to a feature and accuracy on simple lookups stays fine, but accuracy on multi-step word problems drops sharply. Explain why the *same* schema hurts one task type and not the other. — **Answer:** LLMs reason by generating tokens — "thinking out loud" is how they work through a problem. A multi-step word problem needs that scratchpad; `{"answer": integer}` forces the answer as the first content token, so the model must commit before it has reasoned, and accuracy falls. A simple lookup needs no multi-step reasoning, so emitting the answer immediately costs nothing. The schema's harm scales with how much the task depends on intermediate reasoning.
2. Two schemas both contain a `reasoning` string field and an `answer` field. Schema A lists `answer` first, Schema B lists `reasoning` first. Predict which preserves reasoning quality and explain the mechanism that makes order decisive. — **Answer:** Schema B (reasoning first) preserves reasoning. Generation is autoregressive and constrained decoding produces fields in schema order, so in Schema A the model must emit `answer` before it ever writes into `reasoning` — it commits to the answer with no scratchpad, and the reasoning field is filled too late to help. In Schema B it writes `reasoning` first, restoring the scratchpad, then the answer. Same fields; only order determines whether the reasoning field actually functions.
3. A two-step pipeline (unconstrained reasoning call, then constrained reformat call) and a single call with a leading `reasoning` field both "separate thinking from formatting." Give one practical reason a team might choose the two-step pipeline over the single call. — **Answer:** Both work, but the two-step pipeline fully decouples the hard reasoning from any format pressure — call 1 has *zero* schema constraint, so token-masking cannot push the model off its natural path at all, and the cheap call 2 only reformats. A team might prefer it when the reasoning is hard enough that even an in-schema `reasoning` field's surrounding constraints are a concern, or when they want to reuse the unconstrained reasoning output elsewhere. (Credit for any sound reason: stronger separation, reuse of the raw reasoning, or isolating the constraint to a trivial reformat.)

---

## 11.5 — Parsing and validation (Pydantic / Zod); graceful retries

### Concept

Even with structured outputs, the model's response is **untrusted input** crossing the
boundary into your typed program. The disciplined pattern is: **define the schema once
as code, generate output against it, then parse-and-validate that output back into a
typed object before any downstream code touches it.** Strict mode (11.3) makes this
boundary far more reliable but does not remove it — semantic errors, business-rule
violations, and provider edge cases still occur.

**Schema-as-code with Pydantic / Zod.** Don't hand-write JSON Schema. Define the data
model in your application's type system and *derive* the JSON Schema from it:
- **Pydantic** (Python) — define a `BaseModel`; `model_json_schema()` produces the JSON
  Schema you send to the API, and `model_validate_json()` parses *and* validates the
  response into a typed object.
- **Zod** (TypeScript) — define a `z.object({...})` schema; helpers convert it to JSON
  Schema, and `.parse()` / `.safeParse()` validate the response into a typed value.

This **single source of truth** eliminates drift between "the schema the model sees" and
"the type your code expects," and gives you static types for free. SDKs increasingly
accept a Pydantic/Zod model directly (e.g., `response_format=MyModel`) and hand back a
parsed instance.

**Validation does what the schema can't (the semantic layer).** This is where you encode
what strict mode cannot (11.3):
- **Constraints:** ranges (`confidence` ∈ [0,1]), string formats (real email, ISO date),
  `minLength`, regex patterns.
- **Cross-field invariants:** `end_date > start_date`, `sum(line_items) == total`.
- **Business rules:** the enum value is *allowed in this context*, referenced IDs exist.
- **Coercion / normalization:** trim whitespace, parse a date string into a `date`,
  normalize casing.
A Pydantic validator or a Zod `.refine()` expresses these. The output then either becomes
a trusted typed object or raises a precise, located error.

**Graceful retries — handle the failure path deliberately.** Validation *will* sometimes
fail (a semantic violation, a provider hiccup, JSON mode without strict). A robust
integration:
1. **Catch** the parse/validation error — never let a raw exception or a half-parsed
   object propagate.
2. **Retry with the error fed back.** The most effective pattern is a *corrective* retry:
   send the model its previous (invalid) output **and the specific validation error**
   ("`confidence` was 1.7; it must be ≤ 1.0") and ask it to fix it. The error message is
   a precise repair instruction — far better than a blind retry. (Libraries like
   `instructor` automate exactly this loop.)
3. **Bound the retries.** Cap attempts (e.g., 2–3). Repeated failure is a real signal —
   the schema may be too complex, the task ill-posed, or the input genuinely
   unanswerable.
4. **Define a terminal fallback.** After exhausting retries: route to a human, return a
   typed error/`null`, downgrade to a simpler schema, or escalate. *Never* ship a guessed
   or partially-valid object downstream.
5. **Log and monitor.** Track the schema-failure rate as a production metric. A rising
   rate flags a model regression, a prompt drift, or a schema problem. Streaming adds a
   wrinkle: partial JSON is unparseable mid-stream, so either buffer to completion before
   validating or use a tolerant partial-JSON parser.

**The full picture:** schema-as-code (Pydantic/Zod) → constrained generation against it →
parse + semantic validation at the boundary → on failure, a bounded corrective-retry loop
→ a defined terminal fallback → monitoring. Structured outputs make the happy path
reliable; this discipline makes the *whole* path reliable.

### Key terms

- **Schema-as-code** — defining the data model in Pydantic/Zod as the single source of truth, deriving JSON Schema from it.
- **Pydantic** — Python library: define `BaseModel`, generate JSON Schema, parse-and-validate responses into typed objects.
- **Zod** — TypeScript library: define schemas, derive JSON Schema, validate responses into typed values.
- **Semantic validation** — checks beyond shape: ranges, formats, cross-field invariants, business rules.
- **Corrective retry** — re-prompting the model with its invalid output and the specific validation error so it can fix it.
- **Terminal fallback** — the defined behavior after retries are exhausted (human handoff, typed error, simpler schema).
- **Schema-failure rate** — the monitored fraction of responses that fail parse/validation.

### Common misconceptions

- ❌ "With Pydantic/Zod and strict mode I don't need to handle failures." → ✅ Semantic violations and provider edge cases still occur; you need a bounded corrective-retry loop and a terminal fallback.
- ❌ "On a validation failure, just retry the same request." → ✅ A blind retry repeats the same mistake; feed the model its invalid output and the specific error so the retry is a targeted repair.
- ❌ "Retry until it succeeds." → ✅ Retries must be bounded; persistent failure is a signal (too-complex schema, ill-posed task, bad input) and needs a terminal fallback, not an infinite loop.

### Worked example

Extracting structured data with Pydantic and a corrective retry (Python-style sketch):

```python
class Invoice(BaseModel):
    invoice_number: str
    total: float = Field(gt=0)            # semantic: must be positive
    currency: Literal["USD", "EUR", "GBP"]
    due_date: date                        # semantic: must parse as a real date

schema = Invoice.model_json_schema()      # single source of truth → sent to the API

def extract(text, attempts=3):
    messages = build_prompt(text, schema)
    last_error = None
    for _ in range(attempts):
        raw = call_model(messages, strict_schema=schema)   # constrained decoding
        try:
            return Invoice.model_validate_json(raw)        # parse + semantic validation
        except ValidationError as e:
            last_error = e
            messages += [
                {"role": "assistant", "content": raw},
                {"role": "user",
                 "content": f"That output was invalid: {e}. Return corrected JSON."},
            ]
    log_schema_failure(text, last_error)
    return None        # terminal fallback — caller routes to human review
```

Strict mode guarantees the shape (`total` is a number, `currency` is in the enum).
Pydantic catches what strict mode can't: `total` ≤ 0, or `due_date` = `"next month"`
failing to parse as a `date`. On failure the model is re-prompted *with the exact error*;
after 3 attempts the system fails closed to human review — never shipping a bad invoice.

### Check questions

1. A team hand-writes the JSON Schema they send to the model and *separately* hand-writes the Pydantic model their code validates against. What specific class of bug does this invite, and how does schema-as-code remove it? — **Answer:** It invites *schema drift*: the schema the model is constrained to and the type the code expects are two artifacts that can silently diverge — someone edits one and forgets the other, so the model produces output that is "valid" against the API schema but fails the code's validator (or vice versa). Schema-as-code makes the Pydantic/Zod model the single source of truth and *derives* the JSON Schema from it, so the two cannot disagree; you also get static types and a parse-and-validate path for free.
2. A pipeline retries failed extractions by re-sending the exact same request. On a genuinely hard input it can retry many times and still fail. Identify both flaws and give the corrected design. — **Answer:** Flaw 1 — blind retry: re-sending the identical request reproduces the same mistake; the retry should be *corrective*, including the model's previous invalid output and the specific validation error as a precise repair instruction. Flaw 2 — unbounded retries: it loops indefinitely, spiking cost/latency; retries must be *bounded* (e.g., 2–3). Corrected design: bounded corrective-retry loop, and treat persistent failure as a signal (too-complex schema, ill-posed task, bad input) that triggers a terminal fallback rather than another attempt.
3. A team's structured-output pipeline, after exhausting retries, falls back to "return the last invalid object anyway so the pipeline doesn't break." Critique this and give the correct fallback options. — **Answer:** This is wrong — shipping a guessed or partially-valid object downstream propagates bad data into systems that trust the typed boundary, often causing a worse, harder-to-trace failure later than an honest stop. The correct terminal fallback is a *defined* one: route to a human, return a typed error or `null` that callers handle explicitly, or downgrade to a simpler schema — and log the failure / monitor the schema-failure rate. "Fail closed" with a typed signal, never "fail open" with bad data.

---

## 11.6 — The refusal vs. strict-schema conflict

### Concept

Here is a production footgun that surprises teams the first time it bites: **what happens
when a model wants to *refuse*, but you have forced it into a strict schema that has no
way to express a refusal?**

A model declines or cannot answer for many legitimate reasons: the request is unsafe or
violates policy, the input is genuinely unanswerable (a blank document, an image with no
text), the request is out of scope, or the model is simply not confident. In *free-text*
mode the model handles this naturally — it writes "I can't help with that" or "the
document does not contain an invoice number." But **constrained decoding does not let the
model write that sentence.** If your schema is `{"invoice_total": number, "currency":
enum[...], "due_date": string}` with all fields required, the token-level FSM (11.2)
*physically prevents* the model from emitting a refusal — the only token paths that exist
lead to a schema-conforming object. The refusal has been **masked out of existence.**

**What the model does instead — and why it's dangerous.** Cornered by the schema, the
model cannot say "no," so it does the only thing the FSM permits: it **fills the required
fields anyway.** It emits a schema-valid object built from a guess, a hallucination, or a
placeholder. This is the **valid-but-useless / valid-but-fabricated** output from 11.3,
but now with a sharper cause: the structure *itself* suppressed the model's own signal
that it should not answer. The output passes schema validation perfectly — so a naive
pipeline treats a refusal-that-couldn't-happen as a successful extraction and ships
fabricated data downstream. A strict schema can thus **convert an honest "I don't know"
into a confident wrong answer.** That is strictly worse than a refusal.

There is also a genuine *tension with safety*. Provider safety training pushes the model
to refuse harmful requests; strict structured output pushes it to always produce the
schema. These two objectives can fight: a tightly-constrained schema can make refusal
harder, and conversely a model determined to refuse may produce a degenerate schema
object. You should not rely on a strict schema to also enforce safety, and you should not
assume strict mode will never interfere with a refusal.

**The fix: design an escape hatch into the schema.** The output contract must include a
*legitimate, in-schema way to express "I will not / cannot answer."* Common patterns:

- **A status / outcome field.** Make the top-level object a discriminated union:
  `{"status": enum["ok","refused","no_answer"], "data": {...} | null,
  "reason": string | null}`. The model can choose `status: "refused"` *within* the
  schema — the refusal is now a first-class, structurally-valid outcome, and downstream
  code branches on `status` instead of trusting `data` blindly.
- **Make answer fields nullable / optional.** If `invoice_total` may be `null`, the model
  has a legal way to say "not present" instead of fabricating a number. (Recall 11.3:
  with strict mode "optional" is usually expressed as a union with `null`, not by
  omission.)
- **A two-step pipeline.** An unconstrained first call decides "can this be answered, and
  is it safe?"; only on "yes" does a second, constrained call produce the structured
  object. Refusal is handled entirely in the unconstrained step where the model *can*
  say no. (This is the same separate-thinking-from-formatting idea as 11.4.)
- **Prompt for the escape hatch explicitly.** Tell the model, in the prompt, exactly how
  to signal a refusal or no-answer *using the schema* ("if the document has no invoice
  data, set `status` to `no_answer`") — constrained decoding guarantees the shape, not
  the choice, so the model still needs to be told the convention.

The principle: **a strict schema must be designed to represent every outcome the model
legitimately needs to produce — including "no."** A schema that can only express success
does not eliminate failure; it just disguises failure as success. Always give the model
a structurally-valid way out, and have your validation layer (11.5) treat a `refused` /
`no_answer` status as a real, expected branch — not an error.

### Key terms

- **Refusal–schema conflict** — the situation where a strict schema has no way to express a refusal or no-answer, so the model is forced to fabricate a conforming object.
- **Escape hatch** — a deliberate in-schema way to represent "cannot/will not answer" (a status field, nullable fields).
- **Status / outcome field** — a top-level enum (`ok` / `refused` / `no_answer`) making refusal a first-class, schema-valid result.
- **Disguised failure** — a fabricated but schema-valid object that a naive pipeline mistakes for a successful answer.

### Common misconceptions

- ❌ "If the model needs to refuse, it just will — strict mode won't stop it." → ✅ Constrained decoding masks every token, so if the schema has no refusal path the FSM physically prevents the refusal; the model is forced to emit a conforming object instead.
- ❌ "A refused request will fail schema validation, so my pipeline will catch it." → ✅ The forced object is fully schema-valid — it passes validation; the pipeline mistakes a disguised refusal for a success unless the schema has an explicit refusal/no-answer outcome.
- ❌ "A strict schema also helps enforce safety." → ✅ Strict output and safety refusal are competing objectives; a schema with no refusal path can actively make refusal harder. Design an in-schema escape hatch; don't rely on the schema for safety.

### Worked example

A content-moderation feature must extract `{"category": enum[...], "severity": integer,
"rationale": string}` from user-submitted text under strict mode.

- **The conflict.** A user submits text that is itself a request to do something harmful.
  The model's safety training says "refuse." But the strict schema's FSM permits only
  paths that produce a `category`/`severity`/`rationale` object — there is no token path
  to "I can't help with that." So the model emits, e.g.,
  `{"category": "other", "severity": 0, "rationale": "unable to assess"}` — a perfectly
  schema-valid object. Validation passes. The moderation pipeline records a clean,
  low-severity result for content it should have flagged for refusal/escalation.
- **The fix.** Redesign the contract as a discriminated union:
  ```
  { "status": enum["assessed","refused","unprocessable"],
    "assessment": { "category": ..., "severity": ..., "rationale": ... } | null }
  ```
  Now the model has a legal, in-schema way to emit `{"status": "refused",
  "assessment": null}`. The prompt explicitly tells it to use `refused` when it would
  decline. Downstream code branches on `status`: `refused` routes to human escalation,
  `unprocessable` to a different path, `assessed` reads `assessment`. The refusal is now
  a first-class outcome instead of a fabricated low-severity score.

### Check questions

1. An extraction service runs under strict mode with a schema whose every field is required and there is no status field. A user submits a request the model would normally refuse on safety grounds. Describe precisely what the model emits and why a naive pipeline mishandles it. — **Answer:** Constrained decoding's FSM only permits token paths that complete the required-field object — there is no token path to "I can't help with that," so the refusal is masked out of existence. Cornered, the model fills the required fields with a guess/placeholder/hallucination and emits a fully *schema-valid* object. A naive pipeline runs schema validation, sees it pass, and treats the disguised refusal as a successful extraction — shipping fabricated data downstream. The strict schema converted an honest refusal into a confident wrong answer.
2. A team's fix for the refusal–schema conflict is "we'll catch it in validation — a refusal will look malformed." Why does this not work, and what is the correct design? — **Answer:** It does not work because the forced object is not malformed — constrained decoding *guarantees* it conforms to the schema, so it passes validation cleanly; there is nothing for the validator to catch. The correct design is to give the model a structurally-valid way to refuse: add a top-level `status` enum (`ok` / `refused` / `no_answer`) or make answer fields nullable, prompt the model on how to use it, and have downstream code branch on `status` as an expected outcome. The schema must be able to *represent* a refusal, not just detect a broken one.
3. Explain why "use a strict schema" and "let the model refuse unsafe requests" can be competing objectives, and give one design that resolves the tension. — **Answer:** Strict structured output pushes the model to *always* produce a schema-conforming object; safety training pushes it to *refuse* certain requests. When the schema has no refusal path, the constrained-decoding masking enforces "always produce the object" and physically blocks the refusal — the two objectives collide and the schema wins, suppressing a safety behavior. A clean resolution is a two-step pipeline: an unconstrained first call decides whether the request can and should be answered (where the model is free to refuse), and only on approval does a second constrained call produce the structured object — so the refusal is handled where the schema does not interfere. (An in-schema `status: "refused"` outcome also resolves it.)

---

## Topic 11 — Exam Question Bank

A pool the tutor draws from for the gated exam. Mixed-format, scored out of 100, pass
mark 85.

### True / False

1. If a response from JSON mode parses successfully with `JSON.parse()`, you can rely on it having your schema's required fields with the correct types. — **Answer:** False. JSON mode guarantees only *syntactically valid JSON* — it parses — but the field names, required fields, and value types can all still be wrong. Only constrained/strict decoding against your schema guarantees that.
2. Constrained decoding makes the model far more likely to produce schema-conforming output, but a non-conforming token can still occasionally slip through. — **Answer:** False. Constrained decoding *masks* every schema-violating token, giving it zero probability — a non-conforming token is impossible to sample, not merely unlikely. Conformance is a guarantee by construction.
3. A response produced under strict structured outputs can be entirely fabricated yet still pass schema validation. — **Answer:** True. Strict mode guarantees the *shape* (types, required fields, enums) — a structural contract — but says nothing about whether the values are correct or truthful; a perfectly schema-valid object can be wholly hallucinated.
4. Forcing a terse output schema degrades accuracy roughly equally across all task types. — **Answer:** False. For simple extraction/classification the effect is roughly neutral; the measurable accuracy drop appears on *hard multi-step reasoning* tasks, where suppressing the chain-of-thought scratchpad matters.
5. Placing a `reasoning` field *after* the `answer` field in the schema still helps, because the model can revise its answer once it has reasoned. — **Answer:** False. Generation is autoregressive and constrained decoding emits fields in schema order; if `answer` comes first the model commits to it before writing `reasoning`, which is then filled too late to help. The reasoning field must come first.
6. Because strict mode plus a Pydantic model removes shape and parsing errors, a corrective-retry loop becomes unnecessary. — **Answer:** False. Strict mode removes the shape/parsing error class, but semantic errors (out-of-range values, cross-field violations, business-rule failures) and provider edge cases still occur; a bounded corrective-retry loop and a terminal fallback are still required.
7. The most effective response to a validation failure is to re-send the identical request and hope for a better sample. — **Answer:** False. That is a blind retry, which tends to reproduce the same mistake. A corrective retry — re-prompting with the invalid output *and* the specific validation error — is far more effective because the error is a precise repair instruction.
8. Tool-use-for-structure is obsolete and should never be used now that native structured outputs exist. — **Answer:** False. It is a *workaround* for a structured final answer, but it remains appropriate inside genuine agentic flows (where the model is choosing among real tools) and on older models/providers without native structured outputs.
9. Constrained decoding validates the JSON after the model finishes generating and retries if it is wrong. — **Answer:** False. It works *during* generation: the schema is compiled to a grammar then a finite-state machine, and at each step illegal tokens are masked (logits set to −∞) before sampling, so an invalid document is never produced. Generate-then-validate-then-retry is a separate, weaker pattern.
10. Constrained decoding adds essentially no latency because masking tokens is trivial. — **Answer:** False. It has a one-time schema-compilation cost (schema → grammar → FSM) and a recurring per-token mask cost inside the decoding loop. Fast engines (XGrammar, llguidance) shrink the steady-state overhead, but it is not zero, and a naive implementation can be slow.
11. The tokenizer-alignment problem exists because grammar boundaries are defined over characters while the model emits multi-character tokens that need not line up with them. — **Answer:** True. The FSM reasons in characters, but the model emits arbitrary multi-character tokens; for every vocabulary token the engine must decide whether the whole token is a legal FSM continuation — reconciling these mismatched boundaries is the core difficulty of constrained decoding.
12. If a model would refuse a request but you have forced it into a strict schema with no refusal path, the refusal will fail schema validation so your pipeline can catch it. — **Answer:** False. The model, unable to emit a refusal, fills the required fields with a guess/placeholder and emits a fully *schema-valid* object; it passes validation. The pipeline mistakes the disguised refusal for a success unless the schema includes an explicit refusal / no-answer outcome.

### Multiple Choice

1. Three teams want schema-shaped output. Team X carefully prompts "respond only with JSON matching this schema." Team Y uses JSON mode. Team Z uses constrained/strict decoding. Rank them by *strength of the structural guarantee*, strongest first.
   A) X, Y, Z  B) Z, Y, X  C) Y, Z, X  D) All three are equivalent
   — **Answer:** B. Strict/constrained decoding (Z) masks illegal tokens so conformance is guaranteed by construction; JSON mode (Y) guarantees only valid JSON, not schema match; prompting (X) guarantees nothing — the model can ignore it. Z > Y > X.
2. A strict schema declares `room_count` as an `integer`, `floor` as an `integer`, and `room_type` with `enum: ["single","double","suite"]`. Which violation will strict mode reliably *prevent*?
   A) `room_count` being larger than `floor` when business rules forbid it  B) `room_type` being `"penthouse"`  C) `room_count` being a wildly wrong number for the actual room  D) `room_count` and `floor` being internally inconsistent with a separate `address` field
   — **Answer:** B. `"penthouse"` is outside the declared enum, and strict mode makes an out-of-enum value impossible to emit. A, C, and D are semantic/cross-field/correctness concerns strict mode cannot enforce.
3. A classification feature emits `{"label": "...", "confidence": number}` under strict mode and works well. The team adds a feature that solves multi-step logic puzzles with the schema `{"answer": string}` and accuracy drops. The most likely cause is:
   A) String fields are unsupported in strict mode  B) The puzzle schema forces the answer as the first content token, suppressing the chain-of-thought the harder task needs  C) Strict mode is slower for puzzles  D) The classification schema was cached and interfered
   — **Answer:** B. The model reasons by generating tokens; an immediate-answer schema amputates the scratchpad. Classification needs little multi-step reasoning so it is unaffected; the logic puzzle depends on it, so accuracy falls.
4. You must keep strict structured output but the task needs multi-step reasoning. Which option preserves *both*?
   A) Switch to JSON mode so the model has freedom  B) Add a `reasoning` string field before the answer fields, or use an unconstrained reasoning call followed by a constrained reformat call  C) Raise temperature so the model explores more  D) Remove `additionalProperties: false` so the model can add notes
   — **Answer:** B. Separate thinking from formatting: a leading reasoning field lets the model reason inside the strict schema, or a two-step pipeline keeps the hard reasoning unconstrained and reformats afterward. A abandons the guarantee; C and D do not restore the scratchpad.
5. A team hand-maintains the JSON Schema sent to the API and a separate Pydantic model for validation. Adopting schema-as-code (deriving the JSON Schema *from* the Pydantic model) primarily eliminates:
   A) The need for constrained decoding  B) Schema drift between what the model is constrained to and what the code validates against  C) All semantic validation errors  D) The need to handle retries
   — **Answer:** B. One source of truth means the model's schema and the code's validator cannot silently diverge. It does not replace constrained decoding, semantic validation, or retry handling.
6. After a validation failure, which retry design is both effective and safe?
   A) Re-send the identical request, unbounded, until it succeeds  B) Re-prompt with the invalid output and the specific error, bounded to a few attempts, with a terminal fallback  C) Immediately return the last invalid object so the pipeline continues  D) Lower `max_tokens` and retry indefinitely
   — **Answer:** B. Corrective (the error is a repair instruction), bounded (persistent failure is a signal, not something to loop on), and fail-closed via a terminal fallback. A loops forever; C ships bad data; D neither corrects nor bounds.
7. Which statement about provider strict modes is accurate?
   A) They implement the full JSON Schema specification identically across providers  B) They support only a subset of JSON Schema, and the exact subset and limits (e.g., required-properties rules, depth/size caps, unsupported keywords) are provider-specific  C) They guarantee that string fields contain semantically valid content  D) They make a downstream validation layer unnecessary
   — **Answer:** B. Providers honor a limited, provider-specific subset; a schema valid in the spec may be rejected or partially honored, so you must check the provider's docs.
8. An extraction pipeline under strict mode produces objects that always pass schema validation, but a downstream report is full of `"unknown"` and empty strings. This is best described as:
   A) A constrained-decoding bug  B) Valid-but-useless output — schema-conforming objects whose values are placeholders the model emitted because it didn't actually know  C) A streaming/partial-JSON problem  D) A schema-drift problem
   — **Answer:** B. Strict mode forces a value of the right type even when the model has no real answer, so it fills required fields with placeholders. Schema validity is not task success; the fix is semantic validation plus prompt/abstention design.
9. Put the stages of constrained decoding in the correct order:
   A) JSON Schema → token mask → grammar → FSM  B) JSON Schema → grammar → finite-state machine → per-step token masking  C) FSM → grammar → JSON Schema → sampling  D) Grammar → JSON Schema → token mask → FSM
   — **Answer:** B. A declarative JSON Schema is compiled into a formal grammar (BNF/EBNF/GBNF), the grammar is compiled into a finite-state machine that defines legal next tokens per state, and at each decoding step illegal tokens are masked before sampling.
10. Why does a provider's strict mode support only a *subset* of JSON Schema (e.g., restricting unbounded recursion and arbitrary regex patterns)?
   A) To save the customer money  B) Because the schema must be compilable into a grammar/FSM the constraint engine can execute efficiently, and some features are hard or expensive to turn into a fast automaton  C) Because the JSON spec forbids those features  D) Because larger schemas always hallucinate
   — **Answer:** B. Constrained decoding runs by compiling the schema to a grammar then an FSM; features like unbounded recursion or arbitrary `pattern` regexes are hard or costly to compile into a fast automaton, so providers restrict the supported subset.
11. A constrained-decoding engine cannot simply allow or forbid the next *character* because:
   A) Characters are not in the model's vocabulary at all  B) The model emits multi-character tokens that don't align with grammar boundaries, so the engine must decide per-token whether the whole token is a legal FSM continuation  C) JSON has no character-level structure  D) The FSM only tracks whole words
   — **Answer:** B. This is the tokenizer-alignment problem: the grammar/FSM reasons in characters but generation is token-by-token, and tokens span grammar boundaries; the engine must evaluate every vocabulary token against the current FSM state.
12. A fraud-extraction service runs a strict schema with all fields required and no status field. On a request the model would refuse, it emits a schema-valid object with placeholder values, which the pipeline records as a clean result. The root cause is best described as:
   A) A validation-layer bug  B) The refusal–schema conflict — the schema has no in-schema way to express a refusal, so the FSM forces a conforming object and the refusal is disguised as a success  C) Schema drift  D) Reasoning degradation from a terse schema
   — **Answer:** B. Constrained decoding masks every token, so with no refusal path the model cannot decline and is forced to fabricate a conforming object. The fix is an in-schema escape hatch (a `status` enum or nullable fields).

### Short Answer

1. A team's outputs "always parse as JSON" but downstream code still breaks on missing and mistyped fields. Which mechanism are they using, which should they switch to, and what exactly will the switch fix? — **Model answer:** They are using JSON mode, which guarantees only syntactically valid JSON — wrong field names, missing required fields, and wrong types are exactly what it leaves open. They should switch to constrained/strict decoding against their schema, which masks schema-illegal tokens at sampling time so the output conforms to the schema's *structure* (types, required fields, enums). The switch eliminates the missing/mistyped-field class of bug; it does *not* fix semantic errors.
2. A schema constrains an `age` field to be an `integer`. Give one error this prevents and one error of the same field it cannot prevent, and name the contract each belongs to. — **Model answer:** Prevents: `age` being a string like `"forty"` or a float — that is the *structural contract* (correct type). Cannot prevent: `age` being `-5` or `900`, or simply the *wrong* age for the person — those are *semantic contract* concerns (range/correctness) that the type declaration does not express. Strict mode polices the structural envelope, not the meaning of values.
3. Two teams add a `reasoning` field to a strict math schema; one sees an accuracy gain, the other sees none. Their schemas are otherwise identical. What is the most likely difference, and what mechanism explains it? — **Model answer:** The most likely difference is *field order*: the team that gained put `reasoning` before the answer field; the team that didn't put it after. Generation is autoregressive and constrained decoding emits fields in schema order, so a `reasoning` field only restores the scratchpad if it is generated *before* the answer — placed after, the model has already committed to the answer and the reasoning field is filled too late to help.
4. Give two things a Pydantic/Zod validator can enforce that a strict JSON schema generally cannot. — **Model answer:** Any two of: numeric ranges (e.g., `confidence` ∈ [0,1]), cross-field invariants (`end > start`, `sum(items) == total`), real string formats (deliverable email, parseable date), context-dependent business rules, normalization/coercion.
5. What should a robust integration do after exhausting its retry budget? — **Model answer:** Execute a defined terminal fallback — route to a human, return a typed error/`null`, or downgrade to a simpler schema — log the failure, and never ship a guessed or partially-valid object downstream.
6. Why is structured output complicated by streaming? — **Model answer:** A partial JSON fragment received mid-stream is not parseable or schema-validatable until complete; you must either buffer the full response before validating or use a tolerant partial-JSON parser.
7. Why is tool-use-for-structure considered a workaround rather than the preferred approach? — **Model answer:** It repurposes the tool/function-calling feature — designed for the model to request actions — to do output formatting; it can be more brittle and harder to reason about than a dedicated native structured-output mode, which is now preferred for a structured final answer.
8. Describe the three-stage pipeline by which a JSON Schema becomes a hard constraint on generation. — **Model answer:** (1) Schema → grammar: the declarative JSON Schema is compiled into a formal generative grammar (BNF/EBNF; GBNF in llama.cpp) describing exactly which strings are valid. (2) Grammar → finite-state machine: the grammar is compiled into a state machine (a pushdown automaton for JSON's nesting) where each state defines the set of legal next tokens given what has been generated. (3) Per-step token masking: at each decoding step the engine sets the logits of all FSM-illegal tokens to −∞ and samples only from the legal set, so a non-conforming document can never be produced.
9. What is the tokenizer-alignment problem in constrained decoding, and why does it make a naive implementation slow? — **Model answer:** The grammar/FSM reasons over characters, but the model generates multi-character *tokens* that do not align with grammar boundaries (a token may span grammar elements). The engine therefore cannot check one character; for every token in the ~100k+ vocabulary it must determine whether the whole token is a legal continuation from the current FSM state, possibly advancing the FSM through several states. Doing this from scratch at every decoding step is what makes a naive implementation slow; fast engines (XGrammar, llguidance) precompute the token↔FSM mapping so the per-step cost becomes a cheap lookup.
10. Name the two latency costs of grammar-constrained decoding and when each is paid. — **Model answer:** (1) Compilation cost — the one-time work of turning a schema into a grammar and then an FSM; paid the first time a schema is used (then cacheable), so a brand-new schema can stall a latency-sensitive first request. (2) Per-token mask cost — the recurring work of computing the allowed-token mask at every decoding step inside the hot loop; fast engines minimize it but it is never exactly zero.
11. Explain the refusal–strict-schema conflict and give two designs that resolve it. — **Model answer:** When a model would refuse or cannot answer (unsafe request, unanswerable input, out of scope) but is forced into a strict schema with no refusal path, constrained decoding's token masking physically prevents it from emitting a refusal — so it fabricates a schema-valid object from a guess/placeholder, which passes validation and is mistaken for a success. Resolutions (any two): add a top-level `status` enum (`ok` / `refused` / `no_answer`) making refusal a first-class in-schema outcome; make answer fields nullable so "not present" is legal; use a two-step pipeline where an unconstrained call decides answerability/safety first and only then a constrained call structures the result; and prompt the model explicitly on how to signal a refusal using the schema.

### Long Answer

1. Compare JSON mode, constrained decoding, and tool-use-for-structure: how each works, what it guarantees, and when to use it. — **Model answer / rubric:** JSON mode: a flag biasing the decoder to valid JSON; guarantees parseable JSON only, not schema conformance; use for loose JSON you'll validate yourself or as a fallback. Constrained/guided decoding (strict mode): masks schema-illegal tokens at sampling time so output conforms to the schema's structure by construction; strongest guarantee; preferred for a structured final answer. Tool-use-for-structure: define a tool whose argument schema is the desired structure and force the call; the tool arguments are the structured object; a historical workaround, still useful in genuine agentic flows or on providers without native structured outputs. Credit for correct mechanism and guarantee for each, and a sensible selection rule.
2. Explain precisely what strict schema mode guarantees and what it does not, and what follows for system design. — **Model answer / rubric:** Guarantees (structural contract): valid JSON, all required fields present, correct value types, enum values within the allowed set, no extra fields with `additionalProperties:false`, correct nesting. Does not guarantee (semantic): value correctness, cross-field invariants, arbitrary string content/format, the *right* enum choice, truthfulness/no hallucination; also providers support only a subset of JSON Schema. Design consequence: strict mode eliminates the parsing/shape error class but a semantic validation layer (ranges, invariants, business rules) is still required, plus a corrective-retry loop and terminal fallback; treat model output as untrusted input at a typed boundary. Credit for the structural-vs-semantic distinction, concrete examples, and the design implication.
3. Explain how structured output can degrade reasoning and give a full set of workarounds. — **Model answer / rubric:** Cause: LLMs reason by generating tokens; a terse schema forces the answer as the first content token, suppressing chain-of-thought; constrained decoding can also push the model off its natural distribution and the format becomes a competing objective. Effect: roughly neutral for simple extraction/classification, but a measurable accuracy drop on hard reasoning if naively constrained. Workarounds: a `reasoning`/scratchpad field placed *before* the answer fields; a two-step pipeline (unconstrained reasoning call, then a constrained reformat call); using the provider's extended-thinking channel alongside structured output; loosening the schema where free-text is acceptable. Principle: separate thinking from formatting — never force a hard answer as the first token. Credit for the mechanism, the nuanced empirical picture, and multiple correct workarounds.
4. Describe a robust end-to-end structured-output integration, from schema definition to failure handling and monitoring. — **Model answer / rubric:** Define the data model in Pydantic/Zod as the single source of truth; derive the JSON Schema from it and send it to the model with constrained/strict decoding. Parse the response and run semantic validation at the boundary (ranges, formats, cross-field invariants, business rules) into a typed object. On validation failure, run a bounded corrective-retry loop — re-prompt with the invalid output and the specific error — capped at a few attempts. After exhausting retries, execute a terminal fallback (human handoff, typed error, simpler schema) and never ship a bad object. Monitor the schema-failure rate as a production metric; handle streaming by buffering or partial parsing. Credit for schema-as-code, the parse+semantic-validation boundary, bounded corrective retries, the terminal fallback, and monitoring.
5. Explain in full how constrained decoding turns a JSON Schema into a hard guarantee, including the tokenizer-alignment problem and the latency cost. — **Model answer / rubric:** Mechanism: the declarative JSON Schema is compiled into a formal grammar (BNF/EBNF; GBNF for llama.cpp); the grammar is compiled into a finite-state machine — a pushdown automaton, since JSON nesting needs a stack — where each state defines the legal next tokens given what has been generated. At every decoding step the engine intersects the FSM's legal set with the vocabulary, masks all illegal tokens' logits to −∞, and samples; the FSM then advances. An illegal token has zero probability, so conformance holds by construction. Tokenizer-alignment problem: the FSM reasons in characters but the model emits multi-character tokens that do not align with grammar boundaries, so for every vocabulary token the engine must decide whether the whole token is a legal continuation (possibly advancing the FSM through several states) — the core difficulty, and what makes a naive implementation slow. Latency: a one-time schema-compilation cost (paid on first use, cacheable) and a recurring per-token mask cost in the decoding loop; engines like Outlines, XGrammar, and llguidance precompute the token↔FSM mapping to minimize the per-step overhead, but it is not zero. Also explains why providers support only a JSON-Schema subset: a feature must be compilable into an efficiently-executable grammar/FSM. Credit for the three-stage pipeline, the alignment problem, the two latency costs, named systems, and the schema-subset link.
6. Explain the refusal vs. strict-schema conflict: why it happens, why it is dangerous, and how to design around it. — **Model answer / rubric:** Why it happens: a model legitimately needs to refuse or abstain (unsafe/out-of-policy request, unanswerable or empty input, out of scope, low confidence); but constrained decoding masks every token, so if the strict schema has no refusal path the FSM physically prevents the model from emitting a refusal — the only token paths lead to a conforming object. Why dangerous: cornered, the model fills the required fields with a guess/placeholder/hallucination and emits a fully schema-valid object that passes validation, so a naive pipeline mistakes a disguised refusal for a success and ships fabricated data — a strict schema converts an honest "I don't know" into a confident wrong answer; it also competes with safety training. How to design around it: build an in-schema escape hatch — a top-level `status` enum (`ok` / `refused` / `no_answer`) as a discriminated union, and/or nullable answer fields; prompt the model explicitly on how to signal refusal using the schema; or use a two-step pipeline where an unconstrained call handles answerability/safety first; and have the validation layer treat a refusal status as an expected branch, not an error. Principle: a strict schema must be able to represent *every* legitimate outcome including "no." Credit for the masking mechanism, disguised-failure danger, the safety tension, and concrete escape-hatch designs.

### Applied Scenario

1. Your extraction service uses JSON mode and validates with Pydantic. About 3% of responses parse fine but fail Pydantic validation with missing or mistyped fields, breaking downstream code. What is happening and what do you change? — **Model answer / rubric:** JSON mode guarantees only valid JSON, not schema conformance, so the model returns parseable JSON with wrong/missing fields — exactly the 3%. Switch to the provider's strict/constrained structured-output mode against the Pydantic-derived JSON Schema; that masks schema-illegal tokens and eliminates the missing/mistyped-field class. Keep Pydantic for *semantic* validation (ranges, invariants) since strict mode doesn't cover that, and add a bounded corrective-retry loop with a terminal fallback for the residual semantic failures. Credit for diagnosing JSON-mode's weak guarantee and prescribing strict mode plus a retained semantic layer.
2. A fraud-review feature must output `{"decision": enum["approve","reject","escalate"], "risk_factors": string[]}` under strict mode. It works on clear-cut cases but on ambiguous transactions it makes obviously poor decisions, even though the JSON is always valid. The schema lists `decision` first. Diagnose the likely cause and give two distinct fixes that keep the output strict JSON. — **Model answer / rubric:** Cause: ambiguous transactions require multi-step weighing of evidence, but with `decision` first the model must commit to the verdict as its first content token — chain-of-thought is suppressed, so judgment on hard cases degrades; clear-cut cases need little reasoning so they are unaffected. Fixes (any two, output stays strict JSON): (a) reorder the schema so a `reasoning` (or `analysis`) string field comes *before* `decision`, letting the model weigh evidence inside the schema; (b) two-step pipeline — an unconstrained call reasons about the transaction freely, then a constrained call reformats into the schema; (c) use the provider's extended-thinking channel alongside structured output so the thinking is unconstrained. Credit for diagnosing scratchpad suppression on hard-but-not-easy cases and prescribing separate-thinking-from-formatting fixes with correct ordering.
3. A structured-output pipeline catches validation errors and re-sends the *same* request up to 10 times; some inputs fail all 10 and the pipeline then forwards the last (invalid) object downstream so it "doesn't break." Identify every flaw and give the corrected end-to-end failure design. — **Model answer / rubric:** Three flaws: (1) blind retry — re-sending the identical request reproduces the same mistake; make it *corrective* by including the previous invalid output and the specific validation error as a repair instruction. (2) Too-loose bound — 10 attempts spikes cost/latency and ignores that repeated failure is a *signal* (too-complex schema, ill-posed task, bad input); bound to ~2–3. (3) Fail-open — forwarding an invalid object propagates bad data into a typed boundary that trusts it, causing a worse downstream failure; instead fail *closed* with a defined terminal fallback (human handoff, typed error/`null`, or a simpler schema). Also: log and monitor the schema-failure rate so a rising rate flags a regression. Credit for catching all three flaws and the fail-closed terminal fallback.
4. A team wants every model response across their product to be strict JSON — including an open-ended "explain this concept to me" tutoring response. Evaluate this decision. — **Model answer / rubric:** Pushing strict JSON onto genuinely free-form, open-ended generation is a poor fit: the value of strict output is machine-parseability for downstream code, which an explanatory tutoring answer doesn't need; over-constraining can degrade the quality and naturalness of the explanation; and the explanation is ultimately just a string. Better: use strict structured output where the output feeds code (extraction, classification, tool arguments, routing decisions) and use free text (or a loose schema with a large free-text field) for human-facing prose. If a wrapper envelope is wanted, keep the explanation a free `string` field and only constrain metadata. Apply the constraint where it earns its cost. Credit for matching the mechanism to the use case and noting the over-constraining downside.
5. A team adds strict structured outputs and finds malformed-JSON errors disappear entirely — but then asks "why are we still seeing occasional bad data downstream, and why did one complex schema get rejected by the provider outright?" Explain both observations and what they reveal about strict mode's boundaries. — **Model answer / rubric:** Observation 1 (bad data despite strict mode): strict mode is a *structural* contract — it guarantees valid JSON, types, required fields, and enum membership, but not *semantics* — wrong-but-type-valid values, out-of-range numbers, cross-field invariant violations, valid-but-useless placeholder values, and hallucinated content all still pass. That residual bad data is exactly what a semantic validation layer (Pydantic/Zod refinements: ranges, formats, cross-field rules) must catch. Observation 2 (schema rejected): providers support only a *subset* of JSON Schema with provider-specific rules and limits (e.g., all-properties-required, `additionalProperties:false`, nesting-depth and size caps, unsupported keywords); a schema valid in the full spec can be rejected or partially honored — the underlying reason is that the schema must be compilable into a grammar/FSM the provider's constraint engine can execute efficiently. Together they show strict mode removes the parsing/shape error class but is bounded both *above* (no semantics) and *below* (only a schema subset) — design must add semantic validation and respect the provider's supported subset. Credit for explaining the structural-vs-semantic boundary and the schema-subset limitation, and tying each to a concrete design response.
6. An invoice-extraction service runs strict structured outputs with a schema where `invoice_number`, `total`, `currency`, and `due_date` are all required. It works well on real invoices, but when users accidentally upload a blank page or a non-invoice document, the service still returns a confident, schema-valid invoice object full of plausible-looking but fabricated values — and downstream accounting ingests them. Diagnose the root cause and redesign the output contract. — **Model answer / rubric:** Root cause: the refusal–schema conflict. A non-invoice/blank input is genuinely unanswerable, so the model should abstain — but constrained decoding masks every token, and the all-required schema has no path to "this is not an invoice," so the FSM forces the model to emit a conforming object built from hallucinated values. It passes schema validation, so the pipeline treats a disguised no-answer as a success. Redesign: make the output a discriminated union with a top-level `status` enum (`extracted` / `not_an_invoice` / `unreadable`) and an `invoice` field that is the data object or `null`; the model now has a legal in-schema way to report "not an invoice." Prompt it explicitly on when to use each status. Downstream code branches on `status` — only `extracted` is ingested, others route to review. Optionally also a two-step pipeline (an unconstrained "is this an invoice?" classification first). Add semantic validation (e.g., reject placeholder/empty values) as a backstop. Credit for diagnosing the refusal–schema conflict specifically, prescribing an in-schema escape hatch (status union / nullable data), and branching downstream on the status.

---

## Sources

[1] OpenAI — Structured model outputs / JSON mode (API guide) — https://platform.openai.com/docs/guides/structured-outputs
[2] OpenAI — Introducing Structured Outputs in the API — https://openai.com/index/introducing-structured-outputs-in-the-api/
[3] Anthropic — Structured outputs (Claude API Docs) — https://platform.claude.com/docs/en/build-with-claude/structured-outputs
[4] OpenAI Developer Community — Measuring Maximum Depth and Object Properties in Structured Outputs — https://community.openai.com/t/measuring-maximum-depth-and-object-properties-in-structured-outputs/918388
[5] Tam et al. — Let Me Speak Freely? A Study on the Impact of Format Restrictions on Performance of Large Language Models (arXiv:2408.02442) — https://arxiv.org/abs/2408.02442
[6] Willard & Louf — Efficient Guided Generation for Large Language Models (the Outlines FSM-indexing approach) (arXiv:2307.09702) — https://arxiv.org/abs/2307.09702
[7] Dong et al. — XGrammar: Flexible and Efficient Structured Generation Engine for Large Language Models (arXiv:2411.15100) — https://arxiv.org/abs/2411.15100
