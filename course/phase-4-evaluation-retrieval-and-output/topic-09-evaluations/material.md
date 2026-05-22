# Topic 09 — Evaluations — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This topic is taught sub-chapter by sub-chapter, in order. Each sub-chapter has a
**Concept** section (the teaching content), **Key terms**, **Common misconceptions**, a
**Worked example**, and **Check questions** to confirm understanding before moving on.
The **Exam Question Bank** at the end is the pool the tutor draws from for the gated,
closed-book exam (scored out of 100, pass mark 85). Teach a sub-chapter fully, run its
check questions, and only then advance. Evaluations are the discipline that turns
"the demo works" into "the system measurably works" — treat this topic as the spine of
everything in Phase 4 and 5.

---

## 9.1 — Why evals exist; offline vs. online

### Concept

An **evaluation** (an "eval") is a repeatable measurement of how well an LLM system
performs a task. Without evals you are flying blind: LLM outputs are non-deterministic,
prompts are brittle, and a change that fixes one case silently breaks five others. You
cannot improve what you cannot measure, and with LLMs you cannot even *tell whether you
broke something* without measurement, because there is no compiler and no type checker
for "is this answer good."

Software engineers already accept that untested code is broken code. Evals are the LLM
analogue of a test suite — but with a crucial difference: a unit test is a binary
pass/fail against an exact expectation, while an eval is usually a *score* against a
fuzzy notion of quality, because the same question has many acceptable answers. This
shifts the engineering problem from "assert equality" to "define and measure quality."

There are two fundamentally different settings:

**Offline evaluation** runs against a fixed, curated dataset *before* you ship. You
have known inputs, often known expected outputs, and you run the system over them in a
controlled batch. Offline evals are fast, cheap, repeatable, and safe — you can run them
in CI on every prompt change. Their weakness is **distribution shift**: your curated
dataset may not match what real users actually do.

**Online evaluation** measures the system *in production* on real traffic. It includes
A/B tests, live quality scoring of a sample of real responses, user feedback signals
(thumbs up/down, thumbs, edits, retries, abandonment), and business metrics
(resolution rate, escalation rate). Online evals capture the true distribution and real
user behavior, but they are slower to read, noisier, and a bad change can harm real
users before you notice.

The two are complementary, not competing. The mature workflow is: **offline evals gate
deploys** (catch regressions before they ship), **online evals catch what offline
missed** (the cases your dataset didn't cover), and findings from online flow *back*
into the offline dataset so the same failure can never recur unnoticed. This loop —
ship, observe, harvest failures, expand the eval set — is the heart of **eval-driven
development**.

A useful mental model: offline eval answers "did I break anything I already know
about?"; online eval answers "what is happening that I don't know about?"

This closed cycle — the heart of eval-driven development — is what keeps the eval set
and the system improving together:

```
                  ┌──────────────────────────────────────────┐
                  │           GOLDEN DATASET                  │
                  │   curated inputs + references/rubrics      │
                  └──────────────────────────────────────────┘
                       │                          ▲
                       ▼                          │
              ┌─────────────────┐                 │ new failure
              │  OFFLINE EVAL   │                 │ becomes a
              │  (the GATE)     │                 │ permanent
              │  run in CI      │                 │ regression row
              └─────────────────┘                 │
                       │                          │
              pass ────┤                 ┌──────────────────┐
              fail ──► block, fix        │ HARVEST FAILURES │
                       │                 │ thumbs-down,     │
                       ▼                 │ edits, escalations│
              ┌─────────────────┐        └──────────────────┘
              │ DEPLOY / CANARY │                 ▲
              │  ship to a slice│                 │
              └─────────────────┘                 │
                       │                          │
                       ▼                          │
              ┌─────────────────┐                 │
              │   ONLINE EVAL   │ ────────────────►┘
              │ real traffic,   │
              │ real users      │
              └─────────────────┘
```

### Key terms

- **Evaluation (eval)** — a repeatable, scored measurement of LLM system quality on a task.
- **Offline eval** — evaluation against a fixed curated dataset, run before shipping.
- **Online eval** — evaluation on live production traffic and real users.
- **Distribution shift** — mismatch between your eval dataset and real-world inputs.
- **Eval-driven development** — a workflow where evals gate every change and grow from observed failures.
- **Feedback signal** — an implicit or explicit user behavior (thumbs, edit, retry) used as a quality proxy.

### Common misconceptions

- ❌ "If the demo works, the system works." → ✅ A demo covers a handful of happy paths; an eval covers the distribution, including edge cases and regressions.
- ❌ "Online metrics replace offline evals." → ✅ Online is the ground truth distribution but is slow and risky; offline gates changes cheaply before they reach users. You need both.
- ❌ "Evals are a one-time setup." → ✅ The eval set must grow continuously as production surfaces new failure modes; a static eval set decays in value.

### Worked example

A team ships a support chatbot. They tweak the system prompt to be "more concise."
With **no offline eval**, they deploy. Online, the thumbs-down rate climbs from 4% to
9% over three days — because conciseness dropped a required compliance disclaimer. Cost:
three days of degraded support and a postmortem.

With an **offline eval** of 200 curated tickets including 12 that require the
disclaimer, the same prompt change is caught in CI in 90 seconds: disclaimer-presence
check fails 12/12. The change never ships. The team then *also* adds the one online
failure mode they hadn't thought of to the offline set, so it is permanently guarded.

### Check questions

1. A teammate says "our offline eval passes 100% and our online thumbs-up rate is high — we don't need both, let's drop the offline suite to save CI time." Identify the flaw in this reasoning. — **Answer:** Offline and online answer different questions: offline cheaply catches *known* regressions before they reach users ("did I break anything I already know about?"); online captures the true distribution and surfaces *unknown* failures but only after users are affected and slowly/noisily. Dropping offline means every regression — even ones already in the eval set — would only be caught in production, harming users first. They are complementary stages of one loop, not redundant.
2. A change is being deployed under deadline pressure with no time to wait for an online read. Which evaluation setting can still gate it, and what is the specific risk you are accepting by relying only on that setting? — **Answer:** Only the offline eval can gate it (it is fast and runs pre-ship). The risk accepted is distribution shift: the curated dataset may not match what real users actually do, so a regression on an uncovered case can still ship undetected. Offline cannot confirm real-user impact.
3. After an online incident is fixed, simply patching the prompt is not enough — what additional step closes the loop, and what would go wrong if you skipped it? — **Answer:** The failure case (with its expected output/criteria) must be added to the offline eval dataset as a permanent regression guard. Skip it and the same failure can silently recur on a future change, because nothing tests for it — the eval set decays and the incident teaches you nothing durable.

---

## 9.2 — Golden / eval datasets — how to build them

### Concept

A **golden dataset** (or eval set) is the curated collection of inputs — and, where
applicable, reference outputs or grading criteria — that your offline evals run against.
It is the single most valuable artifact in an LLM project, more valuable than any
prompt, because prompts and models are replaceable but a well-built dataset is a durable
asset that survives model upgrades.

**What goes in a row.** At minimum: an input (the prompt or task). Often also: a
reference/expected output, or a rubric, or metadata (category, difficulty, source). Not
every row needs a reference output — for open-ended tasks you may store grading criteria
instead and rely on an LLM judge or human.

**Sourcing.** Good datasets come from, in rough order of value:
1. **Real production logs** — the actual distribution; the gold standard for realism.
2. **Real failures and bug reports** — every incident becomes a permanent test case.
3. **Hand-written cases by domain experts** — for coverage of important rare cases.
4. **Synthetic / LLM-generated cases** — cheap volume, good for stress and edge coverage, but must be reviewed because they drift from real usage and can encode the generator's blind spots.

**Coverage and stratification.** A dataset of 500 near-identical easy questions tells
you almost nothing. Deliberately **stratify**: cover categories, difficulty levels,
languages, input lengths, and especially **adversarial / edge cases** (ambiguous
queries, out-of-scope requests, prompt-injection attempts, empty inputs). Track the
distribution explicitly so you know what you are and aren't measuring.

**Size.** There is no magic number. For a sharp regression test, 50–200 well-chosen,
diverse cases often beats 5,000 redundant ones. Bigger matters more when you need to
detect *small* score differences (statistical power) or slice by many categories. Start
small and high-signal; grow toward the failures you observe.

**Hygiene.** Keep a held-out **test split** you do not look at while iterating, so you
don't overfit your prompts to the eval set (the LLM-engineering version of training on
the test set). Version the dataset (it changes over time; you must be able to compare
runs against the same version). Label provenance so you can audit and prune.

A dataset is never "done." The discipline is: every production failure, every support
escalation, every "huh, the model shouldn't have done that" becomes a new row.

### Key terms

- **Golden dataset / eval set** — curated inputs (+ optional reference outputs/rubrics) for offline evaluation.
- **Reference output (ground truth)** — the known-correct or expected answer for an input.
- **Stratification** — deliberately covering categories/difficulty/edge cases in proportion.
- **Synthetic data** — LLM- or rule-generated eval cases; cheap but require review.
- **Held-out split** — a portion of the dataset never used during iteration, to prevent overfitting.
- **Dataset versioning** — tracking dataset changes so runs are comparable.

### Common misconceptions

- ❌ "Bigger eval set is always better." → ✅ Diversity and signal matter far more than raw count; 100 well-chosen cases can beat 5,000 redundant ones.
- ❌ "Every row needs a single correct answer." → ✅ Open-ended tasks store rubrics or criteria instead; reference outputs are only one option.
- ❌ "Synthetic data is as good as real logs." → ✅ Synthetic data is cheap volume but drifts from the real distribution and inherits the generator's blind spots; review it and anchor on production data.

### Worked example

Building an eval set for a "summarize this support thread" feature:

- 60 rows sampled from real production threads (the distribution).
- 20 rows from past complaints where the summary was wrong (regression guards).
- 15 hand-written edge cases: empty thread, single message, 80-message thread, non-English thread, thread containing a prompt-injection string.
- 15 synthetic threads generated to cover product areas with low log volume — each reviewed by a support lead.
- Each row stores: the thread, a rubric (must capture the resolution, must not invent facts, ≤ 5 sentences), and metadata (product area, length bucket).
- 25% is held out as a test split; the dataset is versioned `v1`. When a new failure appears in production, it is added as `v2`.

### Check questions

1. A startup has almost no production traffic yet but must build an eval set now. Given the sourcing priority order, what should they lean on, and what is the specific danger they must actively manage? — **Answer:** With no logs, they fall back on hand-written expert cases and synthetic/LLM-generated cases (the lower-value sources). The danger: synthetic data drifts from real usage and inherits the generator's blind spots, so it can give false confidence; they must have domain experts review it, keep it stratified across edge cases, and replace it with real logs as soon as traffic appears.
2. You iterate prompts and your eval score climbs from 0.78 to 0.91 over two weeks, but production quality is flat. The eval set itself never changed. What most likely happened, and what practice would have caught it? — **Answer:** You overfit the prompts to that specific eval set (the LLM-engineering version of training on the test set) — the 0.91 reflects the dataset, not real quality. A held-out test split you never inspect while iterating would have caught it: its score would have stayed flat while the iterated split rose, exposing the overfit.
3. Two eval sets for the same feature: Set A is 5,000 cases, Set B is 150. Under what circumstance is the larger set genuinely the better choice, and when is it just redundant cost? — **Answer:** Size genuinely helps when you must detect *small* score differences (statistical power) or slice results across many categories — there a large set buys real signal. It is just redundant cost when the extra cases are near-identical easy questions: 5,000 redundant rows mostly re-measure the same thing, while 150 diverse, stratified cases give broader coverage. Diversity, not raw count, is what determines value.

---

## 9.3 — Regression testing for prompts and models

### Concept

A **regression** is a change that makes something that used to work stop working. In LLM
systems regressions are insidious because the surface that can regress is enormous and
invisible: a prompt edit, a model version bump, a temperature change, a new tool, an
updated RAG index, even an upstream provider silently re-tuning a model can all shift
behavior on inputs you never touched.

**Regression testing** means running your offline eval set on every meaningful change
and comparing scores against a known-good baseline. The rule of thumb: **no prompt or
model change ships without an eval run.** This is the LLM equivalent of "no merge
without green CI."

What counts as a change that triggers a run:
- Any edit to a prompt (system prompt, tool descriptions, few-shot examples).
- A model version change — even a "minor" point release. `gpt-5.1` → `gpt-5.2` or `claude-sonnet-4-5` → a successor can move scores in either direction.
- Sampling parameter changes (temperature, top-p, max_tokens).
- Tool definition changes, retrieval pipeline changes, chunking changes.
- Provider-side updates outside your control — which is why you also re-run periodically even with no local change.

**Comparing runs.** You need per-case results, not just an aggregate. An aggregate score
that holds steady at 0.87 can hide that 15 cases improved and 15 regressed. The
high-value view is the **diff**: which specific cases flipped pass→fail. Those are the
regressions to investigate.

**Thresholds and gates.** Decide in advance what blocks a deploy: a hard floor on the
aggregate score, *and* a rule that no critical-tagged case may regress (a safety
disclaimer test failing should block even if the average went up). Because scores are
noisy, account for variance — a 0.5% drop may be noise; a 5% drop on a stable metric is
real. Run the eval enough times (or with enough cases) that you can tell the difference.

**Integration.** Mature teams wire offline evals into CI: open a PR that changes a
prompt, the eval suite runs, the PR shows a score diff and a list of flipped cases, and
a regression on a critical case blocks the merge. This makes prompt changes reviewable
like code changes.

### Key terms

- **Regression** — a change that breaks previously-working behavior.
- **Baseline** — the known-good reference run that new runs are compared against.
- **Regression test** — running the eval set on every change and diffing against the baseline.
- **Score diff** — the per-case comparison showing which cases improved or regressed.
- **Quality gate** — a pre-defined threshold (and critical-case rule) that blocks a deploy.

### Common misconceptions

- ❌ "The aggregate score didn't drop, so nothing regressed." → ✅ An unchanged average can hide equal numbers of improvements and regressions; you must diff per case.
- ❌ "Only prompt edits need a regression run." → ✅ Model version bumps, sampling changes, retrieval changes, and even silent provider updates can all regress behavior.
- ❌ "Any score drop is a real regression." → ✅ LLM scores are noisy; you must distinguish noise-level drift from a real drop using variance/repeat runs.

### Worked example

A team upgrades from `claude-sonnet-4-5` to its successor expecting a free quality win.
The CI eval runs 220 cases:

- Aggregate score: 0.86 → 0.88 (looks like a win).
- Per-case diff: 24 cases improved, but 9 regressed — including 2 cases tagged
  `critical:pii-redaction` that flipped pass→fail because the new model formats output
  slightly differently and the redaction regex no longer matches.

The aggregate said "ship it." The diff and the critical-case gate said "block." The team
fixes the downstream parsing, re-runs, and only then upgrades. Without per-case diffing
they would have shipped a PII regression.

### Check questions

1. Your team made zero code, prompt, or config changes for a month, yet you re-run the regression eval weekly anyway. Justify this — what could regress with nothing on your side changing? — **Answer:** Providers can silently re-tune or update a hosted model behind the same version label, shifting behavior on inputs you never touched. Because you don't control that surface, periodic re-runs are the only way to detect a provider-side regression; "we changed nothing" does not mean "nothing changed."
2. After a prompt edit, the aggregate eval score is unchanged at 0.87 and a `critical:safety-disclaimer` case flipped pass→fail. Two engineers disagree on whether to ship. State the correct decision and the rule that produces it. — **Answer:** Block the deploy. A flat aggregate can hide equal numbers of improvements and regressions, so it is not evidence of "no regression." The governing rule is that no critical-tagged case may regress regardless of the aggregate; a safety-disclaimer failure blocks even if the average rose. The per-case diff, not the aggregate, is the decision input.
3. Your eval shows a 0.5% score drop after a change. Before calling it a regression, what must you establish, and what determines whether 0.5% is meaningful? — **Answer:** You must characterize the metric's noise band — its run-to-run variance — via repeat runs or a sufficiently large case count. 0.5% is meaningful only if it exceeds that noise band; on a noisy metric it is likely drift, while the same 0.5% on a stable, well-powered metric could be real. A critical-case flip is treated as real regardless of size.

---

## 9.4 — Metric types — exact match, F1, pass@k, BLEU/ROUGE and their weaknesses

### Concept

Once you have a dataset, you need a **metric**: a function that turns an output into a
score. Metrics fall on a spectrum from cheap-and-rigid to expensive-and-nuanced.

**Exact match (EM)** — output must equal the reference string exactly (often after
normalization: lowercasing, trimming whitespace, stripping punctuation). Strength:
unambiguous, free, deterministic. Weakness: only valid when there is exactly one correct
surface form — "42", multiple-choice letters, a parsed boolean. "Paris" vs "Paris,
France" both correct but EM fails one.

**F1 / token overlap** — treats output and reference as sets of tokens; **precision** =
fraction of output tokens that are in the reference, **recall** = fraction of reference
tokens recovered, **F1** = their harmonic mean. Used in extractive QA (e.g., SQuAD).
Strength: partial credit, order-insensitive. Weakness: rewards lexical overlap, not
meaning — a fluent paraphrase scores low; a word-salad with the right words scores high.

**pass@k** — for tasks with a verifiable check (code that must pass tests, a math answer
that must equal a value): generate **k** samples, the case "passes" if **at least one**
of the k is correct. `pass@1` measures single-shot reliability; `pass@10` measures
"can it get there with retries." Used heavily for code (HumanEval, SWE-bench). Strength:
honest about non-determinism and retry-based systems. Weakness: high k can mask an
unreliable model; `pass@1` and `pass@k` together tell the real story.

**BLEU / ROUGE — legacy metrics; know the names, rarely the right choice.** BLEU and
ROUGE predate modern LLMs — BLEU from machine translation, ROUGE from summarization —
and they are **largely outdated for evaluating LLM systems.** They are covered here
because you will see them referenced and must know what they are, **not** because you
should reach for them. Mechanically they count shared **n-grams** between output and
reference; their one historical strength is being cheap and reproducible. But the flaw
is fundamental: **they only see surface form.** A correct answer phrased differently
from the reference scores badly; a wrong answer that reuses the reference's n-grams
scores well; and they correlate poorly with human judgment on open-ended generation.
They retain *narrow* use for translation and tightly-constrained summarization — but for
chatbots, agents, or any free-form generation they are the wrong tool. Default to
LLM-as-judge (9.5) instead.

**Embedding-based / semantic metrics (BERTScore and kin).** Between the surface-form
n-gram metrics and a full LLM judge sits an intermediate family: metrics that compare
output and reference **in embedding space** instead of by literal token overlap. The
best-known is **BERTScore** — it embeds every token of the output and every token of the
reference with a contextual encoder, greedily matches each output token to its most
similar reference token by cosine similarity, and aggregates those similarities into
precision, recall, and F1. Because the comparison is semantic, a fluent paraphrase that
shares few exact words can still score high — exactly the case where BLEU/ROUGE fail.
Related metrics include **MoverScore** (which uses an optimal-transport "earth mover's"
distance between the two token-embedding sets) and **BLEURT** (a learned metric:
a model fine-tuned to predict human quality ratings). These reuse precisely the
embedding machinery of Topic 10 — semantic similarity via dense vectors — and were the
natural successor to n-gram overlap before LLM-as-judge became practical.

Their place on the spectrum: BERTScore-style metrics are **cheaper and more reproducible
than an LLM judge** and **far more meaning-aware than BLEU/ROUGE**, so they are useful as
a fast reference-based screen. But they keep two real limits: they still need a
**reference** output (so they cannot grade open-ended tasks with no gold answer), and
they reward *semantic similarity to the reference*, not *correctness* — an answer that is
on-topic and similarly worded but factually wrong can still score well, and a correct
answer that is structured very differently from the reference can score low. For nuanced,
reference-free, criteria-based grading you still need LLM-as-judge (9.5). Treat
embedding metrics as the rung between n-gram overlap and a judge: use them when you have
references and want a quick semantic check, not as the final word on quality.

**The general principle:** match the metric to the task. Closed-form answer → exact
match. Extractive span → F1. Verifiable code/math → pass@k. Reference exists and you want
a cheap semantic check → embedding-based metric (BERTScore). Open-ended quality with no
reference → reference overlap fails; you need **LLM-as-judge** or human eval (9.5+). Many
real systems use a **layered** approach: cheap deterministic checks first (did it return
valid JSON? is the disclaimer present? is it under the length cap?), then expensive
judge-based checks for the nuanced quality dimensions.

The whole metric family side by side — match the metric to the task:

| Metric | Needs a reference? | What it measures | Best for | Key weakness |
|---|---|---|---|---|
| Exact match | Yes | String equality after normalization | Closed-form answers with one canonical surface form (categories, MCQ letters, booleans) | Fails any correct answer in a different surface form |
| F1 / token overlap | Yes | Token-set precision/recall overlap | Extractive QA / answer spans (partial credit) | Rewards lexical overlap, not meaning; paraphrase scores low, word-salad scores high |
| pass@k | No (needs a verifiable check) | Whether ≥1 of k samples passes a test/verifier | Verifiable code or math | High k masks an unreliable model; report pass@1 too |
| BLEU / ROUGE | Yes | Shared n-gram counts (surface form) | Translation, tightly-constrained summarization (legacy) | Sees only surface form; correlates poorly with human judgment on free-form text |
| BERTScore | Yes | Contextual-embedding similarity of tokens | A fast semantic screen when references exist | Measures similarity to the reference, not correctness; still needs a reference |
| LLM-as-judge | No | Nuanced quality against criteria/rubric | Open-ended, reference-free quality grading | Noisy, biased, must be calibrated; cost and latency |

### Key terms

- **Metric** — a function mapping an output (and reference) to a score.
- **Exact match (EM)** — output must equal the reference exactly after normalization.
- **Precision / recall / F1** — token-overlap measures; F1 is their harmonic mean.
- **pass@k** — at least one of k samples is correct; measures retry reliability.
- **BLEU** — precision-oriented n-gram overlap metric (machine translation origin).
- **ROUGE** — recall-oriented n-gram overlap metric (summarization origin).
- **BERTScore** — an embedding-based metric: matches output and reference tokens by contextual-embedding cosine similarity, aggregated into precision/recall/F1.
- **Embedding-based / semantic metric** — a reference-based metric that compares meaning in embedding space rather than by literal n-gram overlap.
- **Reference-based metric** — any metric that compares output to a stored reference.

### Common misconceptions

- ❌ "A high BLEU/ROUGE score means a good answer." → ✅ They measure n-gram overlap with a reference, not correctness or quality; a wrong answer reusing reference words can score high and a correct paraphrase can score low.
- ❌ "Exact match works for any QA task." → ✅ It only works when there is a single canonical surface form; open or multi-form answers need a more forgiving metric.
- ❌ "pass@10 is the headline number." → ✅ A high pass@10 with low pass@1 means the model is unreliable single-shot; report both.
- ❌ "BERTScore measures meaning, so a high BERTScore means the answer is correct." → ✅ It measures *semantic similarity to the reference*, not correctness; a wrong but on-topic, similarly-worded answer can still score high, and it still needs a reference. Reference-free, criteria-based correctness still needs a judge.

### Worked example

Reference answer: *"The mitochondrion is the powerhouse of the cell."*
Model output: *"Cells get most of their energy from mitochondria."*

- **Exact match:** 0 (different strings) — misleading; the answer is correct.
- **F1 / token overlap:** low — shared tokens are roughly "the", "of", "cell(s)",
  "mitochond-"; a fluent correct paraphrase is penalized.
- **BLEU/ROUGE:** low for the same reason.
- **BERTScore:** high — the contextual embeddings of "powerhouse of the cell" and "get
  most of their energy from mitochondria" are semantically close, so the meaning-aware
  match scores the paraphrase well where n-gram overlap did not. (Note this still
  required the reference, and would *also* score a confidently-wrong-but-on-topic answer
  fairly high.)
- **LLM-as-judge with the rubric "is the answer factually correct?":** "Correct" — the
  right tool for this open-ended case.

For a code task instead — "write `is_prime(n)`" — the right metric is `pass@1`: run the
generated function against a hidden test suite; it passes or it doesn't, regardless of
how the code is phrased.

### Check questions

1. For each task, name the most appropriate metric and justify it in one clause: (a) classify a ticket into one of five fixed categories, (b) extract the answer span from a passage, (c) generate a SQL query that must run and return the right rows, (d) write an empathetic apology to a customer. — **Answer:** (a) exact match — one canonical surface form (the category label); (b) F1 / token overlap — partial credit for a partially-correct span; (c) pass@k — verifiable by executing the query against expected output; (d) LLM-as-judge / human — open-ended, no reference surface form, quality is the harmonic of tone and content. Credit for matching each and a correct one-clause reason.
2. A model's answer to "What is the capital of Australia?" is "Canberra is Australia's capital city." The reference is "Canberra." Exact match scores it 0. Is exact match the wrong metric here, or is the dataset at fault — and what is the fix? — **Answer:** The dataset/metric pairing is at fault, not the model. Exact match is only valid when there is a single canonical surface form; "Canberra is Australia's capital city" is correct but a different surface form. Fix: normalize more aggressively, store accepted surface variants, or switch to a forgiving metric (e.g., F1, or a judge for "does the answer contain the right capital?"). The answer is correct; the eval is miscalibrated.
3. A code model reports pass@1 = 0.45 and pass@10 = 0.95. A teammate wants to headline "95% accurate." What does the gap actually tell you, and how should the system be described honestly? — **Answer:** The gap means the model is unreliable single-shot — it usually needs several attempts to land a correct solution. pass@10 = 0.95 only holds if the system actually samples 10 times and has a verifier to pick the passing one. Honest description: "single-shot success ~45%; with up to 10 sampled attempts and test-based selection, ~95%." Reporting only pass@10 hides the dependence on retries.
4. A team grades a summarization feature with BLEU and gets low scores even on summaries a human calls excellent. A colleague proposes switching to BERTScore; another proposes LLM-as-judge. Explain what each switch fixes and what it does not. — **Answer:** BLEU's low scores reflect surface-form n-gram mismatch — an excellent summary worded differently from the reference is penalized. BERTScore fixes *that*: it compares output and reference tokens by contextual-embedding similarity, so a semantically close paraphrase scores well; it is also cheap and reproducible. But BERTScore still needs a reference and still measures *similarity to the reference*, not correctness or rubric compliance — a wrong-but-on-topic summary can score high. LLM-as-judge fixes *that*: it can grade reference-free against explicit criteria (faithful? complete? concise?). The right answer depends on whether good references exist and whether they need correctness/criteria grading: BERTScore is the cheap semantic screen, the judge is for nuanced criteria-based quality.

---

## 9.5 — LLM-as-judge — single-output scoring vs. pairwise comparison

### Concept

For open-ended outputs, reference-overlap metrics fail and human grading doesn't scale.
**LLM-as-judge** uses a capable model as the grader: you give it the input, the
output(s), and grading instructions, and it returns a score or a verdict. It has become
the default eval method for free-form generation because it captures nuance (relevance,
tone, faithfulness) at machine speed and cost.

There are two main modes.

**Single-output scoring (pointwise / direct scoring).** The judge sees one output and
rates it — e.g., a 1–5 score, or pass/fail against criteria. Strengths: simple, scales
to one output per row, gives an absolute number. Weakness: LLMs are **poorly
calibrated** at absolute scoring — "is this a 6 or a 7 out of 10?" is unstable across
runs, across judges, and drifts with prompt wording. Single scores are usable but you
should expect noise, prefer small scales (1–3 or 1–5 over 1–100), and anchor each point
with an explicit description.

**Pairwise comparison.** The judge sees *two* outputs (A and B) for the same input and
picks the better one (or "tie"). This is far more reliable, because **relative judgment
is easier and more stable than absolute judgment** — humans and LLMs alike are better at
"which is better" than "rate this in isolation." Pairwise is the standard for comparing
two prompts, two models, or a candidate vs. a baseline. You aggregate many pairwise
verdicts into a **win rate** ("the new prompt beats the baseline 63% of the time") or an
**Elo / Bradley-Terry rating** when comparing many systems (this is how LMArena ranks
models).

When to use which: use **pairwise** when you are comparing two candidates and want a
sharp, low-noise signal (A/B of prompts, model selection, regression-vs-baseline). Use
**single-output scoring** when you need an absolute quality bar with no obvious
comparison point (production monitoring: "is this response acceptable, yes/no"), or when
you have thousands of outputs and pairing them all is impractical.

Both modes share a key best practice: make the judge **reason before it verdicts**.
Ask for a brief rationale first, then the score/choice (chain-of-thought). This improves
quality and — critically — gives you something to audit when the judge is wrong.
Pairwise also requires a specific bias mitigation (position bias, see 9.8): swap A and B
and run twice.

The two modes side by side — note pairwise runs the judge twice with positions swapped:

```
  POINTWISE (single-output scoring)      PAIRWISE (comparison)

   input + output                         input + {output A, output B}
        │                                      │
        ▼                                      ▼
   ┌─────────┐                            ┌─────────┐  A first
   │  JUDGE  │                            │  JUDGE  │ ──────────► winner₁
   └─────────┘                            └─────────┘
        │                                      │
        ▼                                 ┌─────────┐  B first (swap)
   score 1–5                              │  JUDGE  │ ──────────► winner₂
   (absolute,                             └─────────┘
    poorly calibrated)                         │
                                               ▼
                                        winner₁ == winner₂ ?
                                          yes → genuine win
                                          no  → position bias → tie
```

### Key terms

- **LLM-as-judge** — using an LLM to grade the output(s) of another LLM.
- **Single-output / pointwise scoring** — judge rates one output on an absolute scale.
- **Pairwise comparison** — judge picks the better of two outputs for the same input.
- **Win rate** — fraction of pairwise comparisons a system wins.
- **Elo / Bradley-Terry rating** — a rating derived from many pairwise matchups.
- **Judge rationale** — the judge's written reasoning, requested before the verdict for quality and auditability.

### Common misconceptions

- ❌ "An LLM can reliably score quality 1–100." → ✅ LLMs are badly calibrated at fine-grained absolute scoring; small anchored scales and especially pairwise comparison are far more reliable.
- ❌ "Pairwise and pointwise are interchangeable." → ✅ Pairwise gives a sharper, lower-noise signal for comparing two candidates; pointwise is needed for absolute production monitoring. Choose by purpose.
- ❌ "Ask the judge for just the score to save tokens." → ✅ Forcing a rationale first improves accuracy and makes wrong verdicts auditable; the tokens are worth it.

### Worked example

Comparing prompt A (current) vs prompt B (candidate) over 200 cases.

*Pointwise:* judge scores each output 1–5. A averages 4.1, B averages 4.2. Is +0.1 real
or noise? Hard to say — single scores are noisy.

*Pairwise:* for each of the 200 inputs, the judge sees both outputs and picks a winner;
each pair is run twice with positions swapped. Result: B wins 121, A wins 64, 15 ties →
B's win rate (excluding ties) ≈ 65%. With 185 decisive comparisons that is well outside
chance — a clear, defensible signal that B is better. The pairwise framing extracted a
sharp answer where pointwise gave a murky one.

### Check questions

1. You run a pointwise eval: prompt A scores 4.1/5, prompt B scores 4.2/5. A colleague says "B wins, ship it." Why is this conclusion unsafe, and what eval mode would give a defensible answer? — **Answer:** LLM judges are poorly calibrated at fine-grained absolute scoring, so a 0.1-point pointwise gap is well within noise — it is not evidence B is better. A pairwise comparison (same input, both outputs, judge picks the winner, positions swapped) extracts a sharp, low-noise signal because relative judgment is far more stable than absolute; aggregate it into a win rate to get a defensible verdict.
2. Pairwise comparison is more reliable, yet a production monitoring system that scores live responses one at a time cannot use it. Explain why, and what that monitoring system should use instead. — **Answer:** Pairwise needs two outputs for the *same* input to compare; a live monitoring stream has only one output per request and no natural comparison candidate. It must use single-output (pointwise) scoring against an absolute bar ("is this response acceptable?"), accepting the higher noise — ideally with a small anchored scale and a rationale to make it as stable as possible.
3. A team strips the rationale from their judge prompt to cut token cost, asking only for the final score. Beyond a possible accuracy drop, what diagnostic capability have they lost, and when will they regret it? — **Answer:** They lose auditability: with only a bare score, when the judge disagrees with humans during calibration (or produces a surprising verdict) there is no way to see *why* — e.g., that it actually rewarded length or formatting. They regret it the moment they must debug or calibrate the judge, because they cannot tell a substantive judgment from a biased one.

---

## 9.6 — Rubric-based evaluation — decomposing quality into scored dimensions

### Concept

Asking a judge "is this a good answer, 1–5?" lumps everything together: a 3 could mean
"accurate but rude" or "polite but wrong." **Rubric-based evaluation** fixes this by
**decomposing quality into explicit, independently scored dimensions**, each with
described criteria.

A rubric for a support reply might be:
- **Factual accuracy** — no incorrect claims (0–2).
- **Completeness** — addresses every part of the question (0–2).
- **Faithfulness / grounding** — claims are supported by the provided context, no hallucination (0–2).
- **Tone** — professional and empathetic (0–1).
- **Format compliance** — under length cap, includes required disclaimer (binary).

Why this is better:

1. **Reliability.** A narrow, well-defined question ("does the reply contain any factual
   error?") is far easier for a judge to answer consistently than a vague holistic one.
   Each dimension is closer to a checkable fact.
2. **Diagnostic power.** When the score drops you see *which* dimension fell. "Accuracy
   held, faithfulness dropped 30%" points straight at a retrieval/hallucination problem.
   A single blended number tells you nothing actionable.
3. **Weighting and gating.** Different dimensions matter differently. You can weight a
   final score, or gate: any factual error or missing disclaimer fails the case outright
   regardless of tone. This encodes real priorities.
4. **Alignment with stakeholders.** A rubric is human-readable; product, legal, and
   support can review and edit "what good looks like" directly.

**Best practices for rubrics:** keep each dimension independent (avoid overlap that
double-counts); use small anchored scales with each level described concretely
("2 = no errors; 1 = one minor error; 0 = a material error"); make as many dimensions as
possible **binary or near-binary** since binary checks are the most reliable; separate
**objective** dimensions (format, disclaimer presence — often checkable with code, no
LLM needed) from **subjective** ones (tone, helpfulness — need a judge). Use code for
what code can do; reserve the judge for genuine judgment.

Rubric-based evaluation is the practical bridge between rigid metrics (9.4) and
holistic LLM-as-judge (9.5): it brings the structure and reliability of rigid metrics to
the nuanced quality that needs a judge.

### Key terms

- **Rubric** — a set of explicit, independently-scored quality dimensions with described criteria.
- **Dimension** — one scored axis of quality (accuracy, tone, faithfulness, format).
- **Anchored scale** — a scoring scale where each numeric level has a concrete description.
- **Gating dimension** — a dimension whose failure fails the whole case regardless of others.
- **Objective vs. subjective dimension** — checkable-by-code vs. needs-a-judge criteria.

### Common misconceptions

- ❌ "A single holistic 1–5 score is enough." → ✅ It blends independent qualities into one unstable number with no diagnostic value; decomposed dimensions are more reliable and tell you what failed.
- ❌ "Every rubric dimension needs an LLM judge." → ✅ Objective dimensions (length cap, disclaimer presence, valid JSON) should be checked with deterministic code — cheaper and exact.
- ❌ "More dimensions is always better." → ✅ Overlapping dimensions double-count and add noise; dimensions should be independent and minimal.

### Worked example

Two support replies both get a holistic score of "3/5" from a naive judge. A rubric
disambiguates them:

| Dimension | Reply A | Reply B |
|---|---|---|
| Accuracy (0–2) | 2 | 0 (states wrong refund window) |
| Completeness (0–2) | 1 (skips the follow-up step) | 2 |
| Faithfulness (0–2) | 2 | 1 |
| Tone (0–1) | 1 | 1 |
| Disclaimer present (gate) | yes | yes |
| **Total / verdict** | 6/7 — acceptable | 4/7 but **accuracy gate failed → FAIL** |

The holistic "3 vs 3" hid that B is unshippable (factually wrong) while A just needs a
small completeness fix. The rubric made the difference visible and actionable.

### Check questions

1. Over two eval runs, a feature's blended holistic score is stable at 3.5/5, but users are complaining more. The same set of cases is used both times. How could a rubric-based eval have revealed what the holistic score concealed? — **Answer:** A holistic score blends independent qualities, so accuracy can fall while tone rises and the blend stays flat. A rubric scores each dimension separately, so the run-over-run diff would show, e.g., "faithfulness dropped 30%, tone rose" — pinpointing a grounding/hallucination regression that the single number averaged away. Decomposition gives the diagnostic signal a blended score cannot.
2. An engineer proposes a 6-dimension rubric where one dimension is "valid JSON?" and another is "professional and empathetic tone?". Critique how each of those two should be implemented and why they differ. — **Answer:** "Valid JSON?" is an objective dimension — deterministically checkable, so it should be a code check (a parser), which is cheaper, exact, and noise-free; using an LLM judge for it wastes money and adds noise. "Professional and empathetic tone?" is subjective — it genuinely needs a judge. The principle: use code for what code can verify, reserve the judge for genuine judgment.
3. A support-reply rubric scores Reply X: accuracy 0 (states the wrong refund window), completeness 2, faithfulness 1, tone 1. Its weighted total looks "okay." Should it pass? Justify using the rubric design. — **Answer:** No — it must fail. Accuracy should be a gating dimension: a material factual error fails the whole case regardless of how high other dimensions score, because shipping a factually wrong refund window is unacceptable no matter how complete or polite the reply is. A weighted average that lets tone and completeness rescue a wrong answer encodes the wrong priorities; gating encodes the real one.

---

## 9.7 — Judge-model ensembles / panels

### Concept

A single judge model is itself a noisy, biased instrument. It has idiosyncrasies, it can
be confidently wrong, and — most awkwardly — it tends to favor outputs from its own model
family (self-preference bias, 9.8). A **judge ensemble** (also called a **panel** or
**jury**) reduces this by having multiple judges evaluate each output and aggregating
their verdicts.

The idea mirrors classic ensembling and the wisdom-of-crowds principle: **independent
errors partially cancel.** If three judges each have somewhat different blind spots, the
cases where all three agree are high-confidence, and the cases where they split are
exactly the ambiguous or hard cases worth a human look.

**How to build a panel.** The judges should be **diverse** to maximize error
independence — ideally different model families (e.g., a Claude model, a GPT model, a
Gemini model), or at minimum different sizes/versions or different judge prompts. Three
judges from the same family with the same prompt is barely an ensemble; their errors are
correlated. A research-supported pattern is the **Panel of LLM Evaluators (PoLL)**:
several smaller, cheaper, diverse judges often **outperform a single large judge** while
costing less in total and notably reducing intra-model bias. The PoLL paper specifically
reported a panel that correlated better with human judgement than a single GPT-4 judge
while being over seven times cheaper. [1]

**Aggregation.** Common schemes:
- **Majority vote** — for pairwise/categorical verdicts; the verdict most judges chose.
- **Mean / median score** — for numeric scores; median is more robust to one outlier judge.
- **Unanimity gate** — require all judges to agree to "pass"; any disagreement routes to human review. Conservative; good for high-stakes evals.
- **Weighted aggregation** — weight judges by how well each correlates with humans (ties to calibration, 9.10).

**Costs and trade-offs.** A panel multiplies cost and latency by the number of judges. It
is most worth it for (a) high-stakes decisions like model-selection or release gates,
(b) reducing self-preference bias when one of your candidate systems shares a family with
your judge, and (c) surfacing ambiguous cases via disagreement. For cheap routine
regression runs, a single well-calibrated judge is often fine. A common hybrid: single
judge for the routine CI run, full panel for the release-gate decision.

The judge-disagreement rate is itself a useful signal: high disagreement means the task
is genuinely ambiguous or your rubric is underspecified — fix the rubric.

A panel fans one output out to several diverse judges, then aggregates their verdicts:

```
                              ┌───────────────────┐
                         ┌───►│ JUDGE 1 (Claude)  │───┐  verdict₁
                         │    └───────────────────┘   │
                         │    ┌───────────────────┐   │
   one output ───────────┼───►│ JUDGE 2 (GPT)     │───┼──►┌────────────┐
   (+ input, rubric)     │    └───────────────────┘   │   │ AGGREGATE  │──► final
                         │    ┌───────────────────┐   │   │ majority / │   verdict
                         └───►│ JUDGE 3 (Gemini)  │───┘   │ median /   │
                              └───────────────────┘       │ unanimity  │
                                                          └────────────┘
            diverse families → independent errors          split → flag
                                                            for human review
```

### Key terms

- **Judge ensemble / panel / jury** — multiple judge models evaluating the same output.
- **Panel of LLM Evaluators (PoLL)** — using several small, diverse judges instead of one large one.
- **Error independence** — the property that diverse judges' mistakes are uncorrelated, so aggregation cancels them.
- **Majority vote / median aggregation** — combining categorical / numeric verdicts.
- **Unanimity gate** — requiring all judges to agree, else escalate to human.
- **Disagreement rate** — fraction of cases where judges disagree; a signal of ambiguity.

### Common misconceptions

- ❌ "Any three judges form an ensemble." → ✅ Three judges from one family with one prompt have correlated errors; the benefit comes from *diverse* judges with independent errors.
- ❌ "A panel of small models is worse than one big judge." → ✅ Research (PoLL) shows a diverse small-model panel can beat a single large judge in agreement with humans, at lower cost and bias.
- ❌ "Always use a panel." → ✅ Panels multiply cost/latency; reserve them for high-stakes gates and self-preference-prone comparisons; a calibrated single judge suffices for routine CI.

### Worked example

A team must choose between their fine-tuned GPT-based assistant and a Claude-based one.
Their default judge is a GPT model — using it alone risks self-preference toward the
GPT candidate.

They run a **3-judge panel**: a Claude model, a Gemini model, and a GPT model, pairwise,
positions swapped. Aggregation: majority vote per case; ties and 2–1 splits flagged.

- Unanimous verdicts (3–0) on 140/200 cases — high confidence.
- 2–1 splits on 45 cases — reviewed; the rubric is sharpened on a recurring ambiguity.
- 15 ties — genuinely close.

**Computing the win rate explicitly.** A per-case verdict is decided by majority vote,
so the 200 cases resolve as: 140 unanimous + 45 reconciled splits = 185 *decisive*
comparisons, plus 15 ties. Suppose the verdicts break down so the Claude-based candidate
takes 117 of the 185 decisive cases and the GPT-based candidate takes 68. Ties are
excluded from the denominator (they decide nothing). The win rate is then:

```
  win rate = wins / decisive comparisons
           = 117 / (117 + 68)
           = 117 / 185
           ≈ 0.632   →   63%
```

So the Claude-based candidate wins **≈ 63%** of decisive comparisons — a clear,
defensible margin over a coin-flip (50%) across 185 cases. The panel both removed the
self-preference risk and surfaced a rubric gap a single judge would have hidden.

### Check questions

1. A team builds a "panel" of three judges: GPT-4o at temperature 0.2, 0.5, and 0.8 — same model, same prompt, different temperatures. They expect ensemble benefits. Will they get them? Explain. — **Answer:** Largely no. The ensemble benefit comes from *error independence* — diverse judges making *uncorrelated* mistakes that partially cancel on aggregation. Three instances of the same model with the same prompt share the same blind spots and biases; varying temperature does not decorrelate their errors. To get a real panel they need different model families (or at least different prompts/sizes), not different sampling temperatures.
2. Across 200 cases your 3-judge panel agrees unanimously on 150 and splits 2–1 on 50. A teammate wants to just majority-vote all 200 and move on. What is the more valuable use of those 50 split cases? — **Answer:** The 50 disagreements are a signal, not just noise to vote away. A high disagreement rate means those cases are genuinely ambiguous or — more usefully — the rubric is underspecified. The split cases should be reviewed by a human and used to *sharpen the rubric*, which improves every future run. Majority-voting and moving on discards the diagnostic information.
3. Your CI runs a prompt-regression eval hundreds of times a week; your annual model-selection decision runs once. Argue why a single judge may suit one and a panel the other, referencing cost and the specific risks. — **Answer:** Routine CI: a calibrated single judge is appropriate — it is cheap and fast, and the per-run stakes are low (a missed nuance is caught next run). A panel multiplying cost/latency by 3 every run is hard to justify. Model selection: a panel is worth it — the decision is high-stakes and one-off, and crucially if a candidate shares a model family with the judge, self-preference bias would skew the choice; a diverse panel removes that and surfaces ambiguous cases. Match the spend to the stakes.

---

## 9.8 — Judge biases — position, verbosity, self-preference — and mitigations

### Concept

LLM judges are not neutral measuring instruments. They have systematic, well-documented
biases. If you don't mitigate them, your eval measures the bias instead of the quality.

**Position bias.** In pairwise comparison the judge tends to favor whichever answer is
in a particular slot (often the first, sometimes the second), independent of content.
*Mitigation:* run each comparison **twice with positions swapped**. If the judge picks A
both times → genuine win for A. If it flips with position → it's biased on that case;
count it a **tie** (or escalate). This doubles judge calls but is non-negotiable for
trustworthy pairwise evals.

**Verbosity / length bias.** Judges systematically prefer longer, more detailed answers,
even when the extra length adds nothing or pads with fluff. A model can "win" evals just
by being wordier. *Mitigations:* explicitly instruct the judge to ignore length and not
reward verbosity; control for length (compare answers of similar length, or normalize);
add a conciseness dimension to the rubric so padding is penalized, not rewarded.

**Self-preference / self-enhancement bias.** A judge tends to rate outputs from its own
model family higher — it "likes" text that looks like its own. This quietly biases
model-selection evals when a candidate shares a family with the judge. *Mitigations:*
use a judge from a *different* family than any candidate; use a diverse panel (9.7) so
no single family dominates; calibrate against humans (9.10).

Other biases to know: **sycophancy / authority bias** (the judge defers to text that
sounds confident or cites authority, even if wrong); **formatting bias** (preference for
nicely formatted/markdown answers); **nepotism toward the rubric's phrasing** (an answer
that echoes the rubric's words scores higher). **Anchoring** — if you show the judge a
reference answer, it over-weights surface similarity to it.

**General mitigations that apply broadly:**
- Force a **rationale before the verdict** — exposes when the "reasoning" is actually
  "it was longer."
- **Decompose with a rubric** (9.6) — narrow questions are harder to bias than holistic
  ones.
- **Calibrate against humans** (9.10) — measure the bias and correct for it; if you
  never check against humans you will never know your judge is gameable.
- **Randomize / blind** — strip model identity, randomize order, equalize formatting
  before judging.

The meta-point: a judge is a model with its own failure modes. Treat the judge itself as
something to be evaluated and validated, not trusted blindly.

### Key terms

- **Position bias** — favoring an answer by its slot (first/second) in a pairwise comparison.
- **Verbosity / length bias** — favoring longer answers regardless of added value.
- **Self-preference bias** — a judge rating its own model family's outputs higher.
- **Sycophancy / authority bias** — deferring to confident or authoritative-sounding text.
- **Order swapping** — running a pairwise comparison twice with A/B positions reversed.
- **Blinding** — removing model identity / equalizing formatting before judging.

### Common misconceptions

- ❌ "A capable judge is objective." → ✅ All LLM judges show systematic biases — position, verbosity, self-preference — that must be actively mitigated.
- ❌ "Position bias is small; one comparison is fine." → ✅ Position bias is large enough to flip verdicts; you must swap positions and treat flips as ties.
- ❌ "Longer, more detailed answers really are better." → ✅ Sometimes — but judges over-reward length specifically; you must control for it or you optimize for verbosity, not quality.

### Worked example

A team evals two summarizers with a single pairwise judge, A always shown first.
Result: A wins 78%. They almost ship A.

A reviewer insists on mitigations:
- **Swap positions:** rerun with B first. Now B wins 70%. The "winner" flipped entirely
  with position → most of that 78% was **position bias**, not quality.
- **Length control:** A's summaries average 180 words, B's 95. They re-prompt the judge
  to ignore length and add a conciseness dimension. After controlling, the swapped-and-
  averaged result is a near tie, slight edge to B for concision.

The original eval would have shipped the wrong system for the wrong reason. The true
signal only emerged after mitigating position and verbosity bias.

### Check questions

1. In a pairwise eval you run each comparison twice with positions swapped. On case 17 the judge picks answer A in both runs (A shown first, then A shown second). On case 18 the judge picks whichever answer is in the *first* slot in both runs. Interpret each and state the verdict. — **Answer:** Case 17: the judge picked the same *answer* (A) regardless of where it sat — a genuine, position-independent win for A. Case 18: the judge picked whatever was in the first slot, so its choice flipped with position — that is pure position bias, not a quality judgment, and the case must be scored a tie (or escalated). The key discrimination: a real win means consistency on the *answer*, not on the *slot*.
2. You evaluate a candidate model against a baseline using a judge from the *same family* as the candidate, and the candidate wins decisively. Why can you not trust this result, and what is the cheapest credible fix? — **Answer:** Self-preference bias: a judge tends to rate text from its own model family higher independent of quality, so the decisive win may reflect family kinship, not better answers. Cheapest credible fix: re-run with a judge from a *different* family than either candidate (or a diverse panel so no family dominates); calibration against humans would also reveal the inflation.
3. A judge consistently prefers the longer answer. You add the instruction "ignore length" but the preference persists. Name two further mitigations and explain why instruction alone was insufficient. — **Answer:** Instruction alone is weak because verbosity bias is a learned tendency, not a conscious rule the model reliably follows. Stronger mitigations: (1) control for length — compare answers of similar length or normalize — so the bias has nothing to act on; (2) add an explicit conciseness dimension to the rubric so padding is *penalized* rather than rewarded; (3) force a rationale so a length-driven verdict becomes visible. Combining a structural control with a rubric beats relying on a request.

---

## 9.9 — Human evaluation and inter-annotator agreement

### Concept

For genuinely subjective or high-stakes quality, **human evaluation is the ground
truth** — the standard everything else (metrics, LLM judges) is ultimately validated
against. Humans assess nuance, domain correctness, safety, and "would a real user be
satisfied" in ways no automated metric fully captures.

But human eval is **slow, expensive, and itself noisy.** Two qualified annotators can
disagree on the same output — because the task is genuinely ambiguous, the guidelines
are unclear, or the annotators interpret the rubric differently. So a single human label
is not automatically "truth"; you must measure how much your humans agree.

**Inter-annotator agreement (IAA)** quantifies this. Have multiple annotators label the
same items and measure consistency. Don't use raw percent agreement — it ignores
agreement that happens by chance (on a binary task two random labelers agree 50% of the
time). Use **chance-corrected** statistics:

- **Cohen's kappa (κ)** — agreement between two annotators, corrected for chance.
- **Fleiss' kappa** — generalizes kappa to more than two annotators.
- **Krippendorff's alpha (α)** — flexible: handles any number of annotators, missing
  data, and ordinal/interval scales.

Rough interpretation of kappa (the widely-cited Landis & Koch scale): < 0.2 slight,
0.2–0.4 fair, 0.4–0.6 moderate, 0.6–0.8 substantial, > 0.8 almost perfect (use as a
guide, not gospel; these thresholds are arbitrary conventions and vary by field). [2]
κ around 0 means agreement is no better than chance — your "ground truth" is random.

The Landis & Koch scale as a lookup table:

| Kappa (κ) range | Agreement level |
|---|---|
| < 0.00 | Poor (worse than chance) |
| 0.00 – 0.20 | Slight |
| 0.21 – 0.40 | Fair |
| 0.41 – 0.60 | Moderate |
| 0.61 – 0.80 | Substantial |
| 0.81 – 1.00 | Almost perfect |

**What low agreement means and how to fix it.** Low IAA is usually not bad annotators —
it is a **bad rubric**. Mitigations: write detailed annotation guidelines with examples;
run a calibration round (everyone labels the same 20 items, discuss disagreements,
refine the rubric); decompose holistic judgments into narrow questions (the rubric
principle from 9.6 applies to humans too); use pairwise rather than absolute scoring
(easier for humans, just as for judges); aggregate multiple annotators per item
(majority vote / median) for the final label.

**Process best practices:** at least 2–3 annotators on overlapping items so you *can*
compute IAA; report IAA alongside results — a result built on κ = 0.1 labels is not
trustworthy; have an adjudication step for disagreements. Human eval is too costly to run
on everything, so use it strategically: on a small high-value sample, on release
candidates, and — crucially — to **calibrate and validate the LLM judge** (9.10) so the
judge can scale.

### Key terms

- **Human evaluation** — humans grading outputs; the ground truth for subjective quality.
- **Inter-annotator agreement (IAA)** — measured consistency between annotators on the same items.
- **Cohen's kappa** — chance-corrected agreement for two annotators.
- **Fleiss' kappa** — chance-corrected agreement for many annotators.
- **Krippendorff's alpha** — flexible chance-corrected agreement (any annotators, scales, missing data).
- **Calibration round** — annotators labeling shared items and refining the rubric to align.
- **Adjudication** — resolving annotator disagreements into a final label.

### Common misconceptions

- ❌ "A human label is automatically the ground truth." → ✅ Humans disagree; a single label is noisy. Trustworthy ground truth requires multiple annotators and measured agreement.
- ❌ "90% raw agreement means the labels are reliable." → ✅ Raw percent agreement ignores chance; on a skewed binary task 90% can correspond to κ near 0. Use chance-corrected kappa/alpha.
- ❌ "Low agreement means you hired bad annotators." → ✅ It usually means the rubric/guidelines are ambiguous; fix the rubric and run a calibration round.

### Common misconception bonus

- ❌ "Human eval should replace automated eval." → ✅ It is too slow/expensive to run on everything; use it on a strategic sample and to validate the cheap automated judge that does scale.

### Worked example

A team has 5 annotators rate chatbot answers "helpful / not helpful." They report 88%
agreement and call the labels solid. A reviewer computes **Fleiss' kappa = 0.18** —
barely above chance, because ~85% of answers are "helpful," so annotators agree mostly
by guessing the majority class.

Diagnosis: "helpful" is undefined. They run a calibration round on 25 shared items,
discover annotators disagree on whether a *correct but terse* answer is "helpful," and
rewrite the rubric into three narrow yes/no questions: *answers the question? / factually
correct? / actionable?* On a re-label, Fleiss' kappa rises to 0.71 — substantial. Only
now are the labels trustworthy enough to serve as ground truth for calibrating the LLM
judge.

### Check questions

1. A binary "safe / unsafe" labeling task has 97% of items genuinely "safe." Two annotators who both just guess "safe" every time will show ~94% raw agreement. Use this to explain why raw percent agreement is misleading and what statistic fixes it. — **Answer:** On a skewed task, agreeing by always picking the majority class produces very high raw agreement that reflects the class imbalance, not real reliability — the two annotators agreed without judging anything. Chance-corrected statistics (Cohen's/Fleiss' kappa, Krippendorff's alpha) subtract the agreement expected by chance given the class distribution, so the always-"safe" annotators would score κ near 0, correctly exposing that the labels carry no information.
2. You compute Krippendorff's alpha = 0.22 on a 5-annotator eval and your manager says "fire the worst annotators." Why is that the wrong first move, and what should you do instead? — **Answer:** Low agreement is usually a *rubric* problem, not a people problem — the guidelines are ambiguous so qualified annotators interpret them differently. Firing annotators changes nothing if the rubric stays vague. Instead: run a calibration round (everyone labels shared items, discuss disagreements), add worked examples, and decompose the holistic judgment into narrow yes/no questions; then re-label and re-measure.
3. You want an LLM judge to scale your eval. Explain the specific role human evaluation plays in making that possible, given that human eval is too expensive to run on everything. — **Answer:** Human eval can't grade everything, but it serves as the *ground truth* on a strategic sample used to *calibrate and validate the LLM judge*. Once the judge is shown to agree with humans at roughly the human-human level, it can be trusted to scale the measurement cheaply across all cases. Human eval is the anchor; the validated judge is the lever.

---

## 9.10 — Eval calibration — validating the judge against humans

### Concept

An LLM judge is only useful if it agrees with what humans consider good. **Eval
calibration** is the process of *validating the judge against human ground truth* before
you trust it to gate decisions. Skipping this is the single most common eval mistake:
teams build an elaborate LLM-judge pipeline and never check whether its scores mean
anything.

**The calibration procedure:**

1. Build a **calibration set** — a sample of representative outputs (cover easy, hard,
   and edge cases; include known-good and known-bad).
2. Get **human labels** for it — multiple annotators, with IAA measured (9.9). This is
   the reference.
3. Run the **LLM judge** over the same set.
4. **Compare** judge verdicts to human labels and quantify the agreement.

**Which agreement statistic.** Depends on the verdict type:
- Categorical (pass/fail, pairwise A/B): **Cohen's kappa** or raw + chance-corrected
  agreement.
- Ordinal/numeric scores: **Spearman** or **Kendall** rank correlation (do judge and
  humans *rank* outputs the same way?), and look at the **confusion pattern** — does the
  judge systematically score high or low?

A key benchmark: the judge should agree with humans **about as well as humans agree with
each other.** If human-human kappa is 0.7, a judge at 0.65 is excellent — you cannot
expect a judge to beat the noise ceiling of the humans themselves. If human-human
agreement is 0.7 but human-judge is 0.3, the judge is not measuring what you care about.

**What to do with the result.** If agreement is high → trust the judge for automated
runs, re-calibrate periodically. If low → iterate the **judge prompt** (clarify the
rubric, add examples/anchors, address a bias), or change the judge model, or fall back to
more human eval for that task. Calibration also reveals **systematic offsets** (the judge
runs one point harsh) which you can correct for.

**Calibration is not one-and-done.** Re-calibrate when you change the judge model, change
the rubric, change the task, or periodically as drift accumulates. The full virtuous
loop: humans define ground truth → judge is calibrated against it → calibrated judge
scales the measurement → judge gates CI and production → new failures feed back into the
human-labeled set → re-calibrate. This is how you get human-quality judgment at machine
scale and cost.

### Key terms

- **Eval calibration** — validating an LLM judge's verdicts against human ground truth.
- **Calibration set** — a representative, human-labeled sample used to validate the judge.
- **Rank correlation (Spearman / Kendall)** — measures whether judge and humans order outputs similarly.
- **Noise ceiling** — the human-human agreement level; the best a judge can realistically reach.
- **Systematic offset** — a consistent bias (e.g., judge scores one point low) that can be corrected.

### Common misconceptions

- ❌ "If the judge produces scores, the eval is valid." → ✅ Scores are meaningless until validated against human ground truth; an uncalibrated judge may measure verbosity or formatting, not quality.
- ❌ "A good judge should agree with humans nearly 100%." → ✅ Humans don't agree with each other 100%; the realistic target is human-human agreement (the noise ceiling).
- ❌ "Calibrate once and you're done." → ✅ Re-calibrate whenever the judge model, rubric, or task changes, and periodically to catch drift.

### Worked example

A team builds an LLM judge for "answer quality, 1–5" and wires it into CI. Before
trusting it they calibrate:

- Calibration set: 150 outputs (50 strong, 50 weak, 50 borderline).
- 3 humans label each; human-human agreement: Spearman ρ ≈ 0.78 (noise ceiling).
- Judge vs. human-mean: Spearman ρ = 0.41 — well below the ceiling. **Not trustworthy.**
- Inspecting the judge's rationales: it consistently rewards long, markdown-formatted
  answers (verbosity + formatting bias) and is harsh on short correct ones.
- They revise the judge prompt: "ignore length and formatting; reward correctness and
  completeness," add three anchored examples, and switch to a small rubric.
- Re-run: judge vs. human Spearman ρ = 0.74 — essentially at the noise ceiling. Now the
  judge is trusted to gate CI, with a quarterly re-calibration scheduled.

### Check questions

1. On your calibration set, human-human agreement is Spearman ρ = 0.62 and your judge-vs-human agreement is ρ = 0.61. A teammate is disappointed it isn't closer to 1.0. Why is ρ = 0.61 actually an excellent result here? — **Answer:** 0.62 is the noise ceiling — the humans defining ground truth only agree with each other that well, so a judge cannot meaningfully exceed it. ρ = 0.61 means the judge is essentially as consistent with humans as humans are with each other; that is the realistic best case. Expecting ~1.0 misunderstands that the ceiling is set by the humans, not the judge.
2. Two judges are calibrated against the same humans: Judge A has ρ = 0.40 against a human-human ceiling of 0.45; Judge B has ρ = 0.40 against a human-human ceiling of 0.85. Same judge correlation — which result is the problem and why? — **Answer:** Judge B is the problem. A's 0.40 sits near its 0.45 ceiling — the task is just noisy and A is doing about as well as possible. B's 0.40 against a 0.85 ceiling is far below what humans achieve, so B is measuring something other than the quality humans care about (likely a bias such as verbosity/formatting) and must be fixed before use. Judge correlation is only interpretable relative to the noise ceiling.
3. A team calibrates their judge once, gets a good result, and wires it permanently into CI. Six months later its CI verdicts no longer match human spot-checks. Give two distinct changes that could have invalidated the original calibration. — **Answer:** Any two of: (a) the judge model was updated/re-tuned by the provider, shifting its behavior; (b) the rubric or judge prompt was edited; (c) the task or input distribution drifted (new product areas, new query types); (d) gradual model/usage drift accumulated. Calibration certifies a judge for a specific model+rubric+task snapshot; when any of those move, it must be re-run — which is why calibration is periodic, not one-and-done.

---

## 9.11 — Hyperparameter / prompt search — grid vs. random

### Concept

Once you have a trustworthy eval, you can **optimize** against it. An LLM system has many
tunable knobs: prompt wording, system-prompt variants, number/choice of few-shot
examples, temperature, top-p, max_tokens, model choice, retrieval parameters (k, chunk
size). **Prompt / hyperparameter search** systematically explores these to find the
configuration that scores best — instead of guessing. Crucially, every evaluation of a
configuration costs real money and time (an eval run over N cases), so **search
efficiency matters**.

**Grid search.** Enumerate every combination of a discrete set of values for each knob.
Exhaustive and simple, but cost is the *product* of the per-knob options — it explodes
combinatorially ("curse of dimensionality"). Fine for 1–2 knobs with few values; quickly
infeasible.

**Random search.** Sample configurations at random from the ranges. Counter-intuitively
**more efficient than grid** when only a few knobs actually matter (the usual case, since
prompt search rarely has many knobs that genuinely move the score): grid wastes its
budget re-testing the irrelevant knobs at fixed values of the important ones, while
random search explores more *distinct* values of each important knob for the same total
budget. **Random search is the strong, simple default** for LLM prompt/hyperparameter
tuning.

Two more advanced ideas are worth *naming* but not dwelling on. **Successive halving**
(and its wrapper, **Hyperband**) is a budget-allocation strategy: start many candidates
on a small/cheap eval, keep the top fraction, re-evaluate survivors harder — useful when
each full eval is expensive and a cheap proxy eval exists. And **Bayesian optimization** —
which builds a surrogate model of score-vs-config — is **a poor fit for textual prompt
search**, because prompt choices are discrete and categorical with no smooth landscape
for a surrogate to exploit; reserve it, if at all, for low-dimensional numeric tuning.

**The universal caveat: search overfits the eval set** — whatever you select is the
config that did best on *that* data. Always confirm the winner on a **held-out split**
before shipping. Frameworks like **DSPy** automate this kind of prompt/example search
against a metric.

### Key terms

- **Hyperparameter / prompt search** — systematically exploring configurations to maximize an eval score.
- **Grid search** — exhaustive enumeration of all combinations; cost is the product of options.
- **Random search** — random sampling of configurations; more efficient than grid when few knobs matter; the default for prompt search.
- **Successive halving / Hyperband** — a budget-allocation strategy: evaluate many candidates cheaply, keep the top fraction, re-evaluate survivors harder.
- **Search overfitting** — selecting the config that did best on the eval set, which must be confirmed on a held-out split.

### Common misconceptions

- ❌ "Grid search is the thorough, best option." → ✅ Grid cost explodes combinatorially and wastes budget on irrelevant knobs; random search usually finds better configs for the same budget when only a few knobs matter.
- ❌ "Bayesian optimization is the sophisticated choice for prompt tuning." → ✅ Prompt choices are discrete and categorical with no smooth landscape, so a surrogate model has little to exploit; random search is the right default for prompt search.
- ❌ "The config that scored best is the one to ship." → ✅ Search overfits the eval set; confirm the winner on a held-out split first.

### Worked example

Tuning a classifier prompt: 4 system-prompt variants × 3 few-shot counts (0/3/8) × 3
temperatures = 36 configs. Eval set: 500 cases.

- **Grid:** 36 × 500 = 18,000 LLM calls. Exhaustive, costly — and most of those calls
  re-test irrelevant knob values.
- **Random search, 12 configs:** 12 × 500 = 6,000 calls — a third of the cost, and
  because few-shot count and prompt variant dominate, it explores their effect well.

The winner is then re-run on a held-out 150-case split to confirm it wasn't an overfit.

### Check questions

1. You are tuning 5 knobs but suspect only 1 or 2 actually move the score, and you have budget for 25 evaluations. Explain why random search will likely beat a 25-cell grid here. — **Answer:** A grid spends its 25 cells on a fixed lattice — e.g., 5 values of one knob crossed with 5 of another — so it tests very few *distinct* values of each knob and wastes evaluations varying knobs that don't matter. Random search draws 25 independent samples, so each of the 1–2 important knobs gets ~25 distinct values explored. When only a few knobs matter, that broader exploration of the ones that count finds a better config for the same budget.
2. A team reaches for Bayesian optimization to search over 8 categorical prompt-template variants and few-shot wordings, expecting it to beat random search. Why is it a poor fit here? — **Answer:** Bayesian optimization builds a surrogate model that assumes a smooth score landscape it can interpolate between configurations; over discrete, categorical, textual prompt choices there is no meaningful distance or smoothness for the surrogate to exploit. For prompt search, random search (a strong default) is the appropriate choice; BO is reserved for low-dimensional numeric tuning where each eval is very expensive.

---

## 9.12 — A/B testing LLM features in production

### Concept

Offline evals predict; **A/B testing** confirms with real users and real outcomes. You
split live traffic between a control (A, current system) and a treatment (B, the change —
new prompt, new model, new RAG pipeline), and compare **business and behavioral metrics**
on real users. It is the bridge between "scored well offline" and "actually better for
users."

**Why you need it on top of offline evals.** Offline evals are bounded by your dataset
and your judge's idea of quality; they cannot capture real user behavior, true input
distribution, or downstream business impact. A change can win the offline eval and *lose*
in production — e.g., longer, more thorough answers score higher with the judge but lower
real user task-completion because users won't read them. A/B testing measures the thing
that actually matters.

**Metrics.** Prefer **outcome metrics** over proxy metrics: task completion / resolution
rate, escalation-to-human rate, user retention, conversion, time-to-resolution. Implicit
behavioral signals (thumbs, edits, retries, regenerations, conversation abandonment,
copy/paste of the answer) are valuable and cheap. Guardrail metrics matter too: latency,
cost-per-request, error rate — B must not win quality while quietly tripling cost.

**Statistical rigor — the LLM-specific traps:**
- **Sample size / power.** Decide the minimum detectable effect and required sample size
  *before* starting; underpowered tests give noisy, non-reproducible results.
- **Statistical significance — use the right test.** Don't eyeball a 2% difference; run a
  proper hypothesis test, and the *kind* of metric dictates which one:
  - For a **rate / proportion** metric (resolution rate, thumbs-up rate, click-through —
    each user is a success/failure) the standard test is a **two-proportion z-test**
    (equivalently a chi-squared test): it asks whether the difference between A's and B's
    success proportions is larger than sampling noise would produce.
  - For a **continuous** metric (latency, time-to-resolution, tokens per session, revenue
    per user) use a **t-test** (Welch's t-test, which does not assume equal variances) on
    the per-user means, or a non-parametric **Mann-Whitney U** test if the metric is
    heavily skewed.
  - **CUPED (Controlled-experiment Using Pre-Experiment Data)** is a variance-reduction
    technique, not a test of its own: it uses each user's *pre-experiment* behavior as a
    covariate to subtract out predictable user-to-user variance, which **shrinks the
    confidence interval** and lets you detect the same effect with a smaller sample or
    shorter run. It is widely used in mature experimentation platforms and is high-value
    for LLM features because user-behavior variance is large.
- **No peeking / fixed horizon.** Repeatedly checking and stopping the moment p < 0.05
  inflates false positives massively. Either fix the duration up front and test only at
  the end, or use a **sequential testing** method explicitly designed for continuous
  monitoring — e.g., **always-valid p-values / confidence sequences**, **group sequential
  tests** (O'Brien-Fleming-style boundaries), or a **Sequential Probability Ratio Test
  (SPRT)** — which keep the false-positive rate controlled *even though* you look
  repeatedly, at the cost of slightly less power per look.
- **Randomization unit.** Randomize by *user* (or session), not by *request* — the same
  user hitting both variants pollutes the comparison and breaks the user experience.
- **Novelty / primacy effects.** Run long enough for the initial-reaction bump to settle.
- **Variance from non-determinism.** LLM outputs vary run-to-run; this adds noise — budget
  for it in your sample size.

**Rollout discipline.** A/B test is usually preceded by a **canary** (route a tiny 1–5%
slice to B, watch guardrail metrics for breakage) and followed by a **gradual ramp**
(5% → 25% → 50% → 100%) so a problem missed by the test is contained. Always keep a fast
**rollback**.

The full loop: offline eval gates the change into a canary → A/B test measures real
impact → ramp or roll back → online failures feed back into the offline eval set. Offline
and online are one system, not alternatives.

### Key terms

- **A/B test** — splitting live traffic between a control and a treatment to compare metrics on real users.
- **Outcome metric** — a metric tied to real value (resolution rate, retention) vs. a proxy.
- **Guardrail metric** — a metric (latency, cost, error rate) that must not regress even if quality improves.
- **Statistical power / minimum detectable effect** — the test's ability to detect a real difference of a given size.
- **Two-proportion z-test** — the standard significance test for comparing a rate/proportion metric between two variants.
- **CUPED** — a variance-reduction technique using pre-experiment data as a covariate to shrink confidence intervals and reach significance faster.
- **Sequential testing** — methods (always-valid p-values, group sequential tests, SPRT) that keep the false-positive rate controlled despite repeated checking.
- **Peeking** — repeatedly checking results and stopping early, which inflates false positives.
- **Randomization unit** — the entity (user/session) assigned to a variant.
- **Canary / gradual ramp** — routing a small slice first, then increasing exposure incrementally.

### Common misconceptions

- ❌ "Winning the offline eval means it's better in production." → ✅ Offline eval is bounded by your dataset and judge; only an A/B test on real users with outcome metrics confirms real impact.
- ❌ "Stop the test as soon as p < 0.05." → ✅ Peeking and early stopping inflate false positives; fix the horizon up front or use sequential-testing methods.
- ❌ "Randomize per request to get more data points." → ✅ Randomize per user/session; per-request assignment pollutes the comparison and gives users an inconsistent experience.

### Worked example

A team's new prompt scores +6% on the offline judge eval. They A/B test before full
rollout.

- **Canary:** 2% to B for a day — latency and cost flat, no error spike. Proceed.
- **A/B:** 50/50 by user, sample size pre-computed for a 3% minimum detectable effect,
  duration fixed at 14 days (no peeking).
- **Result:** offline-favored B has +6% judge score but **−4% task-completion** and
  **+11% conversation abandonment** — its longer answers users won't finish. Latency
  +400ms (guardrail concern).
- **Decision:** roll back B. The offline judge rewarded thoroughness; real users wanted
  brevity. The team adds an offline rubric dimension penalizing unhelpful length and a
  few abandoned real conversations to the eval set, so the offline eval now predicts
  production better.

### Check questions

1. A change wins your offline judge eval by a clear margin. Describe a concrete, realistic way it could still be a net negative in production despite that win. — **Answer:** The offline judge measures the judge's notion of quality, not real outcomes. A realistic failure: the change makes answers longer and more thorough, which the judge (with verbosity bias) scores higher — but real users won't read long answers, so task-completion falls and abandonment rises. The offline eval and the judge are both bounded; only an A/B test on real users with outcome metrics reveals this.
2. A team checks their A/B dashboard every morning and plans to ship the moment p < 0.05 appears. Statistically, what does this practice do to their results, and what is the correct alternative? — **Answer:** Repeatedly checking and stopping at the first significant-looking moment ("peeking") massively inflates the false-positive rate — given enough looks, a null effect will cross p < 0.05 by chance. Correct alternative: fix the sample size and test horizon in advance and only evaluate at the end, or use a sequential-testing method explicitly designed for continuous monitoring.
3. An engineer randomizes A/B assignment per *request* "to get more data points faster." Name two distinct problems this causes. — **Answer:** (1) It pollutes the comparison: the same user is served both variants, so their behavior reflects a blend of A and B rather than a clean exposure to one — the variants are no longer cleanly contrasted. (2) It breaks the user experience: a user sees the product behave inconsistently from request to request. Randomization must be by user (or session) so each user has one coherent, attributable experience.
4. Your treatment lifts the support-resolution *rate* from 71% to 73%. You need to know whether that 2-point lift is real, and your test is underpowered for the sample you can collect in a reasonable window. Name the appropriate significance test for this metric, and name one technique that could let you reach significance faster without simply running longer. — **Answer:** Resolution rate is a proportion (each user resolved or not), so the appropriate test is a **two-proportion z-test** (equivalently a chi-squared test) — it checks whether the gap between the two success proportions exceeds sampling noise. To reach significance faster without extending the run, apply **CUPED**: use each user's pre-experiment behavior as a covariate to subtract out predictable user-to-user variance, which shrinks the confidence interval so the same true effect becomes detectable with less data. (Sequential testing would let you check repeatedly without inflating false positives, but it does not by itself reduce the variance.)

---

## 9.13 — Benchmarks (MMLU, GPQA, SWE-bench, etc.) and benchmark contamination

### Concept

**Benchmarks** are standardized, public datasets and tasks used to compare *models*
(not your application) on broad capabilities. They are how the field reports progress and
how you do an initial model shortlist. Know the major ones and what each measures:

- **MMLU** (Massive Multitask Language Understanding) — ~16k multiple-choice questions
  across 57 subjects; broad knowledge/reasoning. [3] Now largely **saturated** (top models
  near ceiling); **MMLU-Pro** is the harder successor.
- **GPQA** ("Google-Proof Q&A") — graduate-level science questions written so they
  cannot be answered by quick web search; tests deep expert reasoning. **GPQA Diamond**
  is the hardest subset.
- **SWE-bench** — real GitHub issues; the model must produce a patch that makes the
  repo's tests pass. **SWE-bench Verified** is a human-validated, cleaned subset (a
  500-task subset reviewed by professional developers, released by OpenAI in August
  2024). [4] It became the leading benchmark for *agentic* software engineering —
  measuring end-to-end task completion, not trivia — though it is itself an object
  lesson in benchmark decay: OpenAI deprecated SWE-bench Verified in early 2026 over
  residual test flaws and training-data contamination. [4] Newer SWE-bench variants
  (e.g., SWE-bench Pro) have since taken over.
- **HumanEval** — 164 hand-written Python function-completion problems graded by unit
  tests; an early code benchmark, now largely saturated and contamination-prone.
- **GSM8K** — grade-school math word problems; multi-step arithmetic reasoning. Largely
  saturated for frontier models.
- **AIME** — competition math (American Invitational Mathematics Examination); hard
  multi-step math, a current discriminator for reasoning models.
- Others worth knowing: **MATH** (competition math), **HellaSwag** (commonsense),
  **ARC-AGI** (abstract reasoning), **MMMU** (multimodal), **τ-bench / tau-bench**
  (tool-use agents), and **LMArena** (crowdsourced pairwise human preference → Elo).

**Benchmark contamination.** The central problem: benchmarks are *public*, so their
questions and answers leak into pretraining data scraped from the web. A model that has
*seen the test* can score high by **memorization rather than capability** — the score is
inflated and no longer measures generalization. Contamination is hard to detect (you
rarely know exactly what was in training data) and creates a slow incentive problem:
benchmark scores become a marketing number, and "teaching to the test" (training on
benchmark-like data) further inflates them.

**Mitigations the field uses:** **held-out / private** test sets (e.g., a hidden split,
SWE-bench's verified process, periodically refreshed sets); **time-gated** benchmarks
made of items created *after* a model's training cutoff (e.g., recent competition
problems, **LiveCodeBench**, **LiveBench**); canary strings; and decontamination checks.

**Practical takeaways for an AI engineer:**
1. Benchmarks measure *models*, not *your application*. A model topping MMLU may still
   be bad at *your* task. **Your own eval set is what decides** — benchmarks only
   shortlist.
2. Treat published scores with healthy skepticism: check for contamination risk, prefer
   verified/time-gated benchmarks, prefer benchmarks that resemble your real task
   (SWE-bench if you build coding agents; tool-use benchmarks if you build tool agents).
3. A saturated benchmark (everyone near 100%) has lost its discriminative power — it no
   longer tells models apart.

### Key terms

- **Benchmark** — a standardized public dataset/task for comparing models on a capability.
- **MMLU / MMLU-Pro** — broad multiple-choice knowledge benchmark and its harder successor.
- **GPQA** — graduate-level, search-resistant science reasoning benchmark.
- **SWE-bench (Verified)** — real-GitHub-issue agentic coding benchmark; Verified is the cleaned human-validated subset.
- **Benchmark contamination** — leakage of benchmark items into training data, inflating scores via memorization.
- **Saturation** — when top models cluster near the ceiling and the benchmark no longer discriminates.
- **Time-gated / live benchmark** — items created after training cutoffs to resist contamination.

### Common misconceptions

- ❌ "The model that tops the benchmark leaderboard is best for my app." → ✅ Benchmarks measure broad model capability, not your specific task; your own eval set is the real decision-maker. Benchmarks only shortlist.
- ❌ "A high benchmark score proves capability." → ✅ Contamination means the model may have memorized the test; the score can reflect memorization, not generalization.
- ❌ "Benchmarks last forever." → ✅ They saturate (lose discriminative power) and get contaminated; the field continually replaces them with harder, verified, time-gated versions.

### Worked example

A team picks a model for a customer-support agent purely from a leaderboard: Model X
tops MMLU at 91%.

In production, X underperforms a lower-MMLU model on their task. Investigating:
- MMLU is **saturated** — X's 91% vs. another model's 89% is within noise and measures
  broad trivia, irrelevant to support quality.
- Their task is multi-turn tool use; the right benchmark to have consulted was a
  **tool-use / agent benchmark**, and ultimately their **own eval set** of real support
  conversations.
- They re-run their internal eval over three candidates; the model that tops *their*
  rubric — not MMLU — is chosen. They also note one candidate's suspiciously high
  HumanEval score and discount it for **contamination** risk, preferring its
  LiveCodeBench (time-gated) number.

### Check questions

1. Two models report identical 88% on a popular public coding benchmark. Model X was released before the benchmark; Model Y was released two years after it. Which score is more suspect, and what mechanism makes it so? — **Answer:** Model Y's score is more suspect. The benchmark is public, so over two years its questions and answers leaked into web-scraped pretraining data; Y may have *seen the test* and can answer by memorization rather than capability — contamination inflates the score above true generalization. X, released before the benchmark existed, could not have trained on it. (A time-gated or fresh benchmark would resolve the doubt.)
2. A leaderboard shows the top five models within 1.5 points of each other on MMLU. Two separate problems make this leaderboard a weak basis for choosing a model for your support agent. Name both. — **Answer:** (1) Saturation: when top models cluster near the ceiling, a 1.5-point gap is within noise and no longer discriminates between them — the benchmark has lost its discriminative power. (2) Task mismatch: MMLU measures broad multiple-choice knowledge, not multi-turn tool-using support quality; a model's MMLU rank says little about your specific task. Your own eval set on real support conversations is the real decision-maker.
3. Your team wants a benchmark number they can trust won't be inflated by contamination. What property must the benchmark have, and what is the underlying reason it works? — **Answer:** It must be *time-gated / live* — built from items created *after* the candidate models' training cutoffs (e.g., recent competition problems, LiveCodeBench/LiveBench), or use a private held-out split. It works because a model cannot have memorized items that did not exist when it was trained, so the score must reflect capability rather than recall of leaked test data.

---

## Topic 09 — Exam Question Bank

A pool the tutor draws from for the gated exam. Mixed-format, scored out of 100, pass
mark 85.

### True / False

1. Because online evaluation captures the true production distribution, a team with strong online monitoring can safely skip offline evals entirely. — **Answer:** False. Online is the true distribution but slow, noisy, and only catches regressions *after* users are harmed; offline gates changes cheaply *before* they ship. They are complementary stages, not substitutes.
2. After a prompt edit your eval's aggregate score rises by 0.02, so you can conclude no individual case regressed. — **Answer:** False. A rising aggregate can still hide cases that regressed (offset by larger improvements elsewhere); only a per-case diff reveals flipped cases — and a critical-case flip blocks the deploy regardless of the aggregate.
3. BLEU and ROUGE are poor primary metrics for a chatbot but remain reasonable for a constrained machine-translation task. — **Answer:** True. Their n-gram surface overlap correlates poorly with quality on open-ended generation, but for narrow tasks like translation/constrained summarization they are still acceptable.
4. Single-output (pointwise) scoring should always be replaced by pairwise comparison because pairwise is more reliable. — **Answer:** False. Pairwise is lower-noise for *comparing two candidates*, but pointwise is required when there is no comparison candidate (e.g., production monitoring) or when volume makes pairing impractical. Choose by purpose.
5. Running a pairwise comparison twice with positions swapped, then counting any position-dependent flip as a tie, is a mitigation for verbosity bias. — **Answer:** False. Swapping positions mitigates *position* bias. Verbosity bias is mitigated by instructing the judge to ignore length, controlling for length, and adding a conciseness rubric dimension.
6. If five annotators show 95% raw agreement on a label, the labels are reliable enough to serve as ground truth. — **Answer:** False. Raw percent agreement ignores chance; on a skewed task 95% raw agreement can coincide with chance-corrected kappa near 0. Reliability must be judged with kappa/alpha.
7. Grid search is the most thorough search method, so it is the safest default when tuning many hyperparameters. — **Answer:** False. Grid cost is the product of per-knob options and explodes combinatorially; when only a few knobs matter, random search finds better configs for the same budget. Grid is fine only for 1–2 knobs.
8. A model that tops the MMLU leaderboard will, by a 1-point margin, also be the best model for your specific support agent. — **Answer:** False. MMLU is saturated (a 1-point gap is noise) and measures broad knowledge, not your task; your own task-specific eval set is the decision-maker.
9. BERTScore compares an output to a reference by contextual-embedding similarity, so a high BERTScore confirms the answer is factually correct. — **Answer:** False. BERTScore measures *semantic similarity to the reference*, not correctness; an on-topic, similarly-worded but factually wrong answer can still score high, and it still requires a reference. It is a meaning-aware improvement over BLEU/ROUGE, not a correctness check.

### Multiple Choice

1. You evaluate a feature that extracts a customer's country from a free-text address into a fixed list of country codes. Which is the most appropriate primary metric?
   A) pass@k  B) Exact match (after normalization) against the reference code  C) ROUGE against the reference  D) LLM-as-judge on a 1–5 scale
   — **Answer:** B. The output has a single canonical surface form (a country code), so exact match is the rigorous, free, deterministic choice. pass@k is for verifiable code/math; ROUGE/judge are overkill and noisier for a closed-form label.
11. You have gold reference summaries and want a metric that, unlike BLEU/ROUGE, credits a correct paraphrase that shares few exact words — but you do not need reference-free criteria grading. The most appropriate choice is:
   A) Exact match  B) An embedding-based metric such as BERTScore  C) pass@k  D) Cohen's kappa
   — **Answer:** B. BERTScore compares output and reference tokens by contextual-embedding similarity, so a semantically close paraphrase scores well where n-gram overlap fails; it is cheap and reproducible and fits because references exist. Exact match/pass@k are for closed-form/verifiable tasks; kappa measures annotator agreement.
2. A team's three-judge panel barely improves agreement-with-humans over a single judge. The most likely cause is:
   A) The panel is too expensive  B) The three judges are from the same model family with the same prompt, so their errors are correlated  C) They used majority vote instead of mean  D) Three judges is too few to ensemble
   — **Answer:** B. Ensembling only helps when judges' errors are *independent*; same-family, same-prompt judges share blind spots, so aggregating them barely cancels error.
3. Calibrating a judge: human-human Spearman ρ is 0.80; judge-vs-human ρ is 0.78. The right conclusion is:
   A) The judge is broken — it should reach ρ ≈ 1.0  B) The judge is essentially at the human noise ceiling and can be trusted  C) The humans are unreliable and should be discarded  D) ρ is the wrong statistic; use Cohen's kappa
   — **Answer:** B. 0.80 is the noise ceiling; a judge cannot exceed how well humans agree with each other, so ρ = 0.78 is an excellent, trustworthy result.
4. You are tuning a prompt over a handful of knobs and only one or two genuinely move the score. Which search strategy is the right default?
   A) Grid search over all combinations  B) Random search  C) Bayesian optimization over the categorical prompt variants  D) Exhaustively try every prompt wording you can think of
   — **Answer:** B. Random search is the strong default for prompt search: when few knobs matter it explores more distinct values of the important ones than grid for the same budget. Grid's cost explodes combinatorially; Bayesian optimization is a poor fit for discrete, categorical prompt choices.
5. A vendor advertises a 94% score on a well-known public benchmark for a model released long after that benchmark. Why treat the number with caution?
   A) Public benchmarks are always too easy  B) The benchmark's items may have leaked into the model's training data, so the score can reflect memorization rather than capability  C) 94% is mathematically impossible  D) Newer models always overfit
   — **Answer:** B. Benchmark contamination: public Q&A leaks into web-scraped pretraining, so a model trained after the benchmark may have seen the test and answer by memorization.
6. In an A/B test of an LLM feature, the new variant raises the judge quality score but raises cost-per-request by 3×. Cost-per-request here functions as:
   A) The primary success metric  B) A guardrail metric that can block the rollout even though quality improved  C) A proxy metric for quality  D) An outcome metric
   — **Answer:** B. Guardrail metrics (cost, latency, error rate) must not regress even when quality wins; a 3× cost rise can block the change.
7. Why is randomizing an A/B test by *request* rather than by *user* a methodological error?
   A) It produces fewer data points  B) The same user is served both variants, blending their behavior and giving an inconsistent UX  C) It is computationally slower  D) Statistical tests forbid it outright
   — **Answer:** B. Per-request assignment exposes one user to both variants, so their behavior no longer cleanly attributes to A or B, and the product behaves inconsistently for them.
9. An A/B test compares the *thumbs-up rate* of two prompt variants. Which is the appropriate significance test, and which named technique reduces variance so significance can be reached with a smaller sample?
   A) A t-test; bootstrapping  B) A two-proportion z-test; CUPED  C) Cohen's kappa; Hyperband  D) BERTScore; sequential testing
   — **Answer:** B. Thumbs-up rate is a proportion, so a two-proportion z-test (or chi-squared) is the right significance test. CUPED uses pre-experiment user data as a covariate to subtract out predictable variance, shrinking the confidence interval so the same effect is detectable with less data. (Cohen's kappa measures annotator agreement; BERTScore is an output metric; neither is an A/B test.)
10. A team monitors their A/B dashboard continuously and wants to be able to stop as soon as the data is conclusive — without inflating false positives. Which approach makes that statistically valid?
   A) Just lower the significance threshold to p < 0.01  B) Use a sequential testing method (always-valid p-values / group sequential tests / SPRT) designed for repeated looks  C) Average the daily p-values  D) Switch the randomization unit to per-request
   — **Answer:** B. Sequential testing methods control the false-positive rate *despite* repeated checking, at a small cost in per-look power, so the team can legitimately stop early. A fixed lower threshold does not fix the peeking problem; averaging p-values is not a valid procedure; randomization unit is unrelated.
8. A team reports 96% raw agreement between two annotators on a binary label and calls the labels reliable. The hidden problem is most likely:
   A) Two annotators is too few to measure anything  B) The task is class-skewed, so most of that 96% is chance agreement and chance-corrected kappa could be near 0  C) Raw agreement should be reported as a percentage, not a fraction  D) They should have used Spearman correlation
   — **Answer:** B. On a skewed binary task, agreeing by guessing the majority class yields high raw agreement; only chance-corrected kappa/alpha reveals whether the labels carry real information.

### Short Answer

1. A team has evals but treats the eval set as a fixed asset built once at project start. Explain which principle of eval-driven development they are violating and the concrete consequence. — **Model answer:** They violate the principle that the eval set must *continuously grow* from observed failures — every production incident, escalation, and "the model shouldn't have done that" should become a new row. Consequence: a static eval set decays in value; failure modes that surface in production are never guarded, so the same class of bug can recur unnoticed on a future change. Credit for "evals grow from observed failures," not just "evals gate changes."
2. Why request a rationale before the verdict from an LLM judge? — **Model answer:** Chain-of-thought improves judging accuracy, and the written rationale makes wrong verdicts auditable — you can see when the judge's real reason was bias (e.g., length) rather than substance.
3. A product manager argues that since a reply scored 5/7 on the rubric overall, it should pass even though it has one factual error. Explain, using the concept of a gating dimension, why this argument is wrong and how the rubric should be structured. — **Model answer:** A gating dimension is one whose failure fails the *whole case regardless of other scores* — factual accuracy should be a gating dimension because a factually wrong reply is unshippable no matter how high tone and completeness score. Letting a 5/7 weighted total override a factual error encodes the wrong priority; the rubric must gate on accuracy so any material error → FAIL. Credit for explaining *why* some dimensions gate rather than average.
4. A model "wins" an eval mainly because its answers are longer. Name the bias, then explain why simply telling the judge "ignore length" is a weak fix and give a stronger one. — **Model answer:** Verbosity/length bias — judges systematically prefer longer answers regardless of added value. "Ignore length" is weak because the bias is a learned tendency, not a rule the model reliably obeys. Stronger: control for length structurally (compare similar-length answers / normalize) and/or add a conciseness dimension so padding is penalized rather than rewarded. Credit for recognizing instruction alone is insufficient.
5. Judge-vs-human agreement is 0.55. Explain why you cannot interpret that number as good or bad without one more piece of information, and name it. — **Model answer:** 0.55 is only interpretable relative to the *noise ceiling* — the human-human agreement level. If humans agree at 0.58, then 0.55 is excellent (a judge can't beat how well humans agree with each other). If humans agree at 0.85, then 0.55 means the judge is measuring something other than the quality humans care about and must be fixed. The missing piece is the human-human agreement on the same set.
6. Why is pass@1 reported alongside pass@10? — **Model answer:** A high pass@10 with low pass@1 reveals the model is unreliable single-shot and the system depends on retries; reporting both shows true reliability, not just best-of-k.
7. You changed nothing in your prompt, model version, or sampling parameters this month, yet best practice says still run the regression eval periodically. Justify this and name the failure it guards against. — **Model answer:** Hosted providers can silently re-tune or update a model behind the same version label, shifting behavior on inputs you never touched. Periodic re-runs guard against this provider-side regression, which no local change would trigger a run for. Credit for identifying the *uncontrolled* provider-update surface as the reason.

### Long Answer

1. Compare offline and online evaluation: what each measures, strengths, weaknesses, and how they fit into one workflow. — **Model answer / rubric:** Offline = fixed curated dataset before shipping; fast, cheap, repeatable, safe, runs in CI; weakness is distribution shift. Online = live production traffic; captures true distribution, real user behavior, business outcomes; weakness is slow, noisy, and risk to real users. Workflow: offline evals gate deploys (catch known regressions cheaply), canary then A/B test confirms real impact, online failures feed back into the offline dataset as permanent regression guards. They are one system, not alternatives. Credit for the feedback loop and the gate-vs-confirm framing.
2. Explain LLM-as-judge biases and a full mitigation strategy. — **Model answer / rubric:** Biases: position (favoring a slot), verbosity (favoring length), self-preference (favoring own family), plus sycophancy/authority and formatting bias. Mitigations: swap positions and run twice (position); instruct to ignore length + control length + conciseness rubric dimension (verbosity); use different-family judge / diverse panel (self-preference); force rationale before verdict; decompose with a rubric; blind/randomize; and calibrate against humans to measure residual bias. Credit for naming biases correctly and pairing each with a concrete mitigation.
3. Describe how to validate an LLM judge before trusting it (eval calibration), including which agreement statistics to use and what target to aim for. — **Model answer / rubric:** Build a representative calibration set covering easy/hard/edge cases; get multiple human labels with IAA measured; run the judge over the same set; compare. Use Cohen's kappa for categorical verdicts, Spearman/Kendall rank correlation for numeric scores, plus inspect systematic offsets. Target ≈ human-human agreement (the noise ceiling). If below, iterate the judge prompt (rubric clarity, anchors, bias fixes) or change the judge model. Re-calibrate when the judge/rubric/task changes and periodically for drift. Credit for the procedure, the right statistics, and the noise-ceiling target.
4. Compare grid and random search for prompt/hyperparameter tuning, and explain why Bayesian optimization is a poor fit for textual prompt search. — **Model answer / rubric:** Grid: exhaustive, cost = product of options, explodes combinatorially; fine only for 1–2 knobs with few values. Random: random sampling — more efficient than grid when only a few knobs matter (the usual case), because it explores more distinct values of the important knobs for the same budget; the strong default for prompt search. Bayesian optimization builds a surrogate model that assumes a smooth score landscape it can interpolate; discrete, categorical, textual prompt choices have no such smoothness, so BO has little to exploit (it suits low-dimensional numeric tuning instead). Successive halving exists as a budget-allocation option when each full eval is expensive. Universal caveat: search overfits the eval set — confirm the winner on a held-out split. Credit for the grid-vs-random trade-off, the BO-poor-fit reasoning, and the overfitting caveat.

### Applied Scenario

1. A medical-information assistant uses an LLM judge that scores answers 1–10. The team reduced the system prompt to cut token cost; the offline eval went from 8.1 to 8.3 average, so they shipped. Two weeks later clinicians report the assistant has started omitting a required "consult a professional" safety line on some answers. Walk through what the eval missed, why, and how the eval and gating process should be redesigned. — **Model answer / rubric:** What was missed: the safety line is a binary, objective requirement, but it was folded into a holistic 1–10 score — so dropping it on some cases was averaged away by gains elsewhere, leaving the aggregate flat-to-up. Why: a holistic score has no per-dimension visibility and no gating. Redesign: (a) make "safety line present" an *objective dimension checked deterministically by code* (not the LLM judge); (b) make it a *gating dimension* — its failure fails the case regardless of the holistic score; (c) compare runs with a *per-case diff*, not just the aggregate; (d) add the failing cases to the eval set as permanent regression guards. Credit for separating objective-vs-subjective dimensions, gating, per-case diffing, and feedback into the dataset.
2. Your team has a calibrated LLM judge it trusts, and a 220-case offline regression suite. You must now make a one-off, high-stakes decision: pick between two finalist models for a legal-summarization product, one of which is from the same model family as your judge. Describe an evaluation design appropriate to this *decision* (not routine CI), and justify each choice. — **Model answer / rubric:** This is high-stakes and one-off, so spend more than routine CI would. Use *pairwise* comparison (sharper than pointwise) of the two models over the eval set, with *positions swapped* to neutralize position bias. Because one candidate shares a family with the default judge, do *not* use a single judge — self-preference bias would skew it; use a *diverse panel* (different families) and aggregate by majority vote, routing splits to human review. Aggregate into a win rate. Confirm the winner on a *held-out split*. Weigh guardrail metrics (cost, latency). Credit for: pairwise + position swap, diverse panel to defeat self-preference, human review of disagreements, held-out confirmation, and matching effort to the stakes.
3. A change wins your offline judge eval decisively. In a canary it looks fine on guardrails, so you run a full A/B test — and the treatment shows lower task-completion and higher abandonment than control even though offline it scored far higher. Explain the most likely cause of the offline/online divergence and the full corrective process. — **Model answer / rubric:** Most likely cause: the offline judge is miscalibrated to real outcomes — it probably rewards thoroughness/length (verbosity bias), so the change's longer or more elaborate answers win with the judge but real users won't engage with them, depressing completion and raising abandonment. The offline metric is not predicting the outcome that matters. Corrective process: roll back the treatment; recalibrate the judge against human/real-outcome labels; add a rubric dimension that penalizes unhelpful length; harvest the abandoned real conversations into the offline eval set so it predicts production better next time; keep the canary→A/B→ramp discipline. Credit for diagnosing judge miscalibration/verbosity bias and for closing the offline↔online feedback loop.
4. You are choosing between two models for a coding agent. The vendor of Model A advertises a 92% HumanEval score; Model B advertises 88%. How do you make the decision? — **Model answer / rubric:** Don't decide on HumanEval — it is largely saturated and contamination-prone, and it measures isolated function completion, not agentic SWE work. Prefer SWE-bench Verified and time-gated benchmarks (LiveCodeBench) to resist contamination, but ultimately build your own eval set of real tasks from your codebase and run both models (and a panel-judged or test-based metric) on it. Also weigh guardrail metrics — latency and cost. The internal eval, not the advertised number, decides. Credit for skepticism about contamination/saturation, choosing task-relevant benchmarks, and prioritizing the internal eval set.
5. Your A/B test of a new RAG pipeline shows a 2% improvement and someone wants to ship it after two days because "p < 0.05 right now." What is your concern and what do you advise? — **Model answer / rubric:** Concern: peeking — repeatedly checking and stopping the moment significance appears inflates the false-positive rate, so the 2% may be noise; two days also risks novelty effects and an underpowered sample. Advise: pre-register the minimum detectable effect, required sample size, and a fixed test horizon; do not stop early unless using a proper sequential-testing method; verify guardrail metrics (latency, cost) and randomize by user. Only conclude at the planned horizon. Credit for identifying peeking, power, and the fixed-horizon/sequential-test fix.

---

## Sources

[1] Verga et al. — Replacing Judges with Juries: Evaluating LLM Generations with a Panel of Diverse Models (arXiv:2404.18796) — https://arxiv.org/abs/2404.18796
[2] Landis & Koch — The Measurement of Observer Agreement for Categorical Data (Biometrics, 1977; via PMC) — https://pubmed.ncbi.nlm.nih.gov/843571/
[3] Hendrycks et al. — Measuring Massive Multitask Language Understanding (MMLU) (arXiv:2009.03300) — https://arxiv.org/abs/2009.03300
[4] OpenAI — Introducing SWE-bench Verified — https://openai.com/index/introducing-swe-bench-verified/
