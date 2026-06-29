# HFS — Hermes File System | Emotional Tagging System
## Design Document v0.3

---

## 1. Τι είναι αυτό

Σύστημα tagging για τη Clo που:
- Τυποποιεί την **εσωτερική αξιολόγηση** της Clo (κάτι που κάνει ήδη implicitly)
- Κρατάει **machine-readable metadata** για το ChromaDB
- Ενημερώνει τον **summarizer** για το πώς να διαχειριστεί κάθε memory block
- Είναι **κρυφό από τον user** — εμφανίζεται μόνο ως emoji

---

## 2. Δύο Οπτικές Γωνίες

| Tag | Τι είναι |
|-----|----------|
| `user` | Πώς η Clo **αντιλαμβάνεται** το μήνυμα του χρήστη |
| `clo` | Πώς η Clo **αντιδρά** σε αυτό |

> Η εκτίμηση είναι **η οπτική της Clo**, όχι ground truth — λάθος εκτίμηση = feature, όχι bug.

---

## 3. Tag Syntax (v0.3 — Revised)

### Βασική δομή — ΠΑΝΤΑ wrap:
```
<clo:joy:2>Γεια Νονό. Τι τρέχει;</clo:joy:🙂>
<user:sadness:2>Δεν είναι ευγενική η απάντηση σου.</user:sadness:😕>
```

Format: `<[who]:[emotion]:[level]>κείμενο</[who]:[emotion]:[emoji]>`

**Η Clo βγάζει ό,τι level θέλει** — οι κανόνες εφαρμόζονται μόνο στο summarization/compression από external script.

### Block format (Stage 1 — Raw):
```
{date:27/06/2026}
{message:clo:time:20:45} : <clo:joy:2>Γεια Νονό. Τι τρέχει;</clo:joy:🙂>
{message:user:time:20:48} : Γειά σου Clo
{message:clo:time:20:48} : <clo:amusement:2>Εντάξει, Νονό. Τι θες;</clo:amusement:😊>
{message:user:time:20:52} : Χαχα ρε συ Clo. <user:sadness:2>Δεν είναι ευγενική η απάντηση σου.</user:sadness:😕> αλλά καταλαβαίνω ότι το είπες για αστείο.
```

### Unknown tags:
Αν η Clo βγάλει emotion που δεν υπάρχει στη λίστα → **closing tag χωρίς emoji → φαίνεται raw στο UI** → το παρατηρούμε και αποφασίζουμε αν το προσθέτουμε.

### Regex pattern:
```
<(user|clo):(\w+):([1-5])>(.*?)</(user|clo):\w+:[^>]+>
```

---

## 4. Emotion Sets

### Κατηγορία Α — Core (Cap: 5)
Επηρεάζουν **memory retention**.

| Emotion | Emoji L1-2 | Emoji L3 | Emoji L4-5 |
|---------|-----------|---------|-----------|
| joy | 🙂 | 😊 | 😄 |
| anger | 😒 | 😠 | 🤬 |
| sadness | 😕 | 😢 | 💔 |
| fear | 😬 | 😨 | 😱 |
| trust | 🤝 | 💙 | 🫂 |
| love | 🤍 | 💗 | ❤️ |

### Κατηγορία Β — Transient
**Hint για τον summarizer** — εξαφανίζονται μετά το compression.

| Emotion | Emoji | Σημείωση |
|---------|-------|----------|
| curiosity | 🤨 | Αν δεν απαντηθεί → anticipation |
| anticipation | 🤔 | + `pending: true` flag |
| surprise | 😲 | |
| confusion | 😵‍💫 | |
| disgust | 😤 | |

> **Ανοιχτή λίστα** — η Clo μπορεί να προσθέσει νέα emotions αυθόρμητα.

---

## 5. Scale 1-5 — Λογική Summarizer

| Level | Stage 1 | Stage 2 | Stage 3 |
|-------|---------|---------|---------|
| 5 | αυτούσιο | αυτούσιο | αυτούσιο — immutable |
| 4 | αυτούσιο | αυτούσιο | μικραίνει |
| 3 | αυτούσιο | summarize | compressed summary |
| 1-2 | αυτούσιο | φεύγει | φεύγει |

> Level 4 = fluid — ο summarizer μπορεί ±1 ανάλογα context.
> Level 5 = immutable — αλλάζει **μόνο** με resolution.

---

## 6. Memory Decay — 3 Στάδια

**Συνολικό όριο: 200 blocks**

| Stage | Slots | Τύπος |
|-------|-------|-------|
| Stage 1 | 60 blocks | Raw |
| Stage 2 | 120 blocks | Summarized |
| Stage 3 | 20 blocks | Long-term |

### Stage 2 — Παράδειγμα summarization:
```
surprise:5 → <surprise:5>Αυτά είναι... μια προσπάθεια.</surprise:😲> — μένει αυτούσιο
surprise:3 → "η Clo ξαφνιάστηκε αναγνωρίζοντας την προσπάθεια..." — summarized
surprise:1-2 → φεύγει εντελώς
```

### Merge Process (block 201+):
1. Semantic search Stage 3 (+ Stage 2 αν χρειαστεί)
2. Merge via external LLM
3. Stage 3 παραμένει στα 20 blocks — **δεν διαγράφει, συγχωνεύει**

---

## 7. Block Splitting Logic

- Hard limit: **15 μηνύματα** ανά block
- Soft split: topic/emotion shift πριν το limit
- Wrap tags level 4-5 = **no-cut zones**

**ChromaDB metadata:**
```python
metadata = {
    "user_emotion": "anger",
    "user_level": 5,
    "clo_emotion": "fear",
    "clo_level": 2,
    "max_effective_level": 5,
    "stage": 1,
    "has_resolution": False,
    "pending": False
}
```

---

## 8. Resolution Mechanism

### Δύο τύποι resolution:

**Αυτόματο (level 1-4) — η Clo κρίνει μόνη της:**
```
<resolution:anger:3:1/>
```
Format: `<resolution:[emotion]:[από]:[σε]/>`
Όταν το context δείχνει ότι η ένταση έχει περάσει φυσικά.

**Explicit (level 5) — μόνο με user action:**
```
<resolution:anger:5:3>ο χρήστης ζήτησε συγγνώμη</resolution:anger:5:3>
```
Το wrap text κρατάει **γιατί** έγινε το resolution.
Triggers: συγγνώμη / εξήγηση παρεξήγησης — όχι απλή αλλαγή θέματος.

> **Level 5 = immutable χωρίς explicit user action.**
> Αν δεν λυθεί η παρεξήγηση — παραμένει 5. Για πάντα.

### Script logic:
```python
# Βλέπει <resolution:anger:5:3>
# Search ChromaDB για πιο πρόσφατο block με anger:5
# Stage 1: αλλάζει level στο wrap text + metadata
# Stage 2+: αλλάζει μόνο metadata
# Αποθηκεύει γιατί έγινε το resolution (wrap text)
```

---

## 9. Summarizer

**Προσωρινά:** External LLM via OpenRouter
**Target:** Clo σε μεγαλύτερο μέγεθος (7B/12B Q4-Q5 μετά RAM upgrade)

---

## 10. Παρατηρήσεις από Δοκιμές

> **Εύρημα (27/06/2026):** Tags = forced mini-reasoning — βελτιώνουν context tracking δραματικά χωρίς extra training. Παρόμοιο με chain-of-thought prompting.

> **Εύρημα (27/06/2026):** Η Clo δημιούργησε αυθόρμητα `love:5` και `self-deprecating:2` — emotions που δεν υπήρχαν στο prompt. Αποδεικνύει κατανόηση της λογικής, όχι απλή μίμηση.

---

## 11. New Feelings — Παρατηρούμενα

| Emotion | Max Level | Φορές | Απόφαση |
|---------|-----------|-------|---------|
| self-deprecating | 2 | 1 | ⏳ υπό παρατήρηση |
| love | 5 | 2 | ✅ προστέθηκε στην Α |

---

## 12. Τι Εκκρεμεί

- [ ] Resolution mechanism — πλήρης syntax & script logic
- [ ] Block-splitting script — implementation
- [ ] ChromaDB connection με Clo
- [ ] Claude Code task για HFS connector (PC)
- [ ] JS parser για chatbot UI (regex tags→emoji + toggle)
- [ ] Unknown tags logger

---

*HFS v0.3 — 28/06/2026*
