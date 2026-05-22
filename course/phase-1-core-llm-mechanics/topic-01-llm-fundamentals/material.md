# Topic 01 — LLM Fundamentals — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This file is taught sub-chapter by sub-chapter, in order. Each sub-chapter has core
teaching content, a list of key terms, a list of common misconceptions, a worked
example that makes the idea concrete, and a few check questions to confirm
understanding before moving on. A full exam question bank sits at the end of the file;
the tutor draws from it for the gated topic exam. Do not skip ahead — every later
sub-chapter assumes the ones before it, and Topics 2–16 assume all of Topic 1.

---

## 1.1 — What an LLM is — next-token prediction as the whole job

### Concept

A large language model (LLM) is, mechanically, a function that takes a sequence of
tokens and produces a probability distribution over what the *next* token should be.
That is the entire job. Everything else an LLM appears to do — answering questions,
writing code, holding a conversation, reasoning through a math problem — is an
*emergent consequence* of doing next-token prediction extremely well over a vast amount
of text.

It helps to internalize this literally. Given the input `The capital of France is`, the
model does not "look up" Paris. It computes, for every token in its vocabulary, a score
representing how likely that token is to come next given everything it learned during
training. The token `Paris` gets a high score; `banana` gets a very low one. The model
then picks a token from that distribution, appends it to the input, and repeats.

Why does this simple objective produce something that looks like intelligence? Because
to predict the next token well across trillions of words of human text, the model is
forced to learn an enormous amount of structure: grammar (to predict the next word in a
sentence), facts (to predict `Paris` after `capital of France is`), reasoning patterns
(to predict the conclusion of an argument), code semantics (to predict the next line of
a function), and even style and tone. Next-token prediction is a "pretext task" — the
task itself is trivial to state, but solving it well *requires* learning the world.

This framing has hard practical consequences you should carry into every later topic:

- **The model has no goals, memory, or beliefs of its own.** It is producing the
  statistically plausible continuation of the text it was given. When it "lies" or
  "hallucinates," it is not deceiving you — it is generating a plausible-looking
  continuation that happens to be false. Plausibility and truth are correlated in
  training data but not identical.
- **The model is a fixed function during use.** It does not learn from your
  conversation. (More on this in 1.2 and 1.7.)
- **Output is sampled, not deterministic by default.** Because the model produces a
  *distribution*, the same input can yield different outputs depending on how you draw
  from that distribution (Topic 3).
- **"Reasoning" is just more next-token prediction.** When a model writes out a
  step-by-step solution, each step is another predicted token. Reasoning models (Topic
  14) are trained to produce longer, more useful intermediate token sequences before
  the final answer — but it is still next-token prediction underneath.

A useful mental model: an LLM is an extraordinarily sophisticated autocomplete. That is
not a dismissal — autocomplete that has compressed a large fraction of human written
knowledge into its weights is a remarkable tool. But the autocomplete framing keeps you
honest about what the model is and is not doing.

### Key terms

- **LLM (large language model)** — a neural network trained to predict the next token
  in a sequence, scaled to billions of parameters and trained on a large text corpus.
- **Token** — the unit of text the model reads and writes; roughly a word-piece (full
  treatment in Topic 2).
- **Next-token prediction** — the core objective: given a token sequence, output a
  probability distribution over the next token.
- **Pretext task** — a simple, self-supervised training task (here, predict the next
  token) whose solution forces the model to learn deeper structure.
- **Vocabulary** — the fixed set of all tokens the model can read or emit.
- **Emergent capability** — a behavior (e.g., translation, arithmetic) not explicitly
  trained for, arising as a side effect of scaling up next-token prediction.

### Common misconceptions

- ❌ The model "looks up" answers in a database → ✅ It computes a probability
  distribution from learned weights; there is no retrieval step unless you bolt one on
  (RAG, Topic 10).
- ❌ The model "knows" when it is wrong → ✅ It produces plausible continuations;
  plausibility ≠ truth. Hallucination is the model working as designed.
- ❌ Reasoning models do something other than next-token prediction → ✅ They still
  predict tokens; they are trained to produce more useful intermediate tokens first.
- ❌ The model "understands" your question the way a person does → ✅ Be agnostic about
  "understanding"; mechanically it is pattern-completing.

### Worked example

Take the prompt `2 + 2 =`. The model does not run an arithmetic unit. It produces a
distribution over the vocabulary; in that distribution the token `4` has very high
probability because the pattern `<small number> + <small number> =` was followed by the
correct sum millions of times in training. Now try `4817 + 2956 =`. The model still
only predicts a plausible next token. Because that exact sum appeared rarely (if ever),
the prediction is far less reliable — which is exactly why naive LLMs are bad at
multi-digit arithmetic and why we give them a calculator tool (Topic 7). Same mechanism;
the difference is entirely how well the pattern was represented in training data.

### Check questions

1. A model confidently invents a citation to a paper that does not exist. A colleague
   says "it's broken — patch it to stop lying." Using the next-token-prediction framing,
   is "lying" or "broken" the right description? — **Answer:** Neither. The model is not
   deceiving (it has no goals or beliefs) and it is not malfunctioning — it generated the
   *statistically plausible* continuation of "the citation is...", and a plausible-looking
   citation can be fabricated. Hallucination is the mechanism working as designed:
   plausibility and truth are correlated in training data but not identical. A "patch" at
   the model level misframes the problem.
2. Two facts are equally *true*: the capital of France, and the home address of a
   private individual mentioned once in an obscure document. An LLM reliably produces the
   first and not the second. Why, given that next-token prediction has no notion of
   truth? — **Answer:** Reliability tracks how strongly a pattern was represented in
   training data, not whether it is true. "Paris" follows "capital of France is" in a
   vast number of texts, so it gets a high-probability prediction; the obscure address
   appeared rarely or never, so the prediction is unreliable. The model is doing the same
   thing in both cases — only the statistical support differs.
3. A skeptic says "if it's just predicting the next token, it can't really *reason*
   through a multi-step problem." Where is this reasoning wrong, and where is it
   partly right? — **Answer:** Wrong: producing a correct step-by-step solution *is*
   within the reach of next-token prediction, because each step is a predicted token and
   the corpus contains vast amounts of worked reasoning to learn from. Partly right: there
   is no separate "reasoning engine" — the apparent reasoning is still pattern completion,
   so it is only as reliable as the patterns learned, which is why reasoning models are
   trained specifically to produce more useful intermediate tokens.

---

## 1.2 — Training vs. inference; why weights are frozen when you use the model

### Concept

An LLM's life has two completely separate phases: **training** and **inference**.
Conflating them is one of the most common sources of confusion, so pin them down.

**Training** is the one-time (and very expensive) process of building the model. The
model starts with random parameters. It is shown enormous amounts of text and, for each
position, asked to predict the next token. Its prediction is compared to the actual
next token; the difference is a *loss*. Backpropagation computes how each parameter
contributed to that loss, and an optimizer nudges every parameter slightly to reduce
it. Repeat this trillions of times and the parameters converge to values that encode
the structure of language. Training a frontier model takes weeks to months on thousands
of GPUs and costs tens to hundreds of millions of dollars. After training (and
post-training — SFT, RLHF, etc., covered in Topic 14), the parameters are **frozen**.

**Inference** is what happens every time you call the model. The frozen weights are
loaded, your input runs *forward* through the network, and a next-token distribution
comes out. Inference does **not** run backpropagation. It does **not** update any
parameter. It is a pure forward pass — a deterministic function of (weights, input),
modulo the sampling step (Topic 3).

The single most important consequence: **the model does not learn from your
conversation.** When you correct it ("no, the answer is 7") and it appears to "learn,"
it has not changed a single weight. It is simply that your correction is now part of
the input context for the next forward pass, so the model conditions on it. Close the
conversation, start a new one, and that correction is gone — because nothing in the
weights changed. (This is the bridge to 1.7: weights vs. context.)

This is why "the model remembered what I told it yesterday" is impossible unless your
application explicitly stored that text and re-fed it. The model itself is amnesiac
between calls. Anything that looks like memory is your application re-supplying text
(Topic 4) or a retrieval system fetching it (Topic 10).

Why are weights frozen rather than continuously updated? Several reasons: (1)
backpropagation is far more expensive than a forward pass, so live updates would be
slow and costly; (2) updating weights from arbitrary user input would let users
degrade or poison the model; (3) a frozen model is *reproducible* and *testable* — you
can evaluate it, version it, and reason about its behavior. To genuinely change a
model's weights you must run another training process — fine-tuning (Topic 14) — which
is a deliberate, offline, controlled procedure, not something that happens during a
chat.

### Key terms

- **Training** — the offline process of adjusting parameters via backpropagation to
  minimize next-token prediction loss; done once, very expensive.
- **Inference** — running input through the frozen model to get an output; a forward
  pass only, done on every API call.
- **Forward pass** — input flows through the network to produce an output; no
  parameters change.
- **Backpropagation** — the training-only algorithm that computes parameter gradients
  from the loss.
- **Loss** — a number measuring how wrong the model's predictions were; training
  minimizes it.
- **Frozen weights** — parameters fixed after training; identical for every inference
  request until the model is retrained or fine-tuned.
- **Fine-tuning** — a separate, deliberate training run that *does* change weights
  (Topic 14).

### Common misconceptions

- ❌ The model learns from chatting with me → ✅ Inference never updates weights. The
  model "remembers" only what is re-supplied in the context window.
- ❌ Training and inference are the same kind of operation → ✅ Training runs
  backprop and updates parameters; inference is a forward pass only.
- ❌ If many users tell the model something, it eventually believes it → ✅ No weight
  changes from usage. Such feedback only influences the model if curated into a future
  training run.
- ❌ Fine-tuning happens automatically → ✅ Fine-tuning is a separate, deliberate
  offline process.

### Worked example

You tell ChatGPT "my name is Sam" and it uses your name for the rest of the chat.
Tomorrow, in a fresh chat, it has no idea who you are. Why? Because "my name is Sam"
never touched a weight. Within the first chat, that sentence stayed in the context
window and was re-fed on every turn (Topic 4.2), so each forward pass conditioned on
it. A new chat is a new context window — empty — so the information is simply gone.
(Consumer products add a separate "memory" feature that re-injects saved facts; that is
application-layer engineering, not the model learning.)

### Check questions

1. A product manager asks: "10,000 users corrected the model on the same fact this
   month — surely by now the model has *learned* it?" Explain why this is false and what
   *would* have to happen for those corrections to change the model. — **Answer:**
   Inference never updates weights, so 10,000 corrections during use change nothing —
   each was just transient context for its own request. For the corrections to actually
   change the model, someone would have to *curate* them into a dataset and run a
   deliberate, offline fine-tuning (or post-training) job; usage volume alone has zero
   effect on the weights.
2. Within a single chat, you tell the model your name and it uses it for ten turns —
   yet you never fine-tuned anything. Reconcile "the model did not learn" with "it
   clearly remembers my name." — **Answer:** Both are true because "remember" here is not
   a weight change. Your name stayed in the messages array and was re-fed to every
   forward pass (append-and-resend), so each turn *conditioned* on it. The weights are
   identical to before the chat; "memory" is the application resupplying text, not the
   model storing anything.
3. Frozen weights are sometimes described as a limitation. Give two distinct ways the
   *same* property is actually an asset for an engineer running the model in production.
   — **Answer:** Any two: (a) **reproducibility/testability** — a fixed function can be
   evaluated, versioned, and reasoned about; behavior does not drift mid-deployment;
   (b) **safety against poisoning** — arbitrary user input cannot degrade the model for
   everyone, since no input touches the weights; (c) **cost/latency** — a forward pass is
   far cheaper than backprop, so serving stays fast and predictable.

---

## 1.3 — The transformer at a block level — embeddings → attention → FFN → unembedding

### Concept

Nearly every modern LLM is a **transformer**. You do not need the backprop math, but
you must be able to whiteboard the data flow. Follow a token sequence through the
network.

**1. Embedding.** Each input token (an integer ID from the vocabulary) is mapped to a
vector of numbers — its **embedding** — via a lookup in the embedding matrix. A vector
might have, say, 4096 dimensions. This vector is the model's internal representation of
that token's meaning. Because the transformer has no inherent sense of order,
**positional information** is also injected here (or inside attention, e.g., RoPE) so
the model knows token 5 came before token 6.

**2. Transformer blocks (the bulk of the model).** The sequence of embedding vectors
passes through a stack of identical **transformer blocks** — dozens of them in a large
model. Each block has two main sub-layers:

- **Self-attention** — lets each token's vector pull in information from other tokens
  in the sequence. This is the mechanism by which context flows: the representation of
  the word `it` can incorporate information from the noun it refers to earlier in the
  sentence. (Full detail in 1.4.)
- **Feed-forward network (FFN/MLP)** — a position-wise neural network applied
  independently to each token's vector. If attention *mixes information across
  positions*, the FFN *processes the information at each position*. The FFN holds a
  large share of the model's parameters and is widely understood to be where much
  factual knowledge is stored.

Each sub-layer is wrapped with a **residual connection** (the input is added back to
the output, so information and gradients flow cleanly through a deep stack) and a
**normalization** step (which keeps the numbers in a stable range). Two details of that
normalization are the modern default and worth knowing. First, *what* normalizer:
frontier models have largely moved from classic layer normalization to **RMSNorm**
(root-mean-square normalization), a simpler, cheaper variant that rescales activations
by their root-mean-square without subtracting a mean — it trains comparably and costs
less compute. Second, *where* the norm sits relative to the sub-layer: in **pre-norm**
placement (the modern standard) the norm is applied to the sub-layer's *input*, so the
residual path stays a clean, unnormalized highway from the first block to the last; the
older **post-norm** placement normalized the *output* after the residual add, which
makes very deep stacks much harder to train (gradients are repeatedly rescaled on the
main path). Pre-norm with RMSNorm is the combination you should assume for a current
model. Stacking many blocks lets the model build progressively more abstract
representations: early layers capture surface features, later layers capture semantics
and task structure.

**3. Unembedding (the LM head).** After the final block, each position holds a
context-rich vector. To predict the next token, the model projects the final vector
through the **unembedding matrix** (the "language-model head"), producing one score per
vocabulary token. Those raw scores are the **logits** (1.6). A softmax turns them into a
probability distribution.

So the whole network is: **embed → [attention → FFN] × N → unembed → logits**. Two
intuitions to keep: attention is the *only* place tokens talk to each other (everything
else is position-wise), and the FFN layers carry most of the parameters and most of the
stored knowledge.

### Key terms

- **Transformer** — the neural-network architecture underpinning modern LLMs, built
  from stacked attention + FFN blocks.
- **Embedding** — the vector representation of a token; produced by an embedding-matrix
  lookup.
- **Positional encoding** — information added so the model knows token order; modern
  models often use RoPE (rotary position embeddings).
- **Transformer block** — one repeated unit: self-attention sub-layer + FFN sub-layer,
  with residual connections and layer norm.
- **Feed-forward network (FFN/MLP)** — a position-wise network inside each block;
  processes each token independently; holds most parameters and much factual knowledge.
- **Residual connection** — adds a sub-layer's input to its output; enables training of
  very deep stacks.
- **Normalization (LayerNorm / RMSNorm)** — normalizes activations to keep numerical
  scale stable; frontier models default to **RMSNorm**, a cheaper mean-free variant.
- **Pre-norm vs. post-norm** — whether the normalizer is applied to a sub-layer's
  *input* (pre-norm, the modern default — keeps the residual path clean and lets very
  deep stacks train) or to its *output* after the residual add (post-norm — harder to
  train deep).
- **Unembedding / LM head** — the final projection from a token's vector to one logit
  per vocabulary entry.

### Common misconceptions

- ❌ Attention is the whole transformer → ✅ Attention mixes information across
  positions; the FFN does most of the per-position computation and holds most
  parameters.
- ❌ The FFN looks at other tokens → ✅ The FFN is strictly position-wise; only
  attention moves information between positions.
- ❌ Transformers inherently understand word order → ✅ They are order-agnostic by
  default; positional encoding must be added explicitly.
- ❌ Each layer does a totally different job → ✅ Blocks are architecturally identical;
  they specialize through learned weights, with later layers more abstract.
- ❌ Where the normalization sits is an irrelevant implementation detail → ✅ Pre-norm
  (norm on the sub-layer input) keeps the residual path clean and is what lets very deep
  modern transformers train; post-norm placement makes deep stacks much harder to train.

### Worked example

Trace `The cat sat` predicting the next token. `The`, `cat`, `sat` become three
integer IDs → three embedding vectors, each with positional info. The stack of blocks
runs: in attention, the vector at `sat` attends back to `cat` and `The`, so it now
encodes "a cat is the subject and it sat"; the FFN then refines that representation
per position. After N blocks, the final vector at position `sat` is projected through
the unembedding matrix into ~100k logits. Tokens like ` on`, ` down`, ` quietly` score
high; ` purple` scores low. Softmax → probabilities → sample → next token.

### Check questions

1. Imagine you deleted *all* the self-attention sub-layers from a transformer but kept
   the embeddings and FFNs. What single capability would the model lose, and what would
   it then resemble? — **Answer:** It would lose the ability for any token's
   representation to depend on any *other* token — every position would be processed in
   total isolation. The model would resemble a per-token classifier with no context: it
   could not resolve "it" to an earlier noun, could not condition `Paris` on "capital of
   France," etc. This isolates *why* attention, not the FFN, is the cross-position
   mechanism.
2. A teammate proposes removing residual connections "to simplify the architecture,
   since they're just an addition." Why would a deep transformer become hard to *train*
   without them? — **Answer:** The residual add gives information and, critically,
   gradients a direct path through every block. Without it, gradients must pass through
   dozens of sub-layers' transformations and tend to vanish or explode, so the deep stack
   becomes very hard to optimize. The "just an addition" is load-bearing — it is what
   makes training a deep stack tractable.
3. A model with a 100,000-token vocabulary is asked to predict one next token. Roughly
   how many numbers does the unembedding step produce, what are they *before*
   normalization, and why is that step the same regardless of how confident the model
   is? — **Answer:** It produces ~100,000 numbers — one logit per vocabulary token.
   Before softmax they are raw, unbounded real scores, not probabilities. The unembedding
   always emits the full vector of vocabulary logits every step; "confidence" is a
   property of the *distribution* softmax produces, not of how many logits are computed.
4. A team trains a deep transformer with the normalization placed *after* each
   sub-layer's residual add (post-norm) and finds the deep version barely trains, while
   a shallow version is fine. They have residual connections, so why does depth still
   break training — and what placement is the modern fix? — **Answer:** With post-norm,
   the normalizer sits *on* the main path after the residual add, so the signal is
   re-scaled at every one of the dozens of blocks; the residual is no longer a clean,
   unmodified highway, and gradients/activations are repeatedly distorted as depth grows.
   The modern fix is **pre-norm** placement: normalize each sub-layer's *input* instead,
   leaving the residual path itself untouched from the first block to the last — which is
   precisely what lets very deep stacks train. (Most current models also use RMSNorm
   rather than classic LayerNorm, a cheaper mean-free variant.)

---

## 1.4 — Self-attention — queries/keys/values; why cost is O(n²) in sequence length

### Concept

Self-attention is the mechanism that lets a token's representation depend on the other
tokens in the sequence. It is the heart of the transformer and the source of its main
cost characteristic.

**The Q/K/V mechanism.** For each token, the model derives three vectors from that
token's current representation via learned projection matrices:

- **Query (Q)** — "what information am I looking for?"
- **Key (K)** — "what information do I offer?"
- **Value (V)** — "the actual content I will hand over if selected."

To compute the new representation of a given token, attention takes that token's
**query** and compares it (via dot product) against the **key** of every token in the
sequence. Each dot product is a relevance score: how much should this token attend to
that one? The scores are scaled and passed through a softmax, producing attention
*weights* that sum to 1. The token's output is then the weighted sum of all the
**value** vectors, using those weights. (This is why the mechanism is formally called
**scaled dot-product attention**.)

**Why the scaling step — dividing by √d_k.** The "scaled" in scaled dot-product
attention is not cosmetic. A query–key dot product is a sum over d_k components (d_k is
the per-head key dimension). The more components you sum, the larger the *magnitude* of
a typical dot product tends to be — its variance grows with d_k. If those raw scores are
fed straight into softmax, large-magnitude scores push softmax into a regime where it is
extremely "peaked": almost all the weight collapses onto a single token and the
distribution is nearly one-hot. That is bad for two reasons — the model can barely
*blend* information from multiple tokens, and softmax in that saturated regime has
near-zero gradients, so training stalls. Dividing every score by **√d_k** rescales the
dot products back to a roughly unit-variance range *independent of d_k*, keeping softmax
in a well-behaved, trainable regime. The √d_k factor is exactly the right size because
it cancels the standard-deviation growth of a d_k-term sum. In short: the scaling keeps
attention weights soft and gradients healthy regardless of how wide the heads are.

A useful analogy: a query is a search query, keys are document titles, values are
document contents. You match your query against every title, and you retrieve a blend
of the contents weighted by how well each title matched. This is how the vector for
`it` in "the trophy didn't fit in the suitcase because *it* was too big" can pull in
information from `trophy` — the query at `it` matches the key at `trophy` strongly.

Real models use **multi-head attention**: several independent Q/K/V projections (heads)
run in parallel, each free to learn a different relation (one head tracks syntax,
another tracks coreference, etc.). Their outputs are concatenated.

In an autoregressive LLM, attention is **causal (masked)**: a token may attend only to
itself and earlier tokens, never to future ones — otherwise the model could "cheat" by
peeking at the answer it is meant to predict.

**Why O(n²).** Every token's query is compared against every token's key. For a
sequence of n tokens that is n × n dot products — an n×n attention matrix. Both the
**compute** and the **memory** for that matrix grow with the *square* of the sequence
length. Double the context and attention cost roughly quadruples. This single fact
explains why long context is expensive, why providers price by token, why the KV cache
matters (1.5), and why "lost in the middle" and context-management problems exist
(Topic 4). Production techniques like FlashAttention — an IO-aware, memory-efficient *exact*
attention algorithm that is faster in wall-clock time and reduces memory use from
O(n²) to O(n) [1] — and sparse or sliding-window attention exist largely to soften this
quadratic wall. Note the nuance: FlashAttention does not change the O(n²) *arithmetic*
cost of attention (it still computes every query–key pair); it removes the O(n²)
*memory* footprint and cuts slow memory traffic. Approximate methods (sparse, sliding-
window) do reduce the compute, at some quality cost. Either way, the baseline scaling
of standard attention is O(n²).

### Key terms

- **Self-attention** — mechanism letting each token's representation incorporate
  information from other tokens, weighted by learned relevance.
- **Query (Q) / Key (K) / Value (V)** — three learned vectors per token: what I want,
  what I offer, what I hand over.
- **Attention score / weight** — the (post-softmax) relevance of one token to another;
  weights for a query sum to 1.
- **Scaled dot-product attention / √d_k scaling** — dividing each query–key dot product
  by the square root of the per-head key dimension d_k before softmax, so score
  magnitude stays roughly constant regardless of d_k and softmax stays soft and
  trainable.
- **Multi-head attention** — several parallel attention computations, each learning a
  different relation; outputs concatenated.
- **Causal / masked attention** — restriction that a token attends only to itself and
  earlier tokens, never future ones.
- **O(n²) complexity** — attention compute and memory scale with the square of
  sequence length n.

### Common misconceptions

- ❌ Attention cost grows linearly with context length → ✅ It grows quadratically;
  every token attends to every other token.
- ❌ Q, K, and V are the same vector → ✅ They are three distinct learned projections
  of the token's representation, each with a different role.
- ❌ A token can attend to future tokens → ✅ In a causal LLM, attention is masked;
  only itself and earlier tokens are visible.
- ❌ One attention head does everything → ✅ Multi-head attention runs many heads in
  parallel, each specializing in a different relationship.
- ❌ The √d_k division is an arbitrary normalization constant → ✅ It is sized precisely
  to cancel the variance growth of a d_k-term dot product, keeping softmax out of its
  saturated (near-one-hot, near-zero-gradient) regime so attention stays soft and
  trainable.

### Worked example

Sequence: "The river bank was muddy." To represent `bank`, attention forms a query at
`bank` and dot-products it against the keys of `The`, `river`, `bank`, `was`, `muddy`.
The query matches the key of `river` strongly, so `river` gets a high attention weight
and the value at `bank` becomes a blend dominated by river-related content — the model
disambiguates `bank` toward riverbank rather than financial bank. Now the cost: a
5-token sequence needs 5×5 = 25 query–key comparisons. A 1,000-token prompt needs
1,000,000. A 100,000-token prompt needs 10,000,000,000 — ten billion. That quadratic
blow-up is why long context is genuinely expensive.

### Check questions

1. In the sentence "The lawyer questioned the witness because she was nervous," the
   pronoun `she` should attend strongly to `witness`. In Q/K/V terms, which of the three
   vectors at `she` and which at `witness` do the work, and what is the dot product
   between them measuring? — **Answer:** The **query** at `she` ("what referent am I
   looking for?") is dot-producted against the **key** at `witness` ("what I offer for
   matching"). That dot product is the relevance score; a high score gives `witness` a
   large attention weight, so the **value** at `witness` dominates the blended output
   that updates `she`'s representation. Naming each vector's role on a fresh sentence,
   not reciting definitions, is the point.
2. A team upgrades from an 8k-token prompt to a 64k-token prompt and is shocked the
   attention cost did not merely grow 8×. By what factor did the attention work
   *actually* grow, and why? — **Answer:** Roughly **64×**, not 8×. Attention is O(n²):
   8× more tokens means 8² = 64× more query–key comparisons and a 64× larger attention
   matrix in both compute and memory. The surprise comes from expecting linear scaling
   from a quadratic mechanism.
3. Why must attention be causally masked in a generative LLM, and what specifically
   would go wrong *during training* if it were not? — **Answer:** Masking restricts each
   token to attend only to itself and earlier tokens. Without it, while learning to
   predict token n the model could attend *forward* to token n itself (and beyond) — it
   would simply copy the answer it is supposed to predict, drive the loss to near zero,
   and learn nothing useful for actual left-to-right generation.
4. An engineer removes the √d_k division from scaled dot-product attention "to simplify
   the formula." On a model with wide attention heads, what happens to the attention
   weights and to training, and why does the *width* of the heads make it worse? —
   **Answer:** Without the division, the query–key dot products have a magnitude that
   grows with the head dimension d_k (a sum of more components has larger variance). Fed
   into softmax, those large scores make the distribution extremely peaked — nearly
   one-hot — so the model can barely blend information across tokens, and softmax in that
   saturated regime has near-zero gradients, so training stalls. Wider heads (larger
   d_k) make it worse because the dot-product magnitude grows with d_k; the √d_k factor
   exists precisely to cancel that growth and keep softmax soft and trainable regardless
   of head width.

---

## 1.5 — Autoregressive generation; prefill vs. decode and why the distinction matters

### Concept

LLMs generate text **autoregressively**: one token at a time, each new token appended to
the input and fed back in to produce the next. The model cannot emit a whole sentence
at once — it produces token 1, conditions on it to produce token 2, conditions on
tokens 1–2 to produce token 3, and so on, until it emits an end-of-sequence token or
hits a limit (Topic 3.6). This sequential dependency is fundamental: token n cannot be
computed before token n-1 exists.

A single inference request therefore splits into two phases with very different
performance characteristics.

**Prefill** processes the entire input prompt in **one forward pass**. Because all
prompt tokens are already known, the model can compute their representations *in
parallel* — there is no sequential dependency among input tokens. Prefill is therefore
**compute-bound**: it is doing a lot of matrix multiplication and the GPU's arithmetic
units are the bottleneck. Prefill time is what mostly determines **time to first token
(TTFT)** — the user waits for prefill to finish before the first output token appears.
Prefill also builds the **KV cache**: the key and value vectors for every prompt token,
saved so they need not be recomputed.

**Decode** generates the output tokens one at a time. Each decode step is *one* new
token's forward pass, reusing the KV cache for all prior tokens so attention only
computes the new token's query against cached keys/values. Each step is a small amount
of compute but must **stream the entire model's weights** from GPU memory. Decode is
therefore **memory-bandwidth-bound**: the bottleneck is moving billions of weights from
memory, not arithmetic. Decode speed sets the **inter-token latency / time per output
token (TPOT/ITL)** and the tokens-per-second you observe while text streams.

Why this distinction matters — it explains almost every latency and cost behavior you
will meet:

- **Output tokens are slow and expensive; input tokens are fast and cheap.** A 10,000-
  token prompt is one parallel prefill pass; 1,000 output tokens are 1,000 sequential
  memory-bound steps. This is the mechanical reason output is priced higher than input —
  for example, every current Anthropic model prices output at exactly 5× input [2]. The
  exact multiple is provider- and model-specific; the *direction* (output dearer than
  input) is universal among frontier APIs.
- **TTFT vs. throughput are separate problems.** A long prompt hurts TTFT (more
  prefill); a long answer hurts total time (more decode). Optimizing one does not fix
  the other.
- **Prompt caching (Topic 6) works because of prefill.** If a prefix is unchanged, its
  KV cache can be reused, skipping that part of prefill entirely — a large TTFT win.
- **Continuous batching (Topic 12) works because decode is memory-bound.** Since decode
  underuses arithmetic units, a server can interleave many requests' decode steps and
  amortize the weight load across all of them.

**Why the KV cache fits in memory at all — GQA and MQA.** The KV cache is not free: it
stores a key and a value vector *per token, per layer, per attention head*, so its size
grows with the sequence length and with the number of heads. For long contexts and large
batches, a naively-sized KV cache would simply not fit in GPU memory — and since decode
is memory-bandwidth-bound, a bigger cache also means more data to stream every step.
The standard fix is to *shrink the cache by sharing keys and values across heads*.
Recall from 1.4 that multi-head attention runs many heads in parallel. **Multi-Query
Attention (MQA)** keeps a separate *query* projection per head but uses a *single*
shared key/value pair for all heads — collapsing the K/V part of the cache by the head
count. **Grouped-Query Attention (GQA)** is the middle ground used by most current
frontier models: heads are split into a handful of *groups*, and each group shares one
K/V pair. GQA recovers most of the quality of full multi-head attention while cutting
the KV cache several-fold versus storing K/V per head. This is the architectural reason
million-token context windows are servable at all: without GQA/MQA the KV cache for a
long context would be prohibitively large. It is load-bearing for everything about
long-context cost and the KV cache you will meet in Topics 4 and 6.

### Key terms

- **Autoregressive generation** — generating one token at a time, each conditioned on
  all tokens produced so far.
- **Prefill** — the initial forward pass over the whole prompt; parallel across input
  tokens; compute-bound; builds the KV cache.
- **Decode** — the per-token generation loop; sequential; memory-bandwidth-bound.
- **KV cache** — stored key/value vectors for processed tokens, reused so attention is
  not recomputed from scratch each step; its size grows with sequence length, layers,
  and heads.
- **GQA / MQA (grouped-query / multi-query attention)** — attention variants that share
  one key/value pair across a group of heads (GQA) or across all heads (MQA), shrinking
  the KV cache so long contexts and large batches fit in GPU memory.
- **TTFT (time to first token)** — latency until the first output token; dominated by
  prefill.
- **TPOT / ITL (time per output token / inter-token latency)** — time between
  successive output tokens; set by decode.
- **Compute-bound / memory-bandwidth-bound** — whether the bottleneck is arithmetic
  throughput or moving data from memory.

### Common misconceptions

- ❌ The model writes a whole response at once → ✅ It generates strictly one token at a
  time, each conditioned on the prior ones.
- ❌ Prefill and decode are equally fast per token → ✅ Prefill is parallel and
  compute-bound; decode is sequential and memory-bound. Output tokens are much slower.
- ❌ A long prompt and a long answer hurt latency the same way → ✅ A long prompt
  inflates prefill/TTFT; a long answer inflates decode/total time.
- ❌ The KV cache is an optional optimization with little effect → ✅ Without it every
  decode step would re-process the whole sequence, making generation roughly O(n²) per
  step instead of O(n).
- ❌ The KV cache stores one key/value pair per token, full stop → ✅ It stores K/V per
  token *per layer per head* — which is why modern models use GQA/MQA to share K/V
  across heads and keep the cache small enough to serve long contexts.

### Worked example

Request: 8,000-token prompt, 500-token answer. Prefill runs once over all 8,000 tokens
in parallel — say 400 ms — and the user sees nothing until it finishes; that 400 ms is
roughly the TTFT. Then decode runs 500 sequential steps; at, say, 20 ms/token that is
10 s of streaming. Total ≈ 10.4 s. Notice the asymmetry: 8,000 input tokens took 400 ms
(~0.05 ms/token); 500 output tokens took 10,000 ms (20 ms/token). Per token, producing
output was dramatically slower than ingesting input — decode is sequential and
memory-bandwidth-bound, while prefill is parallel and compute-bound. That per-token gap
is the same reason providers price output tokens higher than input. Cut the prompt to
4,000 tokens and TTFT roughly halves
but the 10 s of decode is unchanged. Cut the answer to 100 tokens and decode drops to
~2 s while TTFT is unchanged. Two separate levers.

### Check questions

1. A new GPU has *much* higher arithmetic throughput but the *same* memory bandwidth as
   the old one. Which improves more on this hardware — prefill or decode — and why? —
   **Answer:** Prefill improves much more. Prefill is compute-bound, so more arithmetic
   throughput directly helps it. Decode is memory-bandwidth-bound — each step's
   bottleneck is streaming the weights from memory — so leaving bandwidth unchanged
   leaves decode speed essentially unchanged. This tests *which* resource bounds each
   phase, not just their names.
2. Two requests take the same total wall-clock time: request A is a 50,000-token prompt
   with a 50-token answer; request B is a 200-token prompt with a 2,000-token answer.
   Which one is doing more sequential work, and which one's latency would a faster
   prefill kernel help? — **Answer:** Request B is doing far more *sequential* work — its
   2,000 output tokens are 2,000 sequential decode steps, versus A's 50. A faster prefill
   kernel helps **request A**, whose time is dominated by prefilling 50,000 input tokens;
   it barely touches B, which is decode-dominated.
3. Suppose decode were somehow made fully parallel like prefill. Which of these would
   change — TTFT, total response time, or output-vs-input pricing — and which would not?
   — **Answer:** Total response time would drop dramatically and the output/input price
   asymmetry would largely collapse (output would no longer be a string of expensive
   sequential steps). TTFT would be roughly *unchanged* — it is already set by prefill,
   which was already parallel. This forces reasoning about what each phase actually
   causes.
4. A team wants to serve a much longer context window but finds the KV cache no longer
   fits in GPU memory. A teammate suggests "just remove some attention heads." Explain
   the better-known fix and why it preserves quality far more than dropping heads. —
   **Answer:** The standard fix is **GQA** (or, more aggressively, MQA): keep all the
   query heads but have groups of heads *share* a single key/value pair, so the K/V part
   of the cache shrinks by the group/head count without removing any query head.
   Dropping heads outright removes representational capacity — each head can learn a
   distinct relation (1.4) — whereas GQA keeps every query head's distinct view and only
   shares the K/V projections, which empirically costs very little quality. GQA/MQA are
   the reason long-context windows are servable at all.

---

## 1.6 — Logits → softmax → probability distribution over the vocabulary

### Concept

At each generation step the model must turn its internal computation into an actual
next-token choice. That happens in three stages: **logits → softmax → distribution →
sample**.

**Logits.** After the final transformer block, the current position's vector is
projected through the unembedding matrix to produce one raw score for *every* token in
the vocabulary. Frontier-model vocabularies are commonly in the ~100,000–200,000 range
(e.g., Llama 3's tokenizer has a 128,256-token vocabulary [3]; exact sizes vary by
model), so the model produces on the order of 100,000–200,000 logits per step. A logit is an unbounded real number — it can be negative, zero, or large
positive. On its own a logit is not a probability; it is just a relative score. Higher
logit = the model considers that token more likely.

**Softmax.** To convert logits into a usable probability distribution, the model applies
the **softmax** function. Softmax (1) exponentiates every logit, which makes all values
positive and amplifies differences — a token with a clearly higher logit gets a
disproportionately higher share — and (2) divides each by the sum of all the
exponentials, so the results are non-negative and sum to exactly 1. The output is a
genuine probability distribution over the entire vocabulary: each token gets a
probability in [0, 1], and the probabilities total 1.

**Distribution → token.** A token is then selected from that distribution by a sampling
or decoding strategy (the entire subject of Topic 3). Greedy decoding takes the
highest-probability token; temperature/top-p/top-k sampling draw stochastically. The key
point for Topic 1: the model's native output is a *distribution*, not a token. The
choice of which token to emit is a separate, configurable step layered on top.

Two facts to carry forward:

- **Temperature acts on the logits before softmax.** Dividing every logit by a
  temperature T sharpens the distribution (T < 1) or flattens it (T > 1) — that is the
  entire mechanism of the temperature knob (Topic 3.1). It is meaningless to talk about
  temperature without the logit→softmax picture.
- **The log of a token's softmax probability is its logprob** (Topic 3.5). Because
  probabilities are tiny, providers report **log**-probabilities, which are easier to
  work with and to sum across a sequence (summing logprobs = multiplying probabilities).

So: the model emits logits; softmax makes them a distribution; sampling picks a token;
repeat (this is the loop from 1.5). Once you see generation as "produce a distribution,
then draw from it," every sampling parameter in Topic 3 becomes obvious.

### Key terms

- **Logit** — a raw, unbounded real-valued score the model produces per vocabulary
  token before normalization.
- **Softmax** — function that exponentiates logits and normalizes them into a
  probability distribution that sums to 1.
- **Probability distribution (over the vocabulary)** — the softmax output: a
  probability for every possible next token.
- **Logprob** — the natural log of a token's probability; what APIs report because it
  is numerically convenient and additive across a sequence.
- **Temperature** — a scalar dividing the logits before softmax, controlling how
  sharp/flat the distribution is (Topic 3).

### Common misconceptions

- ❌ Logits are probabilities → ✅ Logits are unbounded raw scores; softmax is required
  to turn them into a probability distribution.
- ❌ Softmax just picks the largest logit → ✅ Softmax produces a *distribution* over
  all tokens; the *picking* is a separate sampling/decoding step.
- ❌ The model outputs a single token → ✅ The model outputs a full distribution; a
  sampling strategy then selects one token.
- ❌ Temperature changes the model's weights or computation → ✅ Temperature only
  rescales logits before softmax; it changes nothing inside the network.

### Worked example

Suppose at one step the model produces these logits for four candidate tokens (the rest
have very low logits): `mat` 4.0, `rug` 3.0, `couch` 1.0, `idea` -2.0. Softmax
exponentiates: e^4 ≈ 54.6, e^3 ≈ 20.1, e^1 ≈ 2.7, e^-2 ≈ 0.14, summing to ≈ 77.5.
Divide each: `mat` ≈ 0.70, `rug` ≈ 0.26, `couch` ≈ 0.035, `idea` ≈ 0.002. Now you have
a real distribution. Greedy decoding emits `mat`. Sampling at temperature 1 emits `mat`
~70% of the time, `rug` ~26%, `couch` rarely, `idea` almost never. Raise temperature
and the gap narrows (`rug` becomes more competitive); lower it and `mat` approaches
certainty — all by rescaling those logits before softmax.

### Check questions

1. Someone reads off a model's raw logit for one token as `-3.1` and concludes "the
   model thinks that token is very unlikely — about a 3% chance." Identify the two
   things wrong with that statement. — **Answer:** (a) A logit is *not* a probability —
   it is an unbounded raw score, so `-3.1` does not denote 3% (or any percentage) on its
   own. (b) A single logit means nothing in isolation: a token's probability depends on
   *all* the other logits, because softmax normalizes across the whole vocabulary. A
   logit of -3.1 could be the top token if every other logit is even lower.
2. If you added the same constant (say +10) to *every* logit before softmax, would the
   resulting probability distribution change? What does your answer reveal about what
   softmax actually responds to? — **Answer:** No — the distribution is unchanged.
   Softmax exponentiates and then normalizes, so a shared additive constant cancels in
   the ratio. This reveals that softmax responds only to the *differences* (gaps) between
   logits, not their absolute values — which is exactly why temperature, which rescales
   those gaps, is the meaningful knob (Topic 3).
3. Why do APIs return logprobs instead of raw probabilities, and what concretely becomes
   easy *because* logprobs are additive? — **Answer:** Probabilities of long sequences
   are vanishingly tiny and numerically unstable to multiply; logs are stable. Because
   logprobs are additive, the log-likelihood of a whole generated sequence is just the
   *sum* of its per-token logprobs (summing logs = multiplying probabilities) — so
   scoring or comparing entire completions becomes a simple addition.

---

## 1.7 — Parameters vs. context; statelessness of the API

### Concept

An LLM-based system has exactly **two** kinds of "memory," and keeping them straight is
foundational for everything in Topics 4–16.

**Parameters (weights)** are the billions of numbers fixed by training (1.2). They
encode everything the model "knows" in a general sense — language, world facts up to
the training cutoff, reasoning patterns, style. Parameters are *identical for every
request* and *do not change during use*. They are long-term, static, shared knowledge.

**Context** is the token sequence you feed into a specific request — the system prompt,
the conversation history, retrieved documents, tool results, the current user message.
Context is the model's *only* working memory: it is per-request, transient, and the
sole place request-specific information can live. Anything the model should "know" for
*this* request that is not baked into its weights must be in the context window.

The crucial consequence is that **the API is stateless**. The model server keeps no
memory of your previous calls. Each request is handled as a fresh, independent forward
pass over whatever tokens you sent *this time*. The server does not "remember" turn 1
when you send turn 2.

Therefore, to hold a multi-turn conversation, **you (the application) must resend the
entire conversation history on every single call.** Turn 5's request contains the
system prompt plus turns 1–4 plus the new turn-5 message. The model re-reads all of it
from scratch every time. This is not a quirk — it follows directly from frozen weights
(no learning) plus statelessness (no server-side memory). It is also why:

- **Cost grows with conversation length.** Every turn re-bills all prior tokens as
  input (Topic 4.6). A long chat gets progressively more expensive per turn.
- **Conversations are bounded by the context window** (Topic 4.3). You cannot resend
  more history than fits.
- **Context engineering exists** (Topic 4.5). Because context is finite and re-sent
  every turn, deciding *what* to include — and what to drop, summarize, or retrieve — is
  a core engineering discipline.
- **Prompt caching is valuable** (Topic 6). Since you re-send a large stable prefix
  every turn, caching its KV state avoids recomputation.

A clean way to hold it: **parameters are what the model knows; context is what the
model is currently looking at.** Parameters are permanent and shared; context is
temporary and per-request. The model itself is stateless — all conversational state
lives in your application, which reconstructs the context window on every call.

### Key terms

- **Parameters / weights** — the model's permanent, trained knowledge; identical for
  every request; unchanged during inference.
- **Context / context window** — the per-request token sequence the model reads; its
  only working memory; transient.
- **Stateless API** — the server retains nothing between calls; each request is
  independent.
- **Conversation history** — the accumulated prior turns the application must resend on
  every call to maintain continuity.
- **Knowledge cutoff** — the date after which the training data ends; later facts are
  absent from parameters and must be supplied via context.

### Common misconceptions

- ❌ The API server remembers my previous messages → ✅ It is stateless; your
  application resends the full history every turn.
- ❌ Sending a `conversation_id` means the server stores my history → ✅ Any such ID is
  application/SDK convenience; the model still receives the whole history as tokens each
  call.
- ❌ Information in the context window is "learned" by the model → ✅ Context is
  transient working memory; it influences only the current request and changes no
  weight.
- ❌ A longer chat costs the same per turn → ✅ Each turn re-bills all prior tokens as
  input, so per-turn cost rises as history grows.

### Worked example

A support bot, turn 3. The user's third message alone is 20 tokens. But the request the
application sends to the API contains: system prompt (500 tokens) + user turn 1 (40) +
assistant turn 1 (120) + user turn 2 (30) + assistant turn 2 (200) + user turn 3 (20) =
~910 input tokens. The model re-reads all 910 from scratch — it has no memory of turns 1
and 2 otherwise. Turn 10 of the same chat might resend 5,000+ input tokens for a 20-
token user message. This is the mechanical reason long agent conversations get
expensive and why summarization/eviction strategies (Topic 4.7) and prompt caching
(Topic 6) exist.

### Check questions

1. Classify each of the following as living in *parameters* or *context*: (a) the
   grammar of English; (b) the user's name, given two messages ago; (c) a company's
   internal refund policy pasted into the system prompt; (d) the fact that Berlin is in
   Germany. Then state the rule you used. — **Answer:** (a) parameters, (d) parameters —
   general/world knowledge learned in training; (b) context, (c) context — request-
   specific text supplied this call. Rule: if it was baked in by training and is
   identical for every request, it is parameters; if it was supplied in *this* request's
   token sequence, it is context. (Note (c): even a "permanent" company policy lives in
   context unless the model was fine-tuned on it.)
2. A teammate, to save bandwidth, builds a client that sends only the newest user
   message after turn 1, assuming "the model already saw the earlier turns." Describe the
   bug the user will observe and explain it from statelessness. — **Answer:** From turn 2
   on, the model behaves as if the conversation never happened — it cannot reference
   earlier turns, forgets the user's name, loses task context. Because the API is
   stateless, the server kept nothing; the only way the model "sees" prior turns is for
   them to be physically in the array sent *this* call. Sending just the new message
   throws all that context away.
3. A frontier model's knowledge cutoff is mid-2025. You need it to answer using a fact
   that became true in 2026 *and* a niche internal metric your company never published.
   For each, say whether parameters could ever supply it and how you would get it to the
   model. — **Answer:** Neither can come from parameters: the 2026 fact post-dates the
   cutoff, and the internal metric was never in any training corpus. Both must enter via
   the **context window** — supplied directly in the prompt, fetched by a tool call, or
   retrieved (RAG). The 2026 fact *could* in principle enter parameters via a future
   retraining; the private metric realistically never would.

---

## 1.8 — Inference-time quantization and speculative decoding

### Concept

Sub-chapter 1.5 established the two phases of an inference request — compute-bound
**prefill** and memory-bandwidth-bound **decode** — and noted that decode is slow
because every step must stream the whole model's weights from memory. Two widely-used
techniques attack that cost directly. You will not implement them as an API consumer,
but they explain real differences in price, latency, and quality you will observe across
models and providers.

**Inference-time quantization.** A model's weights are numbers, and the *numeric
precision* used to store and compute with them is a choice. Training is typically done
in 16-bit floating point — **FP16** or **BF16** (bfloat16, a 16-bit format with a wider
exponent range that is the common training/serving default). **Quantization** is storing
(and often computing with) the weights at *lower* precision — 8-bit (**INT8** or
**FP8**) or even 4-bit. The point connects straight to 1.5: since decode is
**memory-bandwidth-bound**, halving the bits per weight roughly halves the data streamed
per decode step — so quantization makes decode **faster** and lets a model **fit in less
GPU memory** (or fit a longer context / bigger batch). It also lowers serving cost,
which is part of why cheaper model tiers exist.

The trade-off is precision loss, but the modern picture is much better than older
intuition suggests. **FP8** has become a standard serving precision in 2026 and is
near-lossless for inference. Modern **8-bit** quantization is generally near-lossless.
Even modern **4-bit** schemes (AWQ/GPTQ-class methods) are far closer to lossless than
the "4-bit means noticeable degradation" framing of a couple of years ago — quality loss
is small and often acceptable. The key engineering facts: (1) quantization is an
*inference-time* serving choice, distinct from the model's trained weights; (2) two
deployments of "the same model" at different precisions can differ measurably in latency,
cost, *and* output; (3) a "quantized" or "turbo" model tier is usually trading a little
quality for speed and price.

**Speculative decoding.** Decode's core inefficiency is that it generates **one token
per expensive forward pass** of the full model, and that pass is memory-bound — the
arithmetic units sit mostly idle while weights stream in. **Speculative decoding**
exploits this idle compute. A small, fast **draft model** cheaply proposes the next *k*
tokens; then the large **target model** verifies all *k* in a **single forward pass**
(it can score multiple proposed tokens in parallel, the same way prefill processes many
tokens at once). Every proposed token the target model agrees with is accepted; at the
first disagreement, the target model's own token is used and drafting restarts. The
result is **mathematically identical to normal decoding from the target model** — the
target model has the final say on every token, so output quality is unchanged — but when
the cheap draft guesses well, several tokens are produced per expensive target pass
instead of one. That is a real **latency / throughput** win, and it is invisible to the
API caller except as faster token streaming. The catch: the speed-up depends on the
draft model's *acceptance rate* — on easy, predictable text it guesses well and the
win is large; on hard text it guesses poorly and the gain shrinks.

Both techniques share a theme from 1.5: **decode is the expensive, memory-bound phase,
so the high-leverage optimizations target it** — quantization by shrinking the bytes
streamed per token, speculative decoding by producing more tokens per streamed pass.

### Key terms

- **Numeric precision (FP16 / BF16 / FP8 / INT8 / INT4)** — the number of bits used to
  store and compute with model weights; BF16 is a common training/serving default, FP8
  a standard 2026 serving precision.
- **Quantization** — serving a model at lower numeric precision than it was trained in,
  to cut memory and memory-bandwidth cost; modern 8-bit is near-lossless and modern
  4-bit is close.
- **Draft (small) model / target (large) model** — in speculative decoding, the cheap
  model that proposes tokens and the full model that verifies them.
- **Speculative decoding** — generating with a cheap draft model and verifying its
  proposed tokens in one parallel pass of the target model; speeds up decode with
  *no* change to the target model's output distribution.
- **Acceptance rate** — the fraction of draft-proposed tokens the target model accepts;
  determines how large the speculative-decoding speed-up is.

### Common misconceptions

- ❌ Quantization changes which model you are using → ✅ It is a serving-time precision
  choice for the *same* trained weights; it changes cost/latency and can slightly change
  output, but it is not a different model.
- ❌ 4-bit quantization always badly degrades quality → ✅ That reflects older methods;
  modern 8-bit is near-lossless and modern 4-bit (AWQ/GPTQ-class) is close — the loss is
  usually small.
- ❌ Speculative decoding trades quality for speed → ✅ The target model verifies every
  token, so the output distribution is unchanged; only speed changes.
- ❌ Speculative decoding always gives a fixed speed-up → ✅ The gain depends on the
  draft model's acceptance rate, which is higher on predictable text and lower on hard
  text.

### Worked example

A 70-billion-parameter model served in BF16 needs ~140 GB just for weights, and every
decode step streams all 140 GB from memory. Re-serve it in FP8: weights now take ~70 GB,
each decode step streams half the bytes, so decode is markedly faster and the model fits
on smaller/fewer GPUs — at near-zero quality loss. Now add speculative decoding: a tiny
draft model proposes the next 4 tokens of a fairly predictable sentence; the 70B target
model verifies all 4 in one forward pass and accepts 3 of them before its first
disagreement. Those 3 tokens cost *one* expensive target pass instead of three — roughly
a 3× decode speed-up on that span. On a hard, unpredictable passage the draft model
might get only 1 of 4 accepted, and the speed-up shrinks toward 1×. The user just sees
faster streaming; the produced text is exactly what the target model would have emitted
alone.

### Check questions

1. A team re-serves their model from BF16 to FP8 and observes faster token streaming and
   a lower bill, with negligible quality change. Connect this to the prefill/decode story
   from 1.5 — *why* does cutting the precision speed up generation? — **Answer:** Decode
   is **memory-bandwidth-bound**: each step's bottleneck is streaming the model's weights
   from GPU memory, not arithmetic. FP8 stores each weight in half the bits of BF16, so
   each decode step streams roughly half the bytes — directly faster, since the binding
   resource is memory bandwidth. It also fits in less GPU memory (lower serving cost).
   Modern FP8 is near-lossless, so quality barely moves. The win is mechanical: fewer
   bytes per token through the bottleneck.
2. A teammate refuses to enable speculative decoding because "approximating the output
   with a small model will hurt quality." Identify the misconception and explain why
   speculative decoding does *not* change the output. — **Answer:** The small draft
   model only *proposes* candidate tokens; the large **target** model then **verifies**
   every one of them in a parallel pass and has the final say — any token the target
   model would not have produced is rejected and replaced with the target model's own.
   So the emitted sequence is exactly what the target model would have generated alone:
   the output distribution is unchanged. Speculative decoding is a pure *latency*
   optimization, not an approximation — the draft model affects only *speed* (via its
   acceptance rate), never *which* tokens come out.
3. Speculative decoding gives a big speed-up on boilerplate text but much less on dense,
   surprising text. Explain why, in terms of the draft model's acceptance rate. —
   **Answer:** The speed-up comes from the target model accepting several draft-proposed
   tokens per verification pass. On predictable, boilerplate text the cheap draft model
   guesses the continuation well, so its acceptance rate is high and many tokens are
   confirmed per expensive target pass. On dense or surprising text the draft model
   guesses poorly, the target model rejects most proposals, and you fall back toward one
   token per target pass — the normal decode cost. The gain scales with acceptance rate,
   which tracks how predictable the text is.

---

## Topic 01 — Exam Question Bank

A pool the tutor draws from for the gated exam. Mixed-format, scored out of 100; 85 to
pass. Free-form answers are graded on reasoning quality with partial credit.

### True / False

1. A model that scores 95% on a graduate physics exam must contain an explicit physics
   reasoning module separate from next-token prediction. — **Answer:** False. There is
   no separate module; the physics competence is an emergent consequence of doing
   next-token prediction well over a corpus that includes physics — it is still
   pattern-completion underneath.
2. If you fine-tune a model on Monday, then run 1,000,000 inference requests on Tuesday,
   the weights at the end of Tuesday differ from the weights at the start. — **Answer:**
   False. Inference is a forward pass only and never updates weights; the Tuesday
   requests leave the weights exactly as the Monday fine-tune left them.
3. Removing the FFN sub-layers but keeping self-attention would still let tokens
   exchange information with each other. — **Answer:** True. Cross-position information
   flow is done by self-attention; the FFN is strictly position-wise, so its removal
   does not affect whether tokens can read each other (it would, however, cripple
   per-position processing).
4. Doubling a prompt's length roughly doubles the self-attention compute for that
   prompt. — **Answer:** False. Attention is O(n²), so doubling the length roughly
   *quadruples* the attention compute and memory.
5. In a single inference request, the input prompt and the generated output are
   processed by the same kind of operation with the same performance profile. —
   **Answer:** False. The prompt is handled by parallel, compute-bound prefill; the
   output is generated by sequential, memory-bandwidth-bound decode — different
   operations with very different cost/latency profiles.
6. Because a logit can be a large positive number, a sufficiently large logit by itself
   guarantees that token will be chosen. — **Answer:** False. A logit is meaningful only
   relative to the other logits; whether a token is likely depends on the whole
   post-softmax distribution, and which token is *chosen* also depends on the decoding
   strategy.
7. Passing a `conversation_id` to an API guarantees the model server is storing your
   history for you. — **Answer:** False. The model API is stateless; any such ID is an
   application/SDK convenience, and the full history must still be sent as tokens each
   call.
8. On hardware whose memory bandwidth is the binding constraint, generating output
   tokens is the part of inference most affected. — **Answer:** True. Decode (output
   generation) is memory-bandwidth-bound — each step streams the whole model's weights —
   so it is what suffers most when bandwidth is the limit; prefill is compute-bound.
9. Grouped-query attention (GQA) shrinks the KV cache by removing query heads from the
   model. — **Answer:** False. GQA keeps every query head; it shares a single key/value
   pair across a *group* of heads, shrinking only the K/V part of the cache — no query
   head is removed.
10. Placing the normalization on a sub-layer's input (pre-norm) rather than after the
    residual add (post-norm) is the placement that makes very deep transformers
    trainable. — **Answer:** True. Pre-norm keeps the residual path a clean, unmodified
    highway through the whole stack; post-norm rescales the main path at every block,
    which makes deep stacks much harder to train.
11. Speculative decoding speeds up generation by letting a small draft model approximate
    the output, which slightly degrades quality. — **Answer:** False. The large target
    model verifies every draft-proposed token, so the emitted sequence is exactly what
    the target model would have produced alone — output quality is unchanged; only speed
    changes.
12. Serving a model in FP8 instead of BF16 speeds up decode mainly because decode is
    memory-bandwidth-bound and FP8 halves the bytes streamed per weight. — **Answer:**
    True. Decode's bottleneck is streaming weights from memory; lower precision means
    fewer bytes per token through that bottleneck. Modern FP8 is near-lossless.

### Multiple Choice

1. A base LLM (no tools, no retrieval) gives a correct, up-to-date answer about a
   well-known historical event. Mechanically, the *most accurate* account is that the
   model:
   A) Looked the event up in an internal knowledge base keyed by topic
   B) Computed a next-token distribution in which the correct tokens scored highly
      because that pattern was well-represented in training
   C) Ran a symbolic inference over stored facts
   D) Queried a cached copy of the web
   — **Answer:** B. There is no lookup or symbolic step in a base model; correctness here
   is a high-probability next-token prediction backed by strong training-data support.
2. A model resolves the pronoun "it" in a long sentence to the correct earlier noun.
   The component that makes this cross-token link possible is:
   A) The embedding lookup, which encodes each token's meaning
   B) The FFN sub-layer, which refines each token's representation
   C) Self-attention, the only sub-layer that moves information between positions
   D) Layer normalization, which keeps activations stable
   — **Answer:** C. Embedding and FFN are position-wise and layer norm only rescales;
   only self-attention lets one token's representation incorporate another's.
3. A request's prompt is shortened from 20,000 to 5,000 tokens, but the requested answer
   length is unchanged. The clearest expected effect is:
   A) Total response time drops by roughly the same proportion as the prompt shrank
   B) Time to first token drops markedly while per-output-token speed is ~unchanged
   C) Per-output-token speed improves but TTFT is unchanged
   D) Neither TTFT nor total time changes
   — **Answer:** B. Prefill (hence TTFT) scales with prompt length, so TTFT drops; decode
   speed is set by the memory-bound per-token loop and is unaffected by prompt length.
4. Which change would most directly reduce *time to first token* for a request?
   A) Asking the model for a shorter answer
   B) Reusing a cached KV prefix so most of prefill is skipped
   C) Lowering the sampling temperature
   D) Raising `max_tokens`
   — **Answer:** B. TTFT is dominated by prefill; skipping prefill via a cache hit cuts
   it directly. A shorter answer only affects decode/total time; temperature and
   `max_tokens` do not reduce prefill.
5. Two candidate next tokens have logits 8.0 and 7.0. After softmax, the most accurate
   statement is:
   A) The first token has probability 8/15
   B) The first token is somewhat more probable, but the exact probabilities depend on
      every other token's logit too
   C) The first token has exactly 8.0/(8.0+7.0) probability
   D) Logits of 8.0 and 7.0 are already probabilities
   — **Answer:** B. Softmax normalizes over the *entire* vocabulary, so two logits alone
   do not fix the probabilities; the gap makes the first more probable but the values
   depend on all logits.
6. A teammate wants to cut bandwidth by sending only each new user message after turn 1.
   The result will be:
   A) Identical behavior, since the server retains the earlier turns
   B) Identical behavior, since the model memorized the earlier turns in its weights
   C) The model loses all earlier context from turn 2 onward, because the API is
      stateless and keeps nothing
   D) An API error on every request after turn 1
   — **Answer:** C. The API is stateless and weights do not learn from a chat; dropping
   prior turns simply removes them from the model's view — usually not an error, just
   broken continuity.
7. Which statement correctly distinguishes training from inference?
   A) Both run backpropagation, but training does it more often
   B) Training adjusts weights via backpropagation; inference is a forward pass that
      changes no weights
   C) Inference adjusts weights slowly so the model improves with use
   D) They are the same operation run on different hardware
   — **Answer:** B. Only training/fine-tuning runs backprop and updates weights;
   inference is purely a forward pass.
8. A provider prices output tokens well above input tokens. The best *mechanical*
   justification is:
   A) Output tokens carry more information than input tokens
   B) Output is produced one token at a time in a sequential, memory-bandwidth-bound
      loop, while input is consumed in one parallel, compute-bound prefill pass
   C) Output tokens are drawn from a larger vocabulary
   D) The pricing is arbitrary and unrelated to how inference works
   — **Answer:** B. The per-token cost asymmetry is a direct consequence of the
   prefill/decode split, not of information content or vocabulary.
9. A model must serve a much longer context window, but the KV cache no longer fits in
   GPU memory. The architectural change most directly aimed at this problem is:
   A) Increasing the vocabulary size
   B) Grouped-query attention, which shares one key/value pair across a group of heads
      so the KV cache shrinks while every query head is retained
   C) Raising the sampling temperature
   D) Removing the residual connections
   — **Answer:** B. GQA (and the more aggressive MQA) shrink the K/V part of the cache by
   sharing key/value projections across heads — the reason long contexts are servable.
10. Scaled dot-product attention divides each query–key dot product by √d_k before
    softmax. The purpose is:
    A) To make the dot product faster to compute
    B) To keep the score magnitude roughly constant regardless of head dimension, so
       softmax does not saturate into a near-one-hot, near-zero-gradient regime
    C) To convert the logits into probabilities
    D) To mask out future tokens
    — **Answer:** B. The √d_k factor cancels the variance growth of a d_k-term dot
    product, keeping attention weights soft and gradients healthy.
11. A provider offers a cheaper, faster "turbo" tier of the same model. The most likely
    mechanism behind it is:
    A) The turbo tier is a completely different, smaller model
    B) The turbo tier serves the same weights at lower numeric precision (quantization),
       trading a little quality for speed and lower memory/bandwidth cost
    C) The turbo tier removes the system prompt
    D) The turbo tier disables the KV cache
    — **Answer:** B. Quantization is an inference-time precision choice for the same
    weights; lower precision cuts memory-bandwidth cost (faster decode) and price.
12. Speculative decoding produces a large speed-up on predictable text but a small one
    on dense, surprising text. The reason is:
    A) The target model is slower on surprising text
    B) The draft model's acceptance rate is high on predictable text and low on
       surprising text, and the speed-up scales with acceptance rate
    C) Surprising text uses a larger vocabulary
    D) Speculative decoding is disabled on hard text
    — **Answer:** B. The win comes from the target model accepting several draft-proposed
    tokens per pass; that acceptance rate tracks how predictable the text is.

### Short Answer

1. The objective "predict the next token" sounds trivial, yet training on it yields
   translation, coding, and reasoning. Explain the causal link — *why* does optimizing a
   trivial objective force the model to acquire non-trivial abilities? Use one concrete
   ability as an example. — **Model answer:** The objective is trivial to *state* but
   not to *solve well*: to assign high probability to the genuinely-correct next token
   across trillions of words, the model must internalize the structure that determines
   what comes next. E.g., to predict the closing token of a syntactically valid sentence
   it must encode grammar; to predict the sum after "2 + 2 =" it must encode arithmetic
   patterns. The capabilities are *prerequisites* for low loss, so the optimizer is
   forced to acquire them — they are a means to the prediction end, not separate goals.
2. A fact is wrong in the model's answer. Explain how the *fix* differs depending on
   whether the error is "in the parameters" versus "the relevant info was missing from
   context," and how you would tell which case you are in. — **Model answer:** If the
   model's parameters encode a stale or wrong fact, no prompt change reliably fixes it
   the right way is to supply the correct fact in context (RAG / prompt) so it overrides,
   or to fine-tune. If instead the needed info simply was not in this request's context,
   the fix is purely an application change — include the document, call a tool, retrieve
   the snippet — no model change at all. You distinguish them by testing: give the model
   the correct fact explicitly in context; if it then answers correctly, it was a
   context-coverage problem; if it still errs or contradicts, the parameters are
   asserting something and you have a stronger conflict to manage.
3. Without the KV cache, what would the per-step cost of decode look like as the
   sequence grows, and why? Explain what the cache stores and how it changes that. —
   **Model answer:** Without it, every decode step would re-process the *entire*
   sequence from scratch — so step n costs O(n) attention work and the whole generation
   becomes roughly O(n²). The KV cache stores the key and value vectors for every
   already-processed token, so each decode step only computes the *new* token's query
   against the cached K/V — making each step roughly constant in prior length and the
   generation roughly linear overall.
4. A student claims "softmax just picks the biggest logit." Correct them precisely:
   what does softmax actually output, what does the *picking* of a token, and how does
   that division of labor matter for Topic 3? — **Model answer:** Softmax does not pick
   anything — it transforms the whole logit vector into a probability *distribution*
   (exponentiate, then normalize so values are in [0,1] and sum to 1). A separate
   *decoding/sampling* step then selects a token from that distribution. The division
   matters because Topic 3's knobs (temperature, top-p, greedy vs. sampling) all act on
   that selection step or on the logits before softmax — the model's distribution and
   the choice of token are deliberately decoupled.
5. "Stateless API" and "frozen weights" are two different properties, yet both are
   needed to explain why a model cannot recall a previous separate conversation.
   Explain each property's distinct contribution. — **Model answer:** *Frozen weights*
   means nothing said in any conversation alters the model — so the fact was never
   stored in the network. *Statelessness* means the server keeps no per-user memory
   between calls — so the fact is not stored server-side either. Together they leave
   exactly one place request-specific info can live: the context window you send this
   call. A new conversation is a fresh empty context, so the fact is genuinely gone.
6. Attention and the FFN are both inside every block, but swapping their roles would
   break the model. For each, state what would be lost if that sub-layer alone were
   removed. — **Model answer:** Remove **attention**: tokens can no longer read each
   other — all cross-position information flow (coreference, conditioning a word on
   earlier context) is gone; the model becomes a context-blind per-token function.
   Remove the **FFN**: each position loses the bulk of its per-token processing capacity
   and much of the stored factual knowledge (the FFN holds most parameters); attention
   could still move information around but there is little machinery to transform it.
7. Frozen weights are often presented as a constraint, but list two concrete *benefits*
   for someone operating the model in production, and name the alternative (continuous
   weight updates from user input) and one specific risk it would introduce. — **Model
   answer:** Benefits: (a) reproducibility/testability — a fixed function can be
   evaluated and versioned and will not silently drift; (b) cost/latency — a forward
   pass is far cheaper than backprop, keeping serving fast. The alternative, updating
   weights live from user input, would introduce (among others) a poisoning risk: a
   malicious or simply wrong user could degrade the model's behavior for everyone.
8. Explain why the KV cache's size is a long-context bottleneck, and how GQA/MQA address
   it without simply discarding model capacity. — **Model answer:** The KV cache stores a
   key and value vector per token, *per layer, per attention head*, so it grows with
   sequence length and head count; for long contexts and large batches a per-head cache
   would not fit in GPU memory, and (since decode is memory-bandwidth-bound) a bigger
   cache also slows every decode step. MQA shares a single K/V pair across *all* heads;
   GQA — the common middle ground — splits heads into groups that each share one K/V
   pair. Both keep every *query* head intact, so each head still computes its own
   distinct attention pattern; only the key/value projections are shared. This shrinks
   the cache several-fold at little quality cost, which is what makes million-token
   contexts servable.
9. The attention formula divides query–key dot products by √d_k. Explain what goes wrong
   if this scaling is omitted, and why the problem is tied to the head dimension d_k. —
   **Model answer:** A query–key dot product sums d_k component-wise products, so its
   typical magnitude (variance) grows with d_k. Unscaled, large scores drive softmax into
   a saturated, near-one-hot regime: the model can barely blend information across
   tokens, and softmax there has near-zero gradients, so training stalls. Dividing by
   √d_k rescales scores to roughly unit variance *independent of d_k* — the factor is
   sized exactly to cancel the standard-deviation growth of a d_k-term sum — keeping
   attention soft and trainable however wide the heads are.
10. Inference-time quantization and speculative decoding both make decode faster but in
    very different ways. Explain each mechanism and, for each, state whether it changes
    the model's output. — **Model answer:** Decode is memory-bandwidth-bound (1.5).
    **Quantization** serves the model's weights at lower numeric precision (e.g. FP8 or
    INT8 instead of BF16), so each decode step streams fewer bytes through the
    memory-bandwidth bottleneck — faster decode, less GPU memory, lower cost. It *can*
    slightly change the output, since the computed numbers differ — though modern 8-bit
    is near-lossless and modern 4-bit is close. **Speculative decoding** uses a small,
    fast draft model to propose several tokens, which the large target model then
    verifies in a single parallel pass; accepted tokens cost one target pass instead of
    many. It does *not* change the output: the target model has the final say on every
    token, so the emitted sequence is exactly what it would have produced alone — only
    speed changes, and the gain depends on the draft model's acceptance rate.

### Long Answer

1. A junior engineer hands you a "transformer" they built that produces the *same*
   output regardless of the order of the input tokens, and whose deep version will not
   train past a few layers. Walk the data flow from token IDs to a next-token
   distribution, and at each stage identify which missing or broken component explains
   one of those two symptoms. — **Model answer / rubric:** Data flow: token IDs →
   embedding-matrix lookup → embedding vectors; **positional information must be injected
   here (or in attention via RoPE)** — its absence is exactly why order does not matter,
   since attention is otherwise order-agnostic. Then N stacked blocks, each with
   self-attention (cross-position mixing via Q/K/V, multi-head, causally masked) and a
   position-wise FFN; **each sub-layer needs a residual connection** — its absence is why
   the deep version will not train, since gradients/information cannot flow cleanly
   through the stack. Normalization also supports stable training: modern models use
   **RMSNorm** in **pre-norm** placement (norm on the sub-layer input), which keeps the
   residual path a clean highway — *post-norm* placement is itself a second plausible
   cause of a deep stack failing to train, and a strong answer names it. Finally, final
   vector → unembedding/LM head → one logit per vocabulary token → softmax →
   distribution. Strong answers correctly map symptom 1 → missing positional encoding and
   symptom 2 → missing residual connections (or post-norm placement), and note attention
   is the only cross-position step while the FFN holds most parameters/knowledge.
2. Explain the prefill/decode distinction and use it to account for at least three
   observed latency or cost behaviors. — **Model answer / rubric:** Prefill = one
   parallel forward pass over the prompt, compute-bound, sets TTFT, builds the KV cache.
   Decode = sequential per-token generation, memory-bandwidth-bound, sets TPOT/throughput.
   Behaviors: output priced higher than input (sequential memory-bound steps vs. one
   parallel pass); long prompt inflates TTFT while long answer inflates total time
   (separate levers); prompt caching reuses prefill KV state for a TTFT win; continuous
   batching exploits decode being memory-bound to amortize the weight load.
3. Explain why an LLM "remembering" a fact you told it in a previous, separate
   conversation is impossible without application-layer engineering. — **Model answer /
   rubric:** Inference never updates weights (frozen after training), and the API is
   stateless (no server-side memory between calls). Within one conversation, prior turns
   persist only because the application resends them in the context window. A new
   conversation is a fresh, empty context, so the fact is gone. Genuine cross-session
   recall requires the application to store the fact and re-inject it (a "memory"
   feature) or retrieve it (RAG) — that is engineering around the model, not the model
   learning.
4. Explain why self-attention's O(n²) complexity is the root cause of several practical
   constraints in LLM systems. — **Model answer / rubric:** Each token's query is
   compared to every token's key → n² compute and an n×n memory footprint. Consequences:
   long context is expensive (cost rises super-linearly with context); there is a hard
   context-window limit; "lost in the middle"/context-rot degradation; the motivation
   for KV caching, FlashAttention, sparse/sliding-window attention; and per-token API
   pricing. Strong answers tie the math directly to each downstream concern.

### Applied Scenario

1. A teammate says: "Our chatbot got smarter overnight — a user explained our refund
   policy to it yesterday and today it explains the policy correctly to everyone." What
   is actually happening? — **Model answer / rubric:** The model did not learn anything;
   inference never updates weights and the API is stateless. Either the correct policy
   was already in the model's parameters/context, or — more likely — an engineer added
   the policy to the system prompt or a retrieval store, so it now enters every
   request's context window. A single user's chat cannot change the model. Recommend
   verifying where the policy text now lives (system prompt? RAG index?) and version-
   controlling it.
2. A latency dashboard shows TTFT spiked from 0.4 s to 2.5 s but tokens-per-second is
   unchanged. What changed, and where would you look? — **Model answer / rubric:** TTFT
   is dominated by prefill, which scales with prompt length; unchanged tokens/sec means
   decode is fine. So the *input* grew — a longer system prompt, more conversation
   history, larger retrieved documents, or bigger tool definitions. Investigate input
   token counts per request; consider trimming history, summarization, or prompt caching
   to cut prefill work.
3. A product team finds the model reliably answers questions about famous public
   companies but is wrong or vague about *their own* small startup, and is wrong about
   employee head-count figures they consider "simple." They want to fix this by "using a
   bigger, smarter model." Evaluate that plan and propose what would actually work, for
   *each* of the two failure types. — **Model answer / rubric:** A bigger model does not
   address either failure, because both are structural. (a) The startup is barely or
   never represented in training data, so the parameters simply do not encode reliable
   facts about it — scaling a model trained on the same corpus does not invent that
   knowledge. Fix: supply the company's own information via context (system prompt,
   retrieval). (b) Specific head-count *numbers* are both data-sparse and subject to
   fragmented number tokenization, so they are predicted unreliably regardless of size.
   Fix: pull the figure from an authoritative source/tool rather than expecting the
   model to "know" it. The unifying point: next-token prediction is only as reliable as
   training-data support, so the remedy is supplying the right information at inference
   time, not enlarging the model.
4. Per-turn cost in a long-running agent conversation keeps climbing even though each
   new user message is short. Why, and what would you do? — **Model answer / rubric:**
   The API is stateless, so every turn resends the entire growing history as input
   tokens — the bill scales with accumulated context, not the new message. Mitigations:
   summarize or compact old turns, evict stale content, use retrieval-on-demand instead
   of keeping everything inline (Topic 4.7), and apply prompt caching to the stable
   prefix (Topic 6).
5. A vendor claims their API is "stateful — no need to resend history." What questions
   would you ask? — **Model answer / rubric:** The underlying model is still stateless;
   "stateful" must mean the vendor stores history server-side and re-injects it into the
   context window on each call. Ask: where is history stored and is it secure; is it
   still billed as input tokens each turn (it must be re-fed to the model); how is the
   context-window limit handled as history grows (truncation? summarization?); what is
   the data-retention policy. The convenience is real but the token economics and
   context limits do not disappear.

---

## Sources

[1] Dao, Fu, Ermon, Rudra, Ré — "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness" (NeurIPS 2022) — https://arxiv.org/abs/2205.14135
[2] Anthropic — Pricing (Claude API docs; every current Claude model prices output at 5× input) — https://platform.claude.com/docs/en/about-claude/pricing
[3] Meta AI — Introducing Meta Llama 3 (128K-token tokenizer vocabulary) — https://ai.meta.com/blog/meta-llama-3/
