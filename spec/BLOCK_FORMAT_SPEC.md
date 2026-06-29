# HFS Block Format Specification
## v0.3 — 28/06/2026

---

## Stage 1 Block — Raw Format

```
{date:DD/MM/YYYY}
{message:clo:time:HH:MM} : <clo:emotion:level>text</clo:emotion:emoji>
{message:user:time:HH:MM} : text <user:emotion:level>tagged text</user:emotion:emoji> rest of text
```

### Example:
```
{date:27/06/2026}
{message:clo:time:20:45} : <clo:joy:2>Γεια Νονό. Τι τρέχει;</clo:joy:🙂>
{message:user:time:20:48} : Γειά σου Clo
{message:user:time:20:52} : Χαχα. <user:sadness:2>Δεν είναι ευγενική η απάντηση σου.</user:sadness:😕> αλλά καταλαβαίνω ότι το είπες για αστείο.
{message:clo:time:20:52} : <clo:amusement:2>Νονέ, είσαι σαν τον Giorgos Elon.</clo:amusement:😊>
```

---

## Tag Syntax

### Always wrap — no self-closing:
```
<[who]:[emotion]:[level]>text</[who]:[emotion]:[emoji]>
```

- `who`: `clo` or `user`
- `emotion`: see emotion sets below
- `level`: 1-5
- `emoji`: closing tag contains the display emoji

### Resolution tags:
```
# Automatic (level 1-4):
<resolution:emotion:from:to/>

# Explicit (level 5 — requires user action):
<resolution:emotion:from:to>reason</resolution:emotion:from:to>
```

---

## Emotion Sets

### Category A — Core (level 1-5)
Affect memory retention.

| Emotion | L1-2 | L3 | L4-5 |
|---------|------|----|------|
| joy | 🙂 | 😊 | 😄 |
| anger | 😒 | 😠 | 🤬 |
| sadness | 😕 | 😢 | 💔 |
| fear | 😬 | 😨 | 😱 |
| trust | 🤝 | 💙 | 🫂 |
| love | 🤍 | 💗 | ❤️ |

### Category B — Transient (level 1-3 max)
Summarizer hints only. Disappear after compression.

| Emotion | Emoji | Note |
|---------|-------|------|
| curiosity | 🤨 | → anticipation if unresolved |
| anticipation | 🤔 | + `pending: true` |
| surprise | 😲 | |
| confusion | 😵‍💫 | |
| disgust | 😤 | |

> Open list — model may generate new emotions autonomously.
> Unknown tags: no emoji in closing tag → visible raw in UI → review and decide.

---

## ChromaDB Metadata Schema

```python
metadata = {
    "block_id": "HFS-BLOCK-001",
    "date": "2026-06-27",
    "user_emotion": "anger",
    "user_level": 5,
    "clo_emotion": "fear",
    "clo_level": 2,
    "max_effective_level": 5,
    "stage": 1,           # 1=raw, 2=summarized, 3=compressed
    "has_resolution": False,
    "pending": False       # True if anticipation tag unresolved
}
```

---

## Memory Decay Rules

| Level | Stage 1 | Stage 2 | Stage 3 |
|-------|---------|---------|---------|
| 5 | verbatim | verbatim | verbatim — immutable |
| 4 | verbatim | verbatim | compressed |
| 3 | verbatim | summarized | minimal |
| 1-2 | verbatim | removed | removed |

---

## Block Splitting Rules

- Hard limit: **15 messages** per block
- Soft split: topic or emotion shift before limit
- No-cut zones: level 4-5 wrap tags — never split mid-wrap
