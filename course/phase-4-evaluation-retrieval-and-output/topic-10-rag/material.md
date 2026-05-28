# Topic 10 — RAG (Retrieval-Augmented Generation) — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This topic is taught sub-chapter by sub-chapter, in order. Each sub-chapter has a
**Concept** section (the teaching content), **Key terms**, **Common misconceptions**, a
**Worked example**, and **Check questions** to confirm understanding before moving on.
The **Exam Question Bank** at the end is the pool the tutor draws from for the gated,
closed-book exam (scored out of 100, pass mark 85). Teach each sub-chapter fully, run its
check questions, then advance. RAG is the dominant pattern for grounding LLMs in
external and private knowledge — the goal is to be able to design, debug, and reason
about a RAG pipeline end-to-end, not just describe it.

---

## 10.1 — Why RAG exists — knowledge cutoff, private data, grounding, citations

### Concept

A pretrained LLM's knowledge is **frozen** at its training cutoff and limited to what was
in its training corpus. That creates four hard limitations that **Retrieval-Augmented
Generation (RAG)** exists to solve.

1. **Knowledge cutoff.** The model knows nothing that happened after its training
   cutoff — last week's news, this morning's price, today's incident. You cannot ask it
   about events it never saw.
2. **Private / proprietary data.** Your company's internal wiki, your customer's support
   history, your codebase, this user's documents — none of it was in pretraining. The
   model literally cannot know it.
3. **Hallucination and grounding.** When an LLM doesn't know something it does not
   reliably say "I don't know" — it produces a fluent, confident, plausible-sounding
   guess. **Grounding** means anchoring the model's answer in retrieved source text, so
   it generates *from evidence* rather than from parametric memory.
4. **Citations and verifiability.** Many applications (legal, medical, support, finance)
   require the answer to be **traceable** to a source a human can check. A bare model
   answer is unverifiable; an answer grounded in retrieved passages can carry citations.

The core idea of RAG is simple and powerful: **don't rely on what the model memorized —
fetch the relevant text at query time and put it in the context window**, then ask the
model to answer *using that text*. It turns a closed-book exam into an open-book exam.

The canonical pipeline has two phases:

- **Indexing (offline, one-time / periodic):** take your knowledge corpus → split it
  into **chunks** → compute an **embedding** for each chunk → store the chunks + vectors
  in a **vector database**.
- **Retrieval + generation (online, per query):** embed the user's query → search the
  vector store for the most similar chunks → place the top chunks into the prompt as
  context → the LLM generates an answer grounded in them, ideally with citations.

Why RAG instead of just retraining the model on your data? Updating is cheap and instant
(re-index a document; no training run), the knowledge source is auditable and editable,
access control can be enforced at retrieval time, and you get citations for free. RAG
does **not** teach the model new skills or behaviors — it supplies *knowledge*. That
distinction (knowledge vs. behavior) drives the decision framework in 10.9.

A subtle but important framing: RAG is fundamentally a **search problem wrapped around a
generation step**. The hardest, highest-leverage part is the retrieval — if you retrieve
the wrong chunks, even a perfect LLM gives a wrong answer. "Garbage in, garbage out"
applies with full force.

The full pipeline, with each stage annotated by the sub-chapter that covers it — this
diagram doubles as a topic map for everything that follows:

```
  INDEXING (offline, one-time / periodic)
  ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌────────────────────┐
  │ corpus  │──►│  CHUNK   │──►│  EMBED   │──►│ VECTOR DB / INDEX  │
  │         │   │ §10.3    │   │ §10.2    │   │ §10.4 (HNSW/IVF)   │
  └─────────┘   └──────────┘   └──────────┘   └────────────────────┘
                                                        │
  ─────────────────────────────────────────────────────┼───────────
                                                        ▼
  RETRIEVAL + GENERATION (online, per query)    ┌────────────────┐
  ┌─────────┐   ┌──────────┐   ┌────────────┐   │   the index    │
  │  query  │──►│  EMBED   │──►│ ANN SEARCH │◄──┤  (+ BM25 for   │
  │         │   │ §10.2    │   │ §10.4      │   │   hybrid §10.5)│
  └─────────┘   └──────────┘   └────────────┘   └────────────────┘
                                     │
                                     ▼
                              ┌────────────┐   ┌──────────────┐
                              │  RERANK    │──►│ LLM GENERATE │──► answer +
                              │  §10.6     │   │              │    citations
                              └────────────┘   └──────────────┘
  (failure modes §10.7 · agentic loop §10.8 · RAG vs alternatives §10.9)
```

### Key terms

- **RAG (Retrieval-Augmented Generation)** — fetching relevant external text at query time and putting it in the prompt so the LLM answers from evidence.
- **Knowledge cutoff** — the date after which a model has no training knowledge.
- **Grounding** — anchoring an answer in retrieved source text rather than parametric memory.
- **Indexing** — the offline phase: chunk → embed → store the corpus.
- **Retrieval** — the online phase: find the chunks most relevant to a query.
- **Citation / attribution** — linking an answer's claims back to the source passages.
- **Parametric knowledge** — knowledge stored in the model's frozen weights.

### Common misconceptions

- ❌ "RAG teaches the model new skills." → ✅ RAG supplies knowledge into the context; it does not change model weights or behavior. Skills/behavior are the domain of fine-tuning.
- ❌ "RAG eliminates hallucination." → ✅ It greatly reduces ungrounded hallucination by supplying evidence, but the model can still misread, over-generalize, or ignore the context; RAG mitigates, it does not cure.
- ❌ "RAG is mainly an LLM problem." → ✅ RAG is mostly a search/retrieval problem; the generation step is comparatively easy, and most RAG failures are retrieval failures.

### Worked example

A company has a 4,000-page internal HR handbook. A user asks: *"How many days of
parental leave do I get, and is it paid?"*

- **No RAG:** the base model never saw this handbook. It either refuses or invents a
  plausible-but-wrong number — unacceptable for HR.
- **With RAG:** the handbook was chunked and embedded into a vector store at indexing
  time. At query time the system embeds the question, retrieves the 3 most similar
  chunks (which include the parental-leave policy section), and prompts: *"Using only
  the passages below, answer the question and cite the section. Passages: [...]"*. The
  model answers "16 weeks, fully paid" and cites "§7.3 Parental Leave" — a grounded,
  verifiable answer the model could never have produced from parametric memory.

### Check questions

1. A user asks a base model (no RAG) "what is our company's parental-leave policy?" and gets a confident, specific, completely fabricated answer. Which two of RAG's core motivations does this single failure illustrate, and how does RAG address each? — **Answer:** (1) Private/proprietary data — the policy was never in pretraining, so the model literally cannot know it; RAG fetches the actual policy document at query time. (2) Hallucination/grounding — lacking the knowledge, the model produces a fluent guess instead of "I don't know"; RAG grounds the answer in retrieved source text so it generates from evidence. (Citations/verifiability is also relevant — the RAG answer can cite the policy section.)
2. A teammate says "we should fine-tune the model on our 50,000 help articles so it answers from them — that's basically RAG." Identify the conceptual error. — **Answer:** It conflates knowledge with behavior. RAG supplies *knowledge* into the context window at query time without changing weights — instant updates, auditable sources, citations. Fine-tuning changes the *weights/behavior* and injects facts unreliably (hallucination-prone), freezes them at training time, and gives no citations. "Answer from our articles" is a knowledge problem; RAG, not fine-tuning, is the right tool.
3. Two RAG systems use the identical top-tier LLM and identical prompt. System A answers a question correctly; System B answers it wrong. Given the LLM and prompt are the same, where must the difference lie, and what principle does this illustrate? — **Answer:** The difference must be in *retrieval* — System B fetched chunks that did not contain (or buried) the answer-bearing text, so its LLM had no chance. This illustrates that RAG is fundamentally a search problem wrapped around a generation step: answer quality is bounded by retrieval quality — garbage in, garbage out — so most RAG failures are retrieval failures, not generation failures.

---

## 10.2 — Embeddings — dense vectors, semantic similarity, cosine similarity

> **Tiered sub-chapter — overview required, deep-dive optional.** This chapter has
> two parts: a short **Overview** that everyone needs (taught and tested normally),
> and a longer **Deep dive** with the mechanism details (skip-by-default, available
> on request). After the Overview and its check questions, the tutor will ask
> whether to take the deep dive now, skip it for later, or advance to §10.3. The deep
> dive is interview-bait and worth doing eventually, but it is not required to
> advance, and its content is not in the topic exam.

### Overview — Concept

An **embedding** is a fixed-length list of numbers — a **dense vector** — that
represents the *meaning* of a piece of text. An embedding model maps text to a point in a
high-dimensional space (typically 256–4096 dimensions; e.g., OpenAI's
`text-embedding-3-large` defaults to 3072-dim, many open models are 768 or 1024) such
that **texts with similar meaning land near each other**, regardless of the exact words
used. [2]

"Dense" contrasts with "sparse." A **sparse** representation (like a bag-of-words /
TF-IDF vector) has one dimension per vocabulary word and is mostly zeros — it captures
*which exact words appear*. A **dense** embedding has every dimension populated with a
real number and captures *semantic content* — so "car," "automobile," and "vehicle" map
to nearby vectors even though they share no characters. This is what lets RAG match a
query to a passage that answers it in completely different words.

**Measuring similarity.** Given two vectors you need a number for "how alike." The
standard choice is **cosine similarity**: the cosine of the angle between the two
vectors. It ranges from −1 (opposite) through 0 (unrelated/orthogonal) to 1 (identical
direction). Crucially it measures **direction, not magnitude** — it ignores how *long*
the vectors are, so it isn't skewed by text length or overall "intensity," only by
semantic orientation. Many embedding models output **normalized** (unit-length) vectors;
for those, cosine similarity, dot product, and (negative) Euclidean distance all rank
results identically, so the choice is mostly about convention and performance.

**Practical realities an engineer must know:**
- **Model consistency.** The query and the documents *must be embedded by the same
  model*. Vectors from different models live in incompatible spaces and cannot be
  compared. Re-indexing the whole corpus is required to switch embedding models.
- **Query/document asymmetry.** A short query and a long passage are different "shapes"
  of text, and embedding models treat them asymmetrically — they are optimized to place a
  query near its *answering passage*, not near other queries. Many embedding models
  support **task instructions / prefixes** ("represent this query for retrieval...") or
  separate query/document encoders to handle this; use them. (*Why* the asymmetry exists
  is in the deep dive — it falls out of how embedding models are trained.)
- **Dimension trade-off.** Higher dimensions can capture more nuance but cost more
  storage and compute; some models (Matryoshka-style — including `text-embedding-3`) let
  you truncate to a smaller dimension via a `dimensions` parameter with graceful quality
  loss. [2]
- **Domain fit.** A general embedding model may be weak on specialized jargon (legal,
  medical, code); domain-specific or fine-tuned embeddings can materially improve recall.
  (*Why* a general model underperforms on a specialized corpus is also a deep-dive
  consequence of the training objective.)

### Overview — Key terms

- **Embedding** — a dense numeric vector representing the meaning of text.
- **Dense vector** — a vector with all dimensions populated, capturing semantic content.
- **Sparse vector** — a mostly-zero vector with one dimension per word (e.g., TF-IDF), capturing exact terms.
- **Embedding model** — a model trained to map text to a semantic vector space.
- **Cosine similarity** — cosine of the angle between two vectors; measures directional/semantic closeness, ignores magnitude.
- **Normalization** — scaling a vector to unit length, after which cosine and dot product rank identically.
- **Embedding dimension** — the length of the vector (e.g., 768, 1536, 3072).
- **Query/document asymmetry** — the fact that embedding models treat short queries and long documents as different roles; handled with task prefixes or separate encoders.

### Overview — Common misconceptions

- ❌ "Cosine similarity measures word overlap." → ✅ It measures the angle between learned semantic vectors; two texts with no shared words can have high cosine similarity if they mean the same thing.
- ❌ "You can mix embeddings from different models." → ✅ Different models produce vectors in incompatible spaces; query and corpus must be embedded by the same model, and switching models requires full re-indexing.
- ❌ "More embedding dimensions are always better." → ✅ Higher dimensions cost more storage/compute and have diminishing returns; the right size depends on the task and budget.
- ❌ "A general embedding model is equally good on any corpus." → ✅ On specialized domains (legal, medical, your jargon) a general model can underperform; fine-tuning on in-domain query→document pairs can materially lift recall.

### Overview — Worked example

Query: *"How do I reset my password?"*

Three candidate passages, their cosine similarity to the query embedding:
- *"To change your account credentials, go to Settings → Security and follow the recovery flow."* → cosine ≈ 0.83. **High** — no word overlap with "reset/password" at all, but semantically it *is* the answer. This is exactly what dense embeddings buy you.
- *"Our password policy requires 12 characters and rotation every 90 days."* → cosine ≈ 0.55. Shares the word "password" but is about policy, not resetting — moderate.
- *"The cafeteria menu changes every Monday."* → cosine ≈ 0.04. Unrelated — near orthogonal.

Geometrically, cosine similarity is the cosine of the *angle* between the query vector
and each passage vector — small angle = high cosine = close meaning:

```
        ▲
        │        P1 "change credentials / recovery flow"
        │       ╱   small angle → cosine ≈ 0.83  (the real answer)
        │      ╱
        │     ╱  ┌─ QUERY "how do I reset my password?"
        │    ╱ ╱
        │   ╱╱        P2 "password policy: 12 chars, 90 days"
        │  ╱╱  ╲     wider angle → cosine ≈ 0.55  (shares a word, wrong topic)
        │ ╱╱     ╲
        │╱╱        ╲
   ─────●───────────────────────► P3 "cafeteria menu changes Mondays"
     origin            ~90° (orthogonal) → cosine ≈ 0.04  (unrelated)
```

A keyword search would have ranked the policy passage above the actual answer (more
literal "password" matches). The embedding ranks by meaning and surfaces the right one.

### Overview — Check questions

1. The query "How do I cancel my subscription?" and the passage "To end your membership, visit Account → Billing and select Close plan." share almost no words. A keyword search ranks this passage low; a dense-embedding search ranks it high. Explain what property of embeddings produces this difference. — **Answer:** Embeddings are dense semantic vectors learned so that texts with similar *meaning* land near each other regardless of shared words. "Cancel subscription" and "end membership / close plan" are semantically close, so their vectors are close and cosine similarity is high. Keyword/sparse search keys on exact terms, so with no overlapping words it scores the passage low. Dense search matches meaning, not surface form.
2. A team embeds short queries and long documents with the same general model and gets mediocre retrieval. Two embeddings can have high cosine similarity yet one is a 5-word query and the other a 500-word passage — why does cosine not penalize this length gap, and what practical step addresses the query/document mismatch? — **Answer:** Cosine similarity measures the *angle* (direction) between vectors, not their magnitude, so it is not skewed by text length — a short query and long passage can point the same direction. But query and document are still different "shapes" of text; the practical fix is to use the model's task instructions/prefixes (e.g., "represent this query for retrieval...") or separate query/document encoders, which the model provides precisely to handle this asymmetry.
3. A team switches from embedding model A to a newer model B but, to save time, only re-embeds *new* documents — old documents keep their model-A vectors. Retrieval quality collapses. Explain precisely why. — **Answer:** Each embedding model defines its own vector space; model-A and model-B vectors have no shared coordinate meaning, so cosine distances *between* them are meaningless. With a query embedded by B compared against a corpus that is half A-vectors and half B-vectors, similarity scores are only valid for the B half — the A documents are effectively unrankable noise. Switching models requires re-embedding the *entire* corpus, not just new additions.

---

### Deep dive — Concept *(optional)*

The Overview took it on faith that an embedding model "places similar meanings near each
other," that there is a query/document asymmetry, and that a general model can underperform
on a specialized corpus. This section explains *why* — and the explanation is one mechanism:
how embedding models are trained. The three practical quirks above are all consequences of
that one objective. This is interview-bait — embedding-model training and the
dual-encoder/contrastive recipe come up in senior interviews — but it is **not required**
to advance and is not in the topic exam.

**The contrastive / dual-encoder objective.** Embeddings are *learned*, and how they are
learned explains the practical behavior you saw in the Overview. The dominant recipe is
**contrastive learning** with a **dual-encoder** (also called a **two-tower**) setup: you
take a large set of *positive pairs* — texts that *should* be close, e.g., a question and
a passage that answers it, a query and a clicked document, a sentence and its paraphrase
— and you train the model so that, for each anchor, its true positive is scored more
similar than a batch of *negatives* (unrelated texts). The training signal is a
softmax-style **contrastive (InfoNCE) loss**: it maximizes the similarity of the positive
pair while minimizing similarity to the negatives. Geometrically, every training step
**pulls a related pair together and pushes unrelated pairs apart**, and after millions of
such updates the space is organized so that closeness = semantic relatedness.

**Why "dual-encoder."** The two roles in a positive pair (query and document) can be encoded
by *separate* encoders (a query tower and a document tower) and compared by vector
similarity. Even when the two towers share weights, the architecture treats query and
document as distinct inputs by design, and similarity is computed *independently per side*
— which is exactly what makes the architecture scalable (document vectors can be precomputed
once and reused, see §10.6 for the contrast with cross-encoders).

Three consequences of this objective matter directly to a RAG engineer:

- **It explains the query/document asymmetry.** The positive pairs in retrieval training
  are typically *short query ↔ long document* — two different shapes of text serving
  different roles. The model is optimized to place a query near its answering passage,
  *not* near other queries. The learned relationship is directional (query→document), not
  a symmetric "are these the same text" relationship — which is why query/document task
  prefixes exist (to tell the model which role a given text is playing) and why some
  models use separate encoder towers.
- **Negatives — especially hard negatives — are what make it sharp.** A model trained
  only against obviously-unrelated negatives learns coarse distinctions. Good embedding
  models are trained with **hard negatives**: passages that look superficially relevant
  but are wrong answers. Hard negatives force the contrastive loss to learn fine-grained
  distinctions, which is exactly what separates a usable retriever from a vague one.
- **It explains the domain-fit problem.** The model only learns the relationships present
  in its *training pairs*. A model trained on general web Q&A has never seen what
  "relevant" means for legal citations, medical coding, or your internal product jargon —
  so its space organizes those poorly. This is why a general embedding model can
  underperform on a specialized corpus, and why **fine-tuning the embedding model on
  in-domain pairs** (your own query→document examples, ideally with hard negatives) can
  materially lift recall: you are giving the contrastive objective the relationships that
  actually matter for *your* retrieval task.

### Deep dive — Key terms

- **Contrastive learning** — training that pulls positive (related) text pairs together and pushes negatives apart, using a contrastive loss.
- **InfoNCE loss** — the softmax-style contrastive loss used in modern embedding training; maximizes the score of the positive pair while pushing down a batch of negatives.
- **Positive pair** — two texts that should be close (e.g., query ↔ answering passage); the supervision signal.
- **Negative / hard negative** — an unrelated (or *superficially relevant but wrong*) text used as the contrast; hard negatives force fine-grained distinctions.
- **Dual-encoder / two-tower** — the architecture for embedding-based retrieval: query and document are encoded independently (possibly by separate towers) and compared by vector similarity.

### Deep dive — Common misconceptions

- ❌ "Embeddings just learn 'meaning' from text — the training is a black box." → ✅ The
  objective is specific: contrastive learning on positive/negative *pairs*. Every behavior
  you see (asymmetry, hard-negative sensitivity, domain-fit failures) is a direct
  consequence of that objective.
- ❌ "Hard negatives are an optimization detail." → ✅ They are load-bearing — easy
  negatives only teach coarse distinctions; a model without hard negatives is a vague
  retriever.
- ❌ "Fine-tuning the embedding model just teaches it new vocabulary." → ✅ It teaches it
  the *relationships* (which queries should be close to which documents) that the
  general-pair training never saw — that is what lifts recall on a specialized corpus.

### Deep dive — Worked example

Take the patent-filing scenario from the Overview's domain-fit point. With the contrastive
objective in view, trace *why* a general embedding model fails and *what* fine-tuning
actually changes:

1. The general model was trained on positive pairs from general web Q&A: "how do I
   reset my password?" ↔ "to change your account credentials, go to Settings..." and
   millions like it. The InfoNCE loss pulled those pairs together and pushed unrelated
   web text apart.
2. The model therefore knows what "related" looks like *for general web text*. It has
   no positive pair that says "patent query about claim construction" ↔ "patent passage
   citing 35 U.S.C. § 112." It never saw that relationship.
3. At inference on patent filings, the model places patent passages in whatever region of
   the space its general-text geometry happens to put them — there is no learned signal
   organizing patent semantics, so the geometry is approximately random for the
   patent-specific distinctions that matter.
4. Fine-tuning supplies new positive pairs (`patent query` ↔ `passage that answers it`)
   and ideally hard negatives (passages that *look* like the right answer but aren't).
   The contrastive loss reorganizes the space so patent queries land near their answering
   passages, exactly as the original pretraining did for general text.

The asymmetry behavior follows the same logic: every training pair is `query ↔ document`,
so the learned closeness is directional, not symmetric — the model never had a "two queries
mean the same thing" signal to learn.

### Deep dive — Check questions

1. A general-purpose embedding model gives strong retrieval on everyday queries but mediocre retrieval on a corpus of patent filings full of specialized legal-technical language. Using how embedding models are trained, explain why, and what would most directly fix it. — **Answer:** Embedding models are trained by contrastive learning on positive *pairs* — they only learn the notion of "related" present in their training data. A general model's pairs come from everyday web text, so it never learned what "relevant" means for patent-style language, and its vector space organizes that jargon poorly. The most direct fix is to fine-tune the embedding model on in-domain pairs (real query→patent-passage examples, ideally with hard negatives), giving the contrastive objective the domain relationships that matter — which materially lifts recall on the specialized corpus.
2. Embedding-based retrieval is called a "dual-encoder" and is described as asymmetric, optimized to place a short query near its answering passage rather than near other queries. Tie this directly back to the training objective. — **Answer:** The contrastive objective is trained on *positive pairs*, and for retrieval those pairs are short query ↔ long answering document — two different roles. The model is optimized to pull a *query toward its answering passage*, not to pull queries toward each other, so the learned relationship is directional (query→document), not a symmetric same-text similarity. "Dual-encoder" captures that the two roles can even use separate encoder towers, and it is why query/document task prefixes exist — to tell the model which role a given text is playing.

---

## 10.3 — Chunking strategies and tradeoffs

### Concept

You cannot embed a 200-page document as one vector — a single vector cannot represent
that much content, and you'd retrieve the whole document when only one paragraph is
relevant. So at indexing time documents are split into **chunks**: the unit of text that
gets its own embedding and is retrieved as a whole. Chunking is one of the highest-
leverage and most underrated decisions in a RAG system, because the chunk is *both* the
unit of retrieval *and* the unit of context the model sees.

**The central tension:**
- **Chunks too large** → each chunk mixes multiple topics, so its embedding is a blurry
  average that matches no query sharply (recall drops); retrieved context is padded with
  irrelevant text that distracts the model and wastes tokens.
- **Chunks too small** → a chunk loses the surrounding context needed to be understood
  (a sentence with an unresolved pronoun, a number with no label); the model gets a
  fragment that is technically relevant but not usable.

You are trading **precision of the embedding** against **completeness of the retrieved
context**.

**Common strategies:**
- **Fixed-size chunking** — split every N tokens (e.g., 256–512). Simple, predictable,
  fast. Weakness: it cuts blindly through sentences, tables, and code, splitting ideas
  mid-thought.
- **Overlapping windows** — fixed-size but consecutive chunks share an overlap (e.g., 50
  tokens / ~10–20%). The overlap means an idea straddling a boundary still appears
  intact in at least one chunk. Cheap and effective; the standard default. Cost: storage
  and some duplication.
- **Recursive / structure-aware chunking** — split on natural boundaries in priority
  order: paragraphs, then sentences, then words, recursing only when a piece is still too
  big. Respects document structure far better than blind fixed-size.
- **Semantic chunking** — embed sentences and start a new chunk when consecutive
  sentences' similarity drops (a topic shift). Produces topically coherent chunks; costs
  more to compute and can be finicky.
- **Document-structure chunking** — split by the document's own structure: markdown
  headings, HTML sections, code by function/class, slides by slide. Usually best when
  the corpus has reliable structure.

**Techniques layered on top:**
- **Metadata enrichment** — store source, title, section heading, date, author,
  permissions with each chunk. Used for filtering (date range, access control) and for
  giving the model provenance.
- **Contextual retrieval / context augmentation** — prepend a short generated summary of
  the chunk's place in the document ("This chunk is from the 'Refund Policy' section of
  the 2025 Terms of Service") to the chunk *before embedding*. This recovers context a
  small chunk lost and measurably improves retrieval — a now-standard technique. (In
  Anthropic's experiments, contextual embeddings cut the top-20 retrieval failure rate
  by 35%, and combining them with contextual BM25 cut it by 49%.) [1]
- **Small-to-big / parent-document retrieval** — embed and *match on* small precise
  chunks, but *return to the model* the larger parent passage they came from. Best of
  both: sharp matching, complete context.

**There is no universal best.** Chunk size and strategy depend on the corpus (prose vs.
code vs. tables vs. chat logs) and the queries (fact lookup vs. summarization). The
correct way to choose is empirical: build an eval set (Topic 9) and measure retrieval
quality across chunking configurations.

### Key terms

- **Chunk** — the unit of text that gets one embedding and is retrieved as a whole.
- **Fixed-size chunking** — splitting at a fixed token count.
- **Chunk overlap** — sharing text between consecutive chunks so boundary-straddling ideas survive.
- **Recursive chunking** — splitting on a priority of natural boundaries (paragraph → sentence → word).
- **Semantic chunking** — splitting at detected topic shifts using sentence similarity.
- **Contextual retrieval** — prepending a chunk-situating summary before embedding to restore lost context.
- **Small-to-big / parent-document retrieval** — matching on small chunks but returning their larger parent passage.

### Common misconceptions

- ❌ "Bigger chunks are safer because they include more context." → ✅ Oversized chunks blur the embedding (worse matching) and pad context with distracting irrelevant text; size is a trade-off, not a one-way dial.
- ❌ "There's an optimal chunk size (e.g., 512 tokens) for everything." → ✅ The right size depends on the corpus and query type; it must be chosen empirically with a retrieval eval.
- ❌ "Chunking is a minor preprocessing detail." → ✅ The chunk is both the retrieval unit and the context unit; bad chunking is a leading cause of RAG failure.

### Worked example

A RAG system over API documentation retrieves poorly for *"What is the default timeout
for the upload endpoint?"*

- **Original config:** fixed 1,000-token chunks. The chunk containing the timeout also
  contains four unrelated endpoints; its embedding is a blurry topic-average, so it
  ranks low for the specific query.
- **Fix 1 — smaller chunks (250 tokens) by endpoint:** now each endpoint is its own
  chunk; the upload-endpoint chunk matches sharply. But some chunks are now just a code
  snippet with no surrounding explanation.
- **Fix 2 — contextual retrieval:** before embedding, each chunk is prefixed with "This
  describes the `POST /upload` endpoint of the File API." Now even snippet-only chunks
  embed with their context.
- **Fix 3 — small-to-big:** match on the 250-token chunk, but feed the model the full
  endpoint section. Retrieval is precise *and* the model gets complete context.

A retrieval eval over 80 doc questions confirms recall@5 rose from 0.61 → 0.92.

### Check questions

1. A RAG system over legal contracts uses very large 2,000-token chunks "to be safe and include lots of context." Retrieval precision is poor. Explain the specific mechanism by which oversized chunks *hurt* retrieval — note it is not only about wasted tokens. — **Answer:** A large chunk mixes several topics, so its single embedding is a blurry *average* of all of them — it points sharply at no specific query, so it ranks low even when it contains the answer (a recall/matching failure, not just a token-cost issue). Separately, when retrieved it pads the context with irrelevant text that distracts the model. The core harm is that the embedding can no longer represent any one idea precisely.
2. A document says "The fee is waived for these accounts." in one chunk and the list of eligible accounts begins in the next chunk. With no overlap, why does retrieval for "which accounts get the fee waived?" fail, and how does overlap fix it? — **Answer:** The answer straddles a chunk boundary: the rule is in chunk N, the eligibility list in chunk N+1, so neither chunk alone is a complete, matchable answer — whichever is retrieved is a fragment. Overlapping windows make consecutive chunks share text, so the boundary-straddling idea appears *intact* in at least one chunk, making it retrievable and usable.
3. Small-to-big retrieval embeds small chunks but returns their larger parent passage. A teammate asks "why not just embed the large parent directly and skip the small chunks?" Answer them. — **Answer:** Because the two roles need different sizes. Matching needs a *small, precise* embedding (a small chunk's vector is sharp and ranks well for a specific query); the model needs *complete context* to answer. Embedding the large parent directly reverts to a blurry multi-topic embedding that matches poorly — you lose retrieval precision. Small-to-big keeps the sharp small-chunk vector for matching and only then expands to the parent for generation.

---

## 10.4 — Vector databases and ANN search (HNSW, IVF)

> **Tiered sub-chapter — overview required, deep-dive optional.** This chapter has
> two parts: a short **Overview** that everyone needs (taught and tested normally),
> and a longer **Deep dive** with the mechanism details (skip-by-default, available
> on request). After the Overview and its check questions, the tutor will ask
> whether to take the deep dive now, skip it for later, or advance to §10.5. The deep
> dive is interview-bait and worth doing eventually, but it is not required to
> advance, and its content is not in the topic exam.

### Overview — Concept

After indexing you have millions of chunk embeddings. At query time you embed the query
and must find the most similar vectors. The naive approach — **exact k-nearest-neighbor
search** — compares the query against *every* stored vector. That is O(N) per query and
becomes far too slow at scale (tens of millions of vectors). The solution is **Approximate
Nearest Neighbor (ANN) search**: trade a small, controllable amount of accuracy
(**recall**) for orders-of-magnitude faster queries. A **vector database** is a system
built to store embeddings and serve ANN search, plus metadata filtering, CRUD, and
operational concerns (persistence, scaling, backups).

**Two index families dominate.** You should be able to pick between them by their
trade-off shape; the internal mechanisms are in the deep dive.

- **HNSW (Hierarchical Navigable Small World)** — a *graph-based* index, the default in
  most modern vector DBs. Properties: **excellent recall/speed**, supports incremental
  inserts. Cost: **high memory** (the graph plus vectors live in RAM). [3]
- **IVF (Inverted File Index)** — a *cluster-based* index. Properties: **lower memory**
  than HNSW, fast builds. Cost: **some recall loss** at cluster boundaries; less friendly
  to heavy incremental updates.

**Query-time recall dials.** Both families expose a knob that trades recall against
latency *at query time, without rebuilding the index* — HNSW's is **`efSearch`**, IVF's
is **`nprobe`**. Higher = more search effort per query = higher recall, higher latency.
This is the single most operationally useful fact: if recall is slightly low and you
can't afford a rebuild, raise the query-time knob.

**The recall/latency/cost/memory triangle** is the heart of operating a vector DB. You
cannot maximize all of: recall, query speed, low memory, fast inserts. HNSW favors
recall and speed at memory cost; IVF favors memory at some recall cost. Compression
techniques (deep dive) buy more memory for some recall. Pick by your constraints and
**measure recall against an exact-search baseline** on your own data.

**Naming the landscape (2026).** **pgvector** — the Postgres extension; production-grade
at significant scale, keeps vectors next to your relational data and access control,
supports HNSW and IVFFlat; the pragmatic default if you already run Postgres.
**Pinecone** — fully-managed, serverless, scales without manual index tuning. **Qdrant**,
**Weaviate**, **Milvus** — feature-rich dedicated vector DBs (Weaviate is noted for
strong native hybrid search; Milvus for very large scale). **Chroma** — lightweight,
developer-friendly, good for prototyping. **FAISS** — Meta's ANN *library* (not a full
DB) — the building block many systems embed. Choose based on scale, whether you want
managed vs. self-hosted, and whether you need vectors co-located with existing data.

### Overview — Key terms

- **Approximate Nearest Neighbor (ANN)** — search that trades a small, tunable amount of recall for large speed gains over exact kNN.
- **Recall** — fraction of the true nearest neighbors that the approximate search actually returns.
- **Vector database** — a system for storing embeddings and serving ANN search with metadata filtering and CRUD.
- **HNSW** — a graph-based ANN index family; default in most vector DBs; excellent recall/speed at high memory cost.
- **IVF** — a cluster-based ANN index family; lower memory than HNSW with some recall loss at cluster boundaries.
- **efSearch / nprobe** — query-time knobs trading recall against latency (HNSW / IVF respectively); the on-the-fly recall dial.
- **Recall/latency/cost/memory trade-off** — the four-way trade you cannot all maximize; index choice picks which to favor.

### Overview — Common misconceptions

- ❌ "Vector DBs find the exact nearest neighbors." → ✅ They use ANN — approximate search that trades a tunable amount of recall for speed; exact search doesn't scale.
- ❌ "HNSW is strictly best." → ✅ HNSW gives excellent recall/speed but is memory-hungry; IVF trades recall for far lower memory. The right choice depends on the recall/latency/cost/memory constraints.
- ❌ "You can't change recall without rebuilding the index." → ✅ Query-time knobs (`efSearch` for HNSW, `nprobe` for IVF) trade recall vs. latency on the fly without rebuilding.

### Overview — Worked example

A corpus of 20M chunk embeddings (1024-dim). Exact kNN: ~20M dot products per query →
hundreds of ms, far too slow.

- **HNSW (`efSearch=64`):** recall ≈ 0.95 vs. exact, query ≈ 5 ms. But the index needs
  ~100+ GB RAM. Raising `efSearch` to 128 lifts recall to ≈ 0.98 at ≈ 9 ms — the
  query-time recall/latency dial in action.
- **IVF (`nprobe=32`, with compression):** memory drops to a fraction of HNSW's, query
  ≈ 8 ms, recall ≈ 0.90. Raising `nprobe` to 128 lifts recall toward 0.96 at higher
  latency.
- **Decision:** if RAM budget is tight, IVF; if recall is paramount and RAM is
  available, HNSW. Either way, recall is measured against an exact-search sample of 500
  queries so the trade-off is a known number, not a guess.

### Overview — Check questions

1. A prototype RAG system with 5,000 chunks uses exact (brute-force) nearest-neighbor search and is perfectly fast. The team scales to 30 million chunks and latency becomes unacceptable. Explain why exact search broke at scale and what specifically ANN gives up to fix it. — **Answer:** Exact kNN compares the query against *every* stored vector — O(N) per query — so going from 5k to 30M vectors multiplies per-query work ~6,000×. ANN builds an index that reaches the likely nearest neighbors without scanning everything, restoring speed; what it gives up is a small, *tunable* amount of recall — it may miss some true neighbors. Exact search is fine at prototype scale and breaks precisely because cost grows linearly with corpus size.
2. Two teams must choose an ANN index. Team A runs on memory-constrained hardware and rarely updates the corpus; Team B needs the best possible recall, has ample RAM, and inserts new documents continuously. Recommend HNSW or IVF for each and justify with the specific trade-off. — **Answer:** Team A → IVF: it is a cluster-based index with lower memory than HNSW and fast builds, and the recall loss at cluster boundaries is acceptable when RAM is the binding constraint and updates are rare (IVF is less update-friendly). Team B → HNSW: a graph-based index with excellent recall/speed and good support for incremental inserts, at the cost of high memory — which Team B can afford. Match the index to the binding constraint.
3. Recall on your HNSW-backed system is slightly too low for a launch, but a full index rebuild would take hours you don't have. What can you change, why does it work without a rebuild, and what does it cost? — **Answer:** Raise `efSearch`, the *query-time* search-effort knob. It works without a rebuild because it controls how hard each query searches the existing graph at run time (not how the graph is built). Higher `efSearch` explores more of the graph, raising recall; the cost is increased per-query latency. It is the on-the-fly recall/latency dial.

---

### Deep dive — Concept *(optional)*

The Overview said "HNSW is a graph-based index" and "IVF is a cluster-based index" and
asked you to take those properties on faith. This section unpacks the mechanisms — what
the graph actually *is*, what the clusters actually *are*, what the build-time knobs do,
and how compression layers on top. It is interview-bait — ANN internals show up in
infra/retrieval interviews — but it is **not required** to advance and is not in the
topic exam.

**HNSW — the layered-graph descent.** HNSW builds a *multi-layer* navigable graph. The
bottom layer (Layer 0) contains *every* vector with **short-range** links to its near
neighbors — dense and precise. Each higher layer is a sparse subset of the layer below
with **longer-range** links — fewer nodes, bigger hops. A search enters at the sparse
top layer, takes long coarse hops toward the query, then descends layer by layer into
progressively denser links for the final precise hop:

```
              query
                ┆
  Layer 2  ●────┆──────────────────●          sparse: long-range links,
  (top)    │    ┆                  │          fast coarse jumps
           └────┆─────●            │
                ┆     │ descend
  Layer 1  ●────┆●────●────●───────●          denser: medium-range links
           │    ┆│         │       │
           └────┆●─────────●       │
                ┆ │ descend
  Layer 0  ●─●──┆●─●─●─●─●─●─●─●─●─●           densest: short-range links,
  (bottom) all vectors live here              precise final neighbors
                ┆
            ┄┄┄►● nearest neighbor found
```

(The dashed line traces the query descending layer to layer to its nearest neighbor.)
The hierarchy is why HNSW achieves roughly **logarithmic** query-time scaling: the long
hops at the top cover the search space fast, then each descent narrows the region. And
because the graph supports insertion (find where a new vector should go and wire up its
links), it supports incremental updates.

**HNSW's tuning knobs.** Three knobs matter:

- **`M`** — the maximum number of links per node. Higher `M` makes the graph denser →
  better recall, slower build, more memory. Build-time only.
- **`efConstruction`** — the search-effort used *during build* to decide each new node's
  neighbors. Higher = better-quality graph at the cost of build time. Build-time only.
- **`efSearch`** — the search-effort used *per query* to explore the graph. Higher =
  better recall, higher per-query latency. **Query-time** — this is the on-the-fly recall
  dial from the Overview, and the reason it works without a rebuild is that it changes
  how hard each query searches the existing graph, not how the graph is built.

**IVF — k-means partitioning and nprobe pruning.** IVF works completely differently. At
build time it runs **k-means** clustering on the corpus, producing `nlist` centroids
(typical: thousands), and each vector is assigned to the partition of its nearest
centroid. At query time the engine computes the query's distance to all `nlist`
centroids — cheap — picks the `nprobe` partitions whose centroids are nearest the query,
and only searches *those* partitions, skipping the rest. If `nlist=4096` and `nprobe=32`,
the search touches roughly `32/4096 ≈ 0.8%` of the corpus, which is the speedup.

The **boundary failure mode** falls directly out of this: a true nearest neighbor sitting
just *across* a partition boundary may live in a partition the engine didn't probe, so
it is silently missed. Raising `nprobe` searches more partitions and recovers more such
neighbors, at proportionally more work. That is why `nprobe` is the query-time recall
dial for IVF. (And why IVF needs training data — k-means must see a representative sample
to choose centroids — and is less friendly to heavy incremental updates: the partitions
were sized for the corpus you trained on.)

**Quantization — compressing the vectors themselves.** Quantization layers on top of
either index family to cut *memory* at a small recall cost:

- **Scalar quantization** — store each dimension as `int8` (1 byte) instead of `float32`
  (4 bytes) → ~4× smaller, near-imperceptible recall hit. The cheap default.
- **Product Quantization (PQ)** — split each vector into `m` sub-vectors, k-means each
  sub-space into a small codebook (e.g., 256 entries → 1 byte per sub-vector), and store
  each vector as a sequence of `m` byte-codes. A 1024-dim float32 vector (4096 bytes)
  becomes, say, 64 bytes — ~64× smaller. Distance is computed by looking up sub-distances
  in precomputed tables. The recall hit is bigger than scalar quantization's but the
  memory saving is dramatic, which is why `IVF-PQ` is the canonical big-corpus
  low-memory combination.

**DiskANN** — for corpora that won't fit in RAM at all, DiskANN keeps the index on SSD,
arranging the graph so a query reads only a small number of disk pages per search. Higher
per-query latency than in-memory HNSW, but it lets you serve corpora orders of magnitude
larger on the same hardware.

All compression and disk-residency tricks trade some recall (or some latency) for cost
or memory — the trade-off triangle, again.

### Deep dive — Key terms

- **Layered/hierarchical graph (HNSW)** — multi-layer graph with sparse long-range links
  at the top and dense short-range links at the bottom; gives logarithmic query-time
  scaling.
- **`M` / `efConstruction`** — HNSW's build-time knobs (links per node / build-time search
  effort); changing them requires rebuilding.
- **k-means partitioning (IVF)** — clusters the corpus into `nlist` centroids at build
  time; each vector belongs to one partition.
- **`nlist` / `nprobe`** — IVF's partition count (build-time) and per-query probe count
  (query-time, the recall dial).
- **Partition-boundary miss** — IVF's specific recall failure: a true neighbor in an
  unprobed partition is silently missed.
- **Scalar quantization** — store dims as int8 instead of float32; ~4× memory cut, tiny
  recall hit.
- **Product Quantization (PQ)** — split a vector into sub-vectors, replace each with a
  small codebook index; large memory cut at some recall cost; the canonical big-corpus
  compression combined with IVF as `IVF-PQ`.
- **DiskANN** — disk-resident ANN index for corpora too large for RAM.

### Deep dive — Common misconceptions

- ❌ "HNSW's `efSearch` and `M` are interchangeable knobs." → ✅ `efSearch` is *query-time*
  (no rebuild) and `M` / `efConstruction` are *build-time* (require rebuild); the
  Overview's "raise efSearch without rebuild" advice depends on this distinction.
- ❌ "Product Quantization loses a little memory, like scalar." → ✅ PQ buys *dramatically*
  more memory savings than scalar quantization (often 10–60×) at a meaningfully larger
  recall hit; it is the right tool when memory is the binding constraint, not when a
  trivial recall hit matters.
- ❌ "DiskANN is just a slower HNSW." → ✅ DiskANN's distinctive feature is fitting
  *bigger-than-RAM* corpora on the same hardware by reading a small number of SSD pages
  per query; it is about scale, not speed.

### Deep dive — Worked example

Trace a single query through each index family with the mechanisms in view.

*HNSW.* Corpus is 20M vectors, `M=16`, `efSearch=64`. The query embedding arrives:
1. Enter at the top layer (a few hundred nodes, very long links). Greedily follow the
   neighbor whose vector is closest to the query, hopping across the space in 2–3 long
   steps.
2. Descend to Layer 1: shorter links, denser graph. A handful of medium-range hops
   refine the position.
3. Descend to Layer 0 (the full graph). `efSearch=64` says "maintain a dynamic candidate
   list of size 64 and keep expanding the most promising unvisited neighbors until the
   list stabilizes." Return the top-k from that list.

Total work scales like O(log N) in the corpus size — the long top-layer links are what
make that true. Raising `efSearch` to 128 simply enlarges that dynamic candidate list at
Layer 0, exploring more of the graph and recovering more true neighbors at higher cost.

*IVF-PQ.* Same corpus, `nlist=4096`, `nprobe=32`, PQ codes of 64 bytes per vector.
1. Compute the query's distance to each of the 4096 centroids (cheap — 4096 dot products).
2. Sort and pick the 32 nearest centroids; their partitions hold ~`32/4096 ≈ 0.8%` of the
   corpus, roughly 160k vectors.
3. Within those 32 partitions, compute query-to-vector distances *using the PQ codebook
   tables* (no float32 vectors to load — just byte-code lookups into precomputed
   sub-distance tables). Take the top-k.

The boundary miss is now concrete: if a true neighbor sat in the 33rd-closest partition,
it's invisible to this query. Raising `nprobe` to 128 makes the engine search 128
partitions instead of 32, recovering most of those missed neighbors at 4× the per-query
work.

### Deep dive — Check questions

1. Explain *why* HNSW achieves roughly logarithmic query-time scaling — what specifically in the index structure provides that property? — **Answer:** HNSW builds a hierarchy of layers: the top layers are sparse subsets of the corpus with long-range links; the bottom layer holds every vector with short-range links. A search starts at the top, where a few long hops cover huge regions of the space; each descent moves to a denser layer with shorter links, narrowing the region. The combination of long top-layer links covering the whole space in O(log N) hops plus dense bottom-layer refinement gives the ~logarithmic query-time scaling. Without the hierarchy you'd have just a flat dense graph, and search cost would degrade.
2. IVF can silently miss a true nearest neighbor even with a well-tuned index, in a way HNSW typically does not. Explain the mechanism — what specifically about IVF causes this failure mode — and how `nprobe` mitigates it. — **Answer:** IVF partitions the corpus by k-means into `nlist` clusters and at query time only searches the `nprobe` partitions whose centroids are nearest the query. If a true nearest neighbor sits just *across* a partition boundary — its centroid is the (`nprobe`+1)-th closest, not in the probed set — the engine never compares against it, so it is silently missed regardless of how close it actually is to the query. Raising `nprobe` searches more partitions and recovers boundary-crossing neighbors, at proportionally more work per query. HNSW's graph descent doesn't have analogous hard partitions, so it isn't subject to this specific failure mode.
3. A team is serving a 500M-vector corpus on memory-constrained hardware. Plain HNSW won't fit. Describe what `IVF-PQ` does — both the IVF part and the PQ part — and explain which trade-off (recall, latency, memory) each is buying and at what cost. — **Answer:** **IVF** clusters the corpus into `nlist` centroids via k-means and at query time only searches the `nprobe` partitions nearest the query; this trades a small recall loss (true neighbors across partition boundaries can be missed) for searching only a small fraction of the corpus — buying *latency / cost* at the cost of some *recall*. **PQ (Product Quantization)** replaces each vector with a sequence of small codebook indices (each sub-vector replaced by 1 byte pointing into a 256-entry codebook), shrinking a vector from kilobytes to tens of bytes; distance is computed via precomputed sub-distance tables. PQ buys *memory* (often 10–60× cut) at the cost of a meaningfully larger recall hit than scalar quantization. Combined as `IVF-PQ`, the two work together: IVF prunes which partitions to search, and within those partitions PQ makes the stored vectors tiny — so a 500M-vector corpus that wouldn't fit as float32 in any reasonable RAM budget becomes serveable. The cost is somewhat lower recall vs. plain HNSW, recovered partially by raising `nprobe`.

---

## 10.5 — Hybrid search — dense + sparse (BM25)

### Concept

Dense embedding search (10.2) is powerful but has a real blind spot: it matches
*meaning* and can **miss exact terms**. Embeddings tend to blur rare, out-of-distribution
strings — a specific error code `ERR_4471`, a part number `SKU-99-X`, a person's
surname, a function name `parseUserToken`, a legal citation. These tokens carry no
"semantic neighborhood"; the user wants that *exact string*, and a dense model may rank a
semantically-similar-but-wrong passage above the literal match.

**Sparse / lexical search** is the complementary tool. **BM25** (Best Matching 25) is the
classic, battle-tested ranking function used by search engines like Lucene/Elasticsearch.
It scores a document by **term frequency** (how often query terms appear) and **inverse
document frequency** (rare terms count more than common ones), with a saturation curve
(diminishing returns for repeated terms) and a length normalization (long documents
don't win just by being long). [4] BM25 excels at exactly what dense search struggles with:
**exact keyword, identifier, and rare-term matching**. Its weakness is the mirror image —
it has no notion of meaning, so a paraphrase with no shared words scores zero.

**Hybrid search** runs both — dense (semantic) and sparse (lexical) — and **combines the
results**. The two methods have **complementary failure modes**, so the union is more
robust than either alone: dense catches "reset my password" → "change credentials";
sparse catches "ERR_4471" → the exact passage. This is why hybrid search is the default
recommendation for production RAG.

**How results are fused:**
- **Reciprocal Rank Fusion (RRF)** — the standard, robust default. Each result gets a
  score of `Σ 1/(k + rank_in_that_list)` summed over the dense and sparse rankings (`k`
  is a small constant; `k = 60` is the value used in the original RRF paper and a common
  default). It uses only *ranks*, so it sidesteps the hard problem that cosine scores and
  BM25 scores are on completely different, non-comparable scales. Simple, no tuning of
  score weights, works well. [5]
- **Weighted score combination** — normalize each method's scores to a common range and
  take a weighted sum (`α · dense + (1−α) · sparse`). More tunable but requires
  normalization and choosing `α`, and is sensitive to score-distribution quirks.

RRF takes two independently-ranked lists and fuses them into one by summing each
document's rank-based scores — using ranks, not raw scores, so the incompatible-scale
problem disappears:

```
   DENSE list (semantic)        SPARSE list (BM25 / lexical)
   rank 1  doc-C                rank 1  doc-A
   rank 2  doc-A                rank 2  doc-D
   rank 3  doc-B                rank 3  doc-C
        │                            │
        └──────────┬─────────────────┘
                   ▼
          ┌──────────────────┐
          │   RRF FUSE       │   score(doc) = Σ  1/(k + rank)
          │   k = 60         │   e.g. doc-A: 1/(60+2) + 1/(60+1)
          └──────────────────┘            = 0.01613 + 0.01639 = 0.03252
                   │
                   ▼
          FUSED list (one ranking)
          rank 1  doc-A   (high in both lists)
          rank 2  doc-C
          rank 3  doc-B / doc-D ...
```

Many 2026 vector DBs support hybrid search natively in a single query — Weaviate, Milvus,
Qdrant, Pinecone, and others — so you often don't assemble it by hand. The retrieved,
fused set is typically then **reranked** (10.6) for final ordering.

A practical note: hybrid search adds an index (a sparse/BM25 index alongside the vector
index) and a fusion step — a little more infrastructure and latency — but the recall gain
on identifier-heavy or technical corpora is usually well worth it.

### Key terms

- **Sparse / lexical search** — retrieval based on exact term matching (mostly-zero term vectors).
- **BM25** — the standard lexical ranking function; term frequency × inverse document frequency, with saturation and length normalization.
- **Hybrid search** — running dense (semantic) and sparse (lexical) retrieval and merging the results.
- **Reciprocal Rank Fusion (RRF)** — merging two ranked lists using `Σ 1/(k + rank)`; rank-based, no score normalization needed.
- **Weighted score combination** — normalizing and taking a weighted sum of dense and sparse scores.
- **Complementary failure modes** — dense and sparse fail on different inputs, so combining them is more robust.

### Common misconceptions

- ❌ "Dense embeddings are strictly better than keyword search, so BM25 is obsolete." → ✅ Dense search blurs rare exact terms (error codes, IDs, names); BM25 nails them. They are complementary, and hybrid beats either alone.
- ❌ "To fuse results, just add the cosine score and the BM25 score." → ✅ Those scores are on different, non-comparable scales; RRF fuses by rank to avoid this, or you must normalize first for a weighted sum.
- ❌ "Hybrid search means running two systems and showing both result lists." → ✅ Hybrid search merges the two rankings into a single fused result set (e.g., via RRF), usually then reranked.

### Worked example

A developer-docs RAG system. Two queries expose the trade-off:

- *"How do I make my app retry failed requests?"* — conceptual. **Dense** wins: it
  retrieves the "Resilience and backoff" section even though it never uses the word
  "retry app." BM25 alone might miss it.
- *"What does error E_RATE_LIMIT_EXCEEDED mean?"* — exact identifier. **BM25** wins: it
  matches the literal token in the error reference table. Dense search may rank a
  general "rate limiting" essay higher and miss the exact entry.

**Hybrid:** run both, fuse with RRF (`k=60`). The error-code query gets the exact table
entry from the sparse list ranked high, and the conceptual query gets the resilience
section from the dense list ranked high. Recall@10 across a 120-query eval rises from
0.74 (dense only) to 0.89 (hybrid). The fused set is then reranked before going to the
LLM.

### Check questions

1. For a developer-docs corpus, predict whether dense or sparse (BM25) retrieval will win each query and say why: (a) "how do I make my service resilient to flaky network calls?" (b) "what does exit code SIGKILL_137 mean?" — **Answer:** (a) Dense wins — it is conceptual; the answer section ("backoff and retries," "resilience") may share no words with the query, and dense search matches meaning. (b) Sparse/BM25 wins — `SIGKILL_137` is a rare exact identifier with no semantic neighborhood; dense search blurs such out-of-distribution tokens and may rank a generic "exit codes" essay above the literal table entry, while BM25 nails the exact-term match.
2. A team fuses dense and sparse results by computing `0.5 × cosine_score + 0.5 × bm25_score` and gets erratic rankings. Diagnose the bug and explain why Reciprocal Rank Fusion avoids it. — **Answer:** Cosine similarity (roughly 0–1) and BM25 scores (unbounded, corpus-dependent) live on completely different, non-comparable scales, so a raw weighted sum is dominated by whichever scale happens to be larger — the 0.5/0.5 weights are meaningless. Either normalize both to a common range first, or use RRF, which fuses using only each result's *rank* in each list (`Σ 1/(k+rank)`) — ranks are scale-free, so the incompatible-score problem disappears entirely.
3. A team runs dense and sparse retrieval separately and shows users two result lists side by side, calling it "hybrid search." Why is this not hybrid search, and what is the actual benefit they are missing? — **Answer:** Hybrid search *merges* the two rankings into a single fused result set (e.g., via RRF), then typically reranks it — it does not just display both lists. The benefit they miss is the union covering *complementary failure modes*: a single fused, reranked top-k passed to the LLM contains both the semantic matches dense found and the exact-term matches sparse found. Two separate lists shifts the fusion burden onto the user and never produces the combined context the LLM needs.

---

## 10.6 — Reranking — cross-encoder rerankers

### Concept

Initial retrieval (dense, sparse, or hybrid) is built for **speed at scale**: it must
score the query against millions of chunks. To do that it uses a **bi-encoder** — the
query and each document are embedded *independently* into vectors, and similarity is a
cheap vector operation. The catch: because query and document are encoded separately,
the model never "looks at them together." It cannot reason about how a *specific* query
term relates to a *specific* document phrase. So bi-encoder retrieval is fast but
**coarse** — great at producing a roughly-relevant top-100, weaker at getting the *exact
ordering of the top few* right.

**Reranking** is a second, more precise stage. The first stage retrieves a generous
candidate set (e.g., top 50–100); the **reranker** re-scores *those* candidates against
the query and returns the best few (e.g., top 5) to send to the LLM.

The standard reranker is a **cross-encoder**. Unlike a bi-encoder, a cross-encoder takes
the **query and the document together as a single joined input** and runs them through
the model, which can apply full attention *across* both — directly judging "does this
document answer this query?" This joint processing makes it **much more accurate** at
relevance scoring. The price: it cannot be precomputed (the score depends on the
specific query-document pair), so it runs at query time, and it is far more expensive per
pair than a vector dot product — which is exactly why you only run it on the ~100
candidates the cheap first stage already narrowed down to, not the whole corpus.

The two architectures side by side — the key difference is *when* the query and document
meet:

```
  BI-ENCODER (first-stage retrieval)      CROSS-ENCODER (reranker)

   query          document                 [ query ⊕ document ]
     │               │                       (joined as one input)
     ▼               ▼                              │
  ┌─────┐         ┌─────┐                           ▼
  │ Enc │         │ Enc │                     ┌───────────┐
  └─────┘         └─────┘                     │ joint Enc │  full attention
     │               │                       │           │  across both
     ▼               ▼                       └───────────┘
   vec_q           vec_d                            │
     │               │                              ▼
     └──── cosine ────┘                         relevance score
       cheap vector op                      accurate, but query-time
                                             ONLY — cannot precompute
   doc vec is PRECOMPUTABLE
   (encoded once at index time)
```

This **two-stage retrieve-then-rerank** pattern is the standard production architecture:
a cheap, high-**recall** first stage (cast a wide net, don't miss the answer) followed by
an expensive, high-**precision** reranker (order the net's catch correctly). It is the
search analogue of "filter cheaply, then refine carefully."

**Why it matters for RAG specifically:** the LLM only sees the top few chunks. If the
right chunk was retrieved at rank 12 but you only pass the top 5, it never reaches the
model — the information was found but not *used*. A reranker's job is to pull the truly
relevant chunk from rank 12 up into the top 5. It also lets you pass *fewer* chunks
(better precision → less distracting context → cheaper, better answers).

**Options (2026):** managed reranker APIs — **Cohere Rerank**, **Voyage rerank**, plus
**Bedrock Rerank** and **Vertex Reranker** — and open-weight cross-encoders (e.g.,
`bge-reranker`, Jina rerankers, MS-MARCO MiniLM cross-encoders). Trade-offs to weigh:
reranking adds latency (one extra model call over ~100 candidates) and cost; tune how
many candidates to rerank (more candidates = better recall into the reranker, more
latency) and how many to keep. There are also faster middle-ground architectures
(**late-interaction** models like ColBERT) that get closer to cross-encoder quality at
lower cost. **LLM-as-reranker** — asking an LLM to rank the candidates — is another
option, more capable but slower and pricier than a dedicated cross-encoder.

### Key terms

- **Bi-encoder** — encodes query and document separately into vectors; fast, used for first-stage retrieval; coarse.
- **Cross-encoder** — processes query and document jointly with full cross-attention; accurate relevance scorer; expensive, query-time only.
- **Reranking** — a second stage that re-scores a retrieved candidate set for precise final ordering.
- **Retrieve-then-rerank** — the two-stage pattern: cheap high-recall retrieval, then expensive high-precision reranking.
- **Recall vs. precision (in retrieval)** — recall = did we fetch the right item at all; precision = is it ranked near the top.
- **Late interaction (ColBERT)** — a middle-ground architecture cheaper than a cross-encoder, more accurate than a bi-encoder.

### Common misconceptions

- ❌ "Reranking replaces vector search." → ✅ Reranking is a second stage *on top of* retrieval; it re-scores the candidates retrieval found — it can't run over the whole corpus because cross-encoders are too expensive.
- ❌ "A cross-encoder is just a better embedding model." → ✅ A cross-encoder produces no reusable embedding; it scores a specific query-document pair jointly and must run at query time per pair.
- ❌ "If retrieval recall is good, you don't need a reranker." → ✅ Retrieval can fetch the right chunk at rank 12, but if you pass only the top 5 to the LLM it never gets used; the reranker fixes ordering so the right chunk reaches the model.

### Worked example

A RAG support bot. First-stage hybrid retrieval over 2M chunks returns the top 50 for the
query *"Can I get a refund after 30 days if the item is defective?"*

- Among the 50, the chunk that *actually* answers it ("Defective items qualify for a
  full refund regardless of the 30-day window — §4.2") sits at **rank 14** — bi-encoder
  similarity ranked several generic "refund policy" chunks above it.
- The system passes only the top 5 to the LLM. Without reranking, chunk 14 never
  reaches the model → the bot answers "no, the 30-day window has passed" — wrong.
- **With a cross-encoder reranker:** the 50 candidates are re-scored jointly with the
  query. The cross-encoder sees the interaction between "defective" and "regardless of
  the 30-day window" and lifts §4.2 to **rank 2**. Now it is in the top 5, the LLM reads
  it, and the bot answers correctly.

Net: same retrieval, same LLM — the reranker turned a wrong answer into a right one by
fixing ordering.

### Check questions

1. A team proposes replacing their bi-encoder vector search with a cross-encoder run over the whole 3M-chunk corpus "because cross-encoders are more accurate." Explain why this is architecturally impossible at that scale, rooted in how a cross-encoder produces its score. — **Answer:** A cross-encoder scores a *specific query-document pair* by processing them jointly with cross-attention — the score is not a reusable property of the document, so it *cannot be precomputed* the way a bi-encoder embedding can. That means a cross-encoder must run once per query-document pair at query time; over 3M chunks that is 3M expensive model calls per query — infeasible. Its accuracy is exactly why you reserve it for the ~100 candidates a cheap first stage already narrowed to.
2. In a retrieve-then-rerank pipeline, the first stage is tuned for *recall* and the reranker for *precision*. Explain why that division of labor is the right one — what goes wrong if you instead optimized stage 1 for precision? — **Answer:** Stage 1 must cast a wide net cheaply: its job is to *not miss* the answer-bearing chunk (recall), because anything it drops is gone forever — the reranker can only reorder what stage 1 returned. If stage 1 were tuned for precision (a small, tightly-ranked set), it would risk excluding the correct chunk before the reranker ever sees it. Precision is the reranker's job: it is expensive but accurate, so it refines the *ordering* of the wide net's catch. Cheap-and-wide then expensive-and-sharp.
3. A RAG system has excellent retrieval recall — the correct chunk is almost always somewhere in the retrieved top-50 — yet answers are often wrong. The team passes only the top 5 chunks to the LLM. Explain how a reranker fixes this *without changing retrieval at all*. — **Answer:** Recall being fine means the right chunk is in the top-50 but not necessarily in the top-5 the LLM actually sees — information was *found but not used*. A cross-encoder reranker re-scores the 50 candidates by judging each jointly against the query and lifts the truly relevant chunk into the top-5, so it reaches the model. Same retrieval, same LLM — fixing the *ordering* turns a wrong answer into a right one.

---

## 10.7 — RAG failure modes

### Concept

RAG has many moving parts; failures are common and it pays to diagnose them by *stage*.
Group them into **retrieval failures** (the wrong context was found) and **generation
failures** (the right context was found but the model misused it).

**Retrieval failures:**
- **Retrieval miss (recall failure)** — the answer-bearing chunk is never retrieved.
  Causes: bad chunking split the answer, vocabulary mismatch between query and document,
  low ANN recall, the document isn't indexed, or an over-aggressive metadata filter
  excluded it. If the chunk isn't retrieved, the LLM has no chance.
- **Low precision / distracting context** — relevant chunks *are* retrieved but mixed
  with irrelevant ones. Extra irrelevant context is not harmless — it actively distracts
  the model, dilutes attention, and can pull the answer off-course. More retrieved
  chunks is not always better.
- **Stale or conflicting sources** — the index contains an outdated document, or two
  documents disagree; the model may surface the wrong one or average them.
- **Chunk fragmentation** — the answer is split across chunks and no single retrieved
  chunk is complete.

**Generation failures (right context retrieved):**
- **Ignoring the context** — the model overrides the retrieved evidence with its
  parametric (memorized) knowledge, especially when the context contradicts what it
  "believes." The prompt must explicitly instruct it to prefer the provided context.
- **Hallucination despite context** — the model invents details not in the passages, or
  over-generalizes from them.
- **"Lost in the middle"** — when many chunks are stuffed into a long context, models
  tend to attend well to the start and end but more poorly to the middle; a relevant
  chunk buried in the middle can get under-used. Fewer, reranked chunks and putting the
  best chunk near the edges mitigate this. (This was characterized in the "Lost in the
  Middle" study; the strength of the effect varies by model and has lessened in some
  newer long-context models, so treat it as a known risk to test for, not a fixed law.) [6]
- **Unfaithful citation** — the model cites a source that doesn't actually support the
  claim, or cites the wrong one.

**Systemic failure modes:**
- **No-answer handling** — when retrieval genuinely finds nothing relevant, a good system
  must *abstain* ("I don't have information on that") rather than answer from a weak or
  empty context. Many naive RAG systems always answer, hallucinating from junk.
- **Over-retrieval / always-retrieve** — retrieving for queries that need no retrieval
  ("thanks!", "rephrase that") wastes tokens and can inject irrelevant noise — motivates
  agentic RAG (10.8).
- **Index drift** — the corpus changes but the index isn't updated; answers go stale.

**The diagnostic discipline:** when a RAG answer is wrong, do not just blame "the LLM."
Check the retrieval first — *was the correct chunk even in the context?* If no → fix
retrieval (chunking, embeddings, hybrid, recall). If yes → fix generation (prompt,
reranking, context length, abstention). This requires **observability**: log the
retrieved chunks for every query. Evaluate retrieval and generation **separately** with
their own metrics.

The single most important diagnostic question splits every RAG failure into two classes:

```
                    ┌─────────────────────┐
                    │  Wrong answer?      │
                    └─────────┬───────────┘
                              │
                              ▼
            ┌─────────────────────────────────────┐
            │ Was the right chunk IN the context?  │
            │ (log the retrieved chunks to check)  │
            └───────────┬─────────────┬────────────┘
                        │             │
                   NO ◄─┘             └─► YES
                        │             │
                        ▼             ▼
            ┌───────────────────┐  ┌────────────────────┐
            │ RETRIEVAL FAILURE │  │ GENERATION FAILURE │
            │ fix: chunking,    │  │ fix: prompt to     │
            │ embeddings,       │  │ prefer context,    │
            │ hybrid, recall,   │  │ reranking, shorten │
            │ metadata filters  │  │ /reorder context,  │
            │                   │  │ citations          │
            └───────────────────┘  └────────────────────┘
```

**Retrieval metrics — what they measure and when to use each.** All assume you have, per
query, a labeled set of which chunks are actually relevant.

- **recall@k** — of the chunks that *are* relevant, what fraction appear in the top-k
  retrieved? It answers "did we *find* the answer at all?" — the metric the cheap
  first-stage retriever must maximize.
- **precision@k** — of the top-k retrieved, what fraction are relevant? It answers "is the
  context clean, or padded with distractors?"
- **MRR (Mean Reciprocal Rank)** — for each query, find the rank of the *first* relevant
  result and take its reciprocal (`1/rank`: rank 1 → 1.0, rank 2 → 0.5, rank 5 → 0.2, not
  found → 0); MRR is the mean of that over all queries. **Intuition:** it rewards putting
  *a* correct chunk high, and only cares about the first one — the natural metric when a
  query has essentially one right answer and you want it near the top.
- **NDCG (Normalized Discounted Cumulative Gain)** — the metric for when *several* chunks
  are relevant, possibly with *graded* relevance (highly relevant vs. somewhat relevant).
  It works in two steps. **DCG** sums the relevance of each retrieved result, each
  *discounted* by how far down it sits — a standard discount is `relevance / log2(rank+1)`,
  so a relevant hit at rank 1 counts fully but the same hit at rank 8 counts much less.
  Then DCG is **normalized**: divide by the *ideal* DCG (the DCG of the perfect ranking,
  best results first), giving a 0–1 score where 1.0 means "perfectly ordered." **Intuition:**
  NDCG rewards putting *more, and more-relevant* chunks *higher* — the right metric for
  multi-relevant retrieval and for evaluating rerankers, where the *ordering* of several
  good chunks is what matters.

In short: recall@k = did we get it; precision@k = is the set clean; MRR = is the one
right answer near the top; NDCG = are the several relevant results ordered well.

The four retrieval metrics side by side:

| Metric | Question it answers | Cares about rank order? | Use when |
|---|---|---|---|
| recall@k | Did we *find* the relevant chunks at all (in the top-k)? | No — only presence in the top-k | Tuning the cheap high-recall first stage; making sure the answer isn't missed |
| precision@k | Of the top-k retrieved, what fraction are relevant? | No — only the fraction relevant | Checking the context is clean, not padded with distractors |
| MRR | How high is the *first* relevant result? | Yes — but only the first hit | Queries with essentially one right answer; want it near the top |
| NDCG | Are *more* and *more-relevant* chunks placed *higher*? | Yes — full ordering, with graded relevance | Multi-relevant retrieval; evaluating rerankers |

For **generation**, use faithfulness, answer relevance, and correctness — measured with
an LLM judge or a framework like RAGAS, which provides reference-free metrics such as
faithfulness and answer relevancy. [7]

### Key terms

- **Retrieval miss** — the answer-bearing chunk is never retrieved (a recall failure).
- **Distracting context** — irrelevant retrieved chunks that degrade the model's answer.
- **Ignoring the context** — the model answering from parametric memory instead of the provided passages.
- **Lost in the middle** — degraded use of information placed in the middle of a long context.
- **Abstention / no-answer handling** — the system declining to answer when retrieval finds nothing relevant.
- **Faithfulness** — whether the answer's claims are actually supported by the retrieved context.
- **Index drift** — the index falling out of sync with the changing source corpus.
- **recall@k / precision@k** — fraction of relevant chunks found in the top-k / fraction of the top-k that are relevant.
- **MRR (Mean Reciprocal Rank)** — the mean of `1/rank` of the *first* relevant result per query; rewards getting the one right answer near the top.
- **NDCG (Normalized Discounted Cumulative Gain)** — a rank-discounted, ideal-normalized (0–1) retrieval metric; rewards placing more and more-relevant chunks higher.

### Common misconceptions

- ❌ "If a RAG answer is wrong, the LLM is to blame." → ✅ Most RAG errors are retrieval failures; always check whether the correct chunk was even in the context before blaming generation.
- ❌ "Retrieving more chunks improves answers." → ✅ Extra irrelevant chunks distract the model and worsen answers; precision matters, and "lost in the middle" makes very long contexts counterproductive.
- ❌ "A RAG system should always produce an answer." → ✅ When retrieval finds nothing relevant, the correct behavior is to abstain; answering anyway hallucinates from junk context.

### Worked example

A RAG bot answers *"What is our 2025 Q3 revenue?"* with a confidently wrong number.
Debugging by stage:

1. **Log the retrieved chunks.** They are all from the *2024* annual report — the 2025
   Q3 report was uploaded but the index was never rebuilt (**index drift**) — and the
   chunks don't even contain the answer.
2. **Diagnosis = retrieval failure**, specifically index drift causing a retrieval miss.
   The LLM was never given the right context; no prompt tweak could have fixed it.
3. **Fixes:** rebuild the index on document upload; add a date-metadata filter; and add
   an **abstention** instruction — "if the passages don't contain the answer, say you
   don't have that information" — so that a future miss yields "I don't have 2025 Q3
   data" instead of a hallucinated number.

Contrast: if the logs *had* shown the correct 2025 Q3 chunk in context and the model
still answered wrong, the diagnosis would be a **generation failure** — fix the prompt,
add reranking, or shorten the context.

### Check questions

1. A RAG answer is wrong. You log the retrieved chunks and find the correct answer-bearing chunk *was* in the context. Which class of failure is this, which class is now ruled out, and where do you look for the fix? — **Answer:** This is a *generation* failure — the right context was retrieved but the model misused it (ignored it in favor of parametric memory, hallucinated, lost-in-the-middle, or cited unfaithfully). A *retrieval* failure is ruled out, because the chunk was present. The fix lies in generation: prompt the model to prefer the provided context, add reranking, shorten/reorder the context, or enforce citations — not in chunking or embeddings.
2. A team "fixes" a RAG quality problem by raising top-k from 5 to 25, reasoning "more context can't hurt." Give two distinct mechanisms by which this can actually *worsen* answers. — **Answer:** (1) Low precision / distraction: the extra 20 chunks are mostly irrelevant; irrelevant context actively dilutes the model's attention and can pull the answer off-course. (2) "Lost in the middle": stuffing many chunks into a long context means the model attends well to the start and end but under-uses the middle, so a relevant chunk buried mid-context is effectively wasted. More retrieved chunks is not more usable information.
3. A naive RAG bot always produces an answer, even for questions its corpus genuinely has no information about. Why is "always answer" a failure mode, and what behavior should replace it? — **Answer:** When retrieval finds nothing relevant, "always answer" forces the model to generate from weak or empty context — it hallucinates a plausible but unsupported answer, which is worse than no answer in high-stakes settings. The correct behavior is *abstention*: detect that retrieval returned nothing relevant and explicitly say "I don't have information on that," typically via a prompt instruction to abstain when the passages don't contain the answer.
4. Two retrieval setups for a FAQ system, where each question has essentially one correct chunk: Setup A retrieves that chunk at rank 1 for most queries; Setup B retrieves it but usually around rank 4–5. Both have identical recall@10. Which metric distinguishes them, and what does it reward? — **Answer:** recall@10 is identical because both *find* the chunk within the top 10 — recall ignores position. **MRR** distinguishes them: it averages `1/rank` of the first relevant result, so Setup A (rank ~1 → reciprocal ~1.0) scores far higher than Setup B (rank ~4–5 → reciprocal ~0.2–0.25). MRR rewards getting the one right answer *near the top* — which matters here because the LLM only sees the highest-ranked chunks. (NDCG would also separate them, but MRR is the natural metric for one-right-answer queries.)
5. You are evaluating a *reranker* over queries that each have several relevant chunks of differing relevance (some highly relevant, some marginally). Why is NDCG a better fit than recall@k or MRR? — **Answer:** recall@k only asks whether relevant chunks are in the top-k — it ignores ordering, which is exactly what a reranker exists to fix. MRR only looks at the *first* relevant result, ignoring the other relevant chunks. NDCG handles both gaps: it credits *every* relevant chunk, weights it by *graded* relevance, *discounts* it by how far down it sits, and normalizes against the ideal ordering — so it directly measures whether more and more-relevant chunks are placed higher, which is the reranker's job.

---

## 10.8 — Agentic RAG

### Concept

**Classic ("naive") RAG** is a fixed, single-shot pipeline: every query → embed → retrieve
top-k → generate. It is simple and predictable but rigid, and that rigidity causes real
problems:
- It **retrieves even when no retrieval is needed** ("thanks!", "summarize that"),
  wasting tokens and injecting noise.
- It does **one retrieval** with the user's raw query — if that query is poorly phrased,
  ambiguous, or multi-part, the single shot fails and there is no recovery.
- It cannot handle **multi-hop** questions that require retrieving, reading, then
  retrieving again based on what was learned.
- It cannot choose **which source** to query or combine sources.

**Agentic RAG** makes retrieval a **tool the LLM controls** inside an agentic loop
(Topic 8), rather than a hardwired preprocessing step. The model *decides*: whether to
retrieve at all, what query to issue, which source/index to hit, whether the results are
sufficient, and whether to retrieve again. Retrieval becomes a deliberate action in a
think → act → observe loop.

Capabilities this unlocks:
- **Retrieval routing / conditional retrieval** — the model skips retrieval for chit-chat
  or queries it can answer directly; retrieves only when external knowledge is needed.
- **Query rewriting / decomposition** — instead of retrieving on the raw query, the
  model rewrites it into a better search query, or splits a multi-part question into
  sub-queries and retrieves for each.
- **Multi-hop retrieval** — retrieve, read, form a follow-up question from what was
  found, retrieve again. ("Who is the CEO of the company that acquired X?" → find the
  acquirer → then find its CEO.)
- **Source selection** — route to the right index/tool: product docs vs. code repo vs.
  ticketing system vs. live web search.
- **Self-correction / iterative retrieval** — the model evaluates whether retrieved
  context actually answers the question and, if not, reformulates and retries
  (the idea behind patterns like Self-RAG [8] and Corrective RAG / CRAG [9]).

**Trade-offs.** Agentic RAG is more capable and robust but: it costs **more LLM calls and
latency** (each loop iteration is a model call), it is **less predictable** (the model
may retrieve too much, too little, or loop), and it is **harder to evaluate** — you now
have a *trajectory* (which retrievals, in what order) to assess, not just a final answer
(trajectory eval vs. outcome eval, Topic 8/9). It inherits all agent failure modes:
loops, goal drift, budget overruns. So it needs the same harness controls — step limits,
retrieval-call budgets, guardrails.

**When to use which.** Use **classic RAG** for high-volume, latency-sensitive,
predictable single-fact lookups (most FAQ-style bots) — it is cheaper and simpler. Use
**agentic RAG** for complex, multi-step, multi-source questions, ambiguous queries, or
research-style tasks where one fixed retrieval is not enough. As with all agent design,
start with the simplest thing that works (classic RAG) and add agency only where the
task genuinely needs it.

**A note on structured retrieval and GraphRAG.** Everything above retrieves over a flat
pool of text chunks. A complementary line of work retrieves over **structured
representations** of the corpus instead. The best-known is **GraphRAG**: an indexing-time
LLM pass extracts *entities* and *relationships* from the documents into a **knowledge
graph**, often with hierarchical community summaries; at query time the system retrieves
*subgraphs* or those summaries rather than (or alongside) raw chunks. The motivation is
that flat chunk retrieval is weak at **global, synthesizing questions** ("what are the
main themes across all these reports?") and at **multi-hop relational** questions, where
the answer is spread across many documents connected by entities — a vector search of
isolated chunks never assembles that whole. Related approaches retrieve from existing
structure directly: **text-to-SQL / text-to-query** over a relational database, or
querying an existing knowledge base. The trade-off is real: building and maintaining a
graph is a substantial extra indexing cost and complexity, and for ordinary local
fact-lookup questions plain chunk-based RAG is simpler and usually sufficient. Treat
GraphRAG and structured retrieval as the right reach when the corpus is highly
interconnected and the questions are global or relational — not as a default.

### Key terms

- **Classic / naive RAG** — a fixed single-shot pipeline: always embed → retrieve top-k → generate.
- **Agentic RAG** — retrieval exposed as a tool the LLM controls within an agentic loop.
- **Conditional retrieval / routing** — the model deciding whether and where to retrieve.
- **Query rewriting / decomposition** — transforming or splitting the query for better retrieval.
- **Multi-hop retrieval** — chained retrievals where each step's question depends on the previous step's findings.
- **Iterative / corrective retrieval** — the model judging sufficiency of retrieved context and retrying if inadequate.
- **GraphRAG / structured retrieval** — retrieving over a knowledge graph or other structured representation instead of (or alongside) flat text chunks; stronger for global and multi-hop relational questions.

### Common misconceptions

- ❌ "Agentic RAG is just better RAG; always use it." → ✅ It is more capable but costs more calls/latency, is less predictable, and harder to evaluate; classic RAG is the right choice for simple, high-volume, latency-sensitive lookups.
- ❌ "In agentic RAG, the pipeline still always retrieves." → ✅ The model decides whether to retrieve at all — it can skip retrieval for queries it can answer directly.
- ❌ "Agentic RAG only changes the retrieval step." → ✅ It turns RAG into a full agentic loop with all the agent concerns — trajectory evaluation, step/budget limits, loop and drift failure modes.
- ❌ "RAG always retrieves over flat text chunks." → ✅ Structured retrieval — notably GraphRAG over a knowledge graph — retrieves over structured representations and is stronger for global, synthesizing, and multi-hop relational questions; the cost is a heavier indexing pipeline, so it is a targeted reach, not a default.

### Worked example

User: *"Did the company that bought our main competitor last year also lay off staff
recently, and how does that compare to our headcount?"*

- **Classic RAG:** embeds the whole sentence, retrieves top-5 chunks once. It gets a
  blurry mix and likely answers incompletely — this is a 3-hop, 2-source question.
- **Agentic RAG:** the model runs a loop:
  1. *Decompose:* sub-question A — "who acquired [competitor] last year?"
  2. *Retrieve* (news index) → "Acquirer = NorthCo."
  3. *Retrieve* (news index) — "Did NorthCo announce layoffs recently?" → "Yes, 8%."
  4. *Route to a different source:* sub-question — "our current headcount?" →
     *retrieve* (internal HR index) → "1,200."
  5. *Synthesize:* compare and answer, with citations to each source.

Each hop's query was formed from the previous hop's result — impossible for a single
fixed retrieval. The cost: ~4 model calls and ~3 retrievals instead of one, and a
trajectory that must be evaluated, not just the final answer.

### Check questions

1. A classic-RAG FAQ bot embeds and retrieves for *every* incoming message, including "thanks, that helped!" and "can you rephrase that?". Name the failure mode this exhibits and the agentic-RAG capability that eliminates it. — **Answer:** This is over-retrieval / always-retrieve — retrieving for queries that need no external knowledge wastes tokens and injects irrelevant noise into the context. Agentic RAG eliminates it via conditional retrieval / routing: retrieval becomes a tool the model *decides* whether to invoke, so it skips retrieval for chit-chat or anything it can answer directly.
2. The question "Who founded the company that acquired Instagram?" fails in classic RAG no matter how good the embeddings or vector DB are. Explain why this is structurally unfixable in classic RAG, and what agentic capability is required. — **Answer:** It is a multi-hop question: you cannot retrieve the founder until you first know *which* company acquired Instagram. Classic RAG does exactly *one* retrieval on the raw query — there is no mechanism to read a result and form a follow-up query, so no embedding/index quality can fix it. It requires multi-hop retrieval: retrieve the acquirer, then form a second query from that result to retrieve its founder.
3. A team replaces classic RAG with agentic RAG everywhere, including a high-volume single-fact FAQ bot, and is surprised by rising cost, latency, and unpredictable behavior. Explain the trade-off they ignored and when classic RAG is the right choice. — **Answer:** Agentic RAG turns retrieval into an agentic loop — each iteration is an extra LLM call, so it costs more calls/latency, behaves less predictably (over/under-retrieval, loops), and is harder to evaluate (you must assess the retrieval *trajectory*, not just the final answer). For high-volume, latency-sensitive, predictable single-fact lookups, classic RAG is cheaper and simpler and is the right choice; add agency only where the task genuinely needs multi-step or multi-source reasoning.
4. A team's RAG bot answers specific local lookups well, but users complain it cannot answer "what are the recurring themes across all our incident reports?" Explain why flat chunk retrieval struggles with this question, and what retrieval approach is designed for it. — **Answer:** A global, synthesizing question has no single answer-bearing chunk — the answer is distributed across many documents and emerges only from connecting them. Flat vector retrieval fetches the top-k *individually* most similar chunks, which never assembles a corpus-wide picture, so it returns a partial, local view. **GraphRAG / structured retrieval** is designed for this: an indexing-time pass extracts entities and relationships into a knowledge graph with hierarchical community summaries, so the system can retrieve those summaries or subgraphs and answer global and multi-hop relational questions. The cost is a heavier indexing pipeline, so it is a targeted choice for interconnected corpora, not a default.

---

## 10.9 — RAG vs. long-context vs. fine-tuning — the decision framework

### Concept

When a model "doesn't know something," there are three main remedies. Confusing them is
a classic interview failure. The clean mental model: **RAG and long-context supply
knowledge; fine-tuning changes behavior.**

**RAG** — retrieve relevant text at query time, put it in the context.
- *Best for:* large, changing, or private knowledge corpora; when you need citations and
  verifiability; when access control matters; when the corpus is far too big to fit in
  any context window.
- *Strengths:* knowledge updates instantly (re-index, no training); auditable, editable
  sources; citations; cost-efficient (only relevant chunks enter the prompt).
- *Weaknesses:* retrieval can miss; pipeline complexity (chunking, embeddings, vector DB,
  reranking); adds retrieval latency; doesn't teach skills or style.

**Long-context** — skip retrieval; just put the relevant documents (or the whole small
corpus) directly into a model with a large context window.
- *Best for:* a corpus small enough to fit; tasks needing holistic reasoning across a
  *whole* document (a contract, a codebase, a long report) where chunking would break
  the cross-references; simplicity over infrastructure.
- *Strengths:* dead simple — no pipeline; the model sees everything, no retrieval miss;
  better at cross-document synthesis within what fits.
- *Weaknesses:* hard limited by the window size; **expensive** — you pay for every input
  token on every call (prompt caching helps for a stable prefix); **slower** TTFT for
  huge inputs; quality degrades on very long contexts ("lost in the middle," context
  rot); no built-in citations.

**Fine-tuning** — further-train the model's weights on task data (Topic 14).
- *Best for:* changing *behavior, format, style, tone*, or a specialized skill; teaching
  a consistent output structure; baking in domain reasoning patterns; reducing prompt
  length / latency by internalizing instructions.
- *Strengths:* changes how the model behaves, not just what it's told; can shrink prompts;
  can improve a narrow task beyond what prompting achieves.
- *Weaknesses:* does **not** reliably inject *factual knowledge* — facts learned via
  fine-tuning are unreliable and prone to hallucination, and the knowledge is frozen at
  fine-tune time; needs a curated training set; a training pipeline; must be redone when
  the base model updates; no citations.

**The decision rule:**
- Need *current, private, or large-corpus facts* with verifiability → **RAG**.
- The relevant content is *small enough to fit* and you want simplicity / whole-document
  reasoning → **long-context** (often with prompt caching).
- Need a *consistent behavior, format, tone, or skill* the model can't be prompted into
  → **fine-tuning**.
- *"I need the model to know our 50,000 support articles and stay current"* → **RAG**,
  not fine-tuning (a near-universal interview trap — fine-tuning is the wrong tool for
  bulk dynamic knowledge).

The three approaches side by side:

| Approach | Supplies knowledge or changes behavior? | Best for | Key strength | Key weakness | Citations? |
|---|---|---|---|---|---|
| RAG | Supplies knowledge | Large, changing, private, access-controlled corpora needing verifiability | Instant updates, auditable/editable sources, per-token cost efficiency | Retrieval can miss; pipeline complexity; retrieval latency; teaches no skills | Yes |
| Long-context | Supplies knowledge | A corpus small enough to fit; whole-document holistic reasoning | Dead simple — no pipeline; no retrieval miss; cross-document synthesis | Hard window limit; expensive per token; slower TTFT; "lost in the middle" | No |
| Fine-tuning | Changes behavior | Consistent format, tone, style, or a specialized skill | Changes how the model behaves; can shrink prompts | Unreliable for facts; knowledge frozen at fine-tune time; needs training pipeline; redo on base-model update | No |

**They are not mutually exclusive — combine them.** A common production stack:
fine-tune for the domain's tone and output format + RAG for current factual grounding +
a long context to hold the retrieved chunks. The frameworks question — RAG vs. long-
context — is also shifting: as context windows grow and prompt caching cheapens stable
prefixes, "just put the docs in context" wins more often for *moderate* corpora; but for
large, dynamic, access-controlled knowledge bases that need citations, RAG remains the
right tool. Decide by corpus size, update frequency, citation needs, cost, and latency —
not by fashion.

### Key terms

- **Long-context approach** — placing the relevant documents directly in a large context window instead of retrieving.
- **Fine-tuning** — further-training a model's weights on task-specific data (changes behavior).
- **Knowledge vs. behavior** — the core distinction: RAG/long-context supply knowledge; fine-tuning changes behavior.
- **Decision framework** — choosing among RAG / long-context / fine-tuning by corpus size, update frequency, citations, cost, latency.

### Common misconceptions

- ❌ "Fine-tuning is how you give a model your company's knowledge." → ✅ Fine-tuning changes behavior/style/skills; it is unreliable for injecting factual knowledge and the facts are frozen and hallucination-prone. For bulk dynamic knowledge, use RAG.
- ❌ "Long context made RAG obsolete." → ✅ Long context wins for small corpora and whole-document reasoning, but RAG still wins for large, dynamic, access-controlled corpora needing citations and per-token cost efficiency. Decide by the corpus and requirements.
- ❌ "You must pick exactly one." → ✅ They compose — e.g., fine-tune for format/tone, RAG for current facts, long context to hold the retrieved evidence.

### Worked example

Three requests, three correct answers:

1. *"The bot should answer from our 80,000 constantly-updated help articles, with a link
   to the source."* → **RAG.** The corpus is huge, changes daily, and citations are
   required — fine-tuning can't stay current and can't cite; the corpus won't fit a
   context window.
2. *"The bot should review one uploaded 60-page contract and answer questions about
   clause interactions."* → **Long-context.** One document, fits the window, and the
   questions need reasoning *across the whole contract* — chunking would sever the
   cross-references RAG would miss.
3. *"The bot keeps replying in long paragraphs; it must always answer in our terse,
   bulleted house style with a fixed JSON envelope."* → **Fine-tuning** (or strong
   prompting first; fine-tune if prompting is insufficient). This is a *behavior/format*
   problem, not a knowledge problem.

A real product might need all three: fine-tune the tone, RAG the help articles, and use
a long context window to hold the retrieved chunks plus an uploaded document.

### Check questions

1. Classify each as primarily a *knowledge* problem or a *behavior* problem, and name the right tool: (a) the bot must always reply in a terse, bulleted house style; (b) the bot must answer questions about a product launched after the model's training cutoff; (c) the bot must adopt a specific clinical reasoning pattern it cannot be prompted into. — **Answer:** (a) Behavior — output style/format → fine-tuning (try strong prompting first). (b) Knowledge — post-cutoff facts → RAG (or long-context if the docs fit). (c) Behavior — a reasoning skill → fine-tuning. The driving distinction: RAG/long-context supply *knowledge* (what the model is told); fine-tuning changes *behavior* (how the model acts).
2. A team fine-tunes a model on a snapshot of their constantly-updated pricing catalog so it "knows the prices." Within a month customers get wrong prices. Give the two distinct reasons fine-tuning failed here. — **Answer:** (1) Fine-tuning injects *facts* unreliably — the model is prone to hallucinating or mixing up specific values like prices even right after training. (2) Fine-tuned knowledge is *frozen at training time*: a constantly-updated catalog means the snapshot is stale within days, and the only way to refresh it is another full fine-tune. (Also: no citations to verify.) Bulk dynamic factual knowledge belongs in RAG, which re-indexes instantly.
3. A team must answer detailed questions about clause interactions within a single uploaded 70-page contract. Argue why long-context can be the better choice than chunked RAG here — and identify the one corpus property that, if it changed, would flip the answer back to RAG. — **Answer:** The contract fits in a large context window, and the task needs *holistic reasoning across the whole document* — clause interactions span sections, and chunking would sever the cross-references RAG retrieves piecemeal. Long-context lets the model see everything at once, with no retrieval miss and no pipeline. The property that would flip it back to RAG: corpus *size* — if instead of one 70-page contract it were thousands of contracts (or a huge, dynamic, access-controlled corpus needing citations), it would no longer fit and RAG would again be required.

---

## Topic 10 — Exam Question Bank

A pool the tutor draws from for the gated exam. Mixed-format, scored out of 100, pass
mark 85.

### True / False

1. After adding a document to a RAG system's corpus and re-indexing it, the underlying model's weights have been updated. — **Answer:** False. RAG only adds the document to the retrieval index; the model's weights are never touched. Updating weights would be fine-tuning — RAG supplies knowledge through the context window, not the weights.
2. If a query and a passage share no words at all, their cosine similarity must be low. — **Answer:** False. Embeddings encode meaning, not words; a query and a passage that mean the same thing land near each other in vector space and have high cosine similarity even with zero shared words.
3. If retrieval precision is poor, increasing chunk size is a reliable fix because each chunk then carries more context. — **Answer:** False. Larger chunks mix multiple topics, so the embedding becomes a blurry average that matches no query sharply — often *worsening* precision — and pad the context with distracting text. Chunk size is a trade-off, not a one-way dial.
4. A vector database returning the "top 10 nearest" chunks is guaranteed to return the 10 truly closest vectors in the corpus. — **Answer:** False. Vector DBs use Approximate Nearest Neighbor search, which trades a tunable amount of recall for speed; it can miss some true neighbors. Exact nearest-neighbor search doesn't scale to large corpora.
5. BM25 will reliably retrieve a paraphrased query even when the query and the answer passage share no vocabulary. — **Answer:** False. BM25 is a lexical ranking function with no notion of meaning; with no shared terms it scores the passage near zero. Matching paraphrases is the job of dense embedding search — this is why the two are combined in hybrid search.
9. [deep-dive] An embedding model trained contrastively on general web Q&A pairs will be equally accurate at retrieval on any specialized corpus, such as legal filings. — **Answer:** False. Contrastive training only teaches the model the "related" relationships present in its training pairs; it never learned what relevance means for specialized domains, so its vector space organizes that jargon poorly. Fine-tuning on in-domain query→document pairs is what lifts recall on a specialized corpus.
6. Because a cross-encoder is more accurate than a bi-encoder, you should precompute cross-encoder document representations at index time for fast retrieval. — **Answer:** False. A cross-encoder produces no reusable per-document representation — it scores a specific query-document *pair* jointly, so the score depends on the query and cannot be precomputed. It must run at query time over a small candidate set.
7. A well-designed RAG system should always return an answer, since returning "I don't know" looks like a failure to users. — **Answer:** False. When retrieval finds nothing relevant, the correct behavior is abstention; forcing an answer from weak or empty context produces a confident hallucination, which is worse than an honest "I don't have that information."
8. For a knowledge base of 200,000 articles that is updated several times a day, fine-tuning the model on the articles is the recommended approach. — **Answer:** False. Fine-tuning injects facts unreliably and freezes them at training time, so a frequently-updated corpus goes stale immediately and gives no citations. RAG keeps such bulk dynamic knowledge current, auditable, and citable.

### Multiple Choice

1. A RAG system over a hardware-parts catalog answers conceptual questions well but fails when users search for exact part numbers like `BRK-22-9F`. The single most appropriate fix is to:
   A) Increase the embedding dimension  B) Add sparse/BM25 retrieval and fuse it with the dense results (hybrid search)  C) Raise the dense-search top-k  D) Switch to a larger LLM
   — **Answer:** B. Exact rare identifiers carry little semantic neighborhood, so dense search blurs them; BM25 matches the literal token. Hybrid search fuses both so exact-term queries and conceptual queries both succeed. A bigger embedding/LLM or larger top-k does not address the lexical-matching gap.
2. A team fuses dense and sparse results with `final = cosine_score + bm25_score` and gets unstable rankings. Why does Reciprocal Rank Fusion avoid this problem?
   A) RRF averages the two embedding vectors before scoring  B) RRF uses only each result's rank in each list, which is scale-free, sidestepping the non-comparable score scales  C) RRF always takes the higher of the two scores  D) RRF retrains the embedding model on both signals
   — **Answer:** B. Cosine and BM25 scores are on different, non-comparable scales, so a raw sum is dominated by the larger scale. RRF combines `Σ 1/(k+rank)` using ranks only, which removes the scale problem.
3. Your HNSW-backed retrieval has slightly insufficient recall, and you cannot afford an index rebuild before launch. Which change raises recall now?
   A) Increase `M` (links per node)  B) Increase `efConstruction`  C) Increase `efSearch`  D) Decrease `nlist`
   — **Answer:** C. `efSearch` is a *query-time* knob — raising it makes each query search the existing graph harder, raising recall at some latency cost. `M` and `efConstruction` are *build-time* parameters that would require a rebuild; `nlist` is an IVF parameter.
4. In a retrieve-then-rerank pipeline, stage 1 is tuned for recall and the reranker for precision. What goes wrong if stage 1 is instead tuned for precision (a small, tight result set)?
   A) The reranker becomes too slow  B) The correct chunk may be excluded before the reranker ever sees it, and the reranker can only reorder what stage 1 returned  C) Cross-encoder scores stop being comparable  D) The embedding model must be retrained
   — **Answer:** B. The reranker can only reorder the candidates retrieval handed it; if a precision-tuned stage 1 drops the answer-bearing chunk, no reranking can recover it. Stage 1's job is to not miss (recall); the reranker supplies precision.
5. A RAG answer is wrong, and your logs show the correct answer-bearing chunk *was* present in the retrieved context. The failure class is:
   A) A retrieval miss  B) Index drift  C) A generation failure (the model misused context that was present)  D) Low ANN recall
   — **Answer:** C. The right chunk was retrieved, so retrieval-class failures (miss, index drift, low ANN recall) are ruled out. The model ignored, misread, or under-used the context — a generation failure; fix the prompt/reranking/context length.
6. Which task is structurally impossible for classic (single-shot) RAG no matter how good its embeddings and index are?
   A) Answering a single-fact lookup from one document  B) Retrieving a passage that paraphrases the query  C) Answering "who founded the company that acquired Instagram?" — where the second retrieval depends on the first's result  D) Returning the top-k most similar chunks
   — **Answer:** C. This is multi-hop: you cannot query for the founder until you know the acquirer. Classic RAG does exactly one retrieval on the raw query and cannot form a follow-up query from a result; it requires agentic, multi-hop retrieval.
11. A corpus of thousands of interconnected research reports must support global, synthesizing questions like "what themes recur across all reports?" Which retrieval approach is purpose-built for this, and what is its main cost?
   A) Raising top-k on flat vector search; no extra cost  B) GraphRAG / structured retrieval — extract entities and relationships into a knowledge graph with community summaries; cost is a heavier indexing pipeline and added complexity  C) Switching the embedding model; cost is re-indexing  D) BM25 hybrid search; cost is a second index
   — **Answer:** B. A global synthesizing question has no single answer-bearing chunk; flat retrieval fetches individually-similar chunks and never assembles a corpus-wide picture. GraphRAG indexes entities/relationships into a knowledge graph with hierarchical community summaries and retrieves subgraphs/summaries — built for global and multi-hop relational questions. Its cost is the substantial extra indexing pipeline and maintenance, so it is a targeted choice, not a default.
7. A RAG system stuffs 25 chunks into a long context and the model under-uses a relevant chunk sitting in the middle. This is "lost in the middle." The most direct mitigation is:
   A) Increase the chunk overlap  B) Rerank and pass fewer chunks, placing the strongest near the start or end  C) Switch the vector DB from HNSW to IVF  D) Lower the embedding dimension
   — **Answer:** B. Models attend better to the start and end of a long context than the middle; passing fewer, reranked chunks and positioning the best near an edge directly addresses the effect. Overlap, index type, and embedding dimension are unrelated.
8. A team must answer questions about clause interactions within a *single* uploaded 50-page contract. Why is long-context preferable to chunked RAG here?
   A) RAG cannot index PDFs  B) The contract fits the context window and the task needs whole-document reasoning that chunking would sever across cross-references  C) Long-context is always cheaper than RAG  D) BM25 does not work on legal text
   — **Answer:** B. One 50-page document fits the window, and clause-interaction reasoning spans the whole document; chunking would cut the cross-references RAG retrieves piecemeal. (Long-context is *not* always cheaper — it is the right tool here specifically because of corpus size and the holistic-reasoning need.)
9. Two retrieval configs have identical recall@10 on a FAQ corpus where each question has one correct chunk, but config A puts that chunk at rank 1 and config B at rank ~5. Which metric exposes the difference and why?
   A) precision@k, because it counts irrelevant chunks  B) MRR, because it averages `1/rank` of the first relevant result and so penalizes a lower rank  C) NDCG, because it requires graded relevance labels  D) recall@1, which equals recall@10
   — **Answer:** B. recall@10 ignores position, so it is identical. MRR uses the reciprocal of the rank of the first relevant result, so rank 1 (≈1.0) scores far above rank 5 (≈0.2) — it directly rewards getting the one right answer near the top, which matters because the LLM only sees the highest-ranked chunks.
10. You are evaluating a reranker over queries that each have several relevant chunks with differing degrees of relevance. The most appropriate retrieval metric is:
   A) recall@k, because it ignores ordering  B) Exact match against the top result  C) NDCG, because it credits every relevant chunk, weights by graded relevance, discounts by rank, and normalizes against the ideal ordering  D) MRR, because it captures all relevant chunks
   — **Answer:** C. A reranker's job is *ordering* multiple relevant chunks; recall@k ignores ordering and MRR only looks at the first relevant result. NDCG rewards placing more, and more-relevant, chunks higher and normalizes to a 0–1 score against the ideal ranking — the right fit for graded multi-relevant retrieval.

### Short Answer

1. RAG is often praised for "reducing hallucination," yet the material says RAG does not *eliminate* it. Reconcile these two statements. — **Model answer:** RAG reduces *ungrounded* hallucination by supplying real evidence so the model answers from retrieved text instead of guessing from parametric memory — that is the big win. But the model can still misread the passages, over-generalize from them, ignore them in favor of what it "believes," or cite unfaithfully. So RAG mitigates hallucination by changing the *source* of the answer, but generation failures remain — it is a mitigation, not a cure. Credit for naming a concrete way hallucination survives RAG.
2. Cosine similarity ignores vector magnitude. Give a concrete reason this is *desirable* for retrieval — i.e., describe something that would go wrong if similarity were magnitude-sensitive. — **Model answer:** Cosine measures the angle (direction) between vectors, not their length, so it ranks by semantic orientation alone. This is desirable because a magnitude-sensitive measure could be skewed by text length or "intensity" — e.g., a long, emphatic passage might get a larger-magnitude vector and be scored as "more similar" to a query than a short passage that actually answers it better. Decoupling from magnitude keeps ranking about *meaning*, not verbosity.
3. A team uses fixed-size chunking with zero overlap and finds answers that span paragraph boundaries are frequently missed. Explain the mechanism and how overlap fixes it — and name one cost of adding overlap. — **Model answer:** An idea that straddles a chunk boundary is split across two chunks and incomplete in both, so neither is a matchable, usable answer. Overlapping windows make consecutive chunks share text, so the straddling idea appears intact in at least one chunk. Cost: overlap duplicates text, increasing storage and some redundancy in retrieved context.
4. A bi-encoder and a cross-encoder both score query-document relevance. Explain why only one of them can be used for first-stage retrieval over millions of chunks, grounded in *how each produces its score*. — **Model answer:** A bi-encoder embeds the query and each document *separately* into vectors; document vectors are precomputed once at index time, and query-time scoring is a cheap vector operation — so it scales to millions of chunks. A cross-encoder processes the query and document *jointly* with cross-attention; the score is a property of the specific pair and cannot be precomputed, so scoring N documents means N expensive model calls per query — infeasible as first-stage retrieval. The bi-encoder's separability is exactly what makes it scalable (and coarser).
5. Name three distinct RAG failure modes and classify each as retrieval or generation, and for one of them explain how you would *confirm* it from logs. — **Model answer:** Retrieval: retrieval miss (answer chunk never fetched), distracting/low-precision context, index drift/stale sources. Generation: ignoring the context (answering from memory), hallucination despite context, lost-in-the-middle. Confirmation example — for "ignoring the context": log the retrieved chunks; if the correct chunk is present yet the answer contradicts it, the failure is generation, not retrieval. Credit for three correct classifications plus one log-based diagnostic.
6. A stakeholder objects that abstention ("I don't have that information") makes the bot look unhelpful and wants it removed. Give the argument for keeping abstention. — **Model answer:** Without abstention, when retrieval genuinely finds nothing relevant the model is forced to generate from weak or empty context — it produces a confident, fluent, *wrong* answer. In domains like support, legal, medical, or finance a confidently wrong answer is far more damaging than an honest "I don't have that information," which the user can act on (rephrase, escalate). Abstention trades the appearance of helpfulness for actual trustworthiness.
7. State the core distinction that drives the RAG vs. fine-tuning choice, and give one example task for each that the *other* tool would handle badly. — **Model answer:** RAG supplies *knowledge* into the context; fine-tuning changes *behavior* (style, format, skill). RAG-but-not-fine-tuning example: answering from a large, frequently-updated help center — fine-tuning would inject facts unreliably and freeze them. Fine-tuning-but-not-RAG example: enforcing a consistent terse house style or output format — RAG supplies no behavior change. Knowledge problems → RAG; behavior problems → fine-tuning.
8. [deep-dive] Explain how embedding models are trained and use it to account for two practical retrieval behaviors: the query/document asymmetry, and why a general model can underperform on a specialized corpus. — **Model answer:** Embedding models are trained by *contrastive learning* in a *dual-encoder* setup: positive pairs (e.g., a query and its answering passage) are pulled together and negatives — especially hard negatives — pushed apart, via a contrastive (InfoNCE) loss, so that geometric closeness encodes semantic relatedness. (1) Asymmetry: retrieval training pairs are short query ↔ long document, so the model learns a directional query→document relationship (not a symmetric same-text similarity) — hence dual encoders and query/document task prefixes. (2) Domain fit: the model only learns the relationships present in its training pairs; a model trained on general web Q&A never learned what relevance means for specialized jargon, so it organizes that corpus poorly — fine-tuning on in-domain pairs fixes it. Credit for the contrastive/dual-encoder mechanism and tying both behaviors to it.

### Long Answer

1. Walk through the full RAG pipeline end to end, naming each stage and what can go wrong at it. — **Model answer / rubric:** Indexing (offline): collect corpus → chunk (failure: bad chunk size/fragmentation) → embed each chunk (failure: wrong/weak embedding model, domain mismatch) → store in vector DB (failure: index drift, low ANN recall). Retrieval (online): embed query → ANN search, optionally hybrid dense+sparse (failure: retrieval miss, vocabulary mismatch) → rerank (failure: relevant chunk left outside top-k). Generation: place top chunks in prompt → LLM answers with citations (failure: ignoring context, hallucination, lost-in-the-middle, no abstention). Credit for naming all stages, the indexing/retrieval/generation split, and stage-specific failures.
2. Explain hybrid search: what dense and sparse retrieval each do well and badly, why combining them helps, and how results are fused. — **Model answer / rubric:** Dense (embedding) search matches meaning — catches paraphrases and synonyms — but blurs rare exact terms (error codes, IDs, names). Sparse (BM25) matches exact terms via TF-IDF with saturation and length normalization — nails identifiers — but has no notion of meaning so misses paraphrases. They have complementary failure modes, so their union covers cases each misses. Fusion: Reciprocal Rank Fusion (rank-based, `Σ 1/(k+rank)`, no score normalization needed) is the robust default; weighted score combination requires normalizing the non-comparable score scales. Credit for the complementary-failure-modes argument and correct RRF description.
3. Compare RAG, long-context, and fine-tuning: what each is for, strengths, weaknesses, and the decision rule. — **Model answer / rubric:** RAG: retrieve at query time; for large/dynamic/private corpora needing citations; instant updates, auditable, cost-efficient; weakness: retrieval can miss, pipeline complexity, latency. Long-context: put documents directly in a large window; for small corpora and whole-document reasoning; simple, no retrieval miss; weakness: window-limited, expensive per token, slow TTFT, degrades on very long input. Fine-tuning: train the weights; for behavior/format/style/skill; changes how the model acts, can shorten prompts; weakness: unreliable for facts, frozen knowledge, needs a training pipeline, redone on base-model updates. Rule: knowledge → RAG/long-context (large+dynamic → RAG, small+holistic → long-context); behavior → fine-tuning; they compose. Credit for the knowledge-vs-behavior framing and correct trade-offs.
4. [deep-dive] Explain ANN search and the recall/latency/cost/memory trade-off, contrasting HNSW and IVF. — **Model answer / rubric:** Exact kNN is O(N) per query and doesn't scale; ANN trades a tunable amount of recall for large speed gains via an index. HNSW: multi-layer navigable graph, ~logarithmic query time, excellent recall/speed, supports incremental inserts, but high memory; tune `M`, `efConstruction`, `efSearch`. IVF: clusters vectors into partitions, searches only the `nprobe` nearest, lower memory and fast builds, but can miss neighbors across partition boundaries and is less update-friendly. Quantization (PQ/scalar) cuts memory at a recall cost. You cannot maximize recall, speed, low memory, and fast inserts simultaneously — choose by constraints and measure recall against an exact baseline. Credit for the ANN rationale, the trade-off triangle, and accurate HNSW/IVF contrast.

### Applied Scenario

1. A RAG bot over engineering docs answers conceptual "how-to" questions well but fails on questions that mention exact API names like `fetchUserProfile()` or config keys like `max_retry_window`. Logs show that for the failing queries the answer-bearing chunk is never retrieved. Diagnose the root cause and prescribe fixes spanning at least two sub-chapters' techniques. — **Model answer / rubric:** Diagnosis: a retrieval miss caused by dense search's blind spot — exact identifiers (`fetchUserProfile()`, `max_retry_window`) are out-of-distribution tokens with little semantic neighborhood, so the pure-dense retriever blurs them and ranks similar-but-wrong passages above the literal match (10.5). Fixes: (a) add sparse/BM25 retrieval and fuse with RRF — hybrid search nails exact-term queries while keeping dense for conceptual ones (10.5); (b) add a cross-encoder reranker so a literal match retrieved at low rank still reaches the top-k (10.6); (c) revisit chunking — structure-aware chunking by function/section plus contextual retrieval so identifier chunks embed with their context (10.3); (d) add abstention so a residual miss yields "I don't have that" not a wrong answer (10.7). Verify with a retrieval eval (recall@k). Credit for diagnosing the dense-search lexical gap and prescribing hybrid + rerank + chunking fixes.
2. A new RAG feature has good retrieval recall on its eval set, yet generated answers are frequently wrong or incomplete. The system retrieves the top 30 chunks and passes all of them to the LLM, ordered by raw vector similarity. Diagnose why good recall is not translating into good answers, and redesign the pipeline. — **Model answer / rubric:** Good recall means the right chunk is *retrieved* — but passing 30 raw chunks creates generation failures: low precision (many of the 30 are irrelevant and distract the model / dilute attention) and "lost in the middle" (a relevant chunk buried mid-context is under-used). The information is found but not usable. Redesign: keep retrieving a generous candidate set (preserve recall), then add a cross-encoder reranker and pass only the top ~3–5 reranked chunks; this raises precision, cuts distraction and lost-in-the-middle, and lowers token cost. Position the strongest chunk near the start or end. Evaluate retrieval and generation *separately* so this distinction is visible. Credit for separating recall (fine) from the precision/ordering problem and prescribing rerank-then-trim.
3. Your knowledge assistant handles simple FAQ lookups well, but users increasingly ask questions like "Compare the SLA of the vendor we onboarded most recently against our internal target." Classic RAG returns a blurry, incomplete answer. Diagnose why, propose an architecture, and name the new costs and evaluation challenge it introduces. — **Model answer / rubric:** Diagnosis: this is a multi-hop, multi-source question — you must first identify the most-recently-onboarded vendor, then retrieve *that vendor's* SLA, then retrieve the internal target, possibly from different indices. Classic RAG does one fixed retrieval on the raw query and cannot form follow-up queries from intermediate results, so it retrieves a blurry mix. Architecture: agentic RAG — expose retrieval as a tool in an agentic loop so the model decomposes the question, issues a query per hop using the previous hop's result, routes to the right index/source, and judges sufficiency before answering. New costs: more LLM calls and latency, less predictable behavior (over/under-retrieval, loops) — so add harness controls (retrieval-call budget, step limit, loop detection). Evaluation challenge: you must now evaluate the retrieval *trajectory* (which retrievals, in what order), not just the final answer. Credit for diagnosing multi-hop/multi-source, prescribing agentic RAG with decomposition and routing, and naming both the cost and the trajectory-eval challenge.
4. A stakeholder proposes deleting the RAG pipeline and "just pasting everything into the context window." Rather than answer yes/no, state the corpus characteristics under which they would be *right*, the characteristics under which they would be *wrong*, and how you would actually decide. — **Model answer / rubric:** They are right when the relevant corpus is *small enough to fit* the window and the task benefits from holistic whole-document reasoning (chunking would sever cross-references), and pipeline simplicity is valued — long-context then has no retrieval miss and is simple, especially with prompt caching for a stable prefix. They are wrong when the corpus is *large* (won't fit at all), *dynamic* (RAG re-indexes instantly; long-context re-pays for all tokens every call), *access-controlled* (RAG enforces permissions at retrieval), or *needs citations* — and very long contexts also cost more per token, slow TTFT, and degrade ("lost in the middle"). Decide empirically by corpus size, update frequency, citation/access-control needs, cost, and latency — not by fashion; and note the two compose (a longer window can hold more retrieved chunks). Credit for giving both the right-conditions and wrong-conditions and a concrete decision framework rather than a verdict.
5. Your RAG bot sometimes answers questions about events after the indexed documents' dates, giving wrong info, and sometimes ignores a clearly-correct retrieved passage in favor of its own (outdated) belief. Diagnose both and fix. — **Model answer / rubric:** Two distinct issues. (a) Stale index / index drift or retrieval miss for recent events: the bot answers from parametric memory because no current chunk was retrieved — fix by re-indexing on document updates, date-metadata filtering, possibly a live-search tool (agentic RAG), and an abstention instruction. (b) Ignoring correct retrieved context: a generation failure where the model overrides evidence with parametric belief — fix by an explicit prompt instruction to prefer the provided passages over prior knowledge, citation-forcing (every claim must cite a passage), and reranking so the strongest passage is prominent. Add observability (log retrieved chunks) and separate retrieval vs. generation evals to catch each. Credit for separating the retrieval issue from the generation issue and giving targeted fixes for each.

---

## Sources

[1] Anthropic — Introducing Contextual Retrieval — https://www.anthropic.com/news/contextual-retrieval
[2] OpenAI — New embedding models and API updates (text-embedding-3, Matryoshka dimensions) — https://openai.com/index/new-embedding-models-and-api-updates/
[3] Malkov & Yashunin — Efficient and robust approximate nearest neighbor search using Hierarchical Navigable Small World graphs (arXiv:1603.09320) — https://arxiv.org/abs/1603.09320
[4] Robertson & Zaragoza — The Probabilistic Relevance Framework: BM25 and Beyond — https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf
[5] Cormack, Clarke & Buettcher — Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods (SIGIR 2009) — https://cormack.uwaterloo.ca/cormacksigir09-rrf.pdf
[6] Liu et al. — Lost in the Middle: How Language Models Use Long Contexts (arXiv:2307.03172) — https://arxiv.org/abs/2307.03172
[7] Es et al. — RAGAS: Automated Evaluation of Retrieval Augmented Generation (arXiv:2309.15217) — https://arxiv.org/abs/2309.15217
[8] Asai et al. — Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection (arXiv:2310.11511) — https://arxiv.org/abs/2310.11511
[9] Yan et al. — Corrective Retrieval Augmented Generation (arXiv:2401.15884) — https://arxiv.org/abs/2401.15884
