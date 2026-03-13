# Task 3 — The National Anthem Test: Comparative Script Analysis

> **Tokens Used for Task 3:** ~52,000 (Training, Execution & Linguistic Analysis)

## Experimental Setup

To evaluate script efficiency, we trained a single BPE tokenizer on a mixed-script corpus (English + Hindi) and encoded the first stanza of the Indian National Anthem.

- **Model**: BPE (Byte Pair Encoding)
- **Vocabulary Size**: 400
- **Normalizer**: `DevanagariNormalizer`
- **PreTokenizer**: `DevanagariAwarePreTokenizer`
- **Inputs**:
    - **English Transliteration**: "Jana Gana Mana Adhinayaka Jaya He Bharata Bhagya Vidhata Punjab Sindhu Gujarat Maratha Dravida Utkala Banga"
    - **Devanagari**: "जन गण मन अधिनायक जय हे भारत भाग्य विधाता पंजाब सिंधु गुजरात मराठा द्राविड उत्कल बंग"

## Tokenization Results

| Script | Tokens | Token Count |
| :--- | :--- | :--- |
| **English Transliteration** | `['J', '##an', '##a', 'G', '##an', '##a', 'M', '##an', '##a', ...]` | 67 |
| **Devanagari** | `['जन', 'गण', 'मन', 'अ', '##धि', '##न', '##ा', ...]` | 45 |

## Fertility Comparison

Fertility is calculated as `total_tokens / total_words`. Both inputs consist of **16 words**.

| Script | Word Count | Token Count | Fertility |
| :--- | :--- | :--- | :--- |
| **English Transliteration** | 16 | 67 | **4.19** |
| **Devanagari** | 16 | 45 | **2.81** |

## Why the Results Differ

The Devanagari script achieved **33% better compression** (lower fertility) than the English transliteration. This difference is driven by several architectural factors in `abctokz`:

1.  **Linguistic Information Density**: Each Devanagari character (especially when combined with matras) carries more phonetic information than a single Latin character. While "Jana" is 4 Latin tokens in a small-vocab BPE, "जन" is often a single token or two, because the script structure allows for more compact subword representation.
2.  **Grapheme Cluster Awareness**: The `DevanagariAwarePreTokenizer` ensures that combining marks (matras) and halants stay attached to their base characters. This prevents the "fragmentation" that occurs in the Latin script where every vowel and consonant is a separate byte that BPE must learn to merge.
3.  **Vocabulary learned**: In a small vocabulary (400), BPE prioritizes the most frequent patterns. Since the corpus contained both scripts, the Devanagari forms "जन", "गण", "मन" were likely learned as whole units early on due to their high repetition, whereas long Latin words like "Adhinayaka" were fragmented into character-level pieces.

## External Tokenizer Comparison (Bonus)

While not executed locally, typical behavior for large-scale multilingual models (like OpenAI's GPT-3.5/4) is the opposite:

| Tokenizer | Latin Fertility | Hindi Fertility |
| :--- | :--- | :--- |
| **OpenAI (tiktoken)** | ~1.0 - 1.2 | ~3.0 - 5.0 |
| **abctokz (Task 3)** | 4.19 | **2.81** |

**Insight**: Large external tokenizers are often "English-centric," meaning their vocabularies are dominated by Latin subwords. For them, Hindi is often tokenized at the byte level, leading to extremely high fertility. `abctokz`, being biased towards the local corpus and using script-aware pre-tokenization, demonstrates how **localized training significantly improves efficiency for non-Latin scripts.**

## Key Insights
- **Script Matters**: Devanagari is inherently more efficient for BPE once the basic subword units are learned, as it packs more phonetic value per token.
- **Architectural Bias**: Our results show that `abctokz`'s specific focus on Devanagari (normalization/pre-tokenization) pays off in terms of data compression.
- **Subword Advantage**: Even with a tiny vocab of 400, the model successfully compresses complex Sanskrit/Hindi terms better than their transliterated counterparts.
