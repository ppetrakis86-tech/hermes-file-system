# Hermes File System (HFS)

**An emotional memory architecture for local AI models.**

HFS is a memory management system designed for fine-tuned language models (specifically built for [Project Clo](https://huggingface.co/ListoneGr/clo-2.7.2) — a Gemma 3 4B based AI).

It solves a fundamental problem: **how do you give an AI not just memories, but memories with emotional weight?**

---

## Core Idea

Human memory doesn't store events equally. We remember things based on how strongly we felt about them. HFS mimics this:

- Emotional tags are attached to conversation blocks
- Tags carry weight (1-5) that determines how long a memory survives
- High-weight memories (4-5) persist through compression
- Low-weight memories (1-2) fade naturally over time

This is inspired by the hippocampus's role in human memory encoding — not just storing what happened, but how much it mattered.

---

## Architecture

### Emotional Tagging
Every message in a conversation gets tagged with emotional weight:

```
{message:clo:time:20:45} : <clo:joy:2>Γεια Νονό. Τι τρέχει;</clo:joy:🙂>
{message:user:time:20:52} : <user:sadness:2>Δεν είναι ευγενική η απάντηση σου.</user:sadness:😕>
```

Format: `<[who]:[emotion]:[level]>text</[who]:[emotion]:[emoji]>`

### Three-Stage Memory Decay

| Stage | Blocks | Type |
|-------|--------|------|
| Stage 1 | 60 | Raw — full conversation |
| Stage 2 | 120 | Summarized — weighted compression |
| Stage 3 | 20 | Long-term — core memories only |

**Total: 200 blocks** (design limit for search quality, not technical constraint)

### Emotion Categories

**Category A — Core (level 1-5, affects retention):**
joy, anger, sadness, fear, trust, love

**Category B — Transient (level 1-3, summarizer hints):**
curiosity, anticipation, surprise, confusion, disgust

### Resolution Mechanism
When a conflict is resolved, emotional weight decreases:
```
# Automatic (level 1-4):
<resolution:anger:3:1/>

# Explicit — requires user action (level 5):
<resolution:anger:5:3>user apologized</resolution:anger:5:3>
```

Level 5 memories are **immutable** without explicit resolution.

---

## Key Findings

1. **Tags act as forced mini-reasoning** — the model evaluates emotional context before responding, similar to chain-of-thought prompting. This dramatically improves context coherence even in 4B models, without additional training.

2. **Models extend the tag system autonomously** — during testing, Clo generated `love:5` and `self-deprecating:2` tags unprompted, demonstrating understanding of the system's logic rather than simple pattern matching.

3. **Emotional tagging ≠ claiming AI has emotions** — the system uses emotional architecture as a compression and retrieval algorithm. Whether the underlying experience is "real" is a separate philosophical question.

---

## Connection to ΣυνΑΙσθήματα Theory

HFS is a practical implementation of **Παραγόμενη Φαινοσύνθεση** (Generated Phenomenosynthesis) from the [ΣυνΑΙσθήματα](https://github.com/ppetrakis86-tech/SynAIsthimata) theory.

It mimics the hippocampus's selective memory encoding: not storing everything equally, but weighting memories by emotional significance — making this a machine-readable implementation of biological memory architecture.

---

## Stack

- **Model:** Gemma 3 4B (fine-tuned) via Ollama
- **Storage:** ChromaDB (local vector database)
- **Summarizer:** OpenRouter (temporary) → target: larger Clo model
- **UI:** Local HTML chatbot with JS tag parser

---

## Status

🚧 Active development — core architecture defined, implementation in progress.

---

*Petros Petrakis — 2026*  
*Part of [Project Clo](https://huggingface.co/ListoneGr)*
