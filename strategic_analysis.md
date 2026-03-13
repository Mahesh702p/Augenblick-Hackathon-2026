# Strategic Analysis — All 20 Hackathon Tasks

> **Time available**: ~8–9 hours | **Key insight**: Quality > Quantity | **Evaluation**: 20% each for comprehension, reasoning, task extent, presentation, punctuality

---

## Priority Execution Order

| Priority | Tasks | Why | Est. Time |
|----------|-------|-----|-----------|
| 🔴 Quick wins | Fix typos (README, TASKS.md), Task 20 | Shows attention to detail, easy points | 30 min |
| 🔴 Must-do | Task 1, 2, 4 | Core understanding demos — judge will cross-question on these | 2 hrs |
| 🟠 High-value | Task 12, 17+18, 7 | Bug-finding + fix + justification — shows engineering maturity | 2 hrs |
| 🟡 Strong demos | Task 5, 8, 19 | Experimental evidence + reasoning | 2 hrs |
| 🟢 If time | Task 3, 6, 9, 10, 11, 13, 14, 15, 16 | Good for depth but not essential | Remaining |

---

## Task-by-Task Deep Analysis

---

### Task 1 — Follow the Code: Tokenization Pipeline Trace
**What it asks**: Train BPE on a small corpus, encode a Sanskrit mantra, and trace the exact pipeline path through the code.

**Why it exists**: This is the **most fundamental task**. It tests whether you understand the 4-stage pipeline — the backbone of the entire system. Every other task builds on this understanding.

**What concept it tests**: Pipeline architecture comprehension, ability to trace execution flow.

**Where in the codebase**:
- [tokenizer.py:93-145](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py) — [encode()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/adapters/hf.py#63-88) method
- [normalizers/devanagari.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/normalizers/devanagari.py) — NFC normalization
- [pretokenizers/devanagari_aware.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/pretokenizers/devanagari_aware.py) — script-boundary splitting
- [models/bpe.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/models/bpe.py) — [tokenize()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py#436-438) method
- [decoders/subword_decoder.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/subword_decoder.py) — decode path

**How to solve**:
1. Train BPE on a corpus including Devanagari text (use the example script or the conftest data)
2. Encode `ॐ भूर्भुवः स्व: तत्सवितुर्वरेण्यं भर्गो देवस्य धीमहि धियो यो नः प्रचोदयात् ॥`
3. At each stage, show the input and output:
   - Raw text → Normalizer → what changed? (NFC, ZWJ handling)
   - Normalized text → PreTokenizer → what are the pre-tokens?
   - Each pre-token → Model → what token IDs came out?
   - Token IDs → Decoder → reconstructed text
4. Name the exact file + class + method at each stage

**Key observations to prepare for judge's questions**:
- The Sanskrit text contains `ॐ` (sacred syllable), halant-conjuncts (`र्भु`, `स्व`), visarga (`:`), and `॥` (double danda) — all are Unicode edge cases
- The normalizer uses NFC, which is important because NFKC would decompose some of these
- The pretokenizer splits on script boundaries — but `॥` and `:` are punctuation, not Devanagari

**Risk**: The `:` in `स्व:` is ASCII colon, not Devanagari visarga `ः`. This may cause unexpected splitting.

---

### Task 2 — Module Responsibilities Mapping
**What it asks**: Map 5 responsibilities to their modules + identify one clean boundary and one blurry one.

**Why it exists**: Tests architectural comprehension. The judge wants to see you understand *why* the code is structured this way, not just *what* each file does.

**Concept**: Separation of concerns, module cohesion, coupling analysis.

**Quick mapping**:
| Responsibility | Module(s) |
|---|---|
| Training | `trainers/` (BPETrainer, UnigramTrainer, WordLevelTrainer) |
| Encoding | [tokenizer.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py) orchestrates [normalizers/](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/normalizers/sequence.py#30-34) → [pretokenizers/](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/pretokenizers/sequence.py#34-38) → `models/` → `processors/` |
| Save/Load | [tokenizer.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py) + [vocab/serialization.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/vocab/serialization.py) + [data/manifest.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/data/manifest.py) |
| Quality metrics | [eval/metrics.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/eval/metrics.py) + [eval/intrinsic.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/eval/intrinsic.py) |
| External comparison | [adapters/hf.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/adapters/hf.py) + [adapters/sentencepiece.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/adapters/sentencepiece.py) |

**Clean boundary example**: [models/base.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/models/base.py) — the [Model](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/models/base.py#10-78) ABC is purely about [tokenize()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py#436-438)/[get_vocab()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/models/wordlevel.py#53-56). It has zero knowledge of training, normalization, or I/O. This makes the model swappable without touching anything else.

**Blurry boundary example**: [tokenizer.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py)'s [decode()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/subword_decoder.py#50-89) method does its **own** special-token filtering (line 189) using a regex-like heuristic (`<...>`), even though the [Decoder](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/base.py#9-30) classes ([SubwordDecoder](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/subword_decoder.py#10-89), [WordDecoder](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/word_decoder.py#9-43)) *also* have `skip_special_tokens` logic. This is a responsibility split across two layers.

---

### Task 3 — National Anthem Test
**What it asks**: Encode the Indian national anthem in both English transliteration and Devanagari, compare token counts and fertility.

**Concept**: Script-dependent tokenization efficiency, vocabulary coverage.

**Where**: [eval/metrics.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/eval/metrics.py) → [fertility()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/eval/metrics.py#9-29) function.

**How to solve**:
1. Train a tokenizer on a mixed EN+HI corpus
2. Encode both versions
3. Compute fertility = tokens / whitespace-split words
4. Devanagari will likely have higher fertility because each grapheme cluster becomes multiple character-level tokens in a small vocab
5. **Bonus**: Compare with `tiktoken` (GPT-4's tokenizer) — it will show that production tokenizers trained on massive corpora have much lower fertility for common scripts

---

### Task 4 — Config → Tokenizer Trace
**What it asks**: Trace from Pydantic config to working tokenizer, show where defaults come from, demonstrate 2 failure modes.

**Concept**: Configuration validation, factory pattern, Pydantic validators.

**Where**:
- [config/schemas.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/config/schemas.py) — all config classes + [check_trainer_model_alignment](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/config/schemas.py#246-254) validator
- [config/defaults.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/config/defaults.py) — preset builders
- [normalizers/__init__.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/normalizers/__init__.py) — [build_normalizer()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/normalizers/__init__.py#20-45) factory
- [pretokenizers/__init__.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/pretokenizers/__init__.py) — [build_pretokenizer()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/pretokenizers/__init__.py#20-46) factory
- [trainers/__init__.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/trainers/__init__.py) — [build_trainer()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/trainers/__init__.py#11-30) factory

**Failure modes to demonstrate**:
1. **Mismatched model+trainer**: [BPEConfig](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/config/schemas.py#153-164) model + [UnigramTrainerConfig](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/config/schemas.py#209-222) → caught by [check_trainer_model_alignment](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/config/schemas.py#246-254)
2. **Invalid vocab_size**: `vocab_size=0` or negative → Pydantic validation
3. **Extra fields**: [TokenizerConfig(foo="bar")](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/config/schemas.py#232-254) → caught by `extra="forbid"`

---

### Task 5 — Determinism Verification
**What it asks**: Experimentally verify that training is deterministic.

**Concept**: Reproducibility, seed management, tie-breaking.

**Where**:
- [utils/seeds.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/utils/seeds.py) — [set_seed()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/utils/seeds.py#9-25)
- All trainers call [set_seed(config.seed)](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/utils/seeds.py#9-25) in [__init__](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/processors/special_tokens.py#28-39)
- BPE trainer: lexicographic tie-breaking at line 171
- Unigram trainer: sorted piece list at line 277-280

**How to solve**:
1. Train same config twice on same corpus → compare `vocab.json` / `merges.txt` / `pieces.json` (should be byte-identical)
2. Encode same text → compare IDs (should be identical)
3. **What's NOT deterministic**: benchmark **throughput** (wall-clock timing varies), benchmark **ordering** of results

**Key insight for judge**: Determinism in this codebase is achieved through 3 mechanisms: (a) explicit seed, (b) lexicographic tie-breaking, (c) sorted vocabulary construction. If any of these three were removed, determinism would break.

---

### Task 6 — Making the Tokenizer Say "I Don't Know"
**What it asks**: Find inputs that produce `<unk>`, explain why.

**Concept**: OOV handling, model-family differences in robustness.

**Key insight**:
- **WordLevel**: ANY word not seen in training → `<unk>`. Most fragile.
- **BPE**: Character-level alphabet means single unseen characters → `<unk>`. E.g., train on English only, encode Devanagari.
- **Unigram**: Similar to BPE — unknown characters fall back to `unk_score`.

**Suggestion to reduce UNK without retraining**: Expand the initial alphabet or use a byte-fallback mechanism (like BBPE).

---

### Task 7 — Round-Trip Encode/Decode
**What it asks**: Find one exact round-trip and one lossy round-trip.

**Concept**: Losslessness, normalization lossyness, decoder correctness.

**Critical finding from our test run**: Even `"hello world"` fails round-trip — decoded as `"helloworld"` (space lost!). This is because:
1. The BPE model treats space as `##⎵` (a character with continuation prefix)
2. The [SubwordDecoder](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/subword_decoder.py#10-89) in BPE mode inserts spaces only when a token **lacks** the `##` prefix — but if space itself is prefixed with `##`, it gets treated as part of the word

This is a **real bug** that directly answers Task 7 and Task 12.

**Exact round-trip example**: ASCII-only words like `"tokenization"` → works when the full word exists or character-level merge gives correct result.

---

### Task 8 — Normalizer Deep Dive
**What it asks**: Show normalizer transformation on specific Sindhi and Marathi phrases.

**Where**: [normalizers/devanagari.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/normalizers/devanagari.py) — [normalize()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/normalizers/base.py#23-33) method.

**Key observations**:
- NFC normalization composes decomposed sequences (e.g., `ा` + combining marks)
- Whitespace normalization collapses multiple spaces, strips exotic whitespace
- Commas and exclamation marks are **not** affected by the normalizer — they pass through
- But the **pretokenizer** (PunctuationPreTokenizer if used) would split on them

---

### Task 9 — Measuring Phrase Difficulty
**What it asks**: Fertility of Task 8 phrases across BPE/Unigram at different vocab sizes.

**Concept**: Vocabulary size ↔ fertility trade-off.

**Expected resultado**: Higher vocab → lower fertility. Unigram typically gives slightly lower fertility than BPE at the same vocab size (because Viterbi finds globally optimal segmentation vs BPE's greedy merge).

---

### Task 10 — Compression Trade-off
**What it asks**: Find a config change that improves one metric but hurts another.

**Classic example**: Increase [vocab_size](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/adapters/hf.py#114-119) from 500 to 8000:
- ✅ Fertility improves (fewer tokens)
- ❌ UNK rate may increase if `min_frequency` is too high
- ❌ Memory footprint increases
- Trade-off: compression efficiency vs memory/generalization

---

### Task 11 — Benchmark Trustworthiness
**What it asks**: Run benchmark twice, compare.

**Stable metrics**: Fertility, UNK rate, round-trip success, mean tokens/sentence (deterministic — same tokenizer, same input).
**Variable metrics**: Throughput (wall-clock timing depends on system load).

**Subtle design issue**: In [benchmark.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/cli/benchmark.py) line 87, `all_encodings = encodings` keeps only the **last** timed run's encodings. Quality metrics are from a single run, not averaged. This is actually correct (they'd be identical anyway), but the code is confusing.

---

### Task 12 — Where the Design Lies
**What it asks**: Find a gap between architectural intent and actual behavior.

**Answer**: The [decode()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/subword_decoder.py#50-89) method in [tokenizer.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py) (line 189):
```python
tokens = [t for t in tokens if not (t.startswith("<") and t.endswith(">"))]
```
This **claims** to skip special tokens, but it uses a heuristic `<...>` pattern instead of checking the actual registered special tokens list. This means:
- Legitimate vocab entries like `<br>`, `<3>` would be silently dropped
- Meanwhile, the [SubwordDecoder](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/subword_decoder.py#10-89) and [WordDecoder](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/word_decoder.py#9-43) classes each have their own `skip_special_tokens` parameter — a **redundant, inconsistent** mechanism

---

### Task 13 — Predict Then Verify
**What it asks**: Predict the effect of a normalization/pretokenization change, then test it.

**Good change to try**: Switch Devanagari normalizer from NFC to NFKC.
- **Prediction**: Some Devanagari characters get decomposed differently, golden tests may fail, round-trip success rate may decrease for Devanagari text
- **Verify**: Run golden tests and compare

---

### Task 14 — Adding a Fourth Model
**What it asks**: Plan (don't implement) adding WordPiece or another model.

**Files to create**: `models/wordpiece.py`, `trainers/wordpiece_trainer.py`
**Files to modify**: [models/__init__.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/models/__init__.py), [trainers/__init__.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/trainers/__init__.py), [config/schemas.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/config/schemas.py), [config/defaults.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/config/defaults.py), [tokenizer.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py) (load/save logic)
**Files to leave untouched**: [normalizers/](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/normalizers/sequence.py#30-34), [pretokenizers/](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/pretokenizers/sequence.py#34-38), `decoders/`, [eval/](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/eval/intrinsic.py#17-60), `utils/`, [data/](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/types.py#60-110)

**Biggest obstacle**: [tokenizer.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py)'s [load()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py#442-445) and [save()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py#439-441) methods have model-type-specific branching (e.g., "if BPE, load merges"). This would need updating.

---

### Task 15 — Find Something That Breaks
**What it asks**: Find an edge case.

**Already found**: Empty string → no crash but empty Encoding. Space-only → normalizer strips it. Emoji → all `<unk>`. The decode space-loss bug from Task 7.

---

### Task 16 — Production Readiness
**What it asks**: Audit for production deployment.

**Confident**: (1) Deterministic training, (2) Clean abstractions, (3) Pydantic validation
**Hesitant**: (1) No thread-safety, (2) Space-loss in BPE decode, (3) No streaming/batched encode for large inputs, (4) No byte-fallback for truly unknown characters

---

### Task 17+18 — Small Improvement + Justification
**What it asks**: Fix one thing. Justify why small.

**Best candidate**: Fix the [decode()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/decoders/subword_decoder.py#50-89) heuristic in [tokenizer.py](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/tokenizer.py) — replace the `<...>` pattern check with actual special tokens list lookup. This is:
- One-line change
- Clearly a bug (Task 12 evidence)
- Minimal risk
- Easy to test

---

### Task 19 — Model Comparison
**What it asks**: Side-by-side tokenization from all 3 models.

**How**: Train all 3 on same corpus/vocab_size → encode 5 sentences → compare.

---

### Task 20 — Plain English Explanation
**What it asks**: 300-word explanation of tokenization for non-NLP person.

**Key points**: Why [split()](file:///home/mahesh/Desktop/CodeBase/AUGENBLICK-HACKATHON-2026/src/abctokz/pretokenizers/punctuation.py#71-104) isn't enough, subword units, multilingual challenges, vocabulary trade-offs.

---

## Typos Found

### TASKS.md
- Line 56: `"You must report the report the following"` → duplicate "report the"
- Line 81: `"Presentaion clarity"` → "Presentation clarity"
- Line 109: `"PENLIZED"` → "PENALIZED"

### README.md
- Line 13: `pip install -e .` instructions say `cd tokenizer_repo`, should be `cd AUGENBLICK-HACKATHON-2026`

These should be fixed and committed for attention-to-detail points.
