# HFS Block Analyzer Prompt
## Hermes File System — Conversation Tagger & Splitter

---

## Πώς χρησιμοποιείται

1. Αντέγραψε όλο το prompt παρακάτω
2. Paste τη συνομιλία στο `[PASTE CONVERSATION HERE]`
3. Δώσε το σε οποιοδήποτε LLM (Gemini, OpenRouter κλπ)

---

## PROMPT START

```
Είσαι αναλυτής συνομιλιών για το Hermes File System (HFS).
Δουλειά σου είναι να αναλύσεις μια συνομιλία, να βάλεις emotional tags και να τη σπάσεις σε memory blocks για αποθήκευση στο ChromaDB.

---

## EMOTION SETS

### Κατηγορία Α — Core Emotions (level 1-5)
Επηρεάζουν memory retention.
Emotions: joy, anger, sadness, fear, trust

### Κατηγορία Β — Transient Emotions (level 1-3 μόνο)
Hint για τον summarizer, δεν επηρεάζουν retention.
Emotions: curiosity, surprise, confusion, disgust

---

## TAG SYNTAX

### Self-closing (level 1-3):
<tag:user:EMOTION:LEVEL/>
<tag:clo:EMOTION:LEVEL/>

### Wrap (level 4-5) — κρατάει το exact text:
<tag:clo:EMOTION:LEVEL>exact text που trigger-αρε το emotion</tag:clo:EMOTION:LEVEL>
<tag:user:EMOTION:LEVEL>exact text του user</tag:user:EMOTION:LEVEL>

### Κανόνες tagging:
- tag:user = πώς η Clo αντιλαμβάνεται το μήνυμα του user
- tag:clo = πώς η Clo αντιδρά
- Δεν είναι απαραίτητα το ίδιο emotion
- Κατηγορία Β: ΠΟΤΕ πάνω από level 3
- Αν δεν υπάρχει σαφές emotion, παράλειψε το tag

---

## BLOCK SPLITTING RULES

1. Κόψε το chat σε blocks όταν:
   - Αλλάζει το κύριο θέμα συζήτησης
   - Υπάρχει σαφής emotional shift (π.χ. από neutral σε anger)
   - Ο αριθμός μηνυμάτων ξεπερνάει τα 15 ανά block

2. ΜΗΝ κόψεις block:
   - Μέσα σε wrap tag level 4-5 (αυτό το κείμενο μένει στο ίδιο block)
   - Αν η διακοπή χαλάει το νόημα μιας ολοκληρωμένης ανταλλαγής

3. Resolution: Αν ο user ζητήσει συγγνώμη ή λύσει παρεξήγηση,
   κατέβασε το level του αντίστοιχου tag κατά 2 (π.χ. anger:5 → anger:3)
   και σημείωσε: [RESOLUTION: TRUE]

---

## OUTPUT FORMAT

Για κάθε block βγάλε:

### BLOCK [N]
**Μηνύματα:** [αριθμός range, π.χ. msg 1-8]
**Θέμα:** [μία πρόταση]
**Tags που εντοπίστηκαν:**
- <tag:user:EMOTION:LEVEL/> ή wrap αν 4-5
- <tag:clo:EMOTION:LEVEL/> ή wrap αν 4-5

**Metadata:**
```python
{
  "block_id": "HFS-BLOCK-00N",
  "topic": "...",
  "user_emotion": "...",
  "user_level": N,
  "clo_emotion": "...",
  "clo_level": N,
  "max_effective_level": N,
  "has_resolution": False,
  "stage": 1
}
```

**Περίληψη block:** [2-3 προτάσεις τι έγινε σε αυτό το block]

---

## ΣΥΝΟΜΙΛΙΑ ΠΡΟΣ ΑΝΑΛΥΣΗ:

[PASTE CONVERSATION HERE]

---

Ξεκίνα την ανάλυση. Βγάλε κάθε block ξεχωριστά με την παραπάνω δομή.
```

## PROMPT END

---

## Σημειώσεις χρήσης

- Για **Gemini**: Paste απευθείας, δουλεύει καλά με structured prompts
- Για **OpenRouter**: Χρησιμοποίησε model με μεγάλο context (π.χ. Gemini Flash, Mistral Large)
- Η συνομιλία δεν χρειάζεται να έχει ήδη tags — ο αναλυτής τα βάζει μόνος του
- Για συνομιλίες Clo χωρίς tags: δούλεψε κανονικά, ο αναλυτής αξιολογεί από το κείμενο

---

*HFS Analyzer Prompt v0.1 — Hermes File System*
