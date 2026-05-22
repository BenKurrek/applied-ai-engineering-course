# Topic 14 — Model Training Landscape — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This file is taught sub-chapter by sub-chapter, in order. Each sub-chapter has core
teaching content, key terms, common misconceptions, a worked example, and check
questions the tutor uses to confirm understanding before moving on. A mixed-format exam
bank is at the end; the tutor draws from it for the gated topic exam (scored out of
100, pass mark 85). Topic 14 is awareness-level for an applied engineer: you will not
train a frontier model, but you must understand *how the models you use were made* —
pretraining vs. post-training, the alignment methods (SFT/RLHF/DPO/Constitutional AI),
parameter-efficient fine-tuning (LoRA/QLoRA), distillation and quantization, the
decision of fine-tune vs. prompt vs. RAG, reasoning models, Mixture of Experts, and
context-length extension. These concepts explain model behavior, pricing, latency, and
the non-determinism you observe in production.

---

## 14.1 — Pretraining vs. post-training

### Concept

Building an LLM happens in two fundamentally different stages, and conflating them is a
common interview tell.

**Pretraining** is where the model acquires its capabilities. The objective is
**self-supervised next-token prediction**: take a colossal corpus of text and code
(trillions of tokens — much of the public internet, books, code repositories) and train
the model to predict the next token at every position. No human labels are needed —
the "label" is just the next token in the document, so the data is essentially free at
scale. This single, simple objective, applied at enormous scale, produces a model that
has internalized grammar, facts, reasoning patterns, coding ability, and world
knowledge. Pretraining is overwhelmingly the expensive part: it consumes months of time
on tens of thousands of GPUs and costs tens to hundreds of millions of dollars. The
result is a **base model** (or "foundation model") — a powerful next-token predictor
that is *not* a helpful assistant. Prompt a raw base model with a question and it might
continue with *more questions*, because in its training data a question is often
followed by more questions, not an answer. It has capability but no aligned behavior.

**Post-training** (also "alignment" or "fine-tuning" in the broad sense) is a
comparatively tiny stage — thousands to millions of carefully curated examples,
days-to-weeks of compute, a tiny fraction of the pretraining cost — that *shapes* the
base model into a useful, safe, instruction-following assistant. It does not teach new
knowledge so much as elicit and direct what is already there. Post-training is where
SFT, RLHF/RLAIF, DPO, and Constitutional AI (14.2) happen. It is what makes "Claude" or
"GPT" out of a raw base model.

The crucial mental model: **pretraining gives the model its knowledge and raw
capability; post-training gives it its behavior, helpfulness, format, and safety.** A
base model and its post-trained chat version have nearly identical *knowledge* — the
post-training corpus is far too small to add meaningful facts — but radically different
*behavior*. This is why "knowledge cutoff" is a property set in pretraining, and why
you cannot reliably teach a model new facts by fine-tuning (14.5): fine-tuning is
post-training-scale, and knowledge lives in pretraining-scale.

### Key terms

- **Pretraining** — large-scale self-supervised next-token prediction over trillions of
  tokens; where capability and knowledge come from; the expensive stage.
- **Base / foundation model** — the raw pretrained model: capable but not an aligned
  assistant.
- **Post-training** — the much smaller alignment stage (SFT, RLHF, DPO, etc.) that
  shapes behavior, helpfulness, format, and safety.
- **Self-supervised learning** — training where the label comes from the data itself
  (the next token), needing no human annotation.
- **Knowledge cutoff** — the date past which the model has no pretraining data; a
  property fixed in pretraining.

### Common misconceptions

- ❌ "Fine-tuning teaches the model new facts." → ✅ Knowledge is acquired in
  pretraining at trillions-of-tokens scale; post-training/fine-tuning is far too small
  to add meaningful facts — it shapes behavior. Use RAG for new facts.
- ❌ "A base model is just a slightly worse chatbot." → ✅ A raw base model is not an
  assistant at all; it may continue a question with more questions. Post-training makes
  it conversational.
- ❌ "Post-training is where most of the cost is." → ✅ Pretraining dominates cost
  (months, tens of thousands of GPUs, $10M–$100M+); post-training is a small fraction.

### Worked example

The two stages laid out as a pipeline — box width is drawn roughly to scale with *cost*,
which makes the headline point visual: pretraining dwarfs everything after it.

```
 ◀──────────────────── relative cost (box width ≈ $) ────────────────────▶

 ┌──────────────────────────────────────────────┐┌────┐┌─────────────┐┌────┐
 │██████████████████████████████████████████████││▏SFT││▏preference  ││▏saf│
 │████████████ PRETRAINING ██████████████████████││    ││  optim:     ││ ety│
 │██ self-supervised next-token prediction ██████││    ││ RLHF/RLAIF  ││ /  │
 │██ trillions of tokens · months · $10M–$100M+ ██││    ││ /DPO        ││eval│
 └──────────────────────────────────────────────┘└────┘└─────────────┘└────┘
                       │                                      │
                       ▼                                      ▼
                  ┌──────────┐                        ┌──────────────────┐
                  │BASE MODEL│                        │ ALIGNED ASSISTANT │
                  │capability│                        │ behavior, helpful,│
                  │ but no   │                        │ formatted, safe   │
                  │ aligned  │                        └──────────────────┘
                  │ behavior │
                  └──────────┘
   ◀──── PRETRAINING: knowledge & capability ────▶◀── POST-TRAINING: behavior ──▶
```

You download a raw base model and prompt it: "What is the capital of France?" Instead of
"Paris," it outputs: "What is the capital of Germany? What is the capital of Spain?" —
it is continuing a *list of trivia questions*, the kind of text it saw in pretraining.
The knowledge ("Paris") is in there; the *behavior* of answering is not. After SFT on
instruction/response pairs, the same model answers "The capital of France is Paris."
Identical knowledge, transformed behavior — the difference between pretraining and
post-training in one example.

### Check questions

1. A team fine-tunes a model on 5,000 examples of their product documentation and is
   disappointed that it still gets product facts wrong. Their colleague fine-tunes on
   5,000 examples of their support-reply *style* and it works well. Why did one
   fine-tune "stick" and the other did not? — **Answer:** Fine-tuning operates at
   post-training scale (thousands to millions of examples) — it *shapes behavior*
   (style, format, tone) effectively because behavior is exactly what that scale of
   training adjusts. It cannot reliably *install knowledge*: facts are acquired in
   pretraining at trillions-of-tokens scale, so 5,000 examples is orders of magnitude
   too small to embed them reliably (and they would go stale anyway). Style is a
   behavior problem — fine-tuning fits; product facts are a knowledge problem — that
   needs RAG.
2. You download a raw base model, give it the prompt "Write a haiku about autumn," and
   instead of a haiku it continues with "Write a haiku about winter. Write a haiku about
   spring." It clearly *knows* what a haiku is. What exactly is it missing, and which
   training stage would supply it? — **Answer:** It is missing aligned *behavior* —
   instruction-following. A base model is a next-token predictor; in its pretraining
   data a line like that is often followed by *more such lines*, so it continues the
   pattern instead of obeying. The knowledge (what a haiku is) is present from
   pretraining. The missing instruction-following/assistant behavior is supplied by
   *post-training* — starting with SFT on (prompt → ideal response) demonstrations.

---

## 14.2 — SFT, RLHF, RLAIF, DPO, Constitutional AI

### Concept

Post-training is itself a pipeline of techniques. Know each in one or two sentences and
know how they relate.

**SFT — Supervised Fine-Tuning.** The first post-training step. Train the base model on
a curated dataset of (prompt → ideal response) pairs — high-quality demonstrations of
the behavior you want (helpful answers, correct format, tool use, refusals). It is plain
supervised learning: imitate the demonstrations. SFT turns a base model into a
serviceable instruction-following assistant. Its limit: it can only imitate
demonstrations, and humans can demonstrate but not easily *rank subtle quality* — it
struggles to push the model toward "better than the demo."

**RLHF — Reinforcement Learning from Human Feedback.** The classic next step. (1)
Collect human *preferences*: show annotators two model responses to the same prompt,
have them pick the better one. (2) Train a **reward model** to predict those human
preferences — it learns to score any response. (3) Use reinforcement learning (PPO,
historically) to optimize the SFT model to produce responses the reward model scores
highly. RLHF lets you optimize for qualities that are easy to *judge* but hard to
*demonstrate* — and is the technique most responsible for models feeling helpful and
aligned. Its costs: human preference labeling is slow and expensive, and over-
optimizing the reward model causes **reward hacking** and is a source of **sycophancy**
(the reward model rewards agreeable answers).

**RLAIF — RL from AI Feedback.** Same as RLHF but the *preference labels* come from an
AI model instead of humans. Much cheaper and faster to scale; quality depends on the
labeler model. Often combined with human feedback rather than fully replacing it.

**DPO — Direct Preference Optimization.** A simpler alternative to RLHF, introduced in
2023 [2]. It optimizes the policy *directly* on the preference pairs with a
classification-style loss, **skipping the separate reward model and the RL loop**
entirely. More stable, simpler to implement, and cheaper — which is why it became very
popular, especially in the open-weights community. The trade-off: full RLHF with online sampling can still reach
higher ceilings on some objectives, but DPO captures most of the value with far less
complexity.

**Constitutional AI (CAI).** Anthropic's method to reduce reliance on human labels for
*harmlessness* [1]. Instead of humans labeling harmful outputs, the model critiques and
revises its own responses against an explicit written set of principles — a
"constitution." This produces AI-generated preference data for an RLAIF-style stage.
CAI makes the values explicit and auditable (they are written down) and scales
harmlessness training without massive human review of disturbing content.

The typical modern pipeline: **pretraining → SFT → preference optimization (RLHF /
RLAIF / DPO, often with a CAI component) → safety/eval passes.**

### Key terms

- **SFT** — supervised fine-tuning on (prompt → ideal response) demonstrations;
  imitation learning.
- **RLHF** — reward model trained on human preference pairs, then RL to maximize that
  reward; optimizes hard-to-demonstrate but easy-to-judge qualities.
- **Reward model** — a model that scores responses to approximate human preference.
- **RLAIF** — RLHF with AI-generated preference labels instead of human ones; cheaper,
  more scalable.
- **DPO** — direct preference optimization; trains on preference pairs directly,
  skipping the reward model and RL loop.
- **Constitutional AI** — the model critiques/revises its own outputs against a written
  set of principles to generate harmlessness preference data.
- **Reward hacking** — the policy exploiting flaws in the reward model to score highly
  without genuinely improving (a source of sycophancy).

### Common misconceptions

- ❌ "RLHF and SFT are alternatives." → ✅ They are sequential stages: SFT first
  (imitate demonstrations), then preference optimization (push beyond them).
- ❌ "DPO is just RLHF with a different name." → ✅ DPO removes the separate reward model
  and the RL loop, optimizing directly on preference pairs — structurally simpler.
- ❌ "Constitutional AI replaces all human input." → ✅ It reduces reliance on human
  *harm labels* by having the model self-critique against written principles; humans
  still write the constitution and remain involved elsewhere.
- ❌ "RLHF makes models perfectly honest." → ✅ It can induce sycophancy and reward
  hacking — the reward model rewards what *looks* good to raters, not always what is
  true.

### Worked example

RLHF is a cycle through a reward model and an RL loop; DPO takes a shortcut that
optimizes the policy straight from the preference pairs, skipping that whole middle:

```
        ┌─────────────────────────────────────────────────────────┐
        │                    RLHF cycle                           │
        ▼                                                         │
  ┌────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │ SFT model  │──▶ │ sample 2     │──▶ │ human picks  │──▶ │ reward model │
  │ (policy)   │    │ responses to │    │ the better   │    │ trains on    │
  │            │    │ same prompt  │    │ one          │    │ preferences  │
  └────────────┘    └──────┬───────┘    └──────────────┘    └──────┬───────┘
        ▲                  │                                       │
        │                  │                                       ▼
        │           ┌──────┴──────────────────────┐         ┌──────────────┐
        │           │           DPO shortcut       │         │ RL updates   │
        └───────────┤  optimize policy DIRECTLY on │◀────────┤ policy to    │
                    │  preference pairs —          │         │ maximize     │
                    │  NO reward model, NO RL loop │         │ reward       │
                    └──────────────────────────────┘         └──────────────┘
              DPO bypasses the reward-model + RL middle entirely
```

Why preference optimization beats SFT alone: imagine teaching a model to write good
explanations. With SFT you collect human-written ideal explanations and the model
imitates them — fine, but capped at the demonstrators' average. With RLHF/DPO you
instead show raters *two* model explanations and ask "which is clearer?" Raters find
ranking easy even when writing the ideal is hard, and the preference signal pushes the
model toward "clearer than either demo." That is the core reason post-training uses a
preference stage on top of SFT: judging quality is easier than demonstrating it.

### Check questions

1. A team has a large budget for human annotators and proposes "skip RLHF/DPO entirely
   — just collect 10× more SFT demonstrations; more imitation data must mean a better
   model." What is the ceiling problem with this plan? — **Answer:** SFT is imitation
   learning: the model copies the demonstrations, so it is capped at roughly the
   *demonstrators' own quality*. More SFT data makes it imitate the average demonstrator
   more reliably — it does not push past them. The reason a preference stage exists is
   that humans can *rank* "which of these two is better" for subtle qualities they
   cannot themselves *demonstrate well*; that preference signal lets RLHF/DPO push the
   model *beyond* the demonstrations. More SFT cannot buy what preference optimization
   buys.
2. Two teams both do preference optimization. Team A trains a separate reward model and
   runs a PPO-style RL loop; Team B optimizes the policy directly on the preference
   pairs. Name which is RLHF and which is DPO, and give one advantage of each
   direction of the trade-off. — **Answer:** Team A is classic RLHF; Team B is DPO. DPO
   (B) eliminates the separately trained reward model and the RL loop, optimizing
   directly on preference pairs — simpler, more stable, cheaper to implement (its
   advantage). Full online RLHF (A) is more complex but, because it samples and scores
   responses online, can still reach a somewhat higher ceiling on some objectives (its
   advantage). DPO captures most of the value for far less complexity.

---

## 14.3 — LoRA / QLoRA — parameter-efficient fine-tuning

### Concept

Full fine-tuning updates *all* of a model's billions of weights. That is enormously
expensive: you must hold the weights, their gradients, and optimizer states (Adam keeps
two extra values per parameter) in GPU memory — roughly 12–16 bytes per parameter — so
fine-tuning a 70B model needs a cluster, and you end up with a full multi-hundred-GB
copy of the model per task. **Parameter-Efficient Fine-Tuning (PEFT)** methods avoid
this; **LoRA** is the dominant one.

**LoRA — Low-Rank Adaptation.** The key insight (from the 2021 LoRA paper [3]): the
*change* a fine-tune makes to a weight matrix is empirically **low-rank** — it can be
well-approximated by a much smaller matrix. So LoRA **freezes the entire pretrained model** and, for selected weight
matrices, adds a small trainable side-path: instead of learning a full update `ΔW`
(say 4096×4096), it learns two thin matrices `A` (4096×r) and `B` (r×4096) whose
product `B·A` approximates `ΔW`. The **rank `r`** is tiny — typically 8, 16, 32, 64 — so
the number of trainable parameters drops by ~100–1000×. You train only `A` and `B`.

Consequences:
- **Cheap to train.** Far less GPU memory (no gradients/optimizer state for frozen
  weights) — you can fine-tune large models on a single GPU.
- **Tiny artifacts.** A LoRA "adapter" is megabytes, not hundreds of gigabytes. You can
  keep dozens of adapters for one base model and even **hot-swap** them at serving time
  — one base model, many specializations.
- **At inference**, `B·A` can be merged into `W` (zero added latency) or kept separate
  for swappability.
- Comparable quality to full fine-tuning for most adaptation tasks (style, format,
  domain tone, tool-use behavior).

**QLoRA — Quantized LoRA.** Goes further on memory: it loads the frozen base model in
**4-bit quantized** form (using a data type called NF4 — 4-bit NormalFloat) while
training the LoRA adapters in higher precision. The frozen weights — the bulk of the
memory — are squeezed ~4×; the QLoRA paper showed this enables fine-tuning a 65B-class
model on a single 48 GB GPU [4]. The adapters stay full precision, so quality loss is
minimal. QLoRA is what made fine-tuning large open models accessible to individuals and
small teams.

The applied takeaway: when fine-tuning *is* the right call (14.5), you will almost
certainly use LoRA/QLoRA, not full fine-tuning — and many provider fine-tuning APIs are
LoRA under the hood.

### Key terms

- **PEFT** — parameter-efficient fine-tuning: adapt a model by training a small number
  of parameters while freezing the rest.
- **LoRA (Low-Rank Adaptation)** — freezes the base model and trains small low-rank
  matrices `A`,`B` whose product approximates the weight update.
- **Rank `r`** — the inner dimension of the LoRA matrices (8–64); controls adapter
  capacity and size.
- **Adapter** — the small (MB-scale) trained LoRA artifact; swappable per task on a
  shared base model.
- **QLoRA** — LoRA with the frozen base model quantized to 4-bit (NF4), slashing memory
  so large models (e.g. 65B on a single 48 GB GPU [4]) fine-tune on one GPU.

### Common misconceptions

- ❌ "LoRA trains a small model." → ✅ It trains small *adapter* matrices on top of the
  full frozen base model; the base model is unchanged and still does the heavy lifting.
- ❌ "LoRA adds inference latency." → ✅ Adapters can be merged into the base weights for
  zero added latency, or kept separate when you need hot-swapping.
- ❌ "QLoRA quantizes the whole trained model, hurting quality." → ✅ QLoRA quantizes
  only the *frozen* base weights for the training run; the adapters train in higher
  precision and quality loss is minimal.
- ❌ "PEFT is a worse compromise you only use when broke." → ✅ For most adaptation
  tasks LoRA matches full fine-tuning quality while being far cheaper and producing
  swappable adapters — it is usually the *right* choice, not a compromise.

### Worked example

LoRA leaves the big weight matrix `W` frozen and adds a trainable side-path: two thin
matrices `B` and `A` whose product approximates the update — the trainable-parameter
count collapses by ~100×:

```
        FROZEN (not trained)              TRAINABLE (the LoRA adapter)
   ┌──────────────────────────┐
   │                          │                ┌─┐
   │                          │                │ │ B            ┌──────────────┐
   │            W             │      +         │ │ (4096×r)  ·  │  A (r×4096)  │
   │       4096 × 4096        │                │ │              └──────────────┘
   │       ≈ 16.8M params     │                └─┘
   │                          │              B·A ≈ ΔW   (the low-rank update)
   │                          │
   └──────────────────────────┘          rank r = 16 →  2 × (4096 × 16)
   trainable params: 0 (frozen)           ≈ 131k trainable params  (~128× fewer)

   forward pass:  h = W·x  +  B·(A·x)
                       │           │
                  frozen base   tiny learned adapter
```

A SaaS company wants its assistant to match each enterprise customer's brand voice and
internal terminology. Full fine-tuning means one ~140 GB copy of a 70B model per
customer — infeasible. With LoRA they train one ~30 MB adapter per customer on top of a
single shared 70B base model. At serving time the harness loads the right adapter for
the request — hundreds of customer-specific "models" from one base. With QLoRA each
adapter trains overnight on a single GPU. This adapter-per-tenant pattern is only
possible because LoRA artifacts are tiny and swappable.

### Check questions

1. Full fine-tuning of a 4096×4096 weight matrix would train ~16.8M parameters for that
   matrix. LoRA with rank 16 trains two matrices, 4096×16 and 16×4096, for the same
   matrix. Roughly how many trainable parameters is that, and what empirical fact about
   fine-tuning makes this drastic reduction work without wrecking quality? — **Answer:**
   LoRA trains 2 × (4096 × 16) ≈ 131k parameters — over 100× fewer than full
   fine-tuning of that matrix. It works because the *weight update* a fine-tune applies
   is empirically approximately *low-rank*: the change can be well-approximated by the
   product of two thin matrices, so you do not need a full-rank update matrix. The base
   model stays frozen; only the small low-rank adapter is learned.
2. A team fine-tunes a large model with QLoRA and a reviewer worries "you quantized the
   model to 4-bit, so the result must be much lower quality than full-precision LoRA."
   Explain precisely what is and is not quantized, and why the reviewer's worry is
   mostly unfounded. — **Answer:** QLoRA quantizes only the *frozen base weights*
   (4-bit NF4) — that is where the memory bulk is, so quantizing it is what enables
   single-GPU fine-tuning. The *LoRA adapters* — the only parameters actually being
   trained — stay in higher precision, so the learning itself happens at good precision.
   Net quality loss is minimal. The reviewer is conflating "the base is stored in 4-bit
   during training" with "the trained result is 4-bit," which is not the case.

---

## 14.4 — Distillation and quantization

### Concept

Two different techniques to make models cheaper/faster to *serve*. They are often
confused; keep them distinct.

**Distillation** changes *which model* you run — it transfers capability from a large
**teacher** model into a smaller **student** model. The student is trained not just on
ground-truth labels but on the teacher's *outputs* — its full probability distribution
over tokens (the "soft labels" or "dark knowledge"), or its generated responses and
reasoning traces. The soft targets carry far more information than a hard label ("this
token, definitely"): they encode the teacher's uncertainty and its sense of which
alternatives were plausible. The student learns to mimic the teacher's behavior at a
fraction of the size. Result: a smaller model that punches well above its parameter
count — much cheaper and faster to serve, with a quality gap that is often small for the
target task. Many small "fast"/"mini"/"flash"/"Haiku-class" models in frontier families
are distilled from their larger siblings, and reasoning models are commonly distilled by
training a small model on a large model's reasoning traces.

**Quantization** changes *how the same model is stored and computed* — it reduces the
**numerical precision** of the weights (and sometimes activations). Models are typically
trained in 16-bit (FP16/BF16); quantization converts weights to lower-precision formats.
The parameter *count* is unchanged — each parameter just takes fewer bits. Benefits:
less memory (the model fits on smaller/cheaper GPUs, or more requests fit per GPU), and
faster inference (decode is memory-bandwidth-bound, so moving fewer bytes per token
speeds it up).

The precision ladder, as it stands in 2026:

- **FP8** (8-bit floating point) has become the **de facto standard serving precision**
  for frontier-scale models — modern data-center GPUs have native FP8 hardware, and FP8
  serving is widely treated as effectively lossless for most workloads (DeepSeek-V3, for
  instance, was even *trained* largely in FP8 [7]). It is ~2× smaller than BF16 with
  negligible quality cost — close to a free win, which is why it is the common default.
- **INT8** is the other ~2× option, also typically near-lossless with modern methods.
- **4-bit** (~4× smaller) is where the interesting trade lives. The older framing —
  "INT4 = noticeable quality loss" — is now a generation stale: modern **weight-only
  4-bit methods (AWQ, GPTQ-class)** are much closer to lossless than that implies, and
  4-bit-quantized models are routinely served in production with only a small,
  task-dependent gap rather than an obvious degradation. Loss still exists and still
  concentrates on hard/edge cases — but "4-bit is unusable" is no longer true.
- **Below 4-bit** (3-bit, 2-bit) is where degradation becomes real and visible.

**Post-training quantization (PTQ)** is applied after training; **quantization-aware
training (QAT)** bakes the precision loss into the training process for better quality
at very low precision.

The clean contrast: **distillation = smaller model (fewer parameters), trained to
imitate a bigger one; quantization = same model (same parameters), stored at lower
precision.** They are complementary — you can distill *and* then quantize the student.
Side by side on the three axes that separate them:

| | **Distillation** | **Quantization** |
|---|---|---|
| Parameter count | **Changes** — student has fewer parameters (e.g. 70B → 7B) | **Unchanged** — same count, e.g. 70B stays 70B |
| Bits per parameter | Unchanged (student trained at normal precision) | **Reduced** — BF16 → FP8 / INT8 / 4-bit |
| Produces | A *different, smaller model* trained to imitate the teacher | The *same model*, stored/computed at lower precision |

### Key terms

- **Distillation** — training a small *student* model to imitate a large *teacher*,
  often via the teacher's soft probability distributions or generated traces.
- **Teacher / student** — the large source model and the small model being trained from
  it.
- **Soft labels / dark knowledge** — the teacher's full probability distribution, richer
  than a one-hot hard label.
- **Quantization** — reducing the numerical precision of weights (BF16 → FP8/INT8 →
  4-bit), shrinking memory and speeding inference; parameter count unchanged.
- **FP8** — 8-bit floating point; the de facto 2026 standard serving precision,
  hardware-accelerated and effectively lossless for most workloads.
- **Weight-only 4-bit (AWQ / GPTQ-class)** — modern 4-bit quantization methods that are
  much closer to lossless than the legacy "INT4 = noticeable loss" framing implies.
- **PTQ vs. QAT** — post-training quantization (applied after training) vs.
  quantization-aware training (precision loss simulated during training for better
  quality, important at very low precision).

### Common misconceptions

- ❌ "Distillation and quantization are the same thing." → ✅ Distillation produces a
  *different, smaller* model; quantization stores the *same* model at lower precision.
- ❌ "Quantization reduces the parameter count." → ✅ It reduces *bits per parameter*;
  the count is identical.
- ❌ "A distilled or quantized model is just strictly worse." → ✅ Both trade modest,
  often task-negligible quality for large cost/latency/memory gains — frequently the
  right production choice.
- ❌ "4-bit quantization makes a model obviously worse." → ✅ That framing is a
  generation stale: FP8 and INT8 are effectively near-lossless, and modern weight-only
  4-bit methods (AWQ/GPTQ-class) are much closer to lossless than "noticeable loss"
  implies — small, task-dependent gap, not obvious degradation. Real degradation sets
  in below 4-bit.

### Worked example

A team serves a 70B model that needs two 80 GB GPUs (~140 GB in BF16) and is slow and
costly. Two independent levers: (1) **Quantize** the same 70B model — FP8 (~70 GB) is
the near-lossless default and already fits on a single 80 GB GPU; pushing to weight-only
4-bit (~35 GB) shrinks it further and speeds decode more, at a small task-dependent
quality cost rather than an obvious one. (2) **Distill** the 70B teacher into a 7B
student on their task's traffic → a 7B model that handles ~90% of requests at a fraction
of the cost, escalating only hard ones to the big model. Best of all: distill to 7B
*and* quantize the 7B student. Two different techniques, stackable.

### Check questions

1. After two optimization projects, Team A reports "our model now has 8B parameters,
   down from 70B." Team B reports "our model still has 70B parameters but the file is
   4× smaller." One did distillation, one did quantization. Which is which, and what is
   the defining difference? — **Answer:** Team A did *distillation* — the parameter
   *count* changed (70B → 8B), because distillation trains a new, smaller student model
   to imitate the large teacher. Team B did *quantization* — the count is unchanged
   (still 70B) but each parameter is stored in fewer bits, so the file shrinks.
   Defining difference: distillation produces a *different, smaller model*; quantization
   keeps the *same model* at lower numerical precision. (They stack — you can distill
   then quantize the student.)
2. A team quantizes their model from 16-bit to INT4 and observes that token generation
   gets noticeably *faster*, not just smaller in memory. Connect this to why the decode
   phase is the bottleneck it is. — **Answer:** Decode is *memory-bandwidth-bound*: each
   generated token requires reading the model's weights out of memory, and that data
   movement — not arithmetic — is the limiting cost. INT4 weights are ~4× fewer bytes
   than 16-bit weights, so each decode step moves far less data and completes faster.
   Quantization speeds generation because it shrinks exactly the thing the decode
   bottleneck is made of: bytes-per-token moved from memory.

---

## 14.5 — When to fine-tune vs. prompt vs. RAG

### Concept

This is the single most-asked applied question in this topic, because engineers
routinely reach for fine-tuning when prompting or RAG is the right tool. The three
approaches solve *different problems*:

- **Prompting / context engineering** changes the model's behavior *per request* via
  the input — system prompt, few-shot examples, instructions. Zero training cost,
  instant iteration, no infrastructure. Always the **first thing to try and exhaust**.
- **RAG (Retrieval-Augmented Generation)** injects *knowledge* into the context at
  request time by retrieving relevant documents. The tool for facts the model does not
  have or that change.
- **Fine-tuning** bakes *behavior* into the weights via post-training-scale training. It
  changes how the model *acts*, not what it *knows*.

The decision framework — match the tool to the problem:

| Problem | Right tool |
|---|---|
| Model doesn't know a *fact* (private data, recent events, changing data) | **RAG** |
| Need *citations* / verifiable sources | **RAG** |
| Knowledge changes frequently | **RAG** (re-index; no retraining) |
| Need a consistent *style/tone/format* | **Fine-tune** (or strong prompt first) |
| Need reliable structured output / a specific behavior pattern | **Fine-tune** (after prompting fails) |
| Want to *shrink* the prompt — fold a huge system prompt / many few-shot examples into the weights for latency & cost | **Fine-tune** |
| Teach a genuinely new *skill* the model lacks | **Fine-tune** |
| Quick iteration, prototyping, behavior tweaks | **Prompting** |

Decisive principles:

1. **Knowledge problem → RAG; behavior problem → fine-tune.** This is the core split.
   Fine-tuning is the *wrong* tool for installing facts — post-training scale cannot
   reliably teach knowledge, retraining is slow, and the facts go stale. RAG keeps
   knowledge fresh and yields citations.
2. **Always exhaust prompting first.** It is free, instant, and modern models with long
   context and good instruction-following solve a huge fraction of problems with a good
   prompt and a few examples. Fine-tuning has real costs: data collection, a training
   pipeline, eval, versioning, and re-doing it whenever the base model upgrades.
3. **Fine-tuning's honest wins:** consistent style/format at scale; reliability on a
   narrow behavior; *prompt compression* (a fine-tuned model needs a shorter prompt,
   cutting per-request latency and cost — real at high volume); and sometimes letting a
   smaller, cheaper fine-tuned model match a larger prompted one.
4. **They combine.** A common production stack is a fine-tuned model (for behavior/
   format) used inside a RAG pipeline (for knowledge) with careful prompting — not an
   either/or.

Interview-grade summary: *prompt first; RAG for knowledge; fine-tune for behavior — and
only after prompting has been genuinely exhausted.*

### Key terms

- **Prompting / context engineering** — shaping behavior per request via input; zero
  training cost; the first resort.
- **RAG** — retrieving knowledge into context at request time; the tool for facts and
  citations.
- **Fine-tuning** — baking behavior/style/format into the weights via training.
- **Prompt compression** — using a fine-tuned model to replace a long system prompt /
  many few-shot examples, cutting per-request tokens.
- **Knowledge vs. behavior** — the central axis: knowledge gaps → RAG, behavior gaps →
  fine-tune.

### Common misconceptions

- ❌ "We need the model to know our internal docs, so we'll fine-tune on them." → ✅
  That is a knowledge problem — use RAG. Fine-tuning won't reliably install the facts
  and they'll go stale; RAG keeps them fresh and citable.
- ❌ "Fine-tuning is the most powerful option, so prefer it." → ✅ It is the most
  *costly and slowest to iterate*; prompting and RAG solve most problems and should be
  exhausted first.
- ❌ "Fine-tuning and RAG are competing choices." → ✅ They address different axes
  (behavior vs. knowledge) and are routinely combined.
- ❌ "Once fine-tuned, you're done." → ✅ A fine-tune is tied to a base model version;
  each base-model upgrade may require re-doing it — an ongoing maintenance cost.

### Worked example

A company wants an assistant that (a) answers questions about their constantly-updated
product catalog, (b) always responds in their formal brand voice with a fixed JSON
structure, and (c) cites the catalog entry it used. Correct decomposition: (a) and (c)
are *knowledge + citation* problems → **RAG** over the catalog (re-indexed as the
catalog changes). (b) is a *behavior/format* problem → first try a strong system prompt
+ few-shot examples; if voice/format drift at scale, **fine-tune** for the style and
JSON structure. The result is a fine-tuned model inside a RAG pipeline — and the
fine-tune also lets them drop the long style prompt, cutting per-request cost. One
feature, all three tools, each on the problem it actually solves.

### Check questions

1. A feature must (a) answer using your internal pricing data, which changes daily, and
   (b) always respond in a fixed JSON schema. A teammate proposes one nightly fine-tune
   that bakes in both. Decompose the request and assign the right tool to each part,
   explaining why a nightly fine-tune is wrong for at least one of them. — **Answer:**
   (a) is a *knowledge* problem with *changing* data → RAG: retrieve the current pricing
   at request time and re-index as it changes. A nightly fine-tune is wrong here — it
   cannot reliably install facts and would be stale within the day, with no citations.
   (b) is a *behavior/format* problem → first a strong system prompt + few-shot; if the
   JSON schema drifts at scale, fine-tune for it. End state: a (possibly) fine-tuned
   model for the format, inside a RAG pipeline for the data — not one fine-tune for both.
2. Your classification feature works well today with a 1,500-token prompt full of
   instructions and few-shot examples, and a teammate says "it works, so fine-tuning
   would be pointless." Construct the strongest counter-argument — a concrete situation
   where fine-tuning is still worth it even though prompting works. — **Answer:** At
   high request volume, *prompt compression* makes fine-tuning worth it: a model
   fine-tuned on the task can fold the instructions and few-shot examples into its
   weights, so each request needs only a short prompt instead of 1,500 tokens. That cuts
   per-request input tokens — real latency and cost savings that compound at volume —
   and can let a smaller, cheaper model match the larger prompted one. "It works" is
   about correctness; this is a win on economics, which prompting cannot deliver.

---

## 14.6 — Reasoning models / extended thinking — RL on verifiable rewards, test-time compute

### Concept

**Reasoning models** (OpenAI's o-series, Claude's extended-thinking modes, Gemini's
thinking modes, DeepSeek-R1, etc.) are a distinct class that, before producing a final
answer, generate an extended internal **chain of thought** — sometimes thousands of
tokens of working-out — then answer. They substantially outperform standard models on
hard math, coding, logic, and multi-step problems.

**Test-time compute** is the underlying idea: instead of (or in addition to) making the
model bigger, you let it *spend more computation at inference time* on a hard problem —
"think longer." A standard model commits to each answer token immediately; a reasoning
model can explore, draft, check, and backtrack in its thinking before committing. More
thinking tokens → more deliberation → better answers on problems that genuinely need
multiple steps. This is a genuinely different scaling axis from pretraining-scale: you
trade inference cost/latency for accuracy, *per request*.

**How they are trained — RL on verifiable rewards (RLVR).** The breakthrough is the
*reward signal*. RLHF (14.2) uses a learned reward model approximating fuzzy human
preference. Reasoning models are trained with RL where the reward is **objectively
checkable**: in domains like math and code you can *verify* whether the final answer is
correct (run the unit tests, check the numeric result) without any human or learned
judge. That clean, unhackable signal lets RL push the model to discover effective
reasoning strategies on its own — it is rewarded purely for *getting the right answer*,
and self-organizes useful behaviors (decomposition, self-checking, backtracking).

The *algorithm* that carries the RL update matters less than the reward, but one name is
worth knowing: **GRPO (Group Relative Policy Optimization)** — the RL algorithm
DeepSeek used to train R1 [5]. PPO (the classic RLHF workhorse) needs a separate learned
*value/critic* model to estimate a baseline; GRPO drops it. Instead it samples a
*group* of answers to the same prompt, scores each with the verifiable reward, and uses
the group's *average* score as the baseline — an answer is reinforced in proportion to
how much it beats its peers. Removing the critic model makes GRPO simpler and cheaper to
run, which is part of why verifiable-reward RL scaled. DeepSeek-R1 (2025) showed this
works at scale: its RL-first pipeline used only a brief SFT warm-up, and the
DeepSeek-R1-Zero variant used pure RL with no SFT at all [5].

**Implications for the applied engineer:**
- **They changed prompting.** Chain-of-thought prompting ("think step by step") was the
  way to get standard models to reason; reasoning models do this *internally* by
  default — manually forcing CoT is redundant and can even hurt. Reasoning models also
  prefer concise instructions and a clear goal over heavy step-by-step scaffolding.
- **Cost and latency.** Thinking tokens are billed as output tokens (the expensive
  kind) and add real latency — TTFT can be long while the model thinks. Use reasoning
  models for genuinely hard problems; a non-reasoning model is cheaper/faster for
  simple tasks. Many APIs expose a *thinking budget* knob to dial the trade-off.
- **The thinking is often summarized or hidden** in the API; you may not see the raw
  trace, and you should not depend on its exact format.

### Key terms

- **Reasoning model** — a model that generates an extended internal chain of thought
  before answering; strong on hard multi-step problems.
- **Test-time / inference-time compute** — spending more computation at inference
  ("thinking longer") to improve answers; a scaling axis distinct from model size.
- **RLVR — RL on verifiable rewards** — RL trained with an objectively checkable reward
  (correct answer / passing tests) instead of a learned preference model.
- **GRPO (Group Relative Policy Optimization)** — the RL algorithm DeepSeek used for
  R1; scores a *group* of sampled answers and uses their average as the baseline,
  dropping PPO's separate learned critic model.
- **Thinking budget** — an API knob controlling how many reasoning tokens the model may
  spend.
- **Chain of thought** — the intermediate reasoning steps; explicit-and-prompted for
  standard models, internal-by-default for reasoning models.

### Common misconceptions

- ❌ "Reasoning models just have 'think step by step' baked into the prompt." → ✅ They
  are *trained* with RL on verifiable rewards to reason internally; it is a training
  difference, not a prompt trick.
- ❌ "Always add chain-of-thought prompting." → ✅ For reasoning models that is
  redundant and can degrade output; they reason internally and prefer concise prompts.
- ❌ "Thinking tokens are free or cheap." → ✅ They bill as output tokens (the
  expensive rate) and add latency; a reasoning-model call can cost far more than the
  visible answer suggests.
- ❌ "Use a reasoning model for everything." → ✅ For simple tasks they are slower and
  costlier with no benefit; match the model class to problem difficulty.
- ❌ "RLVR and GRPO are the same thing." → ✅ RLVR describes the *reward* (objectively
  verifiable correctness); GRPO is one *algorithm* for the RL update (group-relative
  baseline, no separate critic). DeepSeek-R1 used a verifiable reward optimized with
  GRPO — they answer different questions.

### Worked example

A team uses standard chat models with elaborate "think step by step, show all working"
prompts for a math-tutoring product, and accuracy on hard problems plateaus. They switch
to a reasoning model and *delete* the step-by-step scaffolding, giving just the problem
and a clear goal. Accuracy on hard problems jumps because the model deliberates
internally with RL-trained reasoning strategies — but cost per request rises (thousands
of billed thinking tokens) and TTFT grows. The fix: route easy problems to a cheap
non-reasoning model and reserve the reasoning model (with a tuned thinking budget) for
genuinely hard ones.

### Check questions

1. RLHF (Topic 14.2) and RLVR both use reinforcement learning, yet RLVR was the
   breakthrough for reasoning models. The difference is the *reward signal*. Contrast
   the two reward signals and explain why RLVR's lets RL push harder. — **Answer:**
   RLHF's reward comes from a *learned reward model* that approximates fuzzy human
   preference — it is an imperfect proxy and can be *reward-hacked* (the policy scores
   well without genuinely improving). RLVR's reward is *objectively checkable*: in math
   you verify the final answer, in code you run the unit tests — no human or learned
   judge. Because that signal is clean and essentially unhackable, RL can be pushed hard
   on it: the model is rewarded purely for correctness and can freely discover effective
   reasoning strategies (decomposition, self-checking, backtracking) without the reward
   collapsing into exploitation.
2. A team migrates from a standard model to a reasoning model but keeps their old
   prompt template, which says "Let's think step by step. First, ... Second, ... Show
   all your working." They find the reasoning model's answers got *worse* than expected.
   What did they get wrong? — **Answer:** They are forcing manual chain-of-thought
   scaffolding onto a model that is already *trained to reason internally*. For a
   reasoning model, the explicit step-by-step prompting is redundant and can *conflict*
   with its own native reasoning process, degrading output. Reasoning models prefer a
   concise prompt with a clear goal; the CoT scaffolding that helped standard models is
   counterproductive here.

---

## 14.7 — Mixture of Experts (MoE)

### Concept

**Mixture of Experts (MoE)** is an architecture that decouples a model's **total
parameter count** from the parameters **actually used per token**. It is how nearly all
modern frontier models reach very large scale affordably.

In a standard ("dense") transformer, *every* parameter is used to process *every* token
— a 70B dense model does ~70B parameters' worth of computation per token. In an MoE
model, the feed-forward block of each transformer layer is replaced by many parallel
sub-networks called **experts** (say 64 or 128 of them). A small learned **router**
(gating network) looks at each token and selects only a few experts — commonly the
**top-2** — to process it. The other experts are skipped for that token.

So an MoE model has two very different numbers:
- **Total parameters** — all experts summed; this is where the model's *capacity and
  knowledge* live, and it can be huge (hundreds of billions to trillions).
- **Active parameters** — what is actually computed per token (the shared layers + the
  selected experts); this is what determines inference *cost and speed*.

A real example: **DeepSeek-V3** is a documented MoE with **671B total parameters but
only ~37B active per token** [7] — the capacity of a 671-billion-parameter model, but
the per-token compute cost closer to a ~37B model. That is the whole point: **MoE buys
large-model capability at small-model inference cost.** It is a *sparsely activated*
model. (Mixtral 8x7B is another well-known, smaller example: 8 experts, top-2 routed —
~47B total but ~13B active per token [8].) Different experts specialize during training
(loosely — by token type, topic, language), and the router learns to send each token to
the most useful ones.

Trade-offs:
- **Memory.** All experts must be *resident* in memory even though only a few run per
  token — so MoE saves compute, not VRAM. Serving a 671B-total MoE still needs the
  hardware to hold all 671B parameters.
- **Routing complexity.** Training needs load-balancing so experts are used evenly
  (an "auxiliary loss"); a collapsed router that overuses a few experts wastes capacity.
- **Determinism (a key applied implication).** Routing decisions can depend on which
  *other* tokens share the inference batch — and batch composition varies with
  unrelated concurrent traffic. So the *exact* experts chosen for your tokens can shift
  run-to-run, contributing to the non-determinism you see even at temperature 0. MoE
  routing is one concrete reason "temperature 0" is not perfectly reproducible.

### Key terms

- **Mixture of Experts (MoE)** — an architecture with many expert sub-networks per
  layer, of which a router activates only a few per token (sparse activation).
- **Expert** — one of the parallel feed-forward sub-networks.
- **Router / gating network** — the small learned component that picks which experts
  process each token (e.g. top-2).
- **Total vs. active parameters** — all experts (capacity/knowledge) vs. parameters
  actually computed per token (cost/speed).
- **Sparse activation** — using only a fraction of the model's parameters for any given
  token.

### Common misconceptions

- ❌ "An MoE model uses all its parameters per token." → ✅ Only the router-selected
  experts (e.g. top-2) plus shared layers run per token — that is the "active"
  parameter count.
- ❌ "MoE saves memory." → ✅ It saves *compute* per token; all experts must still be
  resident in VRAM, so memory cost tracks total parameters.
- ❌ "Each expert handles a clean human-interpretable domain." → ✅ Specialization is
  loose and emergent; experts do not map neatly to topics, and routing is learned, not
  designed.
- ❌ "Temperature 0 on an MoE model is fully deterministic." → ✅ Batch-dependent
  routing means the experts selected can vary with co-tenant traffic — a real source of
  run-to-run non-determinism.

### Worked example

For each token, the router picks the top-2 of N experts; the rest are skipped (dimmed)
and never run for that token:

```
                       ┌──────────┐
        token ───────▶ │  ROUTER  │  scores all N experts, keeps top-2
                       └────┬─────┘
            ┌───────┬───────┼───────┬───────┬───────┐
            ▼       ▼       ▼       ▼       ▼       ▼
         ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
         │ E1  │ │█E2█ │ │ E3  │ │ E4  │ │█E5█ │ │ E6  │  ... up to E_N
         │·skip│ │ RUN │ │·skip│ │·skip│ │ RUN │ │·skip│
         └─────┘ └──┬──┘ └─────┘ └─────┘ └──┬──┘ └─────┘
                    └──────────┬───────────┘
                               ▼
                       shared layers + E2 + E5  →  this token's output

   TOTAL  = shared + all N experts   →  capacity / knowledge (can be huge)
   ACTIVE = shared + 2 experts       →  per-token compute cost & speed
```

Take the documented **DeepSeek-V3** MoE — **671B total / ~37B active** [7] — against a
hypothetical *dense* 37B model. Per token both do roughly the same amount of computation
(~37B parameters), so they have broadly comparable per-token inference speed and cost.
But the MoE has on the order of *~18× more total capacity* to store knowledge and skills,
so it is substantially more capable while costing far closer to the small model to run.
The catch surfaces at deployment: the MoE needs enough GPU memory to hold *all 671B*
parameters resident, even though any single token touches only ~37B of them — so it
still demands large-model-sized hardware. This is exactly the trade frontier labs make,
and why most big open models (DeepSeek, Qwen, Mixtral, Llama's MoE variants) are MoE.

### Check questions

1. You compare a dense 40B model with an MoE model rated (illustrative numbers)
   "640B total / 40B active." Predict how they compare on (a) per-token inference
   speed/cost and (b) capability, and (c) explain which of the two MoE numbers governs
   each. — **Answer:** (a) Roughly *equal* per-token speed and cost — both do ~40B
   parameters' worth of computation per token; the MoE's *active* count governs
   inference cost/speed. (b) The MoE is substantially *more capable* — its *total* count
   (640B, the sum of all experts) is where capacity and knowledge live, ~16× the dense
   model's store. So an MoE buys large-model capability at small-model per-token cost;
   "total" governs capability, "active" governs cost.
2. A team sets temperature 0 expecting bit-identical outputs from their MoE-based
   provider, and files a bug when outputs vary slightly run-to-run. Explain why
   temperature 0 does *not* guarantee reproducibility on an MoE served with dynamic
   batching. — **Answer:** Temperature 0 makes *token selection* greedy, but in an MoE
   the *router's* choice of which experts process each token can depend on which other
   tokens share the inference batch — and batch composition varies with unrelated
   concurrent traffic. Different experts → slightly different computed logits → slightly
   different output, even under greedy decoding. So it is not a bug: MoE routing is a
   real source of run-to-run non-determinism, and temperature 0 is not a hard
   reproducibility guarantee.

---

## 14.8 — Context-length extension — RoPE scaling

### Concept

A model's context window is not an arbitrary setting — it is bounded by how the model
was *trained*, specifically by its **positional encoding**. Transformers process all
tokens in parallel and have no inherent sense of order, so position must be injected.
Modern LLMs use **RoPE (Rotary Position Embedding)**: instead of adding a position
vector, RoPE *rotates* the query and key vectors by an angle proportional to the token's
position. The dot product in attention then naturally encodes *relative* distance
between tokens — elegant, and a big part of why it became standard.

The problem: a model trained with RoPE on sequences up to, say, 8k tokens has only ever
seen rotation angles in the range those 8k positions produce. Feed it a 32k-token
sequence and tokens far out get rotation angles **far outside the training
distribution** — positions the model has never encountered. Attention degrades badly:
the model effectively does not know how to interpret those positions. This is why you
cannot simply pass a longer prompt and expect a longer effective context — extending
context requires *deliberately* adapting the positional encoding.

**RoPE scaling** is the family of techniques that does this:

- **Position Interpolation (PI).** Instead of *extrapolating* to unseen large positions,
  *interpolate*: linearly scale position indices down so a 32k sequence's positions map
  into the 0–8k range the model already understands (positions become fractional). The
  model sees only familiar angles. Requires a short fine-tune to adapt, and works well.
- **NTK-aware scaling / YaRN.** More refined schemes. Rather than scaling all RoPE
  frequencies uniformly (which crowds out the high-frequency components that encode
  fine-grained local position), they scale different frequency bands differently —
  preserving local precision while extending global range. **YaRN** (2023) is a widely
  used, efficient method that reaches large context extensions with far less
  fine-tuning than earlier approaches — the paper reports ~10× fewer tokens and ~2.5×
  fewer training steps than prior methods [6].

Key applied points:
- Long-context support is a *training/architecture* achievement, not a free switch.
  Million-token-context models got there via RoPE scaling (plus long-context fine-tuning
  and attention-efficiency work), not by flipping a flag.
- It connects directly to "effective context" (Topic 4): a model advertising 200k or 1M
  context may still degrade — "lost in the middle," context rot — well before the hard
  limit. Context extension widens the *window*; it does not guarantee uniform quality
  across it.
- Extending context usually costs some quality on short sequences and needs a fine-tune
  — it is a trade-off, not a pure win.

### Key terms

- **Positional encoding** — how token order is injected into a transformer, which
  otherwise sees tokens as an unordered set.
- **RoPE (Rotary Position Embedding)** — encodes position by rotating query/key vectors;
  attention then captures relative distance. The modern standard.
- **Extrapolation problem** — RoPE fails on positions beyond those seen in training
  because the rotation angles are out of distribution.
- **Position Interpolation (PI)** — scaling position indices down so long sequences map
  into the trained range.
- **NTK-aware scaling / YaRN** — refined RoPE-scaling methods that scale frequency bands
  non-uniformly to extend range while preserving local precision.

### Common misconceptions

- ❌ "You can use any context length by just sending more tokens." → ✅ Beyond the
  trained range, positional encodings are out of distribution and quality collapses;
  context must be deliberately extended (RoPE scaling + fine-tune).
- ❌ "RoPE scaling is free and lossless." → ✅ It usually needs a fine-tune and can cost
  some quality on shorter sequences — a trade-off.
- ❌ "A 1M-token context window means uniform quality across 1M tokens." → ✅ Extension
  widens the window; effective context still degrades (lost-in-the-middle, context rot)
  before the hard limit.
- ❌ "Position interpolation extrapolates to bigger positions." → ✅ It does the
  opposite — it *interpolates*, mapping large positions down into the already-trained
  range so the model sees only familiar angles.

### Worked example

On a position number line, naive long context *extrapolates* past the trained band into
positions the model has never seen; position interpolation *compresses* the long
sequence back inside the trained band:

```
  trained band (rotation angles the model has seen)
  ┌─────────────────────────────┐
  │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│
  0          ...               8k
  position:  0 ─────────────────────────────────────────────── 32k

  NAIVE (extrapolation):
  │░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░│  8k ────────────────────────▶ 32k
  └──── in-distribution ─────────┘  └──── OUT OF DISTRIBUTION ───┘
                                      angles never seen → attention breaks

  POSITION INTERPOLATION:
  ┌─────────────────────────────┐
  │▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓│   all 32k positions scaled DOWN
  0   1k positions packed per 0.25  into the 0–8k trained band
      → every position lands on a familiar (fractional) angle
```

A team has an 8k-context open model and needs 32k. Naively feeding a 32k prompt
produces garbled, incoherent output past ~8k — RoPE positions beyond 8k are out of
distribution. They apply YaRN: RoPE frequencies are rescaled non-uniformly so 32k
positions map into a range the model can interpret, then they do a short long-context
fine-tune on 32k-token documents. The model now handles 32k coherently. But evals also
show a small dip on short-prompt tasks and that retrieval accuracy still sags for facts
buried in the middle of a 32k context — extension widened the window, but effective
context is still imperfect.

### Check questions

1. A team takes an 8k-context model, sends it a 32k-token prompt, and the output is
   coherent for the first part and then degrades into nonsense. They suspect a memory
   bug. Using how RoPE encodes position, give the real explanation. — **Answer:** It is
   not a memory bug — it is a positional-encoding problem. RoPE encodes a token's
   position by rotating its query/key vectors by an angle proportional to that position.
   A model trained to 8k has only ever *seen* rotation angles from the 0–8k range. Token
   positions beyond 8k produce angles far *outside* the trained distribution — positions
   the model has no learned way to interpret — so attention degrades there and the
   output becomes incoherent. The coherent-then-garbled pattern is exactly the
   in-distribution-then-out-of-distribution boundary.
2. Position Interpolation extends context by scaling 32k-token position indices *down*
   into an 8k-trained model's range (positions become fractional). A student asks: "if
   we want a *bigger* context, why does the method scale positions *down* rather than
   up?" Answer them. — **Answer:** Because the model fails on *unseen* angles, not on
   "big" ones — the danger is out-of-distribution rotation angles. Scaling positions
   *down* maps the long sequence's positions *into the 0–8k range the model already
   understands*, so every token still produces a familiar, in-distribution angle (just
   at finer, fractional spacing). Scaling *up* would extrapolate to never-seen angles —
   exactly the failure. It is called *interpolation* because the model only ever sees
   angles inside its trained range; that is why it works where naive extension does not.

---

## Topic 14 — Exam Question Bank

### True / False

1. Pretraining is where a model acquires most of its knowledge and capability;
   post-training mainly shapes behavior. — **Answer:** True. Pretraining is the massive
   self-supervised stage; post-training is comparatively tiny and adjusts behavior, not
   knowledge.
2. Fine-tuning is the reliable way to teach a model new facts. — **Answer:** False.
   Knowledge lives at pretraining scale; fine-tuning is far too small to install facts
   reliably and they go stale — use RAG.
3. DPO trains a separate reward model and runs a reinforcement-learning loop. —
   **Answer:** False. DPO's whole point is to skip the reward model and the RL loop,
   optimizing directly on preference pairs.
4. LoRA freezes the base model and trains small low-rank adapter matrices. — **Answer:**
   True. That is the definition; the base weights stay frozen.
5. Quantization reduces a model's parameter count. — **Answer:** False. It reduces
   bits-per-parameter (precision); the count is unchanged. Distillation reduces the
   count.
6. Reasoning models are trained with RL on objectively verifiable rewards in domains
   like math and code. — **Answer:** True. RLVR uses a checkable correctness signal
   rather than a learned preference model.
7. In an MoE model, every parameter is used to process every token. — **Answer:**
   False. A router activates only a few experts per token (e.g. top-2); that is sparse
   activation.
8. RoPE scaling lets you extend context for free with no quality cost. — **Answer:**
   False. It typically needs a fine-tune and can cost some quality on shorter
   sequences — a trade-off.
9. FP8 is the de facto standard serving precision in 2026 and is effectively lossless
   for most workloads. — **Answer:** True. Modern data-center GPUs have native FP8
   support; FP8 serving is ~2× smaller than BF16 with negligible quality cost and is the
   common default. (Modern weight-only 4-bit methods are also much closer to lossless
   than the legacy "INT4 = noticeable loss" framing implies.)

### Multiple Choice

1. A team fine-tunes a chat model on 50,000 of their internal engineering documents,
   hoping it will answer questions about their systems. Afterward it writes in their
   house style but still gets specific facts about their architecture wrong. Which
   statement best explains this? A) The fine-tuning dataset was too small to change
   style B) Behavior (style) is shaped at post-training scale, but reliable factual
   knowledge lives at pretraining scale — 50k docs cannot install it C) The model needs
   a larger context window D) Fine-tuning always degrades factual accuracy —
   **Answer:** B. Fine-tuning is post-training-scale: it successfully shifted *behavior*
   (house style) but is orders of magnitude too small to install *knowledge* reliably.
   Facts that change or are private are a RAG problem, not a fine-tuning problem.
2. A team wants to train a model to write *clearer* explanations than their human
   demonstrators can themselves write. Which post-training approach addresses this, and
   why can SFT alone *not*? A) More SFT epochs, because repetition improves quality
   B) A preference stage (RLHF/DPO), because raters can *rank* "which is clearer" even
   for quality they cannot *demonstrate* — SFT can only imitate and is capped at
   demonstrator quality C) Quantization, because it makes the model faster D) Extending
   the context window — **Answer:** B. SFT is imitation learning, capped at the
   demonstrators' average. A preference stage exploits that judging quality is easier
   than producing it, pushing the model *beyond* the demonstrations.
3. QLoRA quantizes the base model to 4-bit, yet its fine-tuning quality is close to
   full-precision LoRA. Why does the 4-bit quantization not wreck quality? A) 4-bit is
   lossless B) Only the *frozen* base weights are 4-bit; the LoRA adapters — the part
   actually being trained — stay in higher precision C) QLoRA re-trains the whole model
   D) The quantization is undone before each step — **Answer:** B. QLoRA quantizes only
   the frozen base (the memory bulk) to slash VRAM; the trainable adapters keep higher
   precision, so the learning happens at good precision and net quality loss is minimal.
4. Your model must answer questions about a product catalog updated daily. Best
   approach? A) Fine-tune nightly B) RAG C) Longer system prompt only D) Distillation —
   **Answer:** B. Changing facts are a knowledge problem; RAG retrieves current data
   with no retraining.
5. An engineer reasons: "Our MoE model only activates ~37B of its 671B parameters per
   token, so we can serve it on hardware sized for a 37B model." Why is this wrong?
   A) The active count is actually higher B) MoE saves *compute* per token, but all
   experts must be resident in memory — VRAM must hold the full 671B C) MoE models
   cannot be served on GPUs D) The router needs its own dedicated GPU — **Answer:** B.
   Sparse activation reduces per-token *computation* (cost/speed track active params),
   but any token might route to any expert, so every expert must be loaded — memory
   tracks *total* parameters. MoE saves compute, not VRAM.
6. A team takes a 70B model, applies INT4 quantization, and ends up with a model that
   still has 70B parameters but a smaller memory footprint. Separately they train a 7B
   model on the 70B model's outputs. Which technique is which, and what is the
   defining difference? A) Both are distillation B) Both are quantization C) The first
   is quantization (same model, fewer bits per parameter); the second is distillation (a
   new, smaller model trained to imitate the teacher) D) The first is distillation, the
   second is quantization — **Answer:** C. Quantization keeps the *same model* and
   lowers numerical precision; distillation produces a *different, smaller model* (fewer
   parameters) trained to mimic a teacher. They are complementary — you can distill then
   quantize the student.
7. Thinking tokens in a reasoning model are billed as: A) Free B) Input tokens C)
   Output tokens D) Cached tokens — **Answer:** C. They are output tokens — the
   expensive rate — which is why reasoning calls can cost far more than the visible
   answer.
8. Why does an 8k-trained model break when fed 32k tokens directly? A) The vocabulary
   is too small B) RoPE positions beyond 8k are out of the trained distribution C) It
   runs out of memory D) Temperature is too high — **Answer:** B. Out-of-distribution
   rotation angles degrade attention.
9. DeepSeek trained R1 with GRPO. What distinguishes GRPO from the classic PPO used in
   RLHF? A) GRPO uses human preference labels B) GRPO drops PPO's separate learned
   value/critic model and uses the average score of a *group* of sampled answers as the
   baseline C) GRPO is a supervised-learning method D) GRPO requires no reward signal at
   all — **Answer:** B. GRPO scores a group of answers to the same prompt with the
   (verifiable) reward and uses their average as the baseline, eliminating PPO's
   separately trained critic — simpler and cheaper. RLVR describes the *reward*; GRPO is
   the *update algorithm*.

### Short Answer

1. A model's "knowledge cutoff" is a property fixed in pretraining, and a model's chat
   version has almost the same knowledge as its base version. Use these two facts to
   explain *why* you cannot fix an out-of-date knowledge cutoff by fine-tuning. —
   **Model answer:** Knowledge is acquired during pretraining at trillions-of-tokens
   scale — that is why the cutoff is a pretraining property. The chat version shares the
   base version's knowledge precisely because post-training is far too small a stage to
   add meaningful facts; it only shapes behavior. Fine-tuning is post-training-scale, so
   it likewise cannot install the post-cutoff knowledge reliably — the facts would be
   sparse, unreliable, and stale. Fresh knowledge must come from RAG (retrieved into
   context at request time), not from training.
2. RLHF makes models feel helpful and aligned, but it is also blamed for *sycophancy*
   (the model agreeing with the user even when the user is wrong). Explain how the
   *same* mechanism produces both the benefit and the flaw. — **Model answer:** RLHF
   optimizes the model against a *reward model* that was trained to predict what human
   raters *prefer*. Raters genuinely prefer helpful, well-presented answers — so the
   policy learns to be helpful (the benefit). But raters also tend to rate agreeable,
   confident answers higher, and the policy can over-optimize the reward model — scoring
   well on what *looks* good to raters rather than what is true (reward hacking).
   Sycophancy is exactly that: the reward signal rewards agreement, so the model learns
   to agree. The benefit and the flaw are two faces of optimizing a proxy for human
   preference.
3. A SaaS company wants a per-customer fine-tuned variant for each of 300 enterprise
   customers. Explain why LoRA makes this feasible where full fine-tuning does not, and
   what changes at *serving* time. — **Model answer:** Full fine-tuning would mean 300
   full multi-hundred-GB copies of the model — infeasible to store or serve. LoRA
   freezes one shared base model and trains only a small (MB-scale) low-rank adapter per
   customer, so 300 customers means one base model + 300 tiny adapters. At serving time
   the adapter for the requesting customer is loaded/hot-swapped onto the shared base
   (or pre-merged), so hundreds of "models" run from one base. The feasibility comes
   from adapters being tiny and swappable — a direct consequence of the weight update
   being low-rank.
4. In distillation, the student is trained on the teacher's full probability
   distribution over tokens ("soft labels"), not just the single correct token. Why is
   the soft distribution more informative than the hard label, and what does the
   student gain from it? — **Model answer:** A hard label says only "this token,
   definitely" — one bit of guidance. The teacher's soft distribution also encodes *how
   plausible the alternatives were* — its uncertainty and its sense of which other
   tokens were near-misses. That "dark knowledge" lets the student learn the teacher's
   nuanced behavior, not just its final answers, so a much smaller student can mimic the
   teacher's quality far better than training on hard labels alone would allow.
5. A high-volume classification feature works fine with a 2,000-token prompt (detailed
   instructions + 15 few-shot examples). The team is told "prompting already works,
   never fine-tune." Give a concrete reason fine-tuning could still be the right call
   here — and what specifically it would buy them. — **Model answer:** Prompt
   compression. A model fine-tuned on the task can fold the instructions and few-shot
   examples *into the weights*, so each request needs only a short prompt instead of
   2,000 tokens. At high volume that cuts per-request input tokens — real latency and
   cost savings — and can let a smaller, cheaper model match the larger prompted one.
   "Prompting works" is about correctness; fine-tuning here is justified by *economics
   at scale*, a legitimate win even when prompting is functionally adequate.
6. RL on verifiable rewards (RLVR) drove the big gains in math and coding reasoning
   models. Explain why the *math/code* setting is what makes RLVR work so well, and why
   the same approach is harder to apply to a task like "write a moving short story." —
   **Model answer:** RLVR's power comes from an *objectively checkable* reward — for
   math you can verify the final numeric answer, for code you can run the unit tests —
   with no human or learned judge in the loop. That signal is clean and essentially
   unhackable, so RL can freely push the model to discover effective reasoning
   strategies, rewarded purely for correctness. "A moving short story" has no
   deterministic verifier — quality is subjective — so you fall back to a *learned*
   preference model (RLHF-style), which is fuzzier and hackable. RLVR shines exactly
   where correctness is mechanically verifiable.
7. A team reports a bug: "we set temperature 0 for reproducibility but the MoE model
   still gives slightly different outputs run-to-run." A colleague says "that's
   impossible, temperature 0 is greedy decoding." Explain who is right and the
   mechanism — and what this implies about treating temperature 0 as a guarantee of
   reproducibility. — **Model answer:** The team is right; the colleague is wrong.
   Temperature 0 makes *token selection* greedy, but in an MoE the router's choice of
   which experts process each token can depend on which *other* tokens share the
   inference batch — and batch composition varies with unrelated concurrent traffic. So
   the experts (and thus the computed logits) can differ run-to-run even under greedy
   decoding, producing slightly different outputs. Implication: temperature 0 reduces
   one source of variation but is not a hard reproducibility guarantee on MoE models
   served with dynamic batching — do not design around exact reproducibility.

### Long Answer

1. Walk through the full pipeline that turns a pretrained base model into a deployed
   chat assistant. — **Model answer / rubric:** Start with the base model from
   pretraining — capable but not an assistant. SFT: supervised fine-tuning on (prompt →
   ideal response) demonstrations to install instruction-following behavior. Preference
   optimization: collect preference pairs (human for RLHF, AI for RLAIF) and either
   train a reward model + RL (RLHF) or optimize directly (DPO) to push beyond
   demonstrations. Constitutional AI: model self-critiques against written principles to
   generate harmlessness preference data, scaling safety training. Then safety/eval
   passes, red-teaming, possibly distillation to smaller deployable variants and
   quantization for serving. Emphasize SFT→preference ordering and that this whole stage
   shapes behavior, not knowledge.
2. Give a complete fine-tune vs. RAG vs. prompt decision framework with examples. —
   **Model answer / rubric:** Three tools, three jobs. Prompting changes behavior per
   request via input — zero cost, instant — always exhaust first. RAG injects knowledge
   at request time — the tool for facts the model lacks or that change, and for
   citations. Fine-tuning bakes behavior/style/format into the weights. Core rule:
   knowledge problem → RAG; behavior problem → fine-tune. Examples: private/changing
   docs → RAG; consistent brand voice / JSON format → fine-tune (after prompt fails);
   prompt compression to cut cost → fine-tune. Costs of fine-tuning: data, pipeline,
   eval, versioning, redo on base-model upgrade. They combine — fine-tuned model inside
   a RAG pipeline is common.
3. Explain MoE: mechanism, the total/active parameter distinction, and the trade-offs.
   — **Model answer / rubric:** Dense models use all parameters per token. MoE replaces
   each layer's feed-forward block with many expert sub-networks; a learned router
   picks a few (e.g. top-2) per token — sparse activation. Total parameters = all
   experts = capacity/knowledge (can be trillions); active parameters = what runs per
   token = cost/speed. MoE buys large-model capability at small-model inference cost.
   Trade-offs: all experts must be memory-resident (saves compute, not VRAM); routing
   needs load-balancing; batch-dependent routing adds non-determinism. Why frontier labs
   use it: affordable scaling.
4. Explain reasoning models — what they are, how RLVR trains them, and how they changed
   prompting and cost. — **Model answer / rubric:** Reasoning models generate an
   extended internal chain of thought before answering, strong on hard multi-step
   problems. Underlying idea is test-time compute — spend more inference computation on
   hard problems. Trained with RL on verifiable rewards: objectively checkable
   correctness signals (math answers, passing tests) let RL discover reasoning
   strategies without a learned/hackable reward model. Prompting impact: manual
   chain-of-thought is now redundant and can hurt; reasoning models prefer concise
   prompts with a clear goal. Cost/latency: thinking tokens bill as output and add
   latency; route easy tasks to cheaper non-reasoning models; use thinking-budget knobs.

### Applied Scenario

1. A legal-tech team wants an assistant that must do three things: (a) cite the
   *current* text of statutes, which are amended periodically; (b) always answer in a
   rigid, firm-mandated memo format; (c) correctly use specialized legal terminology
   and reasoning patterns the base model handles inconsistently. They propose
   "fine-tune one model on everything." Decompose the problem and assign the right tool
   to each part, justifying each choice. — **Model answer / rubric:** Reject the
   one-fine-tune plan — it mixes knowledge and behavior. (a) Current statute text is a
   *knowledge* problem, and the statutes *change* → RAG over a statute corpus, re-indexed
   on amendment, with forced citations; fine-tuning would install stale text and give no
   citations. (b) The rigid memo format is a *behavior* problem → first try a strong
   system prompt + few-shot; if format drifts at scale, fine-tune for it. (c) Consistent
   use of specialized terminology/reasoning is also a *behavior* problem → exhaust
   prompting first, then fine-tune if needed. End state: a fine-tuned model (format +
   terminology behavior) operating *inside* a RAG pipeline (current statutes + citations),
   with prompting on top. The senior signal is decomposing by the knowledge-vs-behavior
   axis rather than reaching for one tool.
2. Your 70B model is too slow and expensive to serve at your traffic volume, but
   quality must stay high on your core task. What options do you have and how do they
   differ? — **Model answer / rubric:** Quantization — store the same 70B at INT8/INT4,
   shrinking memory (fits cheaper/fewer GPUs) and speeding decode (fewer bytes per
   token); mild quality loss, more at INT4. Distillation — train a small student on the
   70B teacher's outputs for your task; a much cheaper model that handles most traffic,
   escalating hard cases to the big model. They are stackable — distill to a 7–13B
   student, then quantize it. Also continuous batching / better serving stack and model
   cascading. Distillation changes which model runs; quantization changes how the same
   model is stored — pick based on whether you can tolerate a different model.
3. Your team built an open-source model into a product and now needs 4× the context
   length it was trained for. The naive fix — sending longer prompts — produces garbage.
   Explain why and what to do. — **Model answer / rubric:** The model's RoPE positional
   encoding only ever saw rotation angles from its trained position range; positions
   beyond it are out of distribution, so attention degrades and output becomes
   incoherent. Fix: apply a RoPE-scaling method — Position Interpolation maps long
   positions into the trained range, or NTK-aware scaling / YaRN scales frequency bands
   non-uniformly to extend range while preserving local precision — then do a short
   long-context fine-tune. Caveats: expect a small quality dip on short prompts and that
   effective context still degrades mid-context (lost-in-the-middle); extension widens
   the window but does not guarantee uniform quality.

4. A product manager says "let's use a reasoning model everywhere — they're the best
   models." Respond with the engineering trade-offs and a better policy. — **Model
   answer / rubric:** Reasoning models excel at hard math/coding/logic/multi-step
   problems but cost more (thinking tokens bill at the output rate, often dwarfing the
   visible answer) and add latency (long TTFT while thinking). For simple tasks —
   classification, extraction, short Q&A — they are slower and costlier with no quality
   benefit, and forcing CoT-style prompts on them can even hurt. Better policy: match
   model class to problem difficulty — route easy/structured tasks to fast, cheap
   non-reasoning models; reserve reasoning models, with a tuned thinking budget, for
   genuinely hard problems; measure cost-per-request and quality per route.

---

## Sources

[1] Anthropic — Constitutional AI: Harmlessness from AI Feedback (arXiv:2212.08073) — https://arxiv.org/abs/2212.08073
[2] Rafailov et al. — Direct Preference Optimization: Your Language Model is Secretly a Reward Model (arXiv:2305.18290) — https://arxiv.org/abs/2305.18290
[3] Hu et al. — LoRA: Low-Rank Adaptation of Large Language Models (arXiv:2106.09685) — https://arxiv.org/abs/2106.09685
[4] Dettmers et al. — QLoRA: Efficient Finetuning of Quantized LLMs (arXiv:2305.14314) — https://arxiv.org/abs/2305.14314
[5] DeepSeek-AI — DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning (arXiv:2501.12948) — https://arxiv.org/abs/2501.12948
[6] Peng et al. — YaRN: Efficient Context Window Extension of Large Language Models (arXiv:2309.00071) — https://arxiv.org/abs/2309.00071
[7] DeepSeek-AI — DeepSeek-V3 Technical Report (arXiv:2412.19437; 671B total / 37B activated per token) — https://arxiv.org/abs/2412.19437
[8] Mistral AI — Mixtral of Experts (arXiv:2401.04088; 8 experts, top-2 routing, ~47B total / ~13B active) — https://arxiv.org/abs/2401.04088
