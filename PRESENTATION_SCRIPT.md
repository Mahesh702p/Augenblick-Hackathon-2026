# PRESENTATION SCRIPT — 8 Minutes

> **Team**: Mahesh, Astha, Arzoo
> **Task Split**:
> - Astha → 2, 3, 6, 16, 18, 19
> - Mahesh → 1, 5, 7, 8, 15
> - Arzoo → 4, 11, 12, 17, 20

---

## OPENING — Mahesh (1.5 min)

### "What we understood about the codebase"

> "abctokz is a tokenizer library built for multilingual text — especially Hindi, Marathi, and Sindhi. It converts raw text into numeric tokens that AI models can process.
>
> The core architecture is a **4-stage pipeline**:
> 1. **Normalizer** — fixes Unicode inconsistencies, does NFC composition
> 2. **Pre-Tokenizer** — splits text at word boundaries while keeping Devanagari grapheme clusters intact
> 3. **Model** — the actual algorithm (BPE, Unigram, or WordLevel) that breaks words into subword pieces
> 4. **Decoder** — converts token IDs back to readable text
>
> The library supports 3 model types. BPE builds vocabulary bottom-up by merging frequent pairs. Unigram starts with a large vocabulary and prunes it top-down. WordLevel is a simple dictionary lookup.
>
> There are about 73 source files, 157 unit tests, and the whole config system is built on Pydantic with strict validation."

### "How we approached the hackathon"

> "We started by reading every module to understand how data flows through the system. Then we categorized all 20 tasks by difficulty and impact. We decided to focus on **16 tasks** — prioritizing the ones that demonstrate deep understanding: tracing the pipeline, finding real bugs, and running experiments with actual numbers.
>
> We split the tasks based on each person's strength. Let me walk through what each of us found."

---

## MAHESH'S TASKS (2 min) — Tasks 1, 5, 7, 8, 15

### Task 1 — Pipeline Trace
> "I traced a Sanskrit mantra through all 4 stages. The Normalizer applied NFC — it doesn't visibly change the text but unifies how characters are stored internally. The Pre-Tokenizer split at spaces while protecting Devanagari clusters. BPE fragmented rare words like 'भूर्भुवः' into 6 pieces because our vocab was only 300. The Decoder stripped the ## prefix and rejoined everything."

### Task 5 — Determinism
> "I tested whether training the same model twice gives identical results. It does — the SHA-256 hashes of vocab.json were bit-identical. The reason is that BPE uses **lexicographic tie-breaking** — when two merges have equal frequency, it picks alphabetically. There's no randomness at all. The seed parameter in the config is actually unused."

### Task 7 — Round-Trip
> "I tested encode → decode to see if we get back the original text. Single words work perfectly. But 'hello  world' with two spaces became 'hello world' with one space. The bug is in the pre-tokenizer — it uses Python's `str.split()` which discards all whitespace."

### Task 8 — Normalizer
> "I audited the normalizer with Marathi and Sindhi phrases. The library uses NFC, not NFKC, because NFKC is too aggressive for Indic scripts — it can break conjunct forms. I also proved that the `strip_zero_width=False` default is critical. If you strip Zero Width Joiners, Marathi half-forms like half-Ka get visually corrupted."

### Task 15 — Offset Bug (FIXED)
> "This was a critical bug. When BPE splits 'hello' into 5 character tokens, all 5 were assigned the offset (0,5) — pointing to the entire word instead of individual characters. This completely breaks NER and text highlighting. I fixed it by adding a `sub_cursor` variable in `tokenizer.py` that tracks each sub-token's position independently. After the fix, each token gets its precise character boundary."

---

## ASTHA'S TASKS (2 min) — Tasks 2, 3, 6, 16, 18, 19

### Task 2 — Module Mapping
> "I mapped which module handles what. The cleanest boundary is the `Model` base class — it knows nothing about files, normalization, or even spaces. It just takes a pre-tokenized string and returns subword pieces. The messiest boundary is `Tokenizer.load()` — it's doing filesystem management, JSON parsing, and class instantiation all in one method. Ideally that should be a separate `ArtifactStore` class."

### Task 3 — National Anthem Comparison
> "I compared the Hindi and English versions of Jana Gana Mana. Hindi got fertility 2.94 — English got 4.06. Hindi is 33% more efficient because each Devanagari character carries more phonetic information than a Latin letter. We also ran it through **tiktoken, GPT-4's tokenizer**. tiktoken scored 5.31 fertility on Hindi — it literally broke Hindi into raw bytes. So abctokz with just 400 vocab tokens **beats GPT-4 with 100,000 vocab tokens** on Hindi. Localized training wins."

### Task 6 — UNK Handling
> "I mapped when each model produces the unknown token. WordLevel produces `<unk>` for any unseen word — total information loss. BPE almost never produces `<unk>` because it falls back to character-level pieces. The recommended fix is Byte-Level BPE — encode everything as UTF-8 bytes so there are only 256 possible base tokens, giving 0% UNK rate."

### Task 16 — Production Readiness
> "I did an honest audit. Three things give confidence: the 157-test suite, Pydantic's strict validation, and structured logging across 9 modules. Three gaps: there's no input length limit in `encode()` so a massive string could crash the pipeline, training loads the entire corpus into memory with no streaming, and `tokenizer.py` line 361 uses `datetime.utcnow()` which is deprecated in Python 3.12."

### Task 19 — Model Comparison
> "I compared BPE, Unigram, and WordLevel on unseen text. BPE had only 5.7% UNK rate. Unigram had 95% — it was catastrophic because it pruned too many base characters during training. WordLevel had 75% UNK. BPE is clearly the safest choice for production, especially for multilingual text."

### Task 18 — Why Minimal Fix
> "For the ZWJ fix that Arzoo will explain — I justified why we kept it small. A full Unicode overhaul would risk regressions across Latin, Cyrillic, and Arabic scripts. Our one-line change affects only how characters group together — nothing else changes. Zero backward compatibility risk."

---

## ARZOO'S TASKS (2 min) — Tasks 4, 11, 12, 17, 20

### Task 4 — Config Trace
> "I traced how a Python config becomes a working tokenizer. Everything goes through Pydantic validation first. I demonstrated 3 failure modes: pairing a BPE model with a Unigram trainer raises a `ValidationError`, setting `vocab_size=-5` is caught by the `ge=1` constraint, and passing an unknown field like `foobar='baz'` is rejected because `extra='forbid'`. The config is frozen after creation — you can't mutate it during training."

### Task 11 — Benchmark Trust
> "I ran the benchmark twice on the same model and compared results. Fertility was **identical** both times — it's purely mathematical, so it never changes. But throughput jumped from 142 to 227 sentences per second — a 60% difference just from background CPU load. So quality metrics like fertility are fully trustworthy, but speed metrics should only be treated as rough estimates."

### Task 12 — Design Lie (FIXED)
> "I found an architectural violation in the decode method. The code claims that special tokens are managed through a registry, but there was a hardcoded line checking `t.startswith('<') and t.endswith('>')`. This silently deleted any token that looked like an HTML tag — even if it was legitimate data like `<br>`. If you tokenized HTML, the tags would just disappear during decoding. I fixed it by removing the pattern check and only respecting the official `_special_tokens` registry."

### Task 17 — ZWJ Fix
> "I found a bug in the `grapheme_clusters()` function in `utils/unicode.py`. It was treating Zero Width Joiners as new cluster boundaries instead of attaching them to the previous character. This broke Marathi half-forms — 'half-Ka' was getting split into two pieces. The fix was one line: adding `or is_zero_width(char)` to the cluster condition. Before the fix: `['क्', '\u200d']` — broken. After: `['क्\u200d']` — correct."

### Task 20 — Plain English
> "In simple terms: tokenization is how AI reads. It takes messy human text and breaks it into precise numbered pieces. For English, 'playing' becomes 'play' + 'ing' so the model learns the root and tense separately. For Hindi, the challenge is bigger because characters combine with invisible marks and joiners. A bad tokenizer breaks the DNA of the script. That's why abctokz exists — it's built specifically to handle Devanagari correctly."

---

## CLOSING — Mahesh (30 sec)

> "So in total, we completed **16 out of 20 tasks**. We found and fixed **3 real bugs** in the codebase — the offset calculation, the special token filter leak, and the ZWJ cluster fragmentation. We also proved that a small, localized tokenizer can beat GPT-4's tokenizer on Hindi text.
>
> The 4 tasks we skipped were either dependent on external library setup or redundant with what we'd already covered. We prioritized depth of understanding over quantity. Thank you."

---

## TIMING GUIDE

| Section | Speaker | Duration |
|---|---|---|
| Opening + Approach | Mahesh | 1.5 min |
| Tasks 1, 5, 7, 8, 15 | Mahesh | 2 min |
| Tasks 2, 3, 6, 16, 18, 19 | Astha | 2 min |
| Tasks 4, 11, 12, 17, 20 | Arzoo | 2 min |
| Closing | Mahesh | 0.5 min |
| **Total** | | **~8 min** |
