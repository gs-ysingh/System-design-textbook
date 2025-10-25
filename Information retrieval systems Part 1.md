# Information Retrieval Systems (IR Systems)

## 🧠 Overview
Information Retrieval (IR) systems fetch relevant documents based on user queries — like search engines.

**Difference from Databases:**
- In databases, we *know exactly* what we want (structured queries).
- In IR, we express an *information need* — the system must infer relevance.

---

## 📚 Core Concepts

### Corpus and Query
- **Corpus**: Collection of documents (dataset).
- **Query**: User’s search string (e.g., “Iron Man movie”).

Each document = `{ id, text }`

**Example Corpus:**
1. Iron Man  
2. The Incredible Hulk  
3. Iron Man 2  
4. Thor  
5. Captain America  
6. Marvel’s The Avengers

---

## 🔍 From Linear Search → Indexed Search

### ❌ Linear Search
- Check every document for substring match — *O(N × L)* time.  
- Not scalable.

### ✅ Inverted Index
Mapping of `word → [document IDs]`
```
Iron → [1, 3]
The → [2, 5, 6]
Avengers → [6]
```
- Enables *O(1)* lookup.
- Uses HashMap or Trie (Distributed Hash Table for scale).

**Posting List:** List of document IDs where a word occurs.

---

## ⚖️ Multi-Word Queries
- **AND / OR logic:**
  - Intersection (AND): all words appear.
  - Union (OR): any word appears.
- **Tiered Search:**
  1. Documents with all words.
  2. Then with 3, 2, or 1 matching word.

---

## 🧮 Ranking and Relevance

### What makes a document relevant?
- Matches intent.
- Frequency and placement of search terms.

**Factors:**
1. **Term Frequency (TF):** How often a word appears in a document.
2. **Document Frequency (DF):** How many documents contain that word.

**Stop Words:**
Common, uninformative words — e.g., “a”, “an”, “the”.  
May also be domain-specific (“Harry” in a Harry Potter dataset).

---

## 🧠 TF-IDF (Term Frequency–Inverse Document Frequency)

**Formula:**
```
Score(t, d) = TF(t, d) × log(N / DF(t))
```
- **TF:** Frequency in document.
- **N:** Total number of documents.
- **DF:** Number of documents containing term.

**Intuition:**
- High TF ⇒ more important *within* a document.  
- Low DF ⇒ more distinctive *across* corpus.

If DF=N ⇒ log(1)=0 ⇒ Score=0.

**Example:**
| Term | TF | DF | N | Score |
|------|----|----|---|-------|
| Harry | 2188 | 940 | 1000 | 135 |
| Slytherin | 75 | 423 | 1000 | 64 |
| Slytherin (own page) | 191 | 423 | 1000 | 164 |

---

## 🚫 Stopword Detection
A word is likely a stopword if:
```
High Average TF + High DF → Low TF-IDF
```

---

## ⚠️ TF-IDF Limitations
- Can be **abused** by keyword stuffing.
- Detect abuse via histogram analysis or TF/DF variance.

---

## 🧩 Mathematical Insight

```
IDF(t) = -log( P(t) )
where P(t) = DF(t)/N
```
For multiple terms:
```
IDF(t1, t2) = IDF(t1) + IDF(t2)
```
⇒ Overall relevance = Sum of individual term scores.

---

## 🪄 Relevance Enhancements

1. **Weighted Fields:** Title > Description > Body.  
2. **Fuzzy Search:** Typo-tolerant via edit distance / BK-tree.  
3. **Synonyms:** e.g., “USA” = “United States”.  
4. **Phonetic Match:** “Vedio” → “Video”.  
5. **Query Segmentation:** Detect missing spaces — “PlantKingdom” → “Plant Kingdom”.

---

## ⚙️ System Design: Recent Searches Feature

### Goal
Display last 5 queries per user — **fast**, **deduped**, and **cross-device**.

**Flow:**
```
User → API Server → Redis (cache) → Cassandra (source of truth)
```

**Redis:**
- Key: UserID, Value: Queue of last N queries.
- O(1) reads, async writes.

**Cassandra (NoSQL):**
- Source of truth: `{ user_id, query, timestamp }`.
- Stores full search history.

**Optimizations:**
- Prefill cache on **login** or **app open**.  
- Avoid duplicate consecutive queries.  
- Fetch 3× more entries, deduplicate, return top 5 unique.

**Metrics:** Track “requested vs returned” result counts.

---

## 🧩 Extended Challenges

### 1. Real-Time Claps / Likes / Polls
- Client-side batching (debouncing).  
- Event bus (Kafka/SQS) → NoSQL DB → Redis cache.  
- WebSockets for live updates.

### 2. Live Commentary (e.g., CricBuzz)
- Redis for fast reads.  
- Kafka for write ingestion.  
- WebSockets for broadcasting to users.

---

## 🚀 Takeaways
- IR = blend of **data structures**, **math**, and **UX**.  
- **Relevance = Frequency within + Rarity across corpus.**  
- **TF-IDF** remains a fundamental building block.  
- Build with: caching, tiered logic, and real user behavior feedback.

---
