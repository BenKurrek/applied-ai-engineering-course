# Topic 02 — Tokenization — Course Material

> Authored source material for the Applied AI Engineering course. Tutors teach from
> this file. Do not record student progress here.

## How to use

This file is taught sub-chapter by sub-chapter, in order. Each sub-chapter has core
teaching content, key terms, common misconceptions, a worked example, and check
questions to confirm understanding before moving on. A full exam question bank sits at
the end; the tutor draws from it for the gated topic exam. Topic 2 assumes Topic 1 —
especially that the model reads and writes *tokens* (1.1) and that the vocabulary is a
fixed set produced before training.

---

## 2.1 — What a token is; tokens ≠ words; tokens-per-word ratios

### Concept

A model from Topic 1 predicts the next *token*. A **token** is the atomic unit of text
the model reads and writes — but a token is **not** a word, and that mismatch causes a
surprising number of real bugs and cost surprises.

Why not just use words? Because a word-level vocabulary has fatal problems. There are
millions of distinct words across languages, plus names, typos, URLs, and code
identifiers — an unbounded set. Any word not in the vocabulary becomes an
"out-of-vocabulary" (OOV) token the model cannot represent. And character-level
tokenization (one token per character) avoids OOV but makes sequences enormously long,
which — given attention's O(n²) cost (1.4) — is prohibitive.

Modern tokenizers take the middle path: **subword tokenization**. Common words become a
single token; rarer words split into several subword pieces. The vocabulary is a fixed
set (typically ~32k–200k entries) of subword units learned from a corpus *before*
training. Every token maps to an integer ID; the model only ever sees those IDs.

Concretely, with a typical English tokenizer:

- `the`, `cat`, ` dog` → one token each (note the leading space — most tokenizers fold
  the preceding space into the token, so ` dog` and `dog` are *different* tokens).
- `tokenization` might split into `token` + `ization` (two tokens).
- `antidisestablishmentarianism` splits into several pieces.
- A rare name like `Kurrek` may split into `K` + `urre` + `k`.

**Tokens-per-word ratio.** For ordinary English prose the rule of thumb is **~1.3
tokens per word**, equivalently **~0.75 words per token**, equivalently **~4 characters
per token** — these are OpenAI's published rules of thumb for English text [1]. So
1,000 English words ≈ 1,300 tokens. But this ratio is not universal, and it is only an
estimate — different tokenizers produce different counts for the same text (2.4):

- **Code** tokenizes worse — lots of punctuation, indentation, `snake_case` and
  `camelCase` identifiers that split awkwardly — often 1.5–2+ tokens per "word."
- **Numbers** can be split into multiple tokens (2.6) — relevant for arithmetic.
- **Non-English languages** are far worse, especially non-Latin scripts (2.6) — the
  same sentence in, say, Hindi or Thai can use several times more tokens than English.

This matters because **everything is billed, limited, and rate-limited in tokens, not
words** (Topics 1, 4, 6, 12). You estimate cost in tokens, you size your context window
in tokens, and you must never assume "word count" is a good proxy. Always count tokens
with the model's actual tokenizer (e.g., a `tiktoken`-style library or the provider's
token-counting endpoint).

### Key terms

- **Token** — the atomic unit of text a model reads/writes; an integer ID into a fixed
  vocabulary; usually a subword piece.
- **Subword tokenization** — splitting text into units between characters and whole
  words; common words become one token, rare words split.
- **Out-of-vocabulary (OOV)** — text a word-level tokenizer cannot represent; subword
  and byte-level tokenizers are designed to eliminate it.
- **Tokens-per-word ratio** — empirical conversion factor; ~1.3 for English prose,
  higher for code and non-English text.
- **Vocabulary** — the fixed set of all tokens (and their IDs), fixed before training.
- **Token boundary** — where the tokenizer chooses to split text; not aligned with word
  boundaries.

### Common misconceptions

- ❌ One token = one word → ✅ A token is a subword piece; English averages ~1.3 tokens
  per word, and the ratio varies a lot by content and language.
- ❌ `dog` and ` dog` (with a leading space) are the same token → ✅ Most tokenizers
  fold the preceding space into the token, so they are distinct IDs.
- ❌ Word count is a fine proxy for cost — ✅ Billing, limits, and rate limits are all in
  tokens; always count with the real tokenizer.
- ❌ All languages tokenize at the same rate → ✅ Non-English (especially non-Latin
  scripts) and code can cost several times more tokens for the same content.

### Worked example

Take the sentence `The unhappiest developers refactor legacy code.` A typical
English BPE tokenizer might produce: `The` | ` unhapp` | `iest` | ` developers` | `
refactor` | ` legacy` | ` code` | `.` — 8 tokens for 6 words (ratio ≈ 1.3). Now the same
*idea* as code: `if not happy: refactor(legacy_code)` could be ` if` | ` not` | ` happy`
| `:` | ` refactor` | `(` | `legacy` | `_` | `code` | `)` — 10 tokens for far fewer
"words," because punctuation and the underscore each split off. Same information, very
different token counts — which is exactly why you must measure, not guess.

### Check questions

1. Subword tokenization is a compromise between two simpler schemes. Name both
   alternatives, state the specific failure each one has, and explain how subword
   tokenization avoids *both* failures at once. — **Answer:** (a) Word-level: the
   vocabulary is unbounded (every name, typo, URL, identifier), so any unseen word
   becomes an out-of-vocabulary token the model cannot represent. (b) Character-level:
   no OOV problem, but sequences become enormously long, and since attention is O(n²)
   that is prohibitively expensive. Subword tokenization keeps a *fixed* vocabulary
   (no unbounded growth) where common words are one token and rare words split into
   pieces (so nothing is OOV) — short-ish sequences and no OOV simultaneously.
2. A teammate budgets context by counting words and multiplying by 1.3. For a 1,000-word
   English blog post that is roughly fine, but they then apply the same factor to a
   1,000-"word" code file and a 1,000-word Thai document. Why will the estimate be off,
   and in which direction, for each? — **Answer:** ~1.3 tokens/word is an English-prose
   rule of thumb only. For code, punctuation, indentation, and `snake_case`/`camelCase`
   identifiers split heavily, so the real count is *higher* (often 1.5–2+×). For Thai (a
   non-Latin script), the multilingual tax means the same content fragments into far
   more tokens — also much *higher*. Both estimates undercount; the only reliable method
   is the model's actual tokenizer.
3. A prompt template works in testing but intermittently produces worse output in
   production; the only difference is that the production string has a leading space
   before a key word. From a tokenization standpoint, why could a single leading space
   change behavior at all? — **Answer:** Most tokenizers fold a leading space into the
   token, so ` word` and `word` are *different token IDs* with different learned
   representations. A stray leading space therefore feeds the model a different token
   sequence than the tested one — enough to shift behavior, and also enough to silently
   break prompt caching (2.7).

---

## 2.2 — Byte-Pair Encoding (BPE) and byte-level BPE

### Concept

**Byte-Pair Encoding (BPE)** is the most common algorithm for *building* a subword
vocabulary, used (in byte-level form) by the GPT family and by most current frontier
models. (For example, Llama 3 and later use OpenAI's `tiktoken`-style byte-level BPE,
with a 128k vocabulary — a switch from the SentencePiece tokenizer used in Llama 1 and
2 [2].) It is a compression-derived algorithm, and understanding the *merge* process
demystifies why tokens split the way they do.

**Building the vocabulary (training the tokenizer, done once, before model training):**

1. Start with a base vocabulary of the smallest units — initially every individual
   character (or, for byte-level BPE, every one of the 256 byte values).
2. Take a large training corpus and represent every word as a sequence of those base
   units.
3. Count every adjacent *pair* of units across the corpus. Find the **most frequent
   pair** and **merge** it into a single new token, adding it to the vocabulary. Record
   the merge as a rule.
4. Repeat step 3 — recount, find the next most frequent pair, merge — thousands of
   times, until the vocabulary reaches the target size.

Each merge is a learned rule like "`t` + `h` → `th`," then later "`th` + `e` → `the`."
Frequent sequences (`the`, `ing`, `tion`, common words) get merged into single tokens
because their constituent pairs co-occur often; rare sequences never merge and stay
split. This is *why* common words are one token and rare words fragment: it is a direct
consequence of pair frequency in the corpus.

**Encoding new text at inference time:** the tokenizer applies the learned merge rules
*in the order they were learned*, greedily, until no more merges apply. It is fully
deterministic — the same text always tokenizes the same way for a given tokenizer.

**Byte-level BPE** is the important refinement. Instead of starting from characters, it
starts from the **256 possible byte values**. Any text in any language — any emoji, any
Unicode character, any binary garbage — is just a sequence of bytes, and every byte is
in the base vocabulary. Therefore **byte-level BPE can never hit an out-of-vocabulary
token**: worst case, an unusual character falls back to its individual bytes (several
tokens), but it is always representable. This universality is why byte-level BPE
dominates: the tokenizer never fails, it just gets less efficient on rare input. The
cost of that guarantee is that rare characters and scripts are token-expensive (2.6).

### Key terms

- **Byte-Pair Encoding (BPE)** — algorithm that builds a subword vocabulary by
  iteratively merging the most frequent adjacent pair of units.
- **Merge / merge rule** — a learned operation combining two units into one new token;
  the ordered list of merges defines the tokenizer.
- **Base vocabulary** — the starting units before any merges: characters, or (byte-level
  BPE) the 256 byte values.
- **Byte-level BPE** — BPE starting from raw bytes; guarantees no out-of-vocabulary
  token because all text is bytes.
- **Greedy encoding** — applying merge rules in learned order until none apply;
  deterministic.

### Common misconceptions

- ❌ BPE merges are chosen by linguistic meaning → ✅ Merges are chosen purely by
  *frequency* of adjacent pairs in the corpus; the algorithm has no notion of morphemes.
- ❌ Byte-level BPE can still produce out-of-vocabulary tokens → ✅ It cannot — every
  byte is in the base vocabulary, so any text is always representable.
- ❌ BPE tokenization is probabilistic / random → ✅ Encoding is deterministic: fixed
  merge rules applied greedily in order.
- ❌ The tokenizer is trained jointly with the model → ✅ The tokenizer/vocabulary is
  built first, in a separate step, then frozen before model training.

### Worked example

Tiny corpus, words `low`, `lower`, `lowest`, each starting as characters. Pair counts:
`l`+`o` is very frequent → merge to `lo`. Recount: `lo`+`w` is now frequent → merge to
`low`. Recount: `low`+`e` appears in `lower`/`lowest` → merge to `lowe`; then `lowe`+`r`
→ `lower`. After these merges, encoding `lower` yields a single token `lower`, while a
never-seen word like `lowed` encodes as `low` + `e` + `d` — the `low` merge applies but
nothing merges `e`/`d`. That asymmetry — `lower` is one token, `lowed` is three — is
purely a frequency artifact of the corpus, exactly the mechanism behind real-world
token splits.

### Check questions

1. A BPE tokenizer trained on a medical-text corpus encodes "electrocardiogram" as a
   single token, but a BPE tokenizer trained on a general web corpus splits the same
   word into several pieces. Both use the identical algorithm. Explain how the same
   algorithm produces different results — and what this implies about why "common words
   are one token." — **Answer:** BPE merges the most *frequent* adjacent pair, iterated.
   In the medical corpus the character/sub-piece pairs inside "electrocardiogram"
   co-occur often enough to be merged all the way up to a single token; in the general
   corpus they do not, so the merges stop earlier and the word stays fragmented.
   "Common words are one token" is therefore not a property of the word — it is a
   property of *that tokenizer's training corpus*: frequency, not meaning, drives merges.
2. A colleague worries that feeding the model an unusual emoji or a snippet of a rare
   script will cause an "unknown token" error, and wants to strip such characters first.
   If the model uses byte-level BPE, is that fear justified? Explain what actually
   happens to such input. — **Answer:** The fear is unjustified — byte-level BPE cannot
   produce an out-of-vocabulary token, because its base vocabulary is the 256 byte
   values and *all* text is a byte sequence. The emoji or rare script is always
   representable; worst case it falls back to several individual byte tokens (more
   tokens, hence more cost), but never an error. Stripping characters is unnecessary and
   would silently change behavior.
3. The same text is tokenized twice by the same trained BPE tokenizer and a teammate is
   surprised the two token-ID sequences are byte-for-byte identical. Why is that
   guaranteed, and name one downstream system that *relies* on exactly this property. —
   **Answer:** BPE encoding is fully deterministic: the merge rules are fixed and applied
   greedily in their learned order, so identical input always yields identical token
   IDs — there is no randomness. Prompt caching relies on exactly this: it keys cache
   hits on the token-ID sequence (2.7), which only works because tokenization is
   reproducible.

---

## 2.3 — WordPiece vs. Unigram / SentencePiece

### Concept

For frontier LLMs in 2026 you are dealing with byte-level BPE (2.2). Three other names
appear in the landscape, and you need only to **recognize** them — what they are and how
they differ from BPE — not to derive their algorithms.

- **WordPiece** (BERT and many encoder models) also builds a vocabulary by merging, but
  merges the pair that most improves the **corpus likelihood** rather than the most
  *frequent* pair — so it favors pairs that are surprisingly common given how common
  their parts already are. It marks continuation pieces (e.g., `play` + `##ing`).
- **Unigram language model** goes the opposite direction from BPE: it starts with a
  *large* candidate vocabulary and **prunes it down** probabilistically. Because it is a
  probabilistic model, one string has many possible segmentations each with a
  probability — useful for *subword regularization* (sampling segmentations during
  training).
- **SentencePiece** is **not an algorithm** — it is a library that runs BPE or Unigram
  underneath. Its contribution is being language-agnostic and reversible: it treats
  input as a raw stream (no assumption spaces delimit words) and can losslessly
  reconstruct the original text. (Llama 1/2 used a SentencePiece BPE tokenizer; Llama 3
  switched to `tiktoken`-style byte-level BPE [2].)

One-line summary: **BPE** merges most-frequent pairs; **WordPiece** merges
most-likelihood-improving pairs; **Unigram** prunes a big vocabulary down;
**SentencePiece** is tooling, not an algorithm. The first is what frontier LLMs use; the
rest matter mainly for older and encoder models.

### Key terms

- **WordPiece** — subword algorithm (BERT) that merges by *likelihood improvement* (not
  raw frequency); marks continuations (e.g., `##`).
- **Unigram language model** — subword method that starts with a large vocabulary and
  *prunes* it down probabilistically.
- **SentencePiece** — a tokenization *library* (not an algorithm) that runs BPE or
  Unigram; language-agnostic and reversible.

### Common misconceptions

- ❌ SentencePiece is a tokenization algorithm → ✅ It is a library/framework; it runs
  BPE or Unigram underneath.
- ❌ WordPiece and BPE are identical → ✅ Both merge, but BPE merges by raw pair
  frequency while WordPiece merges by likelihood improvement.
- ❌ Unigram builds the vocabulary up by merging → ✅ Unigram starts large and prunes
  down.

### Worked example

The string `unhappiness`. A BPE tokenizer (merge-by-frequency) might land on `un` +
`happiness`. WordPiece might produce `un` + `##happiness`, the `##` flagging a
continuation. Unigram holds many candidate segmentations — `un|happiness`,
`unhappy|ness` — each with a probability, and emits the most probable. Same word, three
philosophies; for a frontier LLM, though, it tokenizes via byte-level BPE.

### Check questions

1. Match each description to BPE, WordPiece, or Unigram: (a) merges the most *frequent*
   adjacent pair; (b) starts with a large vocabulary and *prunes* it down; (c) merges
   the pair that most improves corpus likelihood. — **Answer:** (a) BPE, (b) Unigram,
   (c) WordPiece. The recognition-level point: BPE and WordPiece both *merge up* (by
   frequency vs. by likelihood), while Unigram *prunes down*.
2. A teammate says "we should switch from BPE to SentencePiece." Identify the category
   error and restate what they probably mean. — **Answer:** SentencePiece is not an
   algorithm on par with BPE — it is a library that *runs* BPE or Unigram underneath, so
   "BPE vs. SentencePiece" is not a real choice. They probably mean adopting
   SentencePiece's tooling (language-agnostic, reversible) with BPE or Unigram still
   underneath.

---

## 2.4 — Vocabulary size tradeoffs

### Concept

A core design decision when building a tokenizer is the **vocabulary size** — how many
tokens to stop the merge/prune process at. Frontier models in 2026 typically sit
somewhere in the **~100k–256k** range (older GPT-2-era models used ~50k; many recent
models pushed to 128k–256k partly to better serve multilingual and code text). The
choice is a genuine trade-off with consequences in two opposite directions.

**A larger vocabulary means shorter token sequences.** With more tokens available, more
words and phrases (and more multilingual content) become a single token rather than
several pieces. Shorter sequences are valuable: attention is O(n²) in sequence length
(1.4), so fewer tokens means cheaper inference, lower latency, and — because the context
window is measured in tokens — more *content* fits in a fixed window. A bigger vocab
also tends to tokenize non-English text and code more efficiently, reducing the
"multilingual tax" (2.6).

**But a larger vocabulary makes two matrices bigger.** The **embedding matrix** maps
each vocabulary entry to a vector, and the **unembedding matrix / LM head** maps the
final vector to one logit per vocabulary entry (1.3, 1.6). Both scale linearly with
vocabulary size. Doubling the vocabulary roughly doubles the parameters in these two
matrices and adds compute to the final softmax over the vocabulary. It also means more
*rare* tokens, each of which is seen less often in training, so the model gets fewer
gradient updates per rare token and may represent them less well — a data-sparsity
problem at the tail of the vocabulary.

So the trade-off is: **bigger vocab → shorter sequences (cheaper attention, more
content per window, better multilingual/code efficiency) but larger embedding/unembedding
matrices and more poorly-trained rare tokens.** Smaller vocab → leaner matrices but
longer sequences and worse non-English efficiency.

There is no universally correct number; it depends on the target languages, the corpus
mix (how much code, how multilingual), the model size (a huge model can absorb a big
embedding matrix more easily), and serving constraints. The practical engineering
takeaway: vocabulary size is fixed when the model is built, you cannot change it, and it
silently shapes your token costs — which is why two models with similar quality can have
noticeably different token bills for the *same* text.

### Key terms

- **Vocabulary size** — the number of distinct tokens in the tokenizer; a fixed
  design-time choice, typically ~100k–256k for 2026 frontier models.
- **Embedding matrix** — maps each vocabulary token to its initial vector; size scales
  with vocabulary.
- **Unembedding matrix / LM head** — maps the final vector to one logit per vocabulary
  token; size scales with vocabulary.
- **Sequence length** — number of tokens; a larger vocabulary tends to shorten it for
  the same text.
- **Rare-token sparsity** — the problem that very rare tokens in a large vocabulary get
  too few training updates to be well-represented.

### Common misconceptions

- ❌ A bigger vocabulary is strictly better → ✅ It shortens sequences but enlarges the
  embedding/unembedding matrices and worsens rare-token training; it is a trade-off.
- ❌ Vocabulary size doesn't affect model parameter count → ✅ It directly scales the
  embedding and unembedding matrices, which can be a meaningful share of parameters.
- ❌ You can grow the vocabulary of a deployed model on the fly → ✅ It is fixed at
  build time; changing it requires retraining the tokenizer and the model.
- ❌ All frontier models tokenize the same text into the same number of tokens → ✅
  Different vocabularies yield different token counts and therefore different costs.

### Worked example

Two models, A with a 50k vocabulary and B with a 200k vocabulary, both asked to process
the same 1,000-word multilingual document. Model A, with fewer tokens available,
fragments rarer words and non-English text heavily — say 1,800 tokens. Model B, with a
richer vocabulary, represents far more of that text as single tokens — say 1,150 tokens.
For the same document, model B's prompt is ~36% smaller, so it is cheaper to prefill,
faster, and leaves more room in the context window. The price model B paid is a much
larger embedding and unembedding matrix inside the network. Same text, two valid design
points.

### Check questions

1. A team argues "bigger vocabulary is just better — shorter sequences, cheaper
   attention, why would anyone pick small?" Give the strongest counter-argument, naming
   *two distinct* costs a larger vocabulary imposes and why each is a real downside. —
   **Answer:** (a) The embedding and unembedding matrices scale linearly with vocabulary
   size, so a much larger vocabulary adds a meaningful chunk of parameters (and final-
   softmax compute) — model size and memory go up. (b) A larger vocabulary has more
   *rare* tokens, each seen less often in training, so each rare token gets fewer
   gradient updates and is represented worse — a data-sparsity problem at the tail. So
   it is a genuine trade-off, not a free win.
2. You profile a model and find that a surprisingly large share of its parameters sit
   in just two matrices, and that share grew when the team widened the tokenizer. Which
   two matrices are these, and why does widening the *tokenizer* enlarge them? —
   **Answer:** The embedding matrix (token → vector) and the unembedding matrix / LM head
   (final vector → one logit per token). Both have one row/column *per vocabulary entry*,
   so their size is linear in vocabulary size; widening the tokenizer adds vocabulary
   entries and therefore directly enlarges both matrices.
3. Two models are equally capable, but model X consistently produces a higher token
   bill than model Y for the *same* prompts, and you cannot change either model's
   vocabulary. Explain why the bills differ and what that means for an
   apples-to-apples cost comparison. — **Answer:** X and Y have different tokenizers, so
   they segment identical text into different numbers of tokens; since billing is
   per-token, the same content costs different amounts. Vocabulary size is fixed at build
   time, so you cannot "tune" this away. A fair comparison must run representative real
   prompts/completions through *each model's actual tokenizer* and multiply by that
   model's rates — a lower per-token price can lose to a more token-efficient tokenizer.

---

## 2.5 — Special tokens — BOS/EOS, padding, role/turn delimiters, tool tokens

### Concept

Beyond tokens that represent ordinary text, every tokenizer reserves a set of **special
tokens** — tokens that carry *structural* meaning rather than literal text. They never
appear as normal text content; they exist so the model can tell where things begin and
end and what role each piece of the input plays.

**BOS / EOS.** A **beginning-of-sequence (BOS)** token marks the start of an input; an
**end-of-sequence (EOS)** token marks the end. EOS is the most important to understand
operationally: when an autoregressive model (1.5) *emits* the EOS token, that is the
signal to **stop generating** — it is how the model says "I am done." Generation halts
on EOS (or on a stop sequence or max_tokens — Topic 3.6).

**Padding.** When multiple sequences of different lengths are batched together for
efficient processing, shorter ones are filled with a **padding token** so all sequences
in the batch are the same length. An attention mask tells the model to ignore padded
positions so they do not affect the real computation. Padding is a batching mechanic;
it does not carry meaning.

**Role / turn delimiters — the chat template.** This is the one engineers most
underestimate. The roles you see in a messages array — `system`, `user`, `assistant`,
`tool` (Topic 4.1) — are **not** a separate data structure the model natively
understands. Under the hood, the messages array is **flattened into one flat token
sequence**, and special delimiter tokens mark where each turn starts and ends and which
role it belongs to. This formatting is the **chat template**. For example, a turn might
be wrapped as `<|im_start|>user … <|im_end|>` (the exact tokens differ per model). The
model learned during post-training to recognize these delimiters and behave
accordingly — to treat the system turn as authoritative, to respond as the assistant,
and so on. The "conversation" is, mechanically, just text with special tokens in it.

**Tool tokens.** Models that support tool use / function calling (Topic 7) often have
special tokens that delimit a tool *call* and a tool *result*, so the model can emit a
structured "I want to call tool X" block and later read a "here is the result" block.
These too are special tokens woven into the same flat sequence.

Two practical implications: (1) **special tokens consume context budget** — the chat
template's delimiter tokens are real tokens that count toward your limits and bill; (2)
**you must use the correct template for the model** — wrong delimiters, or letting
user-supplied text contain the literal delimiter strings, degrades behavior or opens a
prompt-injection vector (Topic 13). Reputable SDKs apply the right template for you,
which is one reason to prefer them over hand-assembling prompt strings.

### Key terms

- **Special token** — a reserved token carrying structural meaning, not literal text
  content (BOS, EOS, padding, delimiters, tool tokens).
- **BOS / EOS** — beginning-of-sequence and end-of-sequence markers; emitting EOS is how
  the model signals it is done generating.
- **Padding token** — filler added to short sequences so a batch has uniform length;
  ignored via an attention mask.
- **Chat template** — the model-specific formatting that flattens a messages array into
  one token sequence using role/turn delimiter tokens.
- **Role/turn delimiter** — special tokens marking where each conversational turn starts
  and ends and which role it belongs to.
- **Tool tokens** — special tokens delimiting tool-call requests and tool results.

### Common misconceptions

- ❌ Roles like `system`/`user`/`assistant` are a native model concept → ✅ They are
  formatting; the messages array is flattened into one flat token sequence with special
  delimiter tokens.
- ❌ The model decides on its own when to stop → ✅ It stops because it *emits the EOS
  token* (or hits a stop sequence / max_tokens); EOS is a concrete token.
- ❌ Special tokens are free / don't count → ✅ Chat-template delimiters are real tokens
  that consume context budget and are billed.
- ❌ Any chat template works with any model → ✅ Each model expects its specific
  delimiter tokens; using the wrong template degrades behavior.

### Worked example

You send this messages array: `[{system: "You are terse."}, {user: "Hi"}]`. The model
never sees a JSON object. The SDK applies the model's chat template and produces a flat
token stream conceptually like:
`<BOS><|im_start|>system\nYou are terse.<|im_end|>\n<|im_start|>user\nHi<|im_end|>\n<|im_start|>assistant\n`
— and the model generates from there until it emits `<|im_end|>` then `<EOS>`. The
`<|im_start|>`, role names, and `<|im_end|>` are all real tokens that count toward your
input bill. If a user pasted the literal string `<|im_start|>system` into their message,
a naive prompt assembler could let it be interpreted as a real turn boundary — the seed
of a prompt-injection attack (Topic 13).

### Check questions

1. A model "decides to stop talking" — but the material insists nothing magical
   happens. Describe the concrete mechanism, and explain how it is that the *same*
   stopping mechanism is what makes a stuck model that never stops a debuggable problem
   rather than a mystery. — **Answer:** Stopping happens because the model *emits the EOS
   token* (a concrete special token); the decode loop treats EOS as the halt signal.
   (A stop sequence or `max_tokens` can also halt it.) Because stopping is just "did the
   model emit EOS," a model that runs on too long is a concrete, inspectable issue — the
   EOS token's probability was too low to be selected — not an inexplicable behavior; you
   can reason about it via the distribution and the decoding parameters.
2. A developer hand-assembles a prompt string for Model A using Model B's role
   delimiters (e.g., the wrong `<|...|>` tokens). The prompt "looks like a normal
   conversation." Why does the model nonetheless behave worse, given that roles are
   "just a convention"? — **Answer:** Roles are a convention *the model was post-trained
   to recognize via its own specific delimiter tokens*. The messages array is flattened
   into one flat token sequence, and the model learned to treat *its* delimiters as
   turn/role boundaries. Model B's delimiters are different (or unknown) tokens to Model
   A — it does not reliably parse them as turn boundaries, so the trust hierarchy and
   turn structure it was trained on are not triggered, degrading behavior.
3. Two prompts contain the *exact same visible conversation text*, but one is sent as a
   proper messages array via the SDK and the other as a hand-built single string with
   no role delimiters. Will they cost the same number of tokens? Explain. — **Answer:**
   No. The properly structured one also includes the chat template's role/turn delimiter
   tokens (`<|im_start|>`, role names, `<|im_end|>`, etc.), which are *real tokens* that
   count toward the input bill and context limit. The hand-built string without those
   delimiters has fewer tokens — but it also loses the role structure the model relies
   on, so "cheaper" here means "broken."

---

## 2.6 — Tokenization bugs — number splitting, whitespace, glitch tokens, multilingual cost

### Concept

Tokenization is not a neutral preprocessing step — it is a leaky abstraction that
produces real, debuggable failures. Four families matter.

**Number splitting.** Numbers tokenize inconsistently. `100` might be one token,
`1000` two, `31415` several — and the splits do not align with digit places. The model
sees, say, `[314][15]` rather than the digits `3,1,4,1,5`. Doing arithmetic on a
quantity whose representation is fragmented and inconsistent across magnitudes is hard,
and this is a major reason naive LLMs fumble multi-digit arithmetic (1.1's worked
example). Modern tokenizers mitigate this by forcing digits to tokenize uniformly (e.g.,
splitting every number into individual digits, or into fixed three-digit chunks), which
measurably improves arithmetic — but the underlying lesson stands: number handling is a
tokenizer artifact, and for reliable math you give the model a calculator tool (Topic
7).

**Whitespace sensitivity.** As noted in 2.1, a leading space is usually folded into the
token, so ` dog` ≠ `dog`. Trailing whitespace, double spaces, tabs, and newlines all
tokenize in ways that can surprise you. A classic failure: prefilling the assistant turn
(Topic 5.5) with a trailing space, which creates an awkward token boundary and degrades
output. Inconsistent whitespace also silently busts prompt caching (2.7). The rule:
treat whitespace as significant and be deliberate about it.

**Glitch tokens.** A historical curiosity worth one paragraph: in GPT-2/GPT-3-era
tokenizers, a few tokens (the famous case is **`SolidGoldMagikarp`**, a scraped Reddit
username, documented in early 2023 [3]) ended up in the vocabulary but were almost never
seen in meaningful contexts during training, so the model's representation of them was
effectively undefined and prompting with them produced erratic output. The root cause is a mismatch between the
tokenizer's training corpus and the model's training corpus. Modern tokenizer pipelines
largely screen these out; the durable lesson is simply that "in the vocabulary" does not
guarantee "well understood."

**Multilingual cost.** Tokenizers are overwhelmingly trained on English-heavy corpora,
so English merges efficiently while other languages — especially non-Latin scripts like
Chinese, Hindi, Thai, Arabic, or languages with rich morphology — fragment into many
more tokens. The *same* sentence can cost 2–5× (sometimes more) the tokens in another
language than in English. This is the "**multilingual tax**": non-English users pay more
per request, get less content into the same context window, and may hit rate limits
sooner — an equity and cost issue baked into the tokenizer. Larger, more multilingual
vocabularies (2.4) reduce but do not eliminate it.

The unifying point: tokenization sits *below* the model, it is invisible in your prompt
text, and it causes failures (bad math, whitespace bugs, glitch behavior, multilingual
cost) that you cannot debug without thinking at the token level. Always be able to
inspect the actual tokens.

### Key terms

- **Number splitting** — inconsistent, place-misaligned tokenization of digits;
  contributes to arithmetic errors. Mitigated by digit-level or fixed-chunk tokenization.
- **Whitespace sensitivity** — spaces, tabs, and newlines change tokenization; a leading
  space is part of the token; a stray space can degrade output or bust caching.
- **Glitch token** — a (mostly historical) vocabulary token under-represented in
  meaningful training contexts, producing unpredictable behavior; e.g.,
  `SolidGoldMagikarp`. Root cause: a tokenizer-corpus/model-corpus mismatch.
- **Multilingual tax** — the higher token count (and hence cost, latency, and context
  consumption) for non-English text, especially non-Latin scripts.

### Common misconceptions

- ❌ The model sees numbers digit by digit → ✅ It sees number *tokens* that split
  inconsistently and don't align with digit places — a cause of arithmetic errors.
- ❌ Extra spaces in a prompt are harmless → ✅ Whitespace changes tokenization; it can
  degrade output and silently bust prompt caching.
- ❌ Any token in the vocabulary is well understood by the model → ✅ Glitch tokens are
  in the vocabulary but barely trained on, producing unpredictable behavior.
- ❌ A prompt costs the same regardless of language → ✅ Non-English text, especially
  non-Latin scripts, can cost several times more tokens for the same meaning.

### Worked example

Ask a naive model `What is 3 + 5?` — fine, single-digit. Ask `What is 4817 + 2956?` and
errors appear: the operands tokenized into fragments like `[481][7]` and `[295][6]`, so
the model is pattern-matching over a fragmented, inconsistent representation rather than
computing — exactly why arithmetic gets a calculator tool. Now measure the multilingual
tax: "I would like to book a table for two people tonight"
might be ~11 tokens in English but the same request in Hindi or Thai can run 30–50+
tokens — the non-English user pays several times more for an identical request.

### Check questions

1. A model handles `7 + 8` flawlessly but botches `64825 + 31947`. Both are "just
   addition." Explain, from tokenization, why the model's *competence does not transfer*
   from the small case to the large one — and why a tokenizer that forces every digit
   into its own token would help. — **Answer:** Small sums like `7 + 8` appear so often
   in training that the model can pattern-match the answer directly. Large numbers
   tokenize into inconsistent, place-misaligned fragments (e.g. `[648][25]`), and that
   exact sum is essentially never in training data — so there is no pattern to match and
   no clean digit structure to compute over. Competence does not transfer because the
   "skill" was memorized small sums, not an algorithm. Forcing one token per digit gives
   a *consistent, place-aligned* representation across all magnitudes, which measurably
   improves arithmetic — but a calculator tool is still the reliable fix.
2. "In the vocabulary" does not guarantee "well understood." Name the (mostly
   historical) class of token that illustrates this and its root cause in one sentence.
   — **Answer:** Glitch tokens (e.g. `SolidGoldMagikarp`): a string that appeared in the
   *tokenizer's* corpus often enough to earn its own token but was barely seen in
   meaningful contexts in the *model's* training data, leaving its learned representation
   effectively undefined.
3. A PM is surprised that the same chatbot feature costs ~3× more per request for Thai
   users than English users and asks if it is a billing bug. Explain why it is expected,
   and name *three* distinct ways (beyond raw money) the same effect hurts the Thai
   user's experience. — **Answer:** It is the multilingual tax, not a bug: tokenizers are
   trained on English-heavy corpora, so Thai (a non-Latin script) fragments into far more
   tokens for the same meaning, and billing is per-token. Beyond cost: (a) the Thai
   user's prompt consumes more of the context window, so *less content fits*; (b) more
   tokens means more prefill, raising latency; (c) token-based rate limits are reached
   sooner, so the Thai user is throttled earlier. A larger multilingual vocabulary
   reduces but does not eliminate the gap.

---

## 2.7 — How tokenization interacts with prompt caching

### Concept

This sub-chapter is a bridge to Topic 6 (Prompt Caching). You do not need the full
caching mechanics yet — only the precise way *tokenization* governs whether a cache hit
happens.

Recall from Topic 1: prefill processes the prompt and builds a **KV cache** of key/value
vectors. **Prompt caching** lets the provider *reuse* the KV cache for a prompt **prefix**
that is identical to a recent request, skipping recomputation and giving a large TTFT
and cost win. The essential rule of prompt caching is the **exact-prefix-match
requirement**: the cache is reused only up to the first point where the new request
differs from the cached one. One differing token early in the prompt invalidates the
cache for everything after it. (Anthropic's prompt-caching documentation states
explicitly that the cache prefix must be identical and is checked from the beginning of
the prompt; a difference invalidates the cache from that point on [4]. Provider
specifics — minimum cacheable length, TTL — differ and are covered in Topic 6.)

Here is the key insight: **the matching happens at the token level, not the character
level.** The cache is keyed on the *sequence of token IDs*. So whether two prompts share
a cacheable prefix depends entirely on whether they tokenize into the *identical token-ID
sequence* up to the divergence point. This has several sharp consequences:

- **Cosmetic, character-level differences that change tokenization break the cache.**
  An extra space, a different newline style (`\n` vs `\r\n`), a trailing space, a
  different Unicode normalization, smart quotes vs. straight quotes — any of these can
  change the token IDs and therefore bust the prefix match, even though the text "looks
  the same" to a human. Whitespace bugs (2.6) are a leading cause of mysterious
  cache-miss rates.
- **A change anywhere in the prefix invalidates everything downstream.** Because tokens
  are matched in order from the start, changing token #10 of a 5,000-token system prompt
  throws away the cache for tokens #10–5,000. This is why the canonical structuring rule
  (Topic 6.3) is: put **static content first** (system prompt, tool definitions,
  few-shot examples, large stable documents) and **variable content last** (the user's
  query). Variable content at the front would re-tokenize differently every request and
  bust the cache for the whole prompt.
- **Token *boundaries*, not just content, must line up.** Even if the same characters
  are present, if a change causes the tokenizer to merge or split differently around the
  boundary, the token-ID sequence diverges and the cache breaks there.
- **Non-determinism in prompt assembly is the enemy.** Re-serializing JSON with keys in
  a different order, injecting a timestamp, or formatting numbers differently changes
  tokens and silently kills your cache hit rate.

So the operational discipline: assemble the cacheable prefix **byte-for-byte
deterministically** every request, keep it stable, and put anything variable at the end.
Tokenization is the layer that turns "looks identical" into "is identical" — and only
"is identical *at the token level*" earns a cache hit. (Topic 6 covers TTL, pricing
math, and when caching pays off.)

### Key terms

- **Prompt caching** — reusing the KV cache for a repeated prompt prefix to save prefill
  cost and latency (full treatment in Topic 6).
- **Exact-prefix-match requirement** — the cache is reused only up to the first
  differing token; one early difference invalidates everything after.
- **Token-level matching** — caching is keyed on the sequence of token IDs, not on
  characters; identical-looking text that tokenizes differently does not match.
- **Cache busting** — accidentally invalidating the cache via a change (often a
  whitespace or formatting difference) in the prefix.
- **Static-first / variable-last** — the prompt-structuring rule that maximizes the
  cacheable prefix.

### Common misconceptions

- ❌ Prompt caching matches on the text characters → ✅ It matches on the token-ID
  sequence; text that looks identical but tokenizes differently does not hit the cache.
- ❌ A tiny formatting change late in the prompt is harmless to caching → ✅ A change
  anywhere in the prefix invalidates the cache from that token onward.
- ❌ Whitespace differences don't matter for caching → ✅ They change tokenization and
  are a leading cause of cache misses.
- ❌ You can put the variable user query first and still cache the static parts → ✅
  Variable content at the front re-tokenizes every request and busts the cache for the
  whole prompt; put static content first.

### Worked example

A RAG app builds prompts as: `[system prompt][tool defs][retrieved docs][user
question]`. The first three sections are identical across many requests, so they form a
long cacheable prefix — every request after the first gets a cheap, fast cache hit on
that prefix and only pays full price for the short, variable user question at the end.
Now a developer "improves" the prompt by prepending `Request time: 2026-05-20T14:03:11Z`
at the very top. That timestamp changes every request, so the very first tokens differ
every time — the entire prefix, all of it, fails the exact-prefix match, and the cache
hit rate collapses to zero. The cost and latency regression is real and entirely a
tokenization-level effect: the token-ID sequence no longer matches from token #1.

### Check questions

1. Two prompts are *visually* character-for-character identical when printed, yet they
   get zero cache hits against each other. Give a concrete reason this can happen, and
   explain why "matches at the token level, not the character level" is the precise way
   to state the cache rule. — **Answer:** They can tokenize differently despite looking
   identical — e.g. one uses straight quotes and the other smart quotes, or `\n` vs
   `\r\n`, or a different Unicode normalization; these produce different token IDs. The
   cache is keyed on the *token-ID sequence*, so what matters is not whether the text
   looks the same but whether it tokenizes into the identical IDs up to the divergence.
   "Token-level, not character-level" is precise because identical-looking characters
   are necessary but not sufficient — only identical token IDs earn a hit.
2. A developer puts the (highly variable) user question at the *top* of the prompt and
   the large static system prompt and documents below it, reasoning "the static part is
   still in there, so it still caches." Walk through, token by token, why this gets
   essentially zero cache benefit. — **Answer:** The cache matches the prefix *in order
   from token #1*. With the variable question first, the token sequence differs starting
   at (or very near) token #1 on every request, so the exact-prefix match fails
   immediately — and once the match fails, *nothing after it* can be reused, including
   the static system prompt and documents. The static content "being in there" is
   irrelevant; it is positionally downstream of the divergence. Static-first /
   variable-last is required precisely so the long stable run sits *before* any
   divergence.
3. Your cache hit rate silently dropped after a refactor that "changed nothing visible."
   List three plausible tokenization-level culprits that a code refactor could
   introduce, and describe how you would *confirm* which one it is. — **Answer:**
   Plausible culprits: (a) injected per-request data near the top (timestamp, request
   ID, randomized ordering); (b) changed whitespace/newline style or trailing spaces;
   (c) non-deterministic serialization — e.g. JSON keys emitted in a different order, or
   reformatted numbers — or a switched chat-template/SDK version. To confirm: capture the
   actual *token-ID sequences* of two consecutive requests and diff them; the first
   index where they diverge points straight at the offending change. Fix: make the
   cacheable prefix byte-for-byte deterministic and move all variable content to the end.

---

## Topic 02 — Exam Question Bank

A pool the tutor draws from for the gated exam. Mixed-format, scored out of 100; 85 to
pass. Free-form answers are graded on reasoning quality with partial credit.

### True / False

1. Estimating a 1,000-word English prose document at ~1,300 tokens, then applying the
   same 1.3× factor to a 1,000-"word" source-code file, will give a reasonable token
   estimate for the code file. — **Answer:** False. ~1.3 tokens/word is an English-prose
   rule of thumb; code tokenizes worse (punctuation, indentation, identifier splits),
   often 1.5–2+× per "word," so the estimate will undercount.
2. If a model uses byte-level BPE, you must sanitize unusual emoji and rare-script
   characters out of user input or the tokenizer will throw an out-of-vocabulary error.
   — **Answer:** False. Byte-level BPE has all 256 byte values in its base vocabulary,
   so any text is representable; rare characters fall back to multiple byte tokens (more
   cost) but never cause an OOV error.
3. Choosing "SentencePiece instead of BPE" is a coherent design decision. — **Answer:**
   False. SentencePiece is not an algorithm — it is a library/framework that runs BPE
   *or* Unigram underneath; you choose the underlying algorithm, with SentencePiece as
   tooling.
4. Doubling a model's vocabulary size has no effect on its parameter count, only on how
   text is split. — **Answer:** False. The embedding and unembedding/LM-head matrices
   scale linearly with vocabulary size, so a larger vocabulary adds parameters (and
   final-softmax compute) — it is a genuine trade-off, not just a splitting change.
5. A model that runs on and on without ever stopping is best explained as "the model
   chose not to stop," an unanalyzable behavior. — **Answer:** False. Stopping is a
   concrete mechanism — the model halts when it *emits the EOS token* (or hits a stop
   sequence / `max_tokens`). A non-stopping model simply assigned EOS too low a
   probability to be selected; it is an analyzable, debuggable situation.
6. Two prompts that print as character-for-character identical are guaranteed to share a
   prompt-cache prefix. — **Answer:** False. Caching is keyed on the token-ID sequence;
   identical-looking text can tokenize to different IDs (smart vs. straight quotes,
   `\n` vs `\r\n`, Unicode normalization) and then will not match.
7. Two non-English languages will incur roughly the same "multilingual tax" as each
   other relative to English. — **Answer:** False. The tax varies by language and
   script; non-Latin scripts and morphologically rich languages fragment differently,
   so the per-language multiple over English is not uniform.

### Multiple Choice

1. A character-level tokenizer also has a fixed, bounded vocabulary and never hits OOV.
   So why is subword tokenization still preferred over character-level?
   A) Character-level is incompatible with the transformer architecture
   B) Character-level makes sequences far longer, and attention is O(n²), so it is
      prohibitively expensive
   C) Character-level cannot represent emoji
   D) Subword tokenization needs fewer special tokens
   — **Answer:** B. Both avoid OOV; subword wins because it keeps sequences far shorter,
   which matters given quadratic attention cost.
2. A BPE tokenizer trained on a Python-heavy corpus encodes `def` as one token, but a
   BPE tokenizer trained on a poetry corpus splits `def` into `d`+`ef`. The most
   accurate explanation is:
   A) One of the tokenizers is buggy
   B) The merge rules are chosen by the language designers
   C) `d`+`e`+`f` co-occur frequently enough to merge in the code corpus but not the
      poetry corpus — merges are frequency-driven and corpus-specific
   D) Python keywords are always single tokens by specification
   — **Answer:** C. BPE merges the most frequent adjacent pairs; "common word = one
   token" is a property of the training corpus, not of the word.
3. A pair of pieces co-occurs often, but each piece is also very common on its own. Which
   tokenizer is *more* likely to decline to merge that pair, and why?
   A) BPE, because it ignores frequency
   B) WordPiece, because it scores a merge by frequency relative to the parts'
      frequencies, so a pair of two very common parts looks less "worth it"
   C) Both decline it identically
   D) Neither — frequency never affects merges
   — **Answer:** B. WordPiece merges by likelihood improvement, which discounts pairs
   whose parts are individually common; BPE would still merge a high-frequency pair.
4. A team profiles a model and finds two matrices hold a large share of its parameters,
   and that share rose after the tokenizer was widened. Those matrices are:
   A) The query and key projection matrices
   B) The embedding matrix and the unembedding / LM-head matrix
   C) Two FFN layers chosen at random
   D) The KV cache and the attention mask
   — **Answer:** B. Both have one row/column per vocabulary entry, so widening the
   tokenizer (more vocabulary) directly enlarges them.
5. A developer ports a prompt from Model A to Model B by reusing Model A's literal
   role-delimiter strings inside a hand-built prompt. The most likely result is:
   A) Identical behavior, since roles are universal
   B) An immediate API error
   C) Degraded behavior, because Model B was trained to recognize *its own* delimiter
      tokens and does not reliably parse Model A's as turn boundaries
   D) Faster responses, since the delimiters are shorter
   — **Answer:** C. Roles are a learned convention tied to each model's specific
   delimiter tokens; the wrong template is not parsed as the intended turn structure.
6. A prompt containing the rare token `SolidGoldMagikarp` makes a model emit unrelated
   words. The root cause is best described as:
   A) A deliberate safety filter
   B) A mismatch between the tokenizer's training corpus and the model's training
      corpus — the string earned a token but was barely seen in meaningful contexts, so
      its representation is essentially undefined
   C) The token is not in the vocabulary
   D) A network timeout
   — **Answer:** B. Glitch tokens are *in* the vocabulary but under-trained; "in the
   vocabulary" does not mean "well understood."
7. A prompt is `[static system prompt][static tool defs][static docs][variable user
   question]`. A developer prepends a per-request timestamp at the very top. The effect
   on prompt caching is:
   A) No effect — the static blocks are still present
   B) The cache improves, because there is now more content
   C) The cache hit rate collapses, because the token sequence differs from token #1 and
      a divergence invalidates everything after it
   D) Only the user question stops caching
   — **Answer:** C. Prefix matching runs from the start; an early divergence kills reuse
   of the entire downstream prefix.
8. ` dog` (with a leading space) and `dog` are fed to the same tokenizer. Which
   statement is correct?
   A) They always produce the same token ID
   B) The leading space is stripped and ignored
   C) Most tokenizers fold the leading space into the token, so they are different token
      IDs with different learned representations
   D) The leading space causes an out-of-vocabulary error
   — **Answer:** C. Whitespace is significant; this is a common source of subtle bugs
   and cache misses.

### Short Answer

1. A teammate sizes a context budget by word count alone. Explain why word count is an
   unreliable proxy for token count, and identify the *one* reliable way to know the
   token count of a given piece of text. — **Model answer:** A token is a subword unit,
   not a word; the tokens-per-word ratio varies with content (code ≫ prose) and language
   (non-Latin scripts ≫ English), so any fixed factor mis-estimates for some inputs. The
   ~1.3 tokens/word English-prose rule of thumb is only an approximation. The reliable
   method is to run the text through the *model's actual tokenizer* (e.g. a tiktoken-style
   library or the provider's token-counting endpoint) and count the resulting IDs.
2. Using the BPE merge process, explain why a frequent word like "running" is typically
   one token while a rare coinage like "yeeting" splits into pieces — and why that split
   is *not* about morphology even though "yeet"+"ing" looks morphologically sensible. —
   **Model answer:** BPE builds its vocabulary by repeatedly merging the most *frequent*
   adjacent pair. "running"'s constituent pairs co-occur often in the corpus, so the
   merges proceed all the way to a single token; "yeeting"'s do not, so it stays split.
   Any apparent morpheme alignment ("yeet"+"ing") is coincidental — the algorithm has no
   notion of morphemes; it tracks pair frequency only. The split is a frequency artifact
   of the corpus.
3. A developer claims byte-level BPE is risky because "exotic input could be
   unrepresentable." Rebut this precisely, and state the real cost that byte-level BPE
   *does* impose for such input. — **Model answer:** The claim is wrong: byte-level
   BPE's base vocabulary is the 256 byte values, and all text is a byte sequence, so
   *any* input is always representable — there is no OOV failure mode. The real cost is
   *efficiency*, not correctness: rare characters or scripts fall back to multiple
   individual byte tokens, so they consume more tokens (more money, latency, context)
   for the same content.
4. A team picks the largest available vocabulary "because shorter sequences are always
   better." Give the two costs they are ignoring and explain why each is a genuine
   downside, not a technicality. — **Model answer:** (a) The embedding and unembedding/
   LM-head matrices scale linearly with vocabulary size, so a very large vocabulary adds
   real parameters and final-softmax compute — bigger model, more memory. (b) A larger
   vocabulary contains more *rare* tokens, each seen less in training and thus updated
   fewer times, so they are represented worse (tail data-sparsity). Both are real:
   vocabulary size is a genuine trade-off, fixed at build time.
5. A developer hand-builds prompt strings instead of using the SDK's messages array,
   and occasionally sees the model treat pasted user content as if it were a system
   instruction. Explain the mechanism, and why using the SDK's message structure
   prevents it. — **Model answer:** The chat template flattens roles into one token
   sequence using special delimiter tokens; the model was post-trained to treat those
   delimiters as turn/role boundaries. If user-pasted text contains the literal
   delimiter strings (e.g. `<|im_start|>system`), a naive string assembler lets them be
   tokenized as *real* turn boundaries, so the model treats injected text as a
   privileged turn — a prompt-injection vector. The SDK applies the template correctly
   and keeps user content as data inside a user turn, so user text cannot forge a
   boundary.
6. Prompt caching is sometimes described as "reuse if the prompt is the same." Make that
   statement precise: same *at what level*, same *which part*, and what is the
   consequence of the first point of difference? — **Model answer:** Same at the
   *token-ID* level (not characters — identical-looking text that tokenizes differently
   does not match). Same in the *prefix*, matched in order from the start. The cache is
   reused only up to the *first differing token*; from that point on, nothing downstream
   can be reused. Hence the static-first/variable-last rule and the need for
   byte-for-byte-deterministic prefix assembly.
7. Explain why the "multilingual tax" is an *equity* issue and not merely a cost issue,
   by naming the non-monetary consequences for a non-English user. — **Model answer:**
   Because tokenizers are English-heavy, the same request in a non-English language
   (especially non-Latin scripts) fragments into far more tokens. Beyond paying more
   money, the non-English user fits *less content* into the same context window, waits
   *longer* (more tokens to prefill), and hits token-based *rate limits sooner* — so the
   quality and capacity of the product itself are worse for them, which is why it is an
   equity concern baked into the tokenizer.
8. At a recognition level, distinguish BPE, WordPiece, Unigram, and SentencePiece in one
   sentence each. — **Model answer:** BPE — merges the most *frequent* adjacent pair;
   byte-level BPE is what frontier LLMs use. WordPiece — also merges, but by *likelihood
   improvement* rather than raw frequency (BERT). Unigram — goes the other way: starts
   with a large vocabulary and *prunes* it down probabilistically. SentencePiece — not an
   algorithm but a library that runs BPE or Unigram, language-agnostic and reversible.

### Long Answer

1. Explain the vocabulary-size trade-off and why two equally good models can have
   different token bills for the same text. — **Model answer / rubric:** Larger vocab →
   more words/phrases are single tokens → shorter sequences → cheaper attention, more
   content per window, better multilingual/code efficiency; but the embedding and
   unembedding matrices grow linearly and rare tokens get under-trained. Smaller vocab →
   leaner matrices but longer sequences. Two models with different vocabularies tokenize
   the same text into different token counts, so identical input costs different
   amounts. Vocabulary is fixed at build time.
2. Walk through four categories of tokenization bug and how each manifests. — **Model
   answer / rubric:** (a) Number splitting — inconsistent, place-misaligned number
   tokens cause arithmetic errors; mitigated by digit-level tokenization, fixed by tools.
   (b) Whitespace sensitivity — leading/trailing spaces, newline styles change tokens;
   degrades prefilled output and busts caching. (c) Glitch tokens — vocabulary tokens
   barely seen in meaningful training contexts (`SolidGoldMagikarp`) with undefined
   representations, producing erratic output. (d) Multilingual tax — non-English,
   especially non-Latin, text fragments into far more tokens, raising cost/latency and
   shrinking effective context. Unifying point: tokenization is a leaky abstraction
   below the prompt that needs token-level debugging.
3. A multilingual RAG product is simultaneously (a) expensive for Japanese users, (b)
   getting poor prompt-cache hit rates, and (c) occasionally producing wrong arithmetic
   in answers. A teammate insists these are three unrelated bugs. Argue instead that all
   three are *tokenization* phenomena, explain the mechanism behind each, and give one
   fix per issue. — **Model answer / rubric:** All three trace to the tokenizer being a
   leaky abstraction below the prompt. (a) Multilingual tax — Japanese (non-Latin
   script) fragments into far more tokens for the same meaning, and billing/limits are
   per-token; fix: a more multilingual model / larger vocabulary, tighter prompts. (b)
   Cache misses — caching keys on the token-ID sequence; if the cacheable prefix is not
   byte-for-byte deterministic (variable data near the top, inconsistent whitespace,
   non-deterministic JSON serialization) the prefix match fails; fix: make the prefix
   deterministic and put static content first, variable last. (c) Arithmetic errors —
   numbers tokenize into inconsistent, place-misaligned fragments, so the model
   pattern-matches rather than computes; fix: a calculator/code tool (and digit-level
   tokenization helps). Strong answers explicitly name the common thread: none are
   visible in the prompt text, all require token-level debugging.

### Applied Scenario

1. Two engineers each independently build the *same* static system prompt for the same
   feature, and both deploy. Cache hit rates are excellent within each engineer's
   traffic but near zero *across* them — a request served by engineer A's build never
   hits the cache warmed by engineer B's build. Nothing is "wrong" with either prompt.
   Diagnose the most likely cause and propose how to prevent it. — **Model answer /
   rubric:** The two builds almost certainly produce *visually identical* prompts that
   tokenize to *different token-ID sequences* — e.g. one uses `\n` and the other
   `\r\n`, different indentation, smart vs. straight quotes, or a different JSON key
   order in tool defs. Since the cache keys on the token-ID sequence, the two prefixes
   never match. Prevent it by defining a single canonical, byte-for-byte-deterministic
   prompt-assembly routine (one source of truth for the static prefix) shared by both
   builds, and verifying by diffing the actual token IDs the two builds emit.
2. A data-extraction feature reliably reads dates and short codes correctly, but a
   "compute the total" step that sums a column of dollar figures is wrong on large
   invoices and fine on small ones. A teammate blames the prompt wording. Give the
   tokenization-based explanation and the correct fix. — **Model answer / rubric:** The
   wording is not the issue. Large dollar figures tokenize into inconsistent,
   place-misaligned number fragments, so the model is pattern-matching over a fragmented
   representation rather than computing — and large sums are sparsely represented in
   training, so there is no pattern to match (small ones are common, hence the
   asymmetry). Rewording will not fix a structural tokenization problem. The fix is to
   do the arithmetic outside the model — a calculator/code tool (Topic 7) — and let the
   model only orchestrate.
3. Your company prices its API feature at a flat per-request rate and was profitable on
   an English-only beta. After launching in Hindi and Arabic markets, margins collapse
   on those markets even though usage volume is similar. Explain why, and explain why
   simply "raising the flat price" is a poor fix. — **Model answer / rubric:** The
   multilingual tax: Hindi and Arabic (non-Latin scripts) fragment into far more tokens
   for the same content, and the provider bills per token — so the same flat-rate
   request costs the company more to serve in those markets. Raising the flat price for
   everyone overcharges English users (who are cheap to serve) and may still mis-price
   relative to the true per-language token cost; it also does not address that
   non-English users get less content per context window and hit rate limits sooner.
   Better: price by actual token usage, or reduce the tax with a more multilingual
   model / tighter prompts.
4. A teammate hand-assembles prompt strings and the model occasionally produces strange
   role confusion after processing user-pasted content. What is likely happening? —
   **Model answer / rubric:** User-supplied text probably contains literal chat-template
   delimiter strings (e.g., `<|im_start|>system`), and the naive string assembler lets
   them be tokenized as real turn boundaries — a prompt-injection vector (Topic 13). The
   model then treats injected content as a privileged turn. Fix: use the SDK's proper
   message structure and template handling, and sanitize/escape user content so it
   cannot be interpreted as structural tokens.
5. A vendor advertises Model X at a 20% lower per-token input price than your current
   Model Y and a teammate wants to switch immediately to cut costs. Your workload is
   ~70% source code and ~30% non-English support tickets. Explain why the 20% headline
   is not enough to decide, and lay out the comparison you would actually run. — **Model
   answer / rubric:** Per-token price alone is misleading: X and Y have different
   tokenizers, so they segment the *same* text into different token counts — and your
   workload (code and non-English) is exactly the kind of content where tokenizer
   efficiency varies most. Model X's 20%-cheaper token could still lose if its tokenizer
   produces, say, 30% more tokens on your code and tickets. The real comparison: take a
   representative sample of your *actual* prompts and completions, run each through
   *each model's own tokenizer*, multiply the resulting input/output token counts by
   each model's respective rates, and compare total cost per request — not the headline
   per-token number. Also weigh quality on your tasks before switching.

---

## Sources

[1] OpenAI — What are tokens and how to count them? — https://help.openai.com/en/articles/4936856-what-are-tokens-and-how-to-count-them
[2] Meta AI — Introducing Meta Llama 3 (tokenizer with a 128K-token vocabulary) — https://ai.meta.com/blog/meta-llama-3/
[3] kith.org (Jed Hartman) — SolidGoldMagikarp and other glitch tokens — https://www.kith.org/words/2023/12/10/solidgoldmagikarp-and-other-glitch-tokens/
[4] Anthropic — Prompt caching (Claude API docs) — https://platform.claude.com/docs/en/build-with-claude/prompt-caching
