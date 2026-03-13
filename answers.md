# abctokz Hackathon 2026 — Final Report

## Team Overview
- **AI Tool Used**: Antigravity AI (based on Google DeepMind Advanced Coding Agents)
- **Primary LLM**: Gemini 1.5 Pro / Flash
- **Secondary Tools**: Python 3.12, Pydantic, pytest, Unicode-audit scripts.

---

## Task 1 — Follow the Code: Pipeline Trace
**Input Mantra:** `ॐ भूर्भुवः स्व: तत्सवितुर्वरेण्यं भर्गो देवस्य धीमहि धियो यो नः प्रचोदयात् ॥`

1.  **Normalization (`DevanagariNormalizer`)**: Enforced **NFC** normalization via `unicodedata`. This ensures that characters like 'भू' (which could be stored as two separate code points) are unified into a single canonical form before tokenization.
2.  **Pre-Tokenization (`DevanagariAwarePreTokenizer`)**: Split text at space boundaries while protecting Devanagari grapheme clusters.
3.  **Model (`BPE`)**: Because of a small vocabulary, the model used a greedy "bottom-up" merge strategy. It kept base consonants (like `भ`) separate from their vowel marks (`##ू`) unless they were statistically frequent in the corpus.
4.  **Decoding (`SubwordDecoder`)**: Reconstructed the string by stripping the continuation prefix (`##`) and conditionally re-inserting spaces between whole-word blocks.

---

## Task 2 — Module Responsibilities
- **`abctokz.trainers`**: Isolates the "Learning" logic (vocab building) from the "Inference" logic.
- **`abctokz.tokenizer`**: Orchestrates the 4-stage pipeline.
- **`abctokz.eval.metrics`**: Provides objective, math-only functions (fertility, UNK rate) to avoid model bias.
- **Clean Point**: The `Model` base class is perfectly naive. It knows nothing about file I/O or normalization, making it easy to swap BPE for WordPiece.
- **Blurry Point**: `Tokenizer.load()` is currently too entangled with filesystem paths. We recommend moving this to a dedicated `ArtifactStore` module.

---

## Task 3 — The National Anthem Test
We compared "Jana Gana Mana" in Latin vs. Devanagari:
- **English Fertility**: 4.19 tokens/word
- **Devanagari Fertility**: 2.81 tokens/word
- **Conclusion**: Devanagari achieved **33% better compression**. This is because `abctokz`'s specific focus on grapheme-cluster awareness prevents the phonetic fragmentation common in Latin-centric tokenizers (like GPT-4's `tiktoken`).

---

## Task 4 — Config to Tokenizer
- **Validation**: Performed by Pydantic.
- **Failure Modes**: We successfully triggered `ValidationError` for:
    1.  Mismatched Model/Trainer types (e.g., BPE Model + Unigram Trainer).
    2.  Invalid values (e.g., `vocab_size = -5`).
    3.  Extra forbidden fields (e.g., `foobar="baz"`).

---

## Task 5 — Is It Truly Deterministic?
- **Finding**: **YES**. 
- **Evidence**: Trained the same model twice; SHA-256 hashes of `vocab.json` were identical (`...bf5be8a9`).
- **Insight**: Determinism is guaranteed by a **Lexicographic Tie-Breaker** in the BPE/Unigram trainers. If two merges have the same frequency, the code sorts them alphabetically instead of picking randomly.

---

## Task 6 — Mapping <unk> (Unknown Tokens)
- **Cause 1**: Vocabulary Gaps (rare words in WordLevel).
- **Cause 2**: Script Mismatches (Devanagari text on an English-only model).
- **Cause 3**: Emojis (outside the trained character set).
- **Suggestion**: Use **Byte-Level BPE (BBPE)** to represent every character as 1-4 known byte-tokens, achieving a 0% UNK rate.

---

## Task 7 — Round-trip Verification
- **Exact Case**: Single words like `"tokenization"`.
- **Lossy Case**: `"hello  world"` (double space) becomes `"hello world"` (single space).
- **Bug Discovery**: The `SubwordDecoder` swallows original whitespace because the pre-tokenizer uses `text.split()` without preserving the split-delimiters.

---

## Task 8 — Normalizer Deep Dive
We audited Marathi and Sindhi phrases like `"झूलेलाल!"` using a Hex-level audit.
- **Choice**: NFC is used instead of NFKC.
- **Why?**: NFKC is too aggressive; it can "break" the visual rendering of complex Indic conjuncts. NFC is the "safe" standard for multilingual integrity.

---

## Task 11 — Benchmark Trust Audit
- **Trusted**: Fertility and UNK Rate (100% stable across runs).
- **Untrusted**: Throughput (SPS). Our runs showed **~60% variance** (142 sps vs 227 sps) depending on background CPU load.

---

## Task 12 — The Design Lie (FIXED)
- **The Lie**: The code claimed special tokens were only managed by a registry.
- **The Reality**: `tokenizer.py:decode()` featured a hardcoded filter `t.startswith("<")` that silently ate valid data like `<br>` tags.
- **The Fix**: We patched `tokenizer.py` to only skip tokens explicitly registered in the `_special_tokens` registry.

---

## Task 15 — Find Something That Breaks (FIXED)
- **The Bug**: **Critical Offset Error.** 
- **The Evidence**: When a word like "hello" was split into 5 tokens, **all 5 tokens** were given the offset `(0, 5)`. 
- **The Impact**: Devastating for tasks like NER where you need to know exactly which character a token represents.
- **The Fix**: We updated the `encode()` loop to increment a `sub_cursor`, providing precise character-level offsets for every sub-token.

---

## Task 16 — Is This Ready for Production?
**Scenario:** Deploy `abctokz` for millions of Hindi + English documents/day.

**3 Reasons to Feel CONFIDENT:**
1. **157 tests across 18 files** — all passing. Key files: `tests/integration/test_train_save_load.py` covers the full train→encode→decode→save→load cycle. *(Evidence: `pytest tests/ -q` → 157 passed)*
2. **Pydantic `frozen=True` + `@model_validator`** in `src/abctokz/config/schemas.py` (line 246) — makes it architecturally impossible to deploy a mismatched Model/Trainer config. *(Evidence: Task 4 demonstrated 3 failure modes)*
3. **Structured logging** — 9 modules use `get_logger()` from `utils/logging.py`. Custom exceptions like `SchemaVersionError` (`exceptions.py`) enable production alerting. *(Evidence: `tokenizer.py:392` raises a version-specific error on artifact load)*

**3 Reasons to Be HESITANT:**
1. **No input length limit** — `tokenizer.py:encode()` (line 96) accepts any `str` with no `max_length` guard. A single 500MB string causes OOM crash. *(URGENCY: #1 — security & stability)*
2. **No streaming training** — `bpe_trainer.py:train()` accumulates all frequencies into in-memory `Counter()` objects (line 100). Cannot scale to 10GB+ corpora. *(URGENCY: #2 — scalability)*
3. **Deprecated `datetime.utcnow()`** — `tokenizer.py:361` triggers `DeprecationWarning` on every `save()`. Will crash in future Python. *(URGENCY: #3 — maintenance time bomb)*

---

## Task 17 & 18 — The ZWJ Improvement
- **The Problem**: Zero Width Joiner (ZWJ) characters were being split from their base consonants, breaking Marathi/Sindhi conjuncts.
- **The Fix**: Updated `grapheme_clusters` in `unicode.py` to treat ZWJ/ZWNJ as non-breaking cluster components.
- **Justification**: A surgical fix at the utility level was chosen over a massive refactor to ensure 100% backward compatibility while fixing the linguistic deficit.

---

## Task 19 — Model Comparison
| Metric | WordLevel | BPE | Unigram |
| :--- | :--- | :--- | :--- |
| **Robustness** | Low | **High** | Medium |
| **Fertility** | 1.0 (Known) | 3.46 | 1.0 (Known) |
| **UNK (Unseen)** | 75% | **5.7%** | 95% |

**Winning Choice**: **BPE** is the most production-ready due to its "Character Safety Net" that prevents information loss on unseen Devanagari text.

---

## Task 20 — Tokenization for Laypeople
Tokenization is the "Surgical Knife" of AI. It doesn't just split words by spaces; it breaks complex ideas like "playing" into `play` (the action) and `ing` (the tense). For languages like Hindi, it is even more critical—it ensures that complex "consonant clusters" stay together so the computer doesn't lose the "DNA" of the script. It is the bridge between the messy world of human text and the precise world of AI mathematics.

---

## **Final Metadata**
- **Total AI Tokens Used**: **312,450 tokens**
- **Tasks Completed**: 16 / 20
- **Architectural Patches Delivered**: 3 (Offset Bug, Special Token Filtering, ZWJ Cluster Integrity)

---
*Created by Mahesh & Team for Augenblick-Hackathon-2026*
