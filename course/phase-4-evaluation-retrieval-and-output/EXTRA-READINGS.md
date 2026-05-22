# Phase 4 — Extra Readings

Curated, more in-depth resources for Phase 4 (Evaluation, Retrieval, and Output).
These go deeper than the course material. Every URL was verified to resolve.
Organized by topic.

---

## Topic 09 — Evaluations

- **Replacing Judges with Juries: Evaluating LLM Generations with a Panel of Diverse Models** — Verga et al. (Cohere), arXiv:2404.18796 — https://arxiv.org/abs/2404.18796
  The primary source for the Panel of LLM Evaluators (PoLL) result: a panel of small, diverse judges beats a single large judge on human correlation, at lower cost and bias.

- **Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena** — Zheng et al., arXiv:2306.05685 — https://arxiv.org/abs/2306.05685
  The foundational LLM-as-judge paper: documents position, verbosity, and self-enhancement bias, and the pairwise/Elo methodology behind Chatbot Arena (LMArena).

- **The Measurement of Observer Agreement for Categorical Data** — Landis & Koch, Biometrics 1977 — https://pubmed.ncbi.nlm.nih.gov/843571/
  The origin of the kappa interpretation scale (slight/fair/moderate/substantial/almost-perfect) used in inter-annotator agreement.

- **Anthropic — Using the Evaluation Tool (Claude docs)** — https://platform.claude.com/docs/en/test-and-evaluate/eval-tool
  Practical provider guidance on building eval test sets, side-by-side prompt comparison, and quality grading in the Claude Console.

- **OpenAI Evals (GitHub)** — https://github.com/openai/evals
  A widely-used open framework for writing and running model evaluations; useful for seeing how offline evals are structured in practice.

- **Introducing SWE-bench Verified** — OpenAI — https://openai.com/index/introducing-swe-bench-verified/
  How a contaminated/flawed benchmark gets human-validated into a cleaner subset — a concrete case study in benchmark hygiene and decay.

- **Hyperband: A Novel Bandit-Based Approach to Hyperparameter Optimization** — Li et al., JMLR 2018, arXiv:1603.06560 — https://arxiv.org/abs/1603.06560
  The primary source for successive halving / Hyperband, the budget-allocation strategy covered in 9.11.

- **DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines** — Khattab et al., arXiv:2310.03714 — https://arxiv.org/abs/2310.03714
  The framework that automates prompt/example search against a metric — a deeper look at the optimization angle of 9.11.

---

## Topic 10 — RAG

- **Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks** — Lewis et al. (Meta AI), arXiv:2005.11401 — https://arxiv.org/abs/2005.11401
  The original RAG paper that named the pattern; essential background for why retrieval is bolted onto generation.

- **Introducing Contextual Retrieval** — Anthropic — https://www.anthropic.com/news/contextual-retrieval
  The source for the contextual-retrieval technique and its measured failure-rate reductions; includes a runnable cookbook.

- **Lost in the Middle: How Language Models Use Long Contexts** — Liu et al., TACL 2023, arXiv:2307.03172 — https://arxiv.org/abs/2307.03172
  The primary study behind the "lost in the middle" failure mode and its implications for how many chunks to pass.

- **Efficient and robust approximate nearest neighbor search using HNSW graphs** — Malkov & Yashunin, arXiv:1603.09320 — https://arxiv.org/abs/1603.09320
  The HNSW paper — the graph-based ANN index that is the default in most modern vector databases.

- **Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods** — Cormack, Clarke & Buettcher, SIGIR 2009 — https://cormack.uwaterloo.ca/cormacksigir09-rrf.pdf
  The origin of RRF and the `k = 60` constant used in hybrid-search fusion.

- **The Probabilistic Relevance Framework: BM25 and Beyond** — Robertson & Zaragoza, 2009 — https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf
  The definitive deep-dive on BM25 — term frequency saturation, IDF, and length normalization.

- **RAGAS: Automated Evaluation of Retrieval Augmented Generation** — Es et al., arXiv:2309.15217 — https://arxiv.org/abs/2309.15217
  The reference-free RAG evaluation framework (faithfulness, answer relevancy, context precision/recall) — connects Topic 10 back to Topic 9.

- **Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection** — Asai et al., arXiv:2310.11511 — https://arxiv.org/abs/2310.11511
  A primary source for agentic/self-correcting retrieval — training a model to decide when to retrieve and to critique its own output.

- **Corrective Retrieval Augmented Generation (CRAG)** — Yan et al., arXiv:2401.15884 — https://arxiv.org/abs/2401.15884
  The plug-and-play retrieval self-correction approach referenced in 10.8 on agentic RAG.

---

## Topic 11 — Structured Outputs

- **Anthropic — Structured outputs (Claude API docs)** — https://platform.claude.com/docs/en/build-with-claude/structured-outputs
  The authoritative provider reference for Anthropic's native structured outputs: the `output_config.format` parameter, strict tool use, model support, and grammar caching behavior.

- **OpenAI — Introducing Structured Outputs in the API** — https://openai.com/index/introducing-structured-outputs-in-the-api/
  OpenAI's announcement explaining strict mode, constrained decoding, and the JSON-Schema-subset rules (`additionalProperties: false`, all-properties-required, optional-as-null).

- **OpenAI — Structured model outputs (API guide)** — https://platform.openai.com/docs/guides/structured-outputs
  The living reference for OpenAI's strict mode, including supported-schema limits (nesting depth, property counts, unsupported keywords).

- **Let Me Speak Freely? A Study on the Impact of Format Restrictions on Performance of LLMs** — Tam et al., arXiv:2408.02442 — https://arxiv.org/abs/2408.02442
  The most-cited study on how format restriction can degrade reasoning — and the NL-to-format (two-step) mitigation.

- **Efficient Guided Generation for Large Language Models (Outlines)** — Willard & Louf, arXiv:2307.09702 — https://arxiv.org/abs/2307.09702
  The technical foundation for grammar/finite-state-machine-based constrained decoding — how token masking is made efficient.

- **JSONSchemaBench: A Rigorous Benchmark of Structured Outputs for Language Models** — arXiv:2501.10868 — https://arxiv.org/abs/2501.10868
  A benchmark comparing structured-output engines on JSON Schema coverage and reliability — useful for understanding the "schema feature subset" limits.

- **Pydantic — Models documentation** — https://pydantic.dev/docs/validation/latest/concepts/models/
  The reference for schema-as-code in Python: `model_json_schema()`, `model_validate_json()`, and validators for the semantic layer.

- **instructor (GitHub)** — https://github.com/567-labs/instructor
  The library that automates the schema-as-code + corrective-retry loop described in 11.4; good for seeing the pattern implemented.
