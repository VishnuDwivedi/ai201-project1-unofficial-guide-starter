# Project 1 Planning: The Unofficial Guide — Museum Collections Chatbot

> Write this document before you write any pipeline code.
> Your spec and architecture diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Update the Retrieval Approach and Chunking Strategy sections if you change your approach during implementation.
> Update this file before starting any stretch features.

---

## Domain

This project builds a RAG-powered chatbot for museum collections knowledge. The domain covers artworks, artists, mediums, and acquisition history across three major museums: the Metropolitan Museum of Art (the Met), the Museum of Modern Art (MoMA), and the Carnegie Museum of Art.

This knowledge is valuable because museum websites are designed for browsing individual artworks, not for answering natural language questions like "What Egyptian artifacts does the Met have from before 1000 BC?" or "Which MoMA artists worked in both photography and painting?" Visitors, students, and researchers currently have no way to query across a whole collection in plain language — they must navigate clunky search filters or read through thousands of records manually. A RAG system makes the collection conversationally searchable for the first time.

---

## Documents

All three datasets are CSV files downloaded from Kaggle. They require no API key and are available under open licenses (CC0 or equivalent). Each CSV row represents one artwork and will be converted into a text document during ingestion.

| # | Source | Description | URL or location |
|---|--------|-------------|-----------------|
| 1 | Met Museum – MetObjects.csv | 420,000+ artworks: title, artist, culture, medium, date, department, dimensions | https://www.kaggle.com/datasets/metmuseum/the-metropolitan-museum-of-art-open-access |
| 2 | Met Museum – MetObjects.csv (Egyptian Art dept.) | Subset: rows where Department = "Egyptian Art" | Same file as above, filtered during ingestion |
| 3 | Met Museum – MetObjects.csv (European Paintings dept.) | Subset: rows where Department = "European Paintings" | Same file as above, filtered during ingestion |
| 4 | Met Museum – MetObjects.csv (Asian Art dept.) | Subset: rows where Department = "Asian Art" | Same file as above, filtered during ingestion |
| 5 | MoMA – Artworks.csv | 130,262 artworks: title, artist, date, medium, dimensions, date acquired | https://www.kaggle.com/datasets/momanyc/museum-collection |
| 6 | MoMA – Artists.csv | 15,091 artist records: name, nationality, gender, birth/death year | https://www.kaggle.com/datasets/momanyc/museum-collection |
| 7 | Carnegie Museum – artworks.csv | 28,269 artworks: title, artist, nationality, medium, date, department | https://www.kaggle.com/datasets/mfrancis23/carnegie-museum-of-art |
| 8 | Met Museum – MetObjects.csv (Arms & Armor dept.) | Subset: rows where Department = "Arms and Armor" | Same file as above, filtered during ingestion |
| 9 | MoMA – Artworks.csv (Photography dept.) | Subset: rows where Department = "Photography" | Same file as above, filtered during ingestion |
| 10 | Carnegie Museum – artworks.csv (prints/drawings) | Subset filtered by medium containing "print" or "drawing" | Same file as above, filtered during ingestion |

> **Note on document count:** The three Kaggle CSVs cover 10 distinct document slices after departmental filtering. During ingestion, each CSV row becomes one text document in the format:
> `"[Title] by [Artist] ([Date]). Medium: [Medium]. Department: [Department]. Museum: [Museum name]. [Additional fields if available]."`

---

## Chunking Strategy

**Chunk size:** 300 characters (approximately 50–70 tokens)

**Overlap:** 50 characters

**Reasoning:**

Each row in the CSV datasets is a short, self-contained artwork record — typically 150–400 characters when converted to a text sentence. These are not long-form documents; they're structured facts. This means the right chunking strategy is very different from a guide or FAQ.

- **300 characters** fits roughly one full artwork record per chunk. This keeps each chunk semantically complete — title, artist, date, medium, and museum all stay together. A chunk that cuts "Starry Night by Vincent van Gogh (1889). Medium: Oil on canvas. Department:" in half would lose the most retrievable part (the medium and department) and produce a useless embedding.
- **50-character overlap** handles the edge case where two related fields (e.g., date and medium) land on either side of a chunk boundary. This is a small overlap because these records don't have narrative flow — overlap mainly protects against boundary splits, not lost context.
- If chunks were **too small** (e.g., 100 characters), each chunk would contain only a title and partial artist name — not enough signal for semantic search to distinguish an Egyptian artifact query from a European paintings query.
- If chunks were **too large** (e.g., 1000 characters), multiple unrelated artworks would be merged into one chunk. A query about "French Impressionism" would retrieve a chunk that also contains unrelated armor records, diluting the relevance.

During Milestone 3, I'll print 5 sample chunks and verify each one contains a complete, human-readable artwork description before embedding.

---

## Retrieval Approach

**Embedding model:** `all-MiniLM-L6-v2` via `sentence-transformers`

**Top-k:** 5

**Reasoning for top-k:** Each chunk is one artwork record. To answer a question like "What are some Impressionist paintings at the Met?", the system needs several examples — returning just 1 chunk would give a thin answer. Returning 10 would dilute the prompt with too many loosely relevant records and risk sending the LLM off-target. k=5 gives the LLM enough variety to synthesize a useful answer without overwhelming it.

**Production tradeoff reflection:**

If deploying this for real museum visitors at scale, I would weigh the following tradeoffs in choosing an embedding model:

- **Context length:** `all-MiniLM-L6-v2` handles up to 256 tokens, which is sufficient for 300-character artwork records. For a system that also ingests long curatorial essays or exhibition guides, a model with a longer context window (e.g., `text-embedding-3-large` from OpenAI, 8191 tokens) would be necessary.
- **Multilingual support:** Major museums like the Met serve international visitors. A multilingual model like `paraphrase-multilingual-MiniLM-L12-v2` would allow queries in French, Spanish, or Mandarin to match English-language collection records. This is a real production need I'm not addressing in this project.
- **Domain accuracy:** General-purpose sentence transformers may not understand art-specific terminology well (e.g., "chiaroscuro," "intaglio," "Ukiyo-e"). A fine-tuned model on art/museum text would likely improve retrieval precision, at higher cost.
- **Latency vs. accuracy:** `all-MiniLM-L6-v2` is fast and runs locally with no API cost. OpenAI or Cohere embedding APIs offer higher accuracy but add latency and cost per query — unacceptable for a free student project but reasonable for a production museum application with a budget.

---

## Evaluation Plan

| # | Question | Expected answer |
|---|----------|-----------------|
| 1 | What medium did Vincent van Gogh most commonly use in his MoMA works? | Oil on canvas (based on MoMA Artworks.csv records attributed to van Gogh) |
| 2 | How many departments does the Met Museum collection cover in this dataset? | The dataset includes departments such as Egyptian Art, European Paintings, Asian Art, Arms and Armor, American Decorative Arts, and others — at least 10 distinct departments |
| 3 | Name an artwork from the Carnegie Museum that uses printmaking as its medium. | A specific title from Carnegie artworks.csv where medium contains "print" — e.g., a lithograph or etching by a catalogued artist |
| 4 | What nationality are most of the artists in the MoMA collection? | American (the MoMA Artists.csv shows American artists as the largest nationality group) |
| 5 | Are there any Japanese artworks in the Met's Asian Art department? | Yes — the Met's Asian Art department includes Japanese works; the system should return at least one example with a title and date |

---

## Anticipated Challenges

1. **Incomplete or missing fields in CSVs:** Many rows in these datasets have empty fields — artist names listed as "Unknown," dates listed as ranges like "ca. 1200–1250," or mediums that are blank. If these fields are missing when I construct the text document for each chunk, the resulting embedding will carry very little semantic signal, and retrieval will return incomplete records. I'll handle this by adding a preprocessing step that fills missing fields with "Unknown" or skips rows where both title and artist are empty.

2. **Scale — the Met CSV has 420,000+ rows:** Embedding every row in MetObjects.csv with `all-MiniLM-L6-v2` could take 30–60 minutes on a laptop and produce a ChromaDB collection too large to query quickly. To keep the project manageable, I'll filter the Met data to 4 departments and cap at 5,000 rows per department, giving roughly 20,000 Met records total. This still provides good coverage for evaluation questions while keeping ingestion and embedding time under 10 minutes.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        PIPELINE DIAGRAM                         │
└─────────────────────────────────────────────────────────────────┘

  [Kaggle CSV Files]
  MetObjects.csv, Artworks.csv (MoMA),
  Artists.csv (MoMA), artworks.csv (Carnegie)
          │
          ▼
┌─────────────────────┐
│  1. INGESTION        │  pandas (read_csv)
│  Load CSVs,          │  Filter by department
│  drop empty rows,    │  Cap rows per museum
│  build text strings  │  Output: List of dicts
└────────┬────────────┘  {text, source, museum, dept}
         │
         ▼
┌─────────────────────┐
│  2. CHUNKING         │  langchain RecursiveCharacterTextSplitter
│  chunk_size=300      │  or manual split
│  overlap=50          │  Output: List of chunk strings
│  Verify 5 samples    │  with metadata attached
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  3. EMBEDDING +      │  sentence-transformers
│     VECTOR STORE     │  all-MiniLM-L6-v2 (local)
│                      │  ChromaDB (local, persistent)
│  Store: text,        │  Metadata: source file,
│  embedding, metadata │  museum name, department
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  4. RETRIEVAL        │  ChromaDB .query()
│  User query →        │  top-k = 5
│  embed query →       │  Returns chunks +
│  semantic search     │  distance scores + sources
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  5. GENERATION       │  Groq API
│  Prompt: retrieved   │  llama-3.3-70b-versatile
│  chunks + query      │  System prompt enforces
│  → grounded answer   │  grounding + source citation
│  + source list       │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  6. INTERFACE        │  Gradio (gr.Blocks)
│  Input: text box     │  Runs at localhost:7860
│  Output: answer box  │
│  + sources box       │
└─────────────────────┘
```

---

## AI Tool Plan

**Milestone 3 — Ingestion and chunking:**

I'll give Claude the **Documents section** (CSV file descriptions and fields) and the **Chunking Strategy section** (chunk_size=300, overlap=50, text format per row) from this planning.md, plus the requirement to attach metadata (source filename, museum name, department) to each chunk. I'll ask Claude to implement an `ingest.py` script with two functions: `load_and_clean(csv_path, museum_name, max_rows)` that returns a list of text strings, and `chunk_documents(docs, chunk_size, overlap)` that returns a list of dicts with `{text, source, museum, department}`. I'll verify the output by printing 5 random chunks and confirming each is a complete, readable artwork description with no empty fields or HTML artifacts.

**Milestone 4 — Embedding and retrieval:**

I'll give Claude the **Retrieval Approach section** (model name, top-k=5) and the **Architecture diagram** (stage 3–4) from this planning.md, plus the chunk output format from Milestone 3. I'll ask Claude to implement `embed.py` with a function `build_vector_store(chunks)` that embeds all chunks using `SentenceTransformer("all-MiniLM-L6-v2")` and stores them in a persistent ChromaDB collection with source metadata, and `retrieve.py` with `search(query, k=5)` that returns the top-k chunks and their source info. I'll verify by running 3 of my evaluation questions and checking that returned chunks are visibly on-topic and have distance scores below 0.5.

**Milestone 5 — Generation and interface:**

I'll give Claude the **grounding requirement** (answer only from retrieved context, always cite source museum and file), the **output format** (answer paragraph + bulleted source list), and the Gradio skeleton from the project spec. I'll ask Claude to implement `generate.py` with a `generate_answer(query, chunks)` function that calls Groq's `llama-3.3-70b-versatile` with a system prompt enforcing grounding, and `app.py` wiring the Gradio UI to the full pipeline. I'll verify grounding by asking a question my documents don't cover (e.g., "What exhibits are at the Louvre right now?") and confirming the system says it doesn't have enough information rather than fabricating an answer.