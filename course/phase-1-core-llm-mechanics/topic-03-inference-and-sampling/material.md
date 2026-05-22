# Topic 03 — Inference & Sampling — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This file is taught sub-chapter by sub-chapter, in order. Each sub-chapter has core
teaching content, key terms, common misconceptions, a worked example, and check
questions to confirm understanding before moving on. A full exam question bank sits at
the end; the tutor draws from it for the gated topic exam. Topic 3 builds directly on
Topic 1.6: the model emits **logits**, softmax turns them into a **probability
distribution** over the vocabulary, and then a **token is selected** from that
distribution. Every sampling parameter here is a knob on that selection step — the
model's weights and computation never change.

---

## 3.1 — Temperature and its effect on the logit distribution

### Concept

**Temperature** is the most important sampling parameter and the one most often
misunderstood. It controls *how the probability distribution is shaped before a token is
drawn from it* — it does not make the model "more creative" or "smarter" in any literal
sense; it reshapes a distribution.

Recall the pipeline from 1.6: the model produces logits, and softmax converts them into
probabilities. Temperature inserts one step: **every logit is divided by the
temperature value T before softmax is applied.**

- **T = 1** — logits pass through unchanged; softmax produces the model's "native"
  distribution.
- **T < 1 (e.g., 0.2)** — dividing by a small number *magnifies* the gaps between
  logits. After softmax the distribution becomes **sharper / peakier**: the
  highest-probability tokens get even more probability, the tail gets squashed toward
  zero. Output is more focused, deterministic, repetitive.
- **T → 0** — the distribution collapses onto the single highest-logit token. This is
  effectively **greedy decoding** (3.3): always pick the top token. Most APIs treat
  `temperature=0` as "greedy."
- **T > 1 (e.g., 1.5)** — dividing by a large number *compresses* the gaps between
  logits. After softmax the distribution becomes **flatter**: lower-probability tokens
  become more competitive. Output is more varied, surprising, and — pushed far enough —
  incoherent, because genuinely unlikely tokens start getting selected.

The mental model: temperature is a *contrast knob* on the probability distribution. Low
temperature = high contrast (the favorite dominates). High temperature = low contrast
(everything is closer to equally likely). At the extreme T→∞ the distribution approaches
uniform — the model picks tokens almost at random.

Crucially, **temperature changes only the selection step, not the model.** The logits
are exactly the same; the model "believes" exactly the same thing about what is likely.
Temperature only changes how boldly you draw from that belief. It cannot add knowledge
or correct a wrong ranking — if the right token has a low logit, no temperature setting
reliably surfaces it.

**Practical guidance.** Use low temperature (0–0.3) for tasks with a correct answer:
classification, extraction, structured output, code, factual Q&A, anything you evaluate
against ground truth or want reproducible. Use moderate temperature (0.7–1.0) for
open-ended generation where variety is desirable: brainstorming, creative writing, draft
copy. Temperatures much above ~1.2–1.5 are rarely useful in production — coherence
degrades fast. Note that **providers cap temperature at different maxima and the scale
is not standardized**: Anthropic's Messages API accepts `temperature` in the range
0.0–1.0 [1], while OpenAI's API accepts 0.0–2.0 [2]. A value of 1.0 is the *maximum* on
Anthropic's API but the *midpoint* on OpenAI's, so the same number does not denote the
same behavior across providers — always check the specific API's documented range.

The full sampling-knob landscape across the two major providers (a snapshot — confirm
against current docs; the knobs for 3.2 and 3.4 are previewed here):

| Knob | Anthropic Messages API | OpenAI API |
|---|---|---|
| `temperature` range | 0.0–1.0 (1.0 is the max) | 0.0–2.0 (1.0 is the midpoint) |
| `top_p` | Exposed; advanced-use; cannot set with `temperature` in the same request | Exposed; may set alongside `temperature` (but altering one is advised) |
| `top_k` | Exposed; advanced-use | Not exposed |
| Frequency / presence penalties | Not exposed | Both exposed, range −2.0 to 2.0 |
| `seed` | Not exposed | Exposed (best-effort determinism; `system_fingerprint` reports backend) |

### Key terms

- **Temperature (T)** — a scalar that divides every logit before softmax, controlling
  how sharp or flat the resulting probability distribution is.
- **Sharp / peaked distribution** — low-temperature result; the top token(s) dominate;
  focused, repetitive output.
- **Flat distribution** — high-temperature result; lower-probability tokens become
  competitive; varied, eventually incoherent output.
- **Greedy decoding** — equivalent to T→0: always select the single highest-logit
  token (see 3.3).

### Common misconceptions

- ❌ Temperature changes the model's weights, knowledge, or computation → ✅ It only
  rescales logits before softmax; the model and its logits are unchanged.
- ❌ Higher temperature makes the model "smarter" or "more creative" in a real sense → ✅
  It flattens the distribution, increasing variety and randomness; far enough, it just
  makes output incoherent.
- ❌ Temperature 0 guarantees perfectly identical output every time → ✅ It makes
  selection greedy, but other non-determinism sources remain (see 3.7).
- ❌ A high temperature can surface a correct answer the model ranked low → ✅
  Temperature reshapes the existing ranking; it cannot reliably fix a wrong ranking or
  add knowledge.

### Worked example

A step has logits `cat` 5.0, `dog` 3.0, `bird` 1.0. At **T = 1**, softmax gives roughly
`cat` 0.84, `dog` 0.11, `bird` 0.02 (plus a tiny tail). At **T = 0.5**, every logit
doubles before softmax (10.0, 6.0, 2.0) → roughly `cat` 0.98, `dog` 0.02, `bird` ~0 —
much sharper, `cat` nearly certain. At **T = 2.0**, every logit halves (2.5, 1.5, 0.5) →
roughly `cat` 0.63, `dog` 0.23, `bird` 0.09 — much flatter, `dog` and `bird` now have
real chances.

The same three tokens, three temperatures — watch the distribution go from sharp to flat:

```
 T = 0.5  (sharp — high contrast)
   cat   │███████████████████████████████████████████████ 0.98
   dog   │█                                               0.02
   bird  │                                                ~0

 T = 1.0  (native distribution)
   cat   │██████████████████████████████████████████      0.84
   dog   │█████                                           0.11
   bird  │█                                               0.02

 T = 2.0  (flat — low contrast)
   cat   │███████████████████████████████                 0.63
   dog   │███████████                                     0.23
   bird  │████                                            0.09
```

Same three logits, three different distributions — and three different
behaviors when you sample.

### Check questions

1. The material calls temperature a "contrast knob." Trace *where in the pipeline* it
   acts and explain why "contrast" is an apt metaphor for what it does to the
   distribution at T < 1 versus T > 1. — **Answer:** Temperature acts on the *logits*,
   before softmax: every logit is divided by T. At T < 1, dividing by a small number
   magnifies the gaps between logits, so after softmax the top token(s) dominate even
   more — "high contrast." At T > 1, dividing by a large number compresses the gaps, so
   the distribution flattens and lower-probability tokens become competitive — "low
   contrast." Like a contrast control, it does not add or remove anything; it only
   sharpens or flattens differences that are already there.
2. A team gets a confidently wrong factual answer at temperature 0.7 and a teammate
   says "lower the temperature — that'll make it more accurate." Will it? Explain what
   temperature can and cannot do about a wrong answer. — **Answer:** Lowering temperature
   will not fix it. Temperature only reshapes the *existing* logit ranking — it makes the
   model's current favorite more dominant. If the model already ranks the wrong token
   highest, low temperature makes it *more* likely to emit that wrong token, not less.
   Temperature cannot add knowledge or re-rank tokens; if the correct answer has a low
   logit, no temperature setting reliably surfaces it. The fix lies elsewhere (better
   context, a tool, a different model) — not the temperature knob.
3. You move a working classification prompt from temperature 0 to temperature 1.0 "to
   make it less robotic." Predict what happens to (a) accuracy on a fixed labeled test
   set and (b) run-to-run reproducibility, and explain why neither change is desirable
   *for this task*. — **Answer:** (a) Accuracy will likely drop or become noisy: at
   T = 1.0 the model samples from the flattened distribution, so on borderline cases it
   sometimes emits a non-top (wrong) label. (b) Reproducibility collapses: the same input
   can yield different labels across runs. For a task with a correct answer you *want*
   the top-ranked token chosen consistently; variety is a liability, not a feature, so
   low temperature (0–0.3) or greedy is correct here.

---

## 3.2 — top-p (nucleus) and top-k sampling

### Concept

Temperature reshapes the *whole* distribution but never removes any token — even a
wildly improbable token retains a tiny chance, and at high temperature those tokens get
selected, producing nonsense. **Truncation sampling** methods fix this by *cutting off
the tail* of the distribution before drawing a token. The two standard ones are top-k
and top-p, and they are usually applied together with (and after) temperature.

**top-k sampling.** Keep only the **k highest-probability tokens**, discard the rest,
renormalize the kept probabilities to sum to 1, and sample from those. With `k = 40`,
only the 40 most likely tokens are ever eligible at each step. This guarantees the model
never picks a genuinely absurd token. Its weakness is that **k is fixed regardless of
the distribution's shape.** When the model is very confident (one token at 0.95), k=40
still admits 39 junk tokens; when the model is genuinely uncertain (probability spread
thinly over hundreds of tokens), k=40 may chop off many reasonable continuations.

**top-p (nucleus) sampling.** Instead of a fixed *count*, keep the smallest set of
top tokens whose **cumulative probability** reaches the threshold p — that set is the
"nucleus." With `p = 0.9`, sort tokens by probability and keep adding them until their
probabilities sum to 0.9, discard the rest, renormalize, and sample. The number of
tokens kept is **dynamic and adapts to the model's confidence**: when the model is
confident, the nucleus might be just 1–3 tokens; when it is uncertain, the nucleus might
be dozens. This adaptivity is why top-p is generally preferred over top-k and is the
default truncation method on most APIs.

**How they compose.** A typical sampling pipeline at each step: (1) compute logits, (2)
apply temperature, (3) softmax to probabilities, (4) apply top-k and/or top-p to
truncate the tail and renormalize, (5) sample one token from what remains. Temperature
controls *contrast*; top-k/top-p control *how much tail is allowed*. They are
complementary, not alternatives — though many production setups simply pick a
temperature and a single top-p value.

**Practical guidance.** For deterministic/correct-answer tasks you usually do not need
truncation at all — low temperature already concentrates mass on the top tokens (and
greedy ignores the tail entirely). For open-ended generation, a common starting point is
something like temperature ~0.7–1.0 with top-p ~0.9–0.95 (on providers whose top-p
scale follows this convention). The available knobs and how they interact are
**provider-specific**:

- **Anthropic's Messages API** exposes `temperature`, `top_p`, and `top_k`, but
  documents `top_p` and `top_k` as advanced-use parameters and recommends most callers
  use only `temperature`. It also requires that you set *either* `temperature` *or*
  `top_p` but **not both** in the same request — supplying both returns an error [1].
- **OpenAI's API** exposes both `top_p` and `temperature` and does not error if you set
  both, though OpenAI likewise recommends altering only one of them [2].

As with temperature, the exact numeric scale and behavior are not standardized across
providers, so a top-p value tuned on one API is not guaranteed to behave identically on
another.

### Key terms

- **Truncation sampling** — restricting selection to a subset of high-probability tokens
  before drawing, to exclude the unreliable tail.
- **top-k sampling** — keep the k highest-probability tokens, renormalize, sample; k is
  a fixed count.
- **top-p / nucleus sampling** — keep the smallest set of top tokens whose cumulative
  probability ≥ p, renormalize, sample; the set size adapts to the model's confidence.
- **Nucleus** — the kept set of tokens in top-p sampling.
- **Renormalization** — rescaling the kept tokens' probabilities to sum to 1 after the
  tail is discarded.

### Common misconceptions

- ❌ top-k and top-p are alternatives to temperature → ✅ They are complementary:
  temperature reshapes the distribution; top-k/top-p truncate its tail. They are usually
  applied together.
- ❌ top-p keeps a fixed number of tokens → ✅ top-p keeps a *variable* number — however
  many top tokens it takes to reach cumulative probability p.
- ❌ top-k adapts to how confident the model is → ✅ top-k is a fixed count; it does not
  adapt. top-p is the adaptive one.
- ❌ Truncation sampling changes the relative ranking of tokens → ✅ It only discards the
  tail and renormalizes; the surviving tokens keep their relative order.

### Worked example

The full sampling pipeline at each generation step, with the knob that controls each
stage labeled underneath:

```
  logits ──►[ ÷ T ]──► scaled ──►[ softmax ]──► probs ──►[ top-k / top-p ]──► nucleus ──►[ draw ]──► token
   raw       │          logits        │         full         │            truncated      │
   scores    │                        │         dist.        │            dist.          │
             ▼                        ▼                       ▼                           ▼
        temperature              (no knob —              k  or  p                  (random sample,
        knob (3.1)                fixed function)        knobs (3.2)                or greedy = argmax)
```

At one step, after temperature and softmax, the distribution is `the` 0.50, `a` 0.25,
`an` 0.13, `this` 0.07, then a long tail of ~0.05 spread over hundreds of tokens. With
**top-k = 2**: keep `the` and `a`, drop everything else, renormalize → `the` 0.67, `a`
0.33; `an` and `this` are now impossible even though they were perfectly reasonable.
With **top-p = 0.9**: accumulate `the` (0.50) + `a` (0.75) + `an` (0.88) + `this`
(0.95) — crossing 0.9 at `this` — so the nucleus is those four tokens; the long junk
tail is dropped.

The two truncation rules drawn against the same sorted distribution — top-k cuts at a
fixed *count*, top-p cuts where the *cumulative* probability crosses the threshold:

```
  token  prob   bar                          cumulative
  the    0.50   │█████████████████████        0.50
  a      0.25   │██████████                   0.75
  ───────────────────────────────  ◄── top-k = 2 cutoff (fixed count)
  an     0.13   │█████                        0.88
  this   0.07   │███                          0.95
  ───────────────────────────────  ◄── top-p = 0.9 cutoff (cumulative ≥ 0.90)
  …tail  0.05   │██  (hundreds of tokens, each ~0.0001)
```

top-k = 2 stops after two tokens regardless of shape; top-p = 0.9 keeps going until the
running sum reaches 0.90 — here that is four tokens. On a more confident step the top-p
cutoff would land much earlier; the count is *adaptive*. Now consider a *confident* step: `Paris` 0.97, tail 0.03. top-p = 0.9
keeps only `Paris` (its 0.97 alone exceeds 0.9), correctly admitting nothing else —
whereas top-k = 40 would still admit 39 tail tokens. That adaptivity is top-p's
advantage.

### Check questions

1. Consider two generation steps. Step A: one token at 0.96, the rest a thin tail.
   Step B: probability spread fairly evenly over ~200 plausible tokens. For `top-k = 50`
   and for `top-p = 0.9`, describe what each keeps at *each* step — and use the
   contrast to explain why one method is called "adaptive." — **Answer:** top-k = 50
   keeps exactly 50 tokens at *both* steps: at A that admits 49 junk tail tokens
   alongside the one good one; at B it chops off ~150 perfectly plausible tokens. top-p =
   0.9 keeps the smallest set reaching cumulative 0.9: at A that is just the single
   0.96 token (correctly excluding the tail); at B it is a large set of the ~200-ish
   plausible tokens. top-p is "adaptive" because its kept-set *size* changes with the
   model's confidence, while top-k's count is fixed regardless.
2. A teammate sets `top-k = 1`. Describe the resulting behavior and name the decoding
   strategy it is equivalent to. Then explain why `top-p` cannot be configured to *always*
   behave that way with a single fixed p. — **Answer:** `top-k = 1` keeps only the single
   highest-probability token every step — that is exactly **greedy decoding**. top-p
   cannot universally replicate this with one fixed p: top-p keeps the smallest set
   reaching cumulative p, so it would keep just one token only on steps where the top
   token alone already exceeds p. On a less-confident step, top-p would admit several
   tokens — its kept set is by design *variable*, whereas greedy is always exactly one.
3. A developer reasons: "I'm already using top-p = 0.9, so setting temperature is
   redundant — they do the same job." Identify the conceptual error and describe a
   concrete situation where changing temperature still changes the output even with
   top-p fixed. — **Answer:** The error is conflating two distinct operations.
   Temperature reshapes the *contrast* of the whole distribution (it rescales logits
   before softmax); top-p *truncates the tail* of whatever distribution it is given.
   With top-p fixed at 0.9, raising temperature flattens the distribution, so *more*
   tokens are now needed to reach cumulative 0.9 — the nucleus grows and the relative
   probabilities within it change — so sampled output still shifts. They are
   complementary stages, not substitutes.

---

## 3.3 — Greedy decoding vs. sampling; why beam search is rarely used

### Concept

Once you have a distribution (possibly temperature-scaled and truncated), you must pick
a token. The decoding *strategies* fall into three families.

**Greedy decoding.** At every step, deterministically pick the **single
highest-probability token**. No randomness. Greedy is simple, fast, and reproducible
(modulo the caveats in 3.7), and it is what `temperature=0` effectively gives you. Its
weakness is that locally-best choices are not globally-best: always taking the top token
can walk the model into a bland, repetitive, or dead-end continuation, because it never
explores. Greedy is the right choice for tasks with a target answer — classification,
extraction, structured output, deterministic evals.

**Sampling.** At every step, **draw a token randomly from the (truncated, temperature-
scaled) distribution** in proportion to probability. This is non-deterministic: the same
prompt yields different outputs across runs. Sampling produces varied, natural, less
repetitive text and is the standard choice for open-ended generation. Temperature and
top-p/top-k are the knobs that tune *how adventurous* the sampling is. "Greedy" can be
seen as the zero-temperature limit of sampling.

**Beam search.** Worth knowing by name, but not a tool you reach for. Beam search keeps
the *b* most probable *partial sequences* ("beams") alive at once and extends them,
searching for the highest-*total*-probability sequence rather than committing token by
token. It suited older machine translation, where there is essentially one correct
output. It is **essentially absent from modern open-ended LLM generation** for one core
reason — the **mode-seeking objective is wrong for open-ended text**: the single most
probable sequence is generic, repetitive, and bland ("I don't know. I don't know."),
whereas good human-like text has some surprise. (It is also expensive and interacts
badly with sampling.) The practical landscape in 2026: greedy (temperature 0) for
deterministic tasks, sampling (temperature + top-p) for open-ended generation, beam
search a historical footnote.

### Key terms

- **Greedy decoding** — always select the single highest-probability token;
  deterministic; the T→0 limit of sampling.
- **Sampling** — randomly draw a token from the (truncated, temperature-scaled)
  distribution; non-deterministic; standard for open-ended generation.
- **Beam search** — keeps the b highest-probability partial sequences and extends them,
  searching for the highest-total-probability (mode) output; essentially absent from
  modern open-ended LLM generation because that objective yields bland text.

### Common misconceptions

- ❌ Beam search is the standard decoding strategy for modern LLMs, and its
  highest-total-probability sequence is the best output → ✅ It is essentially absent
  from open-ended LLM generation; the most probable sequence is typically generic and
  repetitive, and good text has some surprise.
- ❌ Greedy decoding looks ahead to avoid dead ends → ✅ Greedy is purely local — it
  commits to the locally-best token each step and never explores.
- ❌ Sampling is just "low-quality randomness" → ✅ Sampling is the standard, deliberate
  choice for natural, varied generation; its adventurousness is tuned by temperature and
  top-p.

### Worked example

Prompt: "Write a one-line product tagline for a coffee brand." With **greedy** decoding
you get the single most probable continuation every time — perhaps a safe, generic
"The best coffee for your morning." Run it 10 times: identical 10 times. With
**sampling** at temperature 0.9 + top-p 0.95 you get 10 different, livelier taglines —
some great, some weak — exactly the variety you want for ideation. (Beam search would do
worse still: searching for the highest-total-probability tagline lands on the blandest
phrasing of all — the concrete reason it fell out of favor for open-ended LLM work.)

The three decoding strategies compared:

| Strategy | Deterministic? | How it picks | Good for | Why rarely used |
|---|---|---|---|---|
| Greedy | Yes (modulo 3.7 caveats) | Single highest-probability token, every step | Correct-answer tasks: classification, extraction, structured output, evals | — (widely used where reproducibility matters) |
| Sampling | No | Random draw from the truncated, temperature-scaled distribution | Open-ended generation: writing, brainstorming, varied copy | — (the standard for open-ended generation) |
| Beam search | Yes | Keeps b best *partial sequences*, extends them, returns highest-*total*-probability one | Older machine translation (one correct output) | Mode-seeking objective yields bland, generic text; expensive; clashes with sampling |

### Check questions

1. For each task, say whether you would pick greedy or sampling, and justify from the
   *task's* needs (not a memorized list): (a) extracting an invoice's total into a JSON
   field; (b) generating five distinct marketing subject lines; (c) a regression test
   that must give the same answer every run. — **Answer:** (a) Greedy — there is one
   correct value; you want the top-ranked token chosen consistently and parseably. (b)
   Sampling — you explicitly need *variety*; greedy would return the same single line
   every time. (c) Greedy — reproducibility is the whole point of a regression test
   (though even greedy is only approximately deterministic — 3.7). The principle:
   greedy when there is a target answer / reproducibility matters; sampling when variety
   is the goal.
2. Beam search searches for the single highest-total-probability sequence. In one or two
   sentences, say why that is the wrong objective for an open-ended chatbot reply. —
   **Answer:** For open-ended text the most probable sequence is generic, repetitive, and
   bland — good human-like text has some *surprise*, which is by definition not maximally
   probable — so beam search suppresses exactly what makes the output good. (It suited
   older machine translation, where there is essentially one correct output.)
3. Is greedy decoding the same as zero-temperature sampling? Explain the relationship
   precisely rather than just yes/no. — **Answer:** Effectively yes, in the limit. As
   temperature → 0, dividing logits by a vanishing T drives the distribution to collapse
   onto the single highest-logit token (probability → 1). Sampling from a distribution
   that puts all mass on one token always returns that token — which is exactly what
   greedy does. So greedy is the T → 0 limiting case of sampling; most APIs implement
   `temperature=0` as greedy for this reason.

---

## 3.4 — Repetition / frequency / presence penalties

### Concept

A known failure mode of autoregressive generation — especially with greedy or
low-temperature decoding — is **degeneration into repetition**: the model gets caught in
a loop, repeating a word, phrase, or sentence ("the the the" or restating the same
clause). This happens because once a token appears, the patterns that produced it make
it likely to appear again, and the loop reinforces itself. **Repetition penalties** are
sampling-time interventions that fight this by *modifying the logits* to discourage
already-used tokens. There are three related but distinct controls; the precise behavior
is provider-specific.

**Presence penalty.** A **flat, one-time** penalty subtracted from the logit of any
token that has *already appeared at least once* in the generated text — regardless of
how many times. It is binary: appeared or not. Effect: encourages the model to introduce
*new* tokens and topics. It says "you've used this word; lean toward something else."

**Frequency penalty.** A penalty subtracted from a token's logit **proportional to how
many times it has already appeared**. The more often a token has been used, the larger
the penalty next time. Effect: specifically targets *over*-use — a word used once is
barely penalized, a word used ten times is heavily penalized. It is the better tool for
stopping runaway repetition of a specific token.

**Repetition penalty.** A related mechanism (common in open-source inference stacks
like Hugging Face; not in the OpenAI/Anthropic core APIs in the same form) that
*divides* (rather than subtracts from) the logits of already-seen tokens by a factor —
multiplicatively down-weighting any token that has occurred.

In short: **presence** = "have you used this token at all?" (flat). **frequency** =
"how often have you used it?" (scaled). **repetition** = a multiplicative variant of the
same idea.

**Practical guidance and cautions.** These penalties are blunt instruments. Set them too
high and you get the opposite problem: the model avoids *necessary* repetition — it will
contort sentences to dodge a word it genuinely needs (a key technical term, a person's
name, common function words), producing stilted or incoherent text. Typical useful
values are small (e.g., frequency/presence penalties around 0.1–0.5 on OpenAI's scale,
where each accepts a value from -2.0 to 2.0 [2]). For most modern frontier models,
well-tuned temperature and top-p plus reasonable `max_tokens` already keep repetition
under control, and these penalties are not needed; reach for them mainly when you
observe a concrete repetition problem. Anthropic's Messages API, notably, does **not**
expose frequency or presence penalties at all [1] — another reminder that the available
knobs differ by provider.

### Key terms

- **Degeneration / repetition loop** — the failure mode where the model repeats a token,
  phrase, or sentence; common with greedy/low-temperature decoding.
- **Presence penalty** — a flat, one-time logit penalty applied to any token that has
  already appeared at least once; encourages new tokens/topics.
- **Frequency penalty** — a logit penalty scaled by how many times a token has already
  appeared; targets over-use specifically.
- **Repetition penalty** — a multiplicative down-weighting of already-seen tokens'
  logits; common in open-source inference stacks.

### Common misconceptions

- ❌ Presence and frequency penalties are the same thing → ✅ Presence is flat (applied
  once for "appeared at all"); frequency scales with the number of occurrences.
- ❌ A higher repetition penalty is always better → ✅ Set too high it suppresses
  *necessary* repetition (technical terms, names, function words), producing stilted or
  incoherent text.
- ❌ These penalties change which token the model "thinks" is best at a deep level → ✅
  They simply adjust logits at sampling time; the model's underlying computation is
  unchanged.
- ❌ Every provider exposes frequency/presence penalties → ✅ They are provider-specific;
  Anthropic's Messages API, for instance, does not expose them.

### Worked example

A summarizer keeps emitting "The report states that... The report states that... The
report states that..." With **no penalty**, "report" and "states" keep winning because
prior usage makes them likely. Apply a **frequency penalty**: the first "report" is
unpenalized, the second has a small logit penalty, the third a larger one — by the
fourth occurrence "report" has been pushed down enough that alternatives ("the
document," "it") win, breaking the loop. But push the frequency penalty too high and the
summary becomes painful: the model refuses to reuse "report" even when it is the only
correct word, twisting into vague paraphrases. The skill is using the smallest penalty
that breaks the loop.

The three repetition controls compared:

| Penalty | Flat or scaled? | Additive or multiplicative? | Best for |
|---|---|---|---|
| Presence | Flat — one-time, applied once a token has appeared at all | Additive (subtracted from the logit) | Gently encouraging *new* tokens/topics |
| Frequency | Scaled — grows with the number of prior occurrences | Additive (subtracted from the logit) | Breaking a runaway loop on one *specific* over-used token |
| Repetition | Flat (applied to any already-seen token) | Multiplicative (divides the logit by a factor) | Same goal in open-source inference stacks (Hugging Face); not in the OpenAI/Anthropic core APIs |

### Check questions

1. A token has already appeared *seven* times in the output. Compare how much a
   presence penalty versus a frequency penalty pushes its logit down on the eighth
   occurrence, and use that to say which control is the better fit for breaking a
   runaway loop on one specific word. — **Answer:** A presence penalty is *flat and
   one-time*: after the token's first appearance it applies the same fixed reduction,
   so on the eighth occurrence the penalty is no larger than it was on the second.
   A frequency penalty *scales with count*: by the eighth occurrence it has accumulated
   a much larger reduction. For a runaway loop on one specific word, the frequency
   penalty is the better fit — it specifically escalates against over-use, whereas the
   presence penalty plateaus.
2. A team adds a strong frequency penalty to stop repetition in a *technical* document
   generator, and the output gets *worse* — stilted and harder to read. Explain the
   failure, and why a technical document is an especially bad place to crank this knob.
   — **Answer:** Set too high, the penalty suppresses *necessary* repetition: the model
   contorts sentences to avoid reusing a word it genuinely needs. Technical writing is
   especially vulnerable because it must repeat exact key terms, product names, and
   defined identifiers — there is often no acceptable synonym — so a strong penalty
   forces vague paraphrases of precise terms, degrading correctness and readability. The
   skill is the *smallest* penalty that breaks an actual loop.
3. A teammate says a repetition penalty "teaches the model not to repeat itself."
   Correct the framing: what does the penalty actually modify, what is left completely
   unchanged, and why does that distinction matter for predicting its effects? —
   **Answer:** The penalty modifies the *logits* at sampling time — it down-weights
   tokens that already appeared (subtractively for presence/frequency, multiplicatively
   for the "repetition penalty" variant). It does not "teach" anything: the model's
   weights and forward computation are completely unchanged. The distinction matters
   because the penalty is a blunt, post-hoc nudge on the output distribution, not a
   change in what the model "knows" — so its effect is mechanical and local, and
   overshooting it predictably suppresses needed repetition rather than improving
   judgment.

---

## 3.5 — logprobs — what they are and using them for confidence & classification

### Concept

A **logprob** is the natural logarithm of the probability the model assigned to a token.
Recall from 1.6: softmax produces a probability for every vocabulary token; the log of
the probability of the token that was actually chosen is its logprob. Logs are used
instead of raw probabilities because probabilities are tiny and awkward to multiply,
whereas logprobs are numerically stable and **additive** — the logprob of a whole
sequence is the *sum* of its per-token logprobs (since multiplying probabilities = adding
logs). Logprobs are always ≤ 0: a probability of 1 → logprob 0; a probability of 0.5 →
about -0.69; a tiny probability → a large negative number.

Most APIs let you request logprobs in the response — for the chosen token and often for
the **top-N alternatives** at each position. This turns the model from a black box that
emits text into an instrument you can *measure*, which unlocks several techniques:

**Confidence scoring.** The logprob of a token (or the average logprob across a span) is
a proxy for how *confident* the model was. A token generated with logprob near 0
(probability near 1) was a near-certain choice; a token with a very negative logprob
(low probability) was a shaky guess where alternatives were competitive. You can flag
low-confidence outputs for human review, abstention, or a fallback path. This is not a
true calibrated probability of *correctness* — models are imperfectly calibrated and can
be confidently wrong — but it is a useful, cheap signal.

**Classification.** For a classification task, instead of asking the model to free-form
generate a label, you can constrain it to emit one token (e.g., `positive` /
`negative` / `neutral`) and read the **logprobs of the candidate label tokens directly**.
This gives you a probability distribution *over the classes* — `positive` 0.82,
`negative` 0.13, `neutral` 0.05 — in a single forward pass. You get not just the answer
but a calibratable score, you can set a confidence threshold, and it is far cheaper and
more deterministic than generating an explanation. This is one of the most powerful
practical uses of logprobs.

**Evals and detection.** Logprobs feed evaluation pipelines (Topic 9): you can score how
likely the model considered a known-correct answer (perplexity-style metrics), detect
uncertainty, or compare two candidate completions by total logprob. Some retrieval and
hallucination-detection methods also key off per-token logprob dips.

Caveats: not every model or mode exposes logprobs (reasoning models and some endpoints
restrict them); logprobs reflect *the model's* probability, which is correlated with but
not equal to correctness; and a low logprob can mean genuine uncertainty *or* simply
that several phrasings were equally valid. Use logprobs as a signal, not as truth.

### Key terms

- **Logprob** — the natural log of the probability the model assigned to a token; always
  ≤ 0; additive across a sequence.
- **Sequence logprob** — the sum of per-token logprobs; the log-likelihood of the whole
  output.
- **Top-N logprobs** — the logprobs of the N most probable tokens at a position,
  optionally returned by the API.
- **Confidence scoring** — using logprobs as a proxy for how certain the model was about
  an output.
- **Calibration** — how well the model's stated probabilities match actual correctness
  rates; LLMs are imperfectly calibrated.

### Common misconceptions

- ❌ A logprob is the probability that the answer is correct → ✅ It is the model's
  probability for that *token*; correlated with correctness but not the same — models can
  be confidently wrong.
- ❌ Logprobs can be positive → ✅ They are logs of probabilities in [0,1], so always ≤
  0; 0 means probability 1.
- ❌ To classify with an LLM you must have it generate an explanation → ✅ You can
  constrain it to a single label token and read the candidate tokens' logprobs directly
  — cheaper, faster, gives a class distribution.
- ❌ Every model exposes logprobs → ✅ Availability is provider- and mode-specific; some
  reasoning models and endpoints do not return them.

### Worked example

You need sentiment classification. Prompt the model to answer with exactly one word and
request logprobs. For the review "It was fine, nothing special," the API returns the
top tokens at the answer position: `neutral` logprob -0.22 (prob ≈ 0.80), `negative`
logprob -1.90 (prob ≈ 0.15), `positive` logprob -2.81 (prob ≈ 0.06). You now have a
*distribution over classes* in one cheap call: predict `neutral` with confidence 0.80.
Set a policy — e.g., if the top class is below 0.6, route to human review. Contrast a
clear case: "Absolutely loved it!" → `positive` logprob -0.02 (prob ≈ 0.98). The
logprob magnitude itself tells you which predictions to trust.

A reference for reading logprobs — probability is just `e^(logprob)`:

| Logprob | Probability = e^(logprob) | Reading |
|---|---|---|
| 0 | 1.00 | Certain — the ceiling; logprobs can never exceed 0 |
| −0.10 | ≈ 0.90 | Very confident |
| −0.69 | ≈ 0.50 | A coin flip |
| −1.20 | ≈ 0.30 | Shaky — competitors are real |
| −2.30 | ≈ 0.10 | Low confidence |
| −4.60 | ≈ 0.01 | Very unlikely |
| very negative (e.g. −20) | ≈ 0 | Effectively ruled out |

### Check questions

1. You see three logprobs: token X at -0.01, token Y at -0.69, token Z at +0.4.
   For each, say what it tells you about the model's probability for that token — and
   identify which value is impossible and why. — **Answer:** Logprob is the natural log
   of a probability, so probability = e^(logprob). X at -0.01 → prob ≈ 0.99 (near
   certain). Y at -0.69 → prob ≈ 0.5 (a coin flip). Z at +0.4 is **impossible**:
   logprobs are logs of probabilities in [0,1], so they are always ≤ 0; a positive
   logprob would imply a probability above 1. Logprob 0 is the ceiling (probability 1).
2. You want a classifier with a *confidence score*, not just a label. Explain why
   reading the logprobs of candidate label tokens beats asking the model to free-form
   "explain its reasoning and then answer" — name at least three concrete advantages. —
   **Answer:** Constrain the model to emit a single label token and read the logprobs of
   the candidate labels; converting them gives a probability distribution over the
   classes. Advantages over free-form: (a) you get a *calibratable confidence number*,
   so you can threshold (route low-confidence cases to humans) — a free-form explanation
   gives no comparable score; (b) it is far *cheaper and faster* — one short forward pass
   instead of generating a paragraph; (c) it is *more deterministic and parseable* — a
   single token versus prose you must extract a label from.
3. A pipeline routes any answer with a logprob below a threshold to a human, assuming
   "low logprob = likely wrong." Give two distinct reasons a *low* logprob might *not*
   mean the answer is wrong, and one reason a *high* logprob does not guarantee it is
   right. — **Answer:** Low logprob ≠ wrong: (a) several phrasings may be equally valid,
   so probability mass is split across good options and no single token scores high even
   though any of them is fine; (b) the token may be inherently open-ended (e.g. a name
   in a story) where low confidence is expected and harmless. High logprob ≠ right:
   models are imperfectly *calibrated* and can be **confidently wrong** — a fluent,
   high-probability hallucination still gets a high logprob. Logprobs are a useful cheap
   signal, not ground truth.

---

## 3.6 — max_tokens, stop sequences

### Concept

Autoregressive generation (1.5) needs a defined way to *terminate*. Three mechanisms end
a generation; an engineer controls two of them.

**Natural stop (EOS).** The model emits the end-of-sequence token (2.5) — its own
signal that the response is complete. This is the "ideal" termination: the model decided
it was done.

**`max_tokens`.** A hard cap on the number of **output tokens** the model may generate
in this response. When the count is reached, generation is **cut off immediately**,
wherever it happens to be — possibly mid-sentence, mid-word, or mid-JSON. `max_tokens`
is a *safety and budget control*, not a length target: it bounds worst-case cost and
latency and prevents a runaway generation, but the model does not "aim" for it. Setting
it too low **truncates** real answers (a notorious cause of broken/unparseable JSON in
structured-output pipelines — Topic 11); setting it sensibly high is cheap because you
are billed for tokens *actually generated*, not for the cap. Most APIs report a
`finish_reason` / `stop_reason` so you can detect *why* generation ended — `stop` (EOS),
`length` (hit max_tokens), `stop_sequence`, etc. **Always check it**: a `length` finish
means your output is probably truncated and you should not trust it as complete.
Reasoning models complicate this — "thinking" tokens count toward the output budget, so
`max_tokens` must leave room for the thinking phase *plus* the visible answer or
generation can be cut off mid-reasoning; the budgeting and latency economics of thinking
tokens are covered in Topic 4.6.

**Stop sequences.** One or more strings you specify that, when generated, cause the API
to **halt generation immediately** — and (typically) the stop string itself is **not
included** in the returned output. Stop sequences let *you* define where a response ends
based on content. Uses: in a prompt formatted as a transcript, set a stop sequence like
`\nUser:` so the model cannot hallucinate the user's next turn; cap output at a known
delimiter; stop after a closing tag like `</answer>`. Stop sequences are matched on the
generated text; they do not retroactively affect anything already streamed before the
match.

The three together: the model **wants** to stop at EOS; **`max_tokens`** is your hard
ceiling that stops it regardless; **stop sequences** are your content-based early exits.
Engineering discipline: set `max_tokens` high enough to never truncate legitimate
answers but low enough to bound cost/latency, *always* inspect the finish reason, and use
stop sequences to prevent the model from running past the intended boundary of a turn.

### Key terms

- **max_tokens** — a hard cap on output tokens for one response; on reaching it
  generation is cut off immediately, possibly mid-token.
- **Stop sequence** — a string that, when generated, immediately halts generation; the
  stop string is normally excluded from the returned output.
- **EOS / natural stop** — the model emitting the end-of-sequence token to signal it is
  finished on its own.
- **finish_reason / stop_reason** — the field reporting why generation ended (`stop`,
  `length`, `stop_sequence`, …); must be checked to detect truncation.
- **Truncation** — an output cut off before completion because `max_tokens` was reached.

### Common misconceptions

- ❌ `max_tokens` tells the model how long to make the answer → ✅ It is a hard cutoff,
  not a target; the model does not aim for it. Too low → truncated output.
- ❌ Setting `max_tokens` high wastes money → ✅ You are billed for tokens actually
  generated, not the cap; a generous cap just prevents truncation.
- ❌ The stop sequence string appears in the returned output → ✅ It is normally
  excluded; generation halts when it is produced.
- ❌ If you get a response back, it is complete → ✅ Check the finish reason: a `length`
  finish means the output was truncated by `max_tokens` and may be unusable.

### Worked example

You ask for a JSON object and set `max_tokens=50`. The model produces
`{"name": "Ada Lovelace", "role": "mathematician", "born": 18` — and stops, because the
50-token cap hit mid-value. The `finish_reason` is `length`. Your JSON parser throws;
the pipeline fails. The bug is not the model — it is the cap. Raise `max_tokens` to a
comfortable 500 and the model finishes the object naturally (EOS), `finish_reason` is
`stop`, and parsing succeeds. Separately: in a chat-transcript prompt you add stop
sequence `"\nHuman:"`. The model writes its reply and then starts to hallucinate
`\nHuman: thanks, also...` — but generation halts the instant `\nHuman:` is produced and
that string is stripped, so you cleanly get just the assistant's turn.

### Check questions

1. A developer sets `max_tokens=4000` hoping the model will "aim for a thorough
   ~4,000-token answer," and separately worries that a high cap wastes money. Correct
   *both* misunderstandings of what `max_tokens` is. — **Answer:** (a) `max_tokens` is
   not a length *target* — the model does not aim for it. It is a hard *ceiling*:
   generation is cut off the instant the count is reached, possibly mid-word or
   mid-JSON. To make answers longer you must prompt for length, not raise the cap. (b) A
   high cap does not waste money: you are billed for tokens *actually generated*, not for
   the cap. A generous cap simply avoids truncating legitimate answers; it costs nothing
   extra when the model stops earlier on its own.
2. A JSON-extraction pipeline parses every response and "works most of the time."
   Explain why ignoring the finish/stop reason is a latent bug, what specific failure it
   will eventually cause, and what a `length` finish reason should trigger instead of
   parsing. — **Answer:** Without checking the finish reason, the pipeline cannot tell a
   *complete* answer from one *truncated* by `max_tokens`. Eventually a large object hits
   the cap and generation stops mid-JSON; the parser throws on invalid JSON — an
   intermittent failure that looks random. A `length` finish reason should be treated as
   a failure signal: do not parse it; retry (with a higher cap) or handle it as
   incomplete. Only a `stop`/EOS finish indicates a naturally complete response.
3. You add the stop sequence `"\n## "` to make the model stop at the next Markdown
   heading. The model generates a paragraph, then `\n## `, then would have continued.
   Describe exactly what the API returns and where generation stopped — and explain one
   risk if your downstream code *assumed* the stop string would be present. — **Answer:**
   Generation halts the instant `\n## ` is produced, and the stop string itself is
   normally *excluded* from the returned text — so you get the paragraph and nothing
   after, with no trailing `\n## `. Risk: code that, say, splits on `\n## ` to separate
   sections, or expects the heading marker as a delimiter, will not find it — because the
   API stripped it. You must not rely on the stop string appearing in the output.

---

## 3.7 — Determinism caveats; seeds and their limits

### Concept

A natural assumption is that **temperature 0 makes the model perfectly deterministic** —
same input, byte-identical output, every time. In practice that is **not reliable**.
Greedy decoding removes the *sampling* randomness, but several other sources of
non-determinism remain, and an engineer must know them.

**Why temperature 0 is not fully deterministic:**

- **Floating-point non-associativity.** Floating-point addition is not associative:
  `(a + b) + c` can differ in the last bits from `a + (b + c)`. GPUs sum massive numbers
  of values in parallel, and the *order* of those reductions can vary between runs,
  hardware, driver versions, or kernel choices. The resulting tiny numerical differences
  occasionally flip which token has the highest logit when two candidates are very close
  — and once one token differs, the rest of the generation can diverge entirely.
- **Batching effects.** Providers serve many requests together with continuous batching
  (Topic 12). The *other* requests sharing your batch, and the batch size, change the
  exact computation path and reduction order — so the same prompt can produce slightly
  different logits depending on what else was running at that moment. As an API consumer
  you do not control this directly — but note (see below) that it is now a *known,
  partly solved* engineering problem, not an immutable law.
- **Mixture-of-Experts (MoE) routing.** Many frontier models are MoE (Topic 14): each
  token is routed to a subset of "expert" sub-networks. In some implementations routing
  decisions can be affected by which other tokens are in the batch, adding another
  source of run-to-run variation.
- **Infrastructure changes.** The provider can update model weights, kernels, hardware,
  or the inference stack at any time; "the same model name" is not guaranteed to be the
  same computation across days.

**Seeds and their limits.** Some APIs expose a `seed` parameter — OpenAI's API is the
common example; Anthropic's Messages API does not currently expose a `seed` parameter
[1][3]. A seed fixes the *pseudo-random number generator* used for **sampling**, so
with the same seed, temperature, and prompt you get more *consistent* sampled outputs.
But a seed only addresses the sampling-randomness source. It does **not** fix
floating-point reduction order, batching effects, MoE routing, or backend changes.
OpenAI itself describes seeded outputs as **best-effort**, explicitly stating
determinism is *not guaranteed*, and exposes a `system_fingerprint` field so you can
detect when the backend changed [3]. With temperature 0 a seed is largely moot — there
is no sampling randomness to fix — yet output can still vary for the other reasons.

**This is partly solvable — it is not a law of nature.** The batching-effect source in
particular is now well understood and addressable. The reason a request's logits depend
on its batch is that common GPU kernels (matmuls, reductions, normalization) sum values
in a *batch-size-dependent order*, and floating-point addition is non-associative.
**Batch-invariant (deterministic) inference kernels** are written so the reduction order
is *fixed regardless of batch size or composition* — at some throughput cost — which
removes the batching-effect source and can make temperature-0 generation bit-exact and
reproducible run to run. This is an active area of inference engineering (2025–2026):
some inference stacks and providers now offer an opt-in deterministic mode. The honest
framing for an engineer: nondeterminism is a *known engineering trade-off* (you can pay
throughput for reproducibility), not something inherently "outside your control" — but
unless you have explicitly enabled such a mode, assume the default behavior is
non-deterministic.

**Practical consequences.** Do not build systems that assume bit-exact reproducibility
from an LLM. Caches should key on inputs, not assume identical outputs. Evals (Topic 9)
must tolerate run-to-run variation — run multiple samples, use tolerant metrics, never
assert exact-string equality on a live model. Agent runs (Topic 8) are inherently
non-reproducible for the same reasons; log full traces rather than expecting to "replay"
a run. The correct mental model: an LLM call is a *probabilistic* operation even at
temperature 0; treat reproducibility as approximate.

### Key terms

- **Determinism** — same input always producing the same output; for LLMs it is only
  approximate, even at temperature 0.
- **Floating-point non-associativity** — `(a+b)+c ≠ a+(b+c)` in floating point; varying
  GPU reduction order yields tiny numerical differences that can flip close logits.
- **Batching effects** — co-located requests and batch size altering the exact
  computation path and thus the logits.
- **MoE routing non-determinism** — token-to-expert routing influenced by batch
  composition, adding run-to-run variation.
- **Seed** — a parameter fixing the sampling RNG; makes sampled output more consistent
  but does not fix the non-sampling sources.
- **system_fingerprint** — an identifier exposed by some APIs so clients can detect
  backend changes.
- **Batch-invariant / deterministic inference kernels** — inference kernels written so
  the floating-point reduction order is fixed regardless of batch size/composition;
  remove the batching-effect source of nondeterminism (at a throughput cost), making
  temperature-0 output reproducible.

### Common misconceptions

- ❌ Temperature 0 guarantees identical output every time → ✅ It removes sampling
  randomness only; floating-point order, batching, MoE routing, and backend changes can
  still cause divergence.
- ❌ A seed makes LLM output fully reproducible → ✅ A seed fixes only the sampling RNG;
  it does not control numerical/batching/routing non-determinism. Determinism is
  best-effort.
- ❌ "Same model name" means the same computation forever → ✅ Providers update weights,
  kernels, and hardware; the backend can change underneath a stable model name.
- ❌ LLM nondeterminism is an unfixable law of nature → ✅ It is a known engineering
  trade-off: batch-invariant (deterministic) inference kernels remove the batching-effect
  source at a throughput cost, and some stacks now offer an opt-in deterministic mode.
- ❌ Evals can assert exact-string equality against a live model → ✅ They must tolerate
  run-to-run variation — sample multiple times and use tolerant metrics.

### Worked example

You run a regression test that calls the model at `temperature=0` with a fixed prompt
and asserts the output equals a saved golden string. It passes for a week, then fails in
CI — yet nothing in your code changed. What happened: the output diverged for one of the
infrastructure reasons. Perhaps your request landed in a differently-sized batch, the
reduction order shifted, and at one step two tokens had near-identical logits — the tie
broke the other way, and the rest of the generation followed a different path.

How one near-tie cascades into a wholly different answer:

```
            two near-equal logits at one step
                  ┌─────────────────────┐
                  │  "down"  logit 7.41 │   ◄── tiny FP perturbation
                  │  "on"    logit 7.40 │       from batch/kernel change
                  └─────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
       run A picks "down"           run B picks "on"
              │                           │
              ▼                           ▼
     "...the cat sat down           "...the cat sat on
        and waited."                  the warm mat."
              │                           │
              ▼                           ▼
       diverging continuations — every later token conditions on
       a different prefix, so the two answers never reconverge
```

One flipped close call near the start propagates through every subsequent step. Or the
provider rolled a new kernel. Adding a `seed` would not have saved the test, because the
divergence was not from sampling. Unless your provider offers an opt-in deterministic
(batch-invariant) mode and you have enabled it, the correct fix is to stop asserting
exact equality: score the output semantically (e.g., an LLM-judge or rubric, Topic 9) or
assert on structured fields, and tolerate variation. (If a deterministic mode *is*
available, enabling it is the other legitimate route — at a throughput cost.)

### Check questions

1. At `temperature=0` there is no sampling randomness, yet two calls with the identical
   prompt can still diverge. Walk through *how* a non-sampling source of variation turns
   into a different final answer — i.e. the chain from a tiny numerical difference to a
   wholly different sentence. — **Answer:** Sources like floating-point non-associativity
   (GPU reduction order varying with batch composition, kernels, hardware) produce tiny
   differences in the computed *logits*. At most steps this is harmless — the top token
   is unchanged. But when two candidate tokens have very *close* logits, a tiny
   perturbation can flip which one is highest. Greedy then picks the other token; that
   different token becomes part of the context for every subsequent step, so the rest of
   the generation conditions on a different prefix and can diverge entirely. One flipped
   close call cascades into a different answer.
2. A teammate, after a flaky test, says "add a `seed` — that makes the model
   deterministic." Evaluate this for a call made at `temperature=0`, and separately for
   a call at `temperature=0.8`. — **Answer:** A `seed` fixes only the *sampling* RNG. At
   `temperature=0` there is no sampling step for the seed to control, so adding a seed
   does essentially nothing — the flakiness is from numerical/batching/routing/backend
   sources the seed cannot touch. At `temperature=0.8` a seed *does* help: it makes the
   sampling draws repeatable, so outputs are more consistent — but still only
   best-effort, since the same non-sampling sources remain. In neither case does a seed
   deliver guaranteed bit-exact reproducibility.
3. A regression suite asserts the model's output exactly equals a saved golden string;
   it passes for weeks, then fails in CI with no code change. Explain the most likely
   cause, why the test design — not the model — is the real defect, and how you would
   rewrite the assertion. — **Answer:** The output diverged for an infrastructure reason
   (a differently-sized batch shifted reduction order and flipped a close logit, or the
   provider rolled a new kernel/weights). The model behaving non-deterministically at
   `temperature=0` is *expected*; the defect is a test that assumes bit-exact
   reproducibility from a probabilistic, infrastructure-dependent system. Rewrite it to
   tolerate variation: assert on structured fields / key properties, or score the output
   semantically (rubric or LLM-judge, Topic 9), and sample multiple times rather than
   demanding one exact string.

---

## Topic 03 — Exam Question Bank

A pool the tutor draws from for the gated exam. Mixed-format, scored out of 100; 85 to
pass. Free-form answers are graded on reasoning quality with partial credit.

### True / False

1. If the correct answer token currently has a low logit, raising the temperature is a
   reliable way to get the model to choose it. — **Answer:** False. Temperature only
   reshapes the *existing* ranking; it cannot re-rank tokens or add knowledge. Raising it
   makes low-probability tokens more competitive in general but does not reliably surface
   one specific correct token.
2. At a generation step where the model is highly confident (one token at 0.97), top-p
   = 0.9 and top-k = 40 keep about the same number of tokens. — **Answer:** False. top-p
   keeps just the single 0.97 token (it alone exceeds 0.9), while top-k = 40 still
   admits 39 tail tokens. top-p's kept-set size adapts to confidence; top-k's does not.
3. On the eighth occurrence of the same token, a presence penalty and a frequency
   penalty push its logit down by roughly the same amount. — **Answer:** False. The
   presence penalty is flat and one-time (same reduction after the first appearance); the
   frequency penalty scales with count, so by the eighth occurrence it is much larger.
4. A token reported with a logprob of +0.5 was assigned a probability of about 1.6 by
   the model. — **Answer:** False. Logprobs are logs of probabilities in [0,1], so they
   are always ≤ 0; a positive logprob is impossible. (e^0 = 1 is the ceiling.)
5. A response that comes back with `finish_reason: "length"` can be trusted as a
   complete answer as long as it parses. — **Answer:** False. A `length` finish means
   `max_tokens` cut generation off — possibly mid-structure. It may *happen* to parse yet
   still be truncated/incomplete; a `length` finish should be treated as a failure.
6. Adding a `seed` to a `temperature=0` request makes its output reliably bit-identical
   across calls. — **Answer:** False. A seed only fixes the sampling RNG, and at
   `temperature=0` there is no sampling to fix; floating-point reduction order, batching,
   MoE routing, and backend changes can still cause divergence.
7. Code that splits a response on the configured stop-sequence string will reliably find
   that string in the returned output. — **Answer:** False. The stop string is normally
   *excluded* from the returned output; generation halts when it is produced, so
   downstream code cannot rely on it being present.
8. The batching-effect source of LLM nondeterminism is an immutable law of physics that
   no engineering can address. — **Answer:** False. It stems from batch-size-dependent
   floating-point reduction order; batch-invariant (deterministic) inference kernels fix
   that order regardless of batch composition and remove the source, at a throughput
   cost — it is a known, partly solved engineering trade-off.

### Multiple Choice

1. You add the same constant to every logit before softmax. The resulting distribution:
   A) Becomes sharper  B) Becomes flatter  C) Is unchanged, because softmax responds
   only to the gaps between logits  D) Sums to more than 1
   — **Answer:** C. A shared additive constant cancels in softmax's normalization; only
   the *differences* between logits matter — which is why temperature, which rescales
   those gaps, is the meaningful knob.
2. A model is uncertain at a step: probability is spread fairly evenly over ~150
   plausible tokens. Compared to a confident step, `top-p = 0.9` will:
   A) Keep the same number of tokens, since p is fixed
   B) Keep many more tokens, because more are needed to reach cumulative 0.9
   C) Keep exactly one token  D) Keep zero tokens
   — **Answer:** B. top-p's kept set grows when the model is uncertain and shrinks when
   it is confident — that adaptivity is its defining property.
3. A team uses beam search for a customer-support chatbot and complains the replies are
   bland and formulaic. The most fundamental reason is:
   A) Beam search is too slow to be creative
   B) Beam search searches for the highest-total-probability sequence, and for
   open-ended text that sequence is generic — good replies need some surprise
   C) Beam search disables the system prompt
   D) Beam search only supports one language
   — **Answer:** B. The blandness is intrinsic to optimizing for the mode; it is the
   wrong objective for open-ended generation.
4. A summarizer is stuck repeating one specific phrase dozens of times. Of the
   repetition controls, the one that escalates *specifically* against that over-used
   phrase (rather than penalizing all repeats equally once) is:
   A) Presence penalty  B) Frequency penalty  C) Temperature  D) A stop sequence
   — **Answer:** B. The frequency penalty scales with occurrence count, so it grows
   against the over-used token; presence is flat after the first appearance.
5. You want a sentiment classifier that also reports a confidence score in one cheap
   call. The best approach is:
   A) Generate a paragraph of reasoning, then a label, at high temperature
   B) Constrain the output to one label token and read the candidate labels' logprobs
   as a probability distribution over classes
   C) Run beam search over the three label strings
   D) Call the model three times and majority-vote
   — **Answer:** B. Reading candidate-token logprobs yields a calibratable class
   distribution from a single short forward pass.
6. A JSON-extraction call returns `finish_reason: "length"`. The correct handling is:
   A) Parse it and use the result if it happens to be valid JSON
   B) Treat it as truncated/incomplete — do not trust it; retry with a higher
   `max_tokens` or handle as a failure
   C) Lower `max_tokens` so it stops sooner
   D) Raise the temperature and retry
   — **Answer:** B. A `length` finish means generation was cut off; the result must not
   be trusted as complete even if it parses.
7. An engineer wants repeatable outputs and sets a `seed`. The `seed` parameter
   primarily makes repeatable:
   A) The floating-point reduction order on the GPU
   B) The draws of the sampling random-number generator
   C) Which other requests share the batch
   D) The provider's backend/kernel version
   — **Answer:** B. A seed fixes only the sampling RNG; the other three are non-sampling
   sources of variation a seed cannot control.
8. For an extraction task scored against ground truth, the most appropriate sampling
   configuration is:
   A) temperature ≈ 1.0 with top_p ≈ 0.95, for natural output
   B) Beam search with b = 5
   C) temperature ≈ 0 (greedy), so the top-ranked token is chosen consistently and
   reproducibly
   D) A high frequency penalty to avoid repeated field names
   — **Answer:** C. The task has a correct answer; you want the model's top choice
   chosen consistently, not variety. (A high frequency penalty would actively harm
   repeated-but-necessary field names.)

### Short Answer

1. Temperature is often described as a "creativity" knob. Explain why that label is
   misleading, and give the precise mechanical description of what raising it actually
   does — including why pushing it far enough produces *incoherent* rather than
   *creative* output. — **Model answer:** "Creativity" implies the model gains new
   ability; it does not. Temperature divides every logit by T before softmax, which
   compresses (T > 1) or magnifies (T < 1) the gaps between logits. Raising it flattens
   the distribution, so genuinely unlikely tokens become competitive — that reads as
   variety at moderate values, but pushed far enough the model starts selecting tokens it
   "believes" are wrong, producing incoherence. It reshapes a fixed distribution; it adds
   nothing.
2. top-k and top-p both truncate the tail, but only one is "adaptive." Explain what
   "adaptive" means here by describing how each behaves on a confident step versus an
   uncertain step, and conclude why top-p is generally preferred. — **Model answer:**
   "Adaptive" means the *number of tokens kept* responds to the model's confidence.
   top-k keeps a fixed count k on every step: on a confident step it admits junk tail
   tokens, on an uncertain step it may chop off many reasonable ones. top-p keeps the
   smallest set reaching cumulative p, so it keeps few tokens when the model is confident
   and many when it is uncertain — matching the distribution's actual shape. That is why
   top-p is generally preferred: it neither admits junk nor over-truncates.
3. Logprobs turn the model "from a black box into an instrument." Explain what a logprob
   is, what makes it numerically convenient, and one concrete measurement it enables
   that raw generated text alone does not. — **Model answer:** A logprob is the natural
   log of the probability the model assigned to a token (always ≤ 0). It is convenient
   because probabilities of long sequences are vanishingly small and unstable to
   multiply, whereas logprobs are stable and *additive* — a sequence's log-likelihood is
   just the sum of its token logprobs. It enables, e.g., confidence scoring: a token's
   logprob is a cheap proxy for how certain the model was, so you can flag low-confidence
   spans for review — something plain text gives you no handle on.
4. A team has a runaway-repetition problem on one specific word and is choosing between
   a presence penalty and a frequency penalty. Explain which fits better and why, by
   contrasting how each one's penalty grows as the word recurs. — **Model answer:** The
   frequency penalty fits better. A presence penalty is flat: after the word's first
   appearance it applies the same fixed reduction every time, so it does not escalate
   against a word used 20 times. A frequency penalty scales with the occurrence count, so
   each repeat of the over-used word is penalized more than the last — which is exactly
   what is needed to break a runaway loop on a specific token. Presence is better suited
   to gently encouraging *new topics*, not stopping one runaway word.
5. `max_tokens` is described as "a safety/budget control, not a length target." Explain
   both halves of that statement, and explain why ignoring the finish reason turns a
   too-low `max_tokens` into an *intermittent* bug rather than an obvious one. — **Model
   answer:** "Not a length target": the model does not aim for `max_tokens`; it is a hard
   ceiling that cuts generation off immediately on being reached, possibly mid-token.
   "Safety/budget control": it bounds worst-case cost and latency and stops runaway
   generations. If you never check the finish reason, you cannot distinguish a complete
   answer (`stop`) from a truncated one (`length`) — so a too-low cap only fails on the
   *occasional* large output, producing sporadic invalid results that look random rather
   than a consistent, obvious failure.
6. `temperature=0` removes sampling randomness, yet output can still vary call to call.
   Name two distinct non-sampling sources of variation and, for *one* of them, explain
   the mechanism by which it changes the final answer. — **Model answer:** Two of:
   floating-point non-associativity (GPU reduction order varying with kernels/hardware/
   batch); batching effects (co-located requests change the computation path); MoE
   routing influenced by batch composition; provider backend/weight/kernel updates.
   Mechanism (e.g. floating-point): a tiny numerical difference perturbs the logits;
   usually harmless, but when two candidate tokens have very close logits the
   perturbation flips which is highest, greedy picks the other token, and that different
   token reshapes every subsequent step's context — so the whole continuation diverges.

### Long Answer

1. A developer reports: "I set temperature low so the output would be focused, *and*
   set top-p very low so only safe tokens survive — but the output is still
   occasionally weird." Walk through the full pipeline from logits to a selected token,
   naming where temperature and top-p each act, and use the ordering to explain why
   stacking two aggressive settings is hard to reason about (and what the recommended
   discipline is). — **Model answer / rubric:** Pipeline: (1) the model produces logits —
   one raw score per vocabulary token; (2) temperature divides every logit by T (T < 1
   sharpens contrast, T > 1 flattens, T → 0 greedy); (3) softmax converts the scaled
   logits into a probability distribution summing to 1; (4) truncation — top-k keeps the
   k highest and/or top-p keeps the smallest set reaching cumulative p, then the survivors
   are renormalized; (5) a token is sampled from what remains (greedy takes the top one).
   Temperature controls *contrast*; top-p controls *how much tail survives* — they act at
   different stages on the *same* distribution, so two aggressive settings interact in
   ways that are hard to predict (a low temperature already concentrates mass, then a low
   top-p truncates the already-sharpened distribution — the combined effect is opaque).
   The recommended discipline is to tune *one* of temperature or top-p at a time. Strong
   answers also note "occasionally weird" output can come from non-sampling sources
   (3.7), not just the knobs.
2. Explain how to build a robust LLM classifier using logprobs, and its advantages over
   free-form generation. — **Model answer / rubric:** Prompt the model to answer with
   exactly one label token from a fixed set; set low temperature; request top-N
   logprobs. Read the logprobs of the candidate label tokens and convert to a probability
   distribution over the classes. Advantages: a calibratable confidence score, not just a
   label; you can threshold (route low-confidence cases to human review or abstention);
   one cheap forward pass instead of generating an explanation; more deterministic and
   parseable. Caveats: logprobs reflect model confidence, not guaranteed correctness
   (imperfect calibration); not all models expose logprobs.
3. Discuss the sources of non-determinism in LLM inference and how an engineer should
   design around them. — **Model answer / rubric:** Sources: sampling randomness
   (removed by greedy/T=0 or fixed by a seed); floating-point non-associativity with
   varying GPU reduction order; batching effects from co-located requests; MoE routing
   influenced by batch composition; provider backend/weight/kernel updates. A seed only
   fixes sampling. Crucially, this is *partly solvable*, not a law of nature: the
   batching-effect source can be removed with **batch-invariant (deterministic) inference
   kernels** that fix the reduction order regardless of batch size — an opt-in mode some
   stacks now offer, at a throughput cost. Design implications: unless a deterministic
   mode is enabled, never assume bit-exact reproducibility; do not assert exact-string
   equality in evals/regression tests — use semantic or structured scoring and multiple
   samples; expect agent runs to be non-reproducible and log full traces; watch
   `system_fingerprint` for backend changes; treat a default LLM call as a probabilistic
   operation even at T=0.
4. A product chatbot running on a low temperature starts producing answers that loop —
   restating the same sentence over and over until `max_tokens` cuts them off. A junior
   engineer's first instinct is to crank a frequency penalty to its maximum. Explain
   *why* low-temperature decoding is prone to this loop in the first place, lay out the
   range of tools available with the trade-off of each, and explain specifically what
   will go wrong with the "max frequency penalty" instinct. — **Model answer / rubric:**
   Why it loops: once a token/phrase appears, the patterns that produced it make it
   likely again; greedy/low-temperature decoding always takes that locally-most-probable
   token, so it reinforces the loop instead of escaping it. Tools and trade-offs: raise
   temperature / use top-p sampling — adds variety so the loop is less self-reinforcing,
   but too much harms focus; presence penalty — flat, one-time, nudges toward new
   topics; frequency penalty — scales with occurrence count, best for a specific runaway
   token; repetition penalty — multiplicative variant (open-source stacks). What goes
   wrong with a maxed frequency penalty: it suppresses *necessary* repetition — the model
   contorts to avoid reusing required terms, names, and common function words, producing
   stilted, incoherent text — trading one failure for another. Correct approach: use the
   *smallest* intervention that breaks the loop; for modern frontier models, well-tuned
   temperature/top-p plus a sensible `max_tokens` often suffices. Note penalties are
   provider-specific — Anthropic's Messages API does not expose them, so on that API the
   temperature/top-p route is the available one.

### Applied Scenario

1. A meeting-notes feature usually produces clean summaries, but a user reports that on
   *long* meetings the summary "just stops mid-sentence" and the last bullet is
   incomplete. Short meetings are fine. The team currently does not inspect any response
   metadata. Diagnose the most likely cause, explain why the length of the *input* meeting
   correlates with the failure, and give the fix — including why the fix is essentially
   free. — **Model answer / rubric:** The summary is being truncated by `max_tokens`:
   long meetings warrant longer summaries, and on those the output hits the cap and
   generation is cut off immediately, mid-sentence — short meetings stay under the cap, so
   they look fine. The team cannot see this because they ignore the finish reason; a
   `length` finish is the tell. Fix: raise `max_tokens` to comfortably exceed the longest
   expected summary, and always inspect the finish reason — treat `length` as a failure
   (retry / handle), not a normal result. The fix is nearly free because billing is for
   tokens *actually generated*, not for the cap — a generous ceiling costs nothing on the
   short meetings.
2. A creative-writing feature produces near-identical text every run; users complain it
   is repetitive and bland. What sampling settings would you change? — **Model answer /
   rubric:** It is almost certainly running greedy / very low temperature, so the single
   most probable continuation is chosen every time. Raise temperature into the ~0.7–1.0
   range and use top-p (~0.9–0.95) to allow varied but still coherent continuations.
   Optionally add a modest presence/frequency penalty if specific phrases recur. Avoid
   temperature far above ~1.2 (incoherence). Verify with sample outputs.
3. Two teammates run the *same* `temperature=0` prompt against the *same* model at the
   same time and get slightly different answers. One concludes "the API is broken"; the
   other says "someone must have changed the prompt." Both are wrong. Explain what is
   actually happening, why `temperature=0` does not prevent it, and what this implies for
   how the team should write any test or cache that depends on the model's output. —
   **Model answer / rubric:** Neither a bug nor a prompt change — this is expected
   non-determinism from *non-sampling* sources: the two requests likely landed in
   different-sized batches, shifting GPU floating-point reduction order (and possibly MoE
   routing), which perturbs logits enough to flip a close token choice and diverge the
   rest of the generation; a backend/kernel update could do the same. `temperature=0`
   only removes *sampling* randomness — it cannot touch these sources, so it does not
   make output bit-exact *by default*. Implication: unless the provider offers an opt-in
   deterministic (batch-invariant) mode and you have enabled it, do not build tests or
   caches that assume identical output. Tests should assert on structured fields or score
   semantically (rubric / LLM-judge) and sample multiple times; caches should key on the
   *inputs*, not assume a reproducible output. Treat a default LLM call as probabilistic
   even at `temperature=0` — while knowing that deterministic kernels make this a
   solvable engineering trade-off, not an immutable law.
4. You need a confidence signal to decide when to escalate an LLM answer to a human.
   How would you implement it and what are the limits? — **Model answer / rubric:**
   Request logprobs and use the chosen token's logprob (or the average logprob across
   the answer, or — for a classifier — the top class's probability) as a confidence
   proxy; set a threshold below which the case is routed to a human. Limits: logprobs
   reflect the model's *self-assessed* probability, which correlates with but is not
   equal to correctness — models can be confidently wrong (imperfect calibration); a low
   logprob may mean genuine uncertainty *or* simply multiple valid phrasings; not all
   models expose logprobs. Calibrate the threshold against labeled data and treat it as
   a useful heuristic, not ground truth.
5. A teammate sets `temperature=1.5` and `top_p=0.5` and `frequency_penalty=1.8` "to get
   the best of all knobs." Critique the configuration. — **Model answer / rubric:** The
   settings fight each other and are extreme. High temperature (1.5) flattens the
   distribution toward incoherence, but top_p=0.5 then aggressively truncates to a small
   nucleus — so you pay the incoherence risk while also over-restricting; tuning *one* of
   temperature or top-p at a time is the recommended discipline. A frequency penalty of
   1.8 is very high and will suppress necessary repetition, producing stilted text.
   Better: pick a single coherent regime — e.g., temperature ≈0.8 with top_p ≈0.9 and no
   penalty — then adjust based on observed output.

---

## Sources

[1] Anthropic — Messages API reference (temperature 0.0–1.0; `top_p`/`top_k` as advanced parameters; no frequency/presence penalty or seed) — https://platform.claude.com/docs/en/api/messages
[2] OpenAI — Create chat completion API reference (temperature 0–2; `top_p`; `frequency_penalty` and `presence_penalty` −2.0 to 2.0) — https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create
[3] OpenAI — Reproducible outputs with the seed parameter (best-effort determinism; `system_fingerprint`) — https://developers.openai.com/cookbook/examples/reproducible_outputs_with_the_seed_parameter
