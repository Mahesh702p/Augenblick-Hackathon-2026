# Task 1 — Follow the Code: What Actually Happens When You Tokenize?

## 1. The Investigation Setup
*Documenting how we trained the model, the corpus we used, and the script we ran to trace the pipeline output.*

## 2. Stage-by-Stage Pipeline Trace
*Tracing the specific string through every step of the pipeline.*

### Stage 1: The Normalizer
- **Involved Files/Classes:** 
- **Input String:**
- **Output String:**
- **What happened & Why:**

### Stage 2: The Pre-Tokenizer
- **Involved Files/Classes:** 
- **Input String:**
- **Output List of Pre-Tokens:**
- **What happened & Why:**

### Stage 3: The Model (BPE tokenize)
- **Involved Files/Classes:** 
- **Input List of Pre-Tokens:**
- **Output Token IDs & Subword Pieces:**
- **What happened & Why:**

### Stage 4: The Decoder
- **Involved Files/Classes:** 
- **Input Token IDs:**
- **Output Reconstructed String:**
- **What happened & Why:**

## 3. Key Insights & Edge Cases Revealed
*Documenting any "wrong turns", hypotheses we tested (like checking if the spaces get lost), or bugs we noticed during the trace.*
