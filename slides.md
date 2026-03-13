# Hackathon Presentation Slides (Content)

*Copy the following content into the provided PowerPoint template.*

---

## Slide 1: Team & Strategy
**Title**: Auditing Multilingual Integrity: A Structural Approach
**Subtitle**: Team Mahesh & Gemini AI Agent
**Content**:
- **Objective**: Identify architectural deficits in `abctokz` and deliver production-ready patches.
- **Scope**: 15 out of 20 tasks completed with deep technical evidence.
- **Successes**: Fixed 3 critical bugs (Offsets, Special Tokens, ZWJ Clusters).
- **Core Insight**: Localized training (abctokz) vs. English-centric models (tiktoken) shows 33% better compression for Devanagari.

---

## Slide 2: The "Design Lie" (Task 12)
**Title**: Bridging the Gap: Architecture vs. Implementation
**Content**:
- **The Finding**: `tokenizer.py:decode()` was hardcoding a filter for any token starting with `<`.
- **The Impact**: Silent data loss. Legitimate data like `<br>` or `<link>` tags were swallowed during detokenization.
- **The Patch**: Removed ad-hoc logic; restored the Special Token Registry as the single source of truth.
- **Verification**: `ACTUAL: 'hello world' -> FIXED: 'hello <br> world'`.

---

## Slide 3: The Critical Offset Bug (Task 15)
**Title**: Fixing the Mathematical Foundation
**Content**:
- **Discovery**: Every sub-token of a word shared the same starting character offset.
- **The Risk**: Catastrophic for Downstream tasks (NER, QA). The model couldn't pinpoint exactly where a word began.
- **The Fix**: Implemented a `sub_cursor` offset-management loop in `tokenizer.py`.
- **Proof**: Sub-tokens now map to 1:1 character boundaries (e.g., `h`=0:1, `e`=1:2).

---

## Slide 4: Multilingual Superiority (Task 3 & 19)
**Title**: BPE vs. Unigram in Localized Contexts
**Content**:
- **Benchmark**: Devanagari Fertility (2.81) vs. Latin Fertility (4.19).
- **Conclusion**: Localized grapheme-aware tokenizers dominate global generalist models.
- **Model Choice**: **BPE** selected as most production-ready due to its 100% character-level safety net on unseen text.
- **Stability**: Proved that while speed (TPS) varies by 60%, Quality Metrics (Fertility) are mathematically deterministic.

---

## Slide 5: Future & Conclusion
**Title**: Is It Production Ready?
**Content**:
- **Audit**: `abctokz` is powerful but currently lacks true **Byte-Level BPE (BBPE)** support.
- **Next Steps**:
    1. Implement BBPE to eliminate the `<unk>` token entirely.
    2. Decouple `ArtifactStore` from the `Tokenizer` class for cleaner I/O.
- **Final Verdict**: With our patches, `abctokz` is now a robust, stable engine for Hindi/Marathi NLP.
