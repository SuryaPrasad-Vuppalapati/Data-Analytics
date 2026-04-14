# GenAI Hackathon (Kaggle Week 2026) - Project Guide

This guide breaks down the complete RAG (Retrieval-Augmented Generation) project into clear, actionable steps based on the hackathon instructions.

## 📂 Recommended Repository Structure

You will use a **private GitHub repository** for the hackathon. Ensure you add `bachtn` and `lachhebo` as collaborators.

```text
DocuMind/
├── .env                          # API keys (DO NOT COMMIT)
├── .gitignore                    # Updated to ignore .env, cache
├── README.md                     # Updated with setup instructions
├── requirements.txt              # Python dependencies
├── cost_tracker.json             # Track API spending
│
├── data/
│   ├── pdfs/                     # Your 7 PDFs (already have)
│   ├── corpus.json               # All extracted text (already have)
│   ├── sample.json               # Sample 10 pages (already have)
│   ├── questions.json            # Eval set (already have)
│   ├── chunks.json               # Output from embedding script
│   └── my_index.faiss            # FAISS index (saved, not committed)
│
├── scripts/
│   ├── extract.py                # PDF extraction (already have)
│   ├── EXTRACTION_REPORT.md      # Report (already have)
│   ├── 01_chunk_corpus.py        # NEW: Chunk text
│   ├── 02_embed_chunks.py        # NEW: Create embeddings
│   ├── 03_build_index.py         # NEW: Build FAISS index
│   └── 04_test_rag.py            # NEW: Test pipeline
│
└── app/
    ├── __init__.py               # Make it a package
    ├── app.py                    # NEW: Main Gradio UI
    ├── rag.py                    # NEW: RAG pipeline core
    ├── utils.py                  # NEW: Helper functions
    └── cost_tracker.py           # NEW: Cost tracking
```

## 🛠 Prerequisites & Environment Setup
1. **Python version:** Use **Python 3.12**.
2. **Package Manager:** Use **uv** for environment management.
3. Setup commands:
   ```bash
   uv venv --python 3.12
   source .venv/bin/activate  # On Windows: .venv\Scripts\activate
   # Recommended base packages
   uv pip install pymupdf openai faiss-cpu tiktoken gradio python-dotenv beautifulsoup4 numpy
   ```

---

## 📝 Step-by-Step Execution Plan

### Step 1: Team Formation & Corpus Collection
**Deadline:** Tuesday, April 7 at 23:42
- Form a team of 4 within your program (AIS or DSA).
- Fill out the Microsoft Form provided by the instructors.
- Choose a topic and collect **at least 100 pages** of factual, verifiable text (PDFs, text files, or scraped web pages). Avoid poetry or opinion pieces.
- Place your collected documents locally inside your `data/sources/` directory.

### Step 2: Corpus Preparation & Text Extraction
**Deadline:** Friday, April 10 at 23:42
- Write `scripts/extract.py` to process your documents from `data/sources/`.
- Use `PyMuPDF` (default) or `pdfplumber` (for heavy tables) for PDFs. Use `BeautifulSoup` for HTML.
- **Clean the text** by removing extra whitespace, repeated headers/footers, and navigation bars.
- Save the full output to `data/corpus.json` and a 10-entry preview to `data/sample.json`.
- Each JSON entry must follow this exact format:
  ```json
  [
    {
      "source": "filename.pdf",
      "page": 1,
      "char_count": 2847,
      "text": "Extracted text..."
    }
  ]
  ```
- Have your script print out the extraction stats (total files, entries, char count, avg chars, empty entries < 50 chars).
- Identify and document your extraction process and any issues in `scripts/EXTRACTION_REPORT.md` (push to GitHub).
- **Upload `corpus.json` and `sample.json` to your Teams channel** (they must NOT be pushed to GitHub).

### Step 3: Prepare Evaluation Questions
**Deadline:** Sunday, April 13 at 23:42
- Write **15 to 20 test questions** manually (Do NOT use GenAI tools for this step — read your corpus).
- Save them in `data/questions.json`.
- Required question distribution:
  - **≥10 Factual questions** (Clear verified answers from a specific section)
  - **≥3 Cross-reference questions** (Requires combining info from multiple parts)
  - **≥3 Out-of-scope questions** (System should answer "I don't have information about this")
  - **≥2 Ambiguous questions** (System should request clarification or give a broad answer)
- Formatting: Include exactly `expected_answer`, `source`, `page`, and `category` alongside the `question` (using nulls for out-of-scope/ambiguous). 

### Step 4: Build the RAG Pipeline
**Constraints:** Raw Python only! Do NOT use frameworks like LangChain or LlamaIndex. Use `gpt-4o-mini` and `text-embedding-3-small`. Team budget is $5.
1. **Chunking & Indexing (Do this ONCE to save API costs):**
   - Create a script (e.g., `app/embed.py`) to split your `corpus.json` text into chunks of 500-1000 characters.
   - Use `text-embedding-3-small` to embed these chunks.
   - Store the vectors in a FAISS index (`faiss-cpu`) and save it to disk (`data/my_index.faiss`). Save chunk metadata to `data/chunks.json`.
   - *Pro-tip: Only ONE teammate should run this. Share the generated `.faiss` and `.json` files on Teams so the team isn't charged repeatedly.*
2. **Retrieval Engine (`app/retrieval.py`):**
   - Embed incoming user queries.
   - Query your FAISS index to retrieve the top-k (e.g., 3-5) closest result chunks.
3. **Generation Engine (`app/llm.py`):**
   - Pass the retrieved chunks + the user query to the `gpt-4o-mini` LLM using an explicit system prompt.
   - Force the LLM to answer *only* using the provided context, include document/page citations, and decline out-of-scope questions gracefully.

### Step 5: Web Interface & Checkpoint
**Mandatory Checkpoint:** Tuesday, 2:00 PM
- Use `gradio` (inside `app/app.py`) to build a Chatbot Web UI.
- The UI must accept a question, execute the retrieval and generation pipeline, and output the response alongside the **source document and page citations**.
- **Crucial:** You must present a fully working Chatbot by the Tuesday 2:00 PM milestone. 

### Step 6: Testing & Final Delivery (Demo Day)
**Deadline:** Friday (Demo Day)
- Start running your `data/questions.json` through the system to evaluate it. Refine your system prompt, chunking size, or text cleaning based on where it fails.
- Complete the code inside the `app/` folder and make sure your `README.md` clearly explains how to install and run the project locally.
- Prepare your 15-minute Demo Day presentation (including a live demo out of your corpus, prepared to answer jury questions).
