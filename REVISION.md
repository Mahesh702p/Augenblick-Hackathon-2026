# FINAL REVISION — abctokz Hackathon Cheat Sheet

---

## PART 1: KEY CONCEPTS

### What is a Token?
The smallest unit that an AI model works with. The model never sees text — it sees numeric IDs. Each token maps to an ID (e.g., "hello" = ID 42).

### What is Tokenization?
Converting raw text → tokens → numeric IDs. It's 4 stages in abctokz:
```
Raw Text → [Normalizer] → [PreTokenizer] → [Model] → [Decoder]
```

### What is BPE (Byte Pair Encoding)?
- **Direction**: Bottom-up
- **How it works**: Starts with individual characters. Finds the most frequent adjacent pair → merges them into one token. Repeats until vocab size is reached.
- **Strength**: Never produces <unk> for known characters — always falls back to character level.
- **Weakness**: High fertility (lots of small pieces) when vocab is small.

### What is Unigram?
- **Direction**: Top-down
- **How it works**: Starts with a huge vocabulary. Uses probability (EM algorithm) to prune the least useful tokens until target vocab size.
- **Strength**: Cleaner segmentations on known text.
- **Weakness**: Aggressive <unk> on unseen text because it may prune too many base characters.

### What is WordLevel?
- **Direction**: Dictionary lookup
- **How it works**: Every word is either in the vocab (whole) or <unk>.
- **Strength**: Simple and fast.
- **Weakness**: Completely useless for unseen words. No subword capability.

### What is Fertility?
```
Fertility = total_tokens / total_words
```
Lower = better compression. Fertility 1.0 means every word stayed whole.

### What is <unk>?
The "Unknown" token. Produced when the model encounters something it can't represent. BPE rarely produces it, WordLevel produces it for any new word.

### What is the ## prefix?
Continuation marker. `##ing` means "this piece is glued to the previous token, not a new word." Defined in `config/defaults.py` as `BPE_CONTINUATION_PREFIX = "##"`.

### What is NFC Normalization?
Unicode Canonical Composition. Combines decomposed characters (base + matra stored separately) into a single code point. Example: `क` + `ा` (2 code points) → `का` (1 code point). The library uses NFC, not NFKC, because NFKC is too aggressive for Indic scripts.

### What is ZWJ / ZWNJ?
- **ZWJ (Zero Width Joiner)**: Forces a half-form. `क + ् + ZWJ = क्‍` (half-Ka)
- **ZWNJ (Zero Width Non-Joiner)**: Prevents a ligature. `क + ् + ZWNJ + ष = क्‌ष` (separate, not conjunct)
- Both are invisible characters that carry meaning. The normalizer preserves them (`strip_zero_width=False`).

### What is a Grapheme Cluster?
A group of Unicode code points that form one visible character. Example: `भू` = `भ` (base) + `ू` (matra) = 1 cluster, 2 code points. The `DevanagariAwarePreTokenizer` keeps these together.

### What is Pydantic?
Python validation library used for configs. `frozen=True` means configs can't be changed after creation. `extra="forbid"` means unknown fields are rejected.

### What is Round-Trip?
`text → encode() → decode() → result`. Ideally result == text. In abctokz, it fails on double spaces because `str.split()` swallows whitespace.

### What is an Offset?
Character position mapping. Token "h" at offset (0,1) means it came from characters 0-1 in the original text. Critical for NER (Named Entity Recognition) and highlighting.

---

## PART 2: THE 4-STAGE PIPELINE

```
"ॐ भूर्भुवः"
     │
     ▼
[1. NORMALIZER]  ──→  NFC normalization, fixes Unicode inconsistencies
     │                  File: src/abctokz/normalizers/devanagari.py
     ▼
[2. PRE-TOKENIZER] ──→  Splits at spaces, keeps grapheme clusters together
     │                   File: src/abctokz/pretokenizers/devanagari_aware.py
     ▼
[3. MODEL (BPE)]  ──→  Breaks words into subword pieces using merge rules
     │                  File: src/abctokz/models/bpe.py
     ▼
[4. DECODER]  ──→  Converts IDs back to text, strips ## prefix, adds spaces
                    File: src/abctokz/decoders/subword_decoder.py
```

---

## PART 3: RESULTS SUMMARY TABLE

| Task | Title | Key Result |
|---|---|---|
| **1** | Pipeline Trace | Traced mantra through all 4 stages. Normalizer did NFC, PreTokenizer split at spaces, BPE fragmented rare words, Decoder stripped ## and rejoined. |
| **2** | Module Mapping | Clean boundary: `Model` class knows nothing about I/O. Blurry boundary: `Tokenizer.load()` is entangled with filesystem paths. |
| **3** | National Anthem | Hindi fertility 2.94 vs English 4.06. tiktoken (GPT-4) gets 5.31 on Hindi. **abctokz beats GPT-4 on Hindi.** |
| **4** | Config Trace | Pydantic catches 3 failure modes: mismatched model/trainer, invalid values, unknown fields. |
| **5** | Determinism | YES — SHA-256 hashes identical across runs. Seed is unused. Tie-breaking is lexicographic. |
| **6** | UNK Mapping | WordLevel = most fragile (any new word → unk). BPE = most robust (character fallback). Suggested fix: Byte-Level BPE. |
| **7** | Round-Trip | Fails on double spaces. `str.split()` swallows whitespace. NFC normalization changes bytes but not visuals. |
| **8** | Normalizer | NFC used, not NFKC. NFKC breaks Indic conjuncts. ZWJ audit proved `strip_zero_width=False` is vital. |
| **11** | Benchmark Trust | Fertility: 100% stable. Throughput: ~60% variance (142 vs 227 sps). Speed metrics are unreliable. |
| **12** | Design Lie (FIXED) | `decode()` had hardcoded `t.startswith("<")` filter. Silently ate HTML like `<br>`. Fixed to only use registry. |
| **15** | Offset Bug (FIXED) | All sub-tokens got same offset (0,5). Fixed with `sub_cursor` — now each token gets precise boundaries. |
| **16** | Production Audit | Confident: 157 tests, Pydantic, logging. Hesitant: No input limit (OOM), no streaming, deprecated `utcnow()`. |
| **17** | ZWJ Fix | `grapheme_clusters()` was splitting ZWJ. 1-line fix: added `or is_zero_width(char)` to the condition. |
| **18** | Why Minimal Fix | Surgical fix chosen over full UAX #29 refactor. Zero regression risk. 100% backward compatible. |
| **19** | Model Comparison | BPE: 5.7% UNK on unseen. Unigram: 95% UNK. WordLevel: 75% UNK. BPE is the safest choice. |
| **20** | Plain English | Tokenization = surgical knife for AI. Subword models balance vocab size with infinite coverage. |

---

## PART 4: THE 3 BUGS WE FIXED

### Bug 1: Sub-Token Offset Error (Task 15)
- **Where**: `tokenizer.py:encode()`, line 134
- **What**: All sub-tokens of "hello" got offset (0,5) instead of (0,1), (1,2), etc.
- **Why it happened**: Code used `len(pre_tok)` (whole word length) instead of `len(tok_str)` (individual piece)
- **Our fix**: Added a `sub_cursor` variable that increments by each piece's length
- **Impact**: Critical for NER, QA, text highlighting

### Bug 2: Special Token Filter Leak (Task 12)
- **Where**: `tokenizer.py:decode()`, line 188
- **What**: Code had `t.startswith("<") and t.endswith(">")` that silently removed valid tokens like `<br>`
- **Why it happened**: Hardcoded pattern matching instead of checking the `_special_tokens` registry
- **Our fix**: Removed the pattern check. Now only tokens in the registry are skipped.
- **Impact**: Data loss in HTML/XML processing

### Bug 3: ZWJ Cluster Fragmentation (Task 17)
- **Where**: `utils/unicode.py:grapheme_clusters()`
- **What**: ZWJ characters were treated as new clusters instead of being attached to the previous character
- **Why it happened**: Condition only checked `is_combining(char)`, not `is_zero_width(char)`
- **Our fix**: Added `or is_zero_width(char)` to the condition
- **Impact**: Marathi/Sindhi half-forms were broken

---

## PART 5: KEY FILES TO KNOW

| File | What it Does |
|---|---|
| `src/abctokz/tokenizer.py` | The main orchestrator. Runs the 4-stage pipeline. Contains `encode()`, `decode()`, `train()`, `save()`, `load()`. |
| `src/abctokz/models/bpe.py` | BPE algorithm. Greedy merge lookup from vocab. |
| `src/abctokz/trainers/bpe_trainer.py` | Counts character pair frequencies, merges the most common, repeats. Lexicographic tie-breaking on line 171. |
| `src/abctokz/config/schemas.py` | Pydantic models. `frozen=True`, `extra="forbid"`, `@model_validator` for trainer/model alignment. |
| `src/abctokz/config/defaults.py` | Preset configs like `bpe_multilingual()`. Stores constants like `BPE_CONTINUATION_PREFIX = "##"`. |
| `src/abctokz/normalizers/devanagari.py` | NFC normalization. `strip_zero_width=False` by default. |
| `src/abctokz/pretokenizers/devanagari_aware.py` | Splits at spaces. Keeps grapheme clusters together. Uses `text.split()` (which is the source of the space-loss bug). |
| `src/abctokz/decoders/subword_decoder.py` | Converts IDs → text. Strips `##` prefix. Adds spaces between non-continuation tokens. |
| `src/abctokz/eval/metrics.py` | Pure math functions: `fertility()`, `unk_rate()`, `round_trip_success_rate()`. |
| `src/abctokz/eval/benchmark.py` | `BenchmarkRunner`. Runs timed encoding loops. Averages elapsed time. |
| `src/abctokz/utils/unicode.py` | `grapheme_clusters()` function. This is where the ZWJ bug was. |
| `src/abctokz/adapters/hf.py` | Wraps HuggingFace tokenizers behind abctokz's interface. |

---

## PART 6: NUMBERS TO REMEMBER

| What | Value |
|---|---|
| Total test functions | 157 |
| Total test files | 18 |
| Tasks completed | 16 / 20 |
| Bugs fixed | 3 |
| abctokz Hindi fertility | **2.94** |
| tiktoken Hindi fertility | **5.31** |
| abctokz English fertility | 4.06 |
| tiktoken English fertility | 2.12 |
| tiktoken vocab size | ~100,000 |
| abctokz vocab size (our experiment) | 400 |
| Benchmark throughput variance | ~60% between runs |
| Fertility variance | 0% (deterministic) |
| BPE UNK on unseen text | 5.7% |
| Unigram UNK on unseen text | 95% |
| WordLevel UNK on unseen text | 75% |

---

## PART 7: TASKS WE SKIPPED & WHY

| Task | Title | Why We Skipped |
|---|---|---|
| 9 | Adapter Deep Dive | Required HuggingFace + SentencePiece setup. Time-intensive for low insight gain. |
| 10 | Benchmark External | Dependent on Task 9. We used tiktoken directly instead in Task 3. |
| 13 | Edge Case Hunt | Covered by Tasks 7. 12, and 15 — we found real bugs instead of theoretical edge cases. |
| 14 | Config Ablation | Interesting but low-impact. Config validation was already proven robust in Task 4. |

**If a judge asks "Why not all 20?":**
> *"We focused on depth over breadth. Instead of shallow answers for 20 tasks, we did deep analysis on 16 — including finding and patching 3 actual bugs. We believe understanding 16 tasks thoroughly is more valuable than surface-level answers for all 20."*
