# System Architecture & Code Flow Reference

This document provides a comprehensive overview of the DocuMind platform's architecture, data flow, technology stack, and a detailed file-by-file function reference.

---

## 1. High-Level Architecture
DocuMind is an end-to-end **Retrieval-Augmented Generation (RAG)** pipeline. It translates raw PDF documents into a machine-readable format, semantically chunks and indexes them, and serves an interactive web UI where an LLM answers user questions strictly grounded in the document context.

### Data Flow
1. **Extraction & Cleaning:** `extract.py` scans `data/pdfs/`, pulls text using PyMuPDF, cleans whitespace/artifacts, and outputs a structured `corpus.json`.
2. **Semantic Chunking:** The pipeline reads `corpus.json` and breaks the text into semantic chunks based on sentence boundaries, keeping context intact.
3. **Embedding:** OpenAI's embedding model translates these chunks into dense vector representations.
4. **Indexing:** FAISS stores these vectors for rapid semantic similarity search. Rank-BM25 also indexes the text tokens for eventual hybrid support.
5. **Retrieval (At Query Time):** User input is embedded, and FAISS fetches the nearest chunks. If the query is a comparison (e.g., "compare A and B"), the retriever performs a batch search and interleaves the top documents for both entities.
6. **Generation:** An OpenAI chat model receives a strict, 5-step system prompt alongside the retrieved text chunks to generate a safe, hallucination-free response with inline citations.
7. **Frontend (Gradio):** Displays the chat, generates real-time highlights on the source PDF pages, and tracks cumulative API costs.

---

## 2. Technology Stack
*   **Python 3:** Core programming language.
*   **Gradio:** Builds the interactive web-based frontend and manages chat state.
*   **PyMuPDF (fitz):** Handles heavy-duty PDF text extraction and generates image renderings of PDF pages with text highlight overlays.
*   **FAISS (cpu):** Facebook AI Similarity Search. Used to store document embeddings and perform hyper-fast vector nearest-neighbor lookups.
*   **OpenAI API:**
    *   `text-embedding-3-small`: Converts text chunks into mathematical vectors capturing semantic meaning.
    *   `gpt-4o-mini`: The Chat LLM responsible for reading the context and generating the final answer.
*   **Rank-BM25:** Builds a token-based inverted index (useful for exact keyword matching and hybrid RRF retrieval architectures).
*   **Regex (re):** Powers the semantic chunker to cleanly cut text along punctuation boundaries instead of cutting words in half.

---

## 3. Comprehensive File & Function Reference

### Frontend (`app/app.py`)
Responsible for running the web server, managing UI components, calling the RAG pipeline, and rendering PDF images.

*   `get_pipeline()`: Initializes or returns the global singleton instance of `RAGPipeline`.
*   `ensure_pipeline_ready()`: Checks if the FAISS index and chunks exist. Returns `(bool, msg)`.
*   `get_header_html() -> str`: Returns the Base64-encoded HTML string used to render the custom UI banner and logo.
*   `get_pdf_page_image(filename: str, page_num: int, text_to_highlight: str) -> PIL.Image`: 
    *   **Inputs:** PDF file name, specific page number, and the exact text string to highlight.
    *   **Outputs:** A PIL Image object of the rendered PDF page with the discovered text highlighted in yellow.
*   `get_cost_markdown() -> str`: Fetches the current API cost from `cost.py` and returns a formatted markdown string.
*   `respond(message: str, chat_history: list) -> generator`: 
    *   **Inputs:** The user's typed message and the ongoing Gradio chat history state.
    *   **Outputs:** Continuously `yields` UI updates (thinking states, final LLM string, PDF gallery images, and cost updates).
*   `build_index_ui() -> dict`: Triggered by a button click. Forces the pipeline to rebuild chunks, vectors, and FAISS index, updating the UI upon completion.
*   `handle_pdf_upload(file_objs)`: Saves uploaded files directly to `data/pdfs/` and instructs the user to rebuild the index.

### RAG Orchestration (`scripts/rag/pipeline.py`)
Acts as the central conductor managing all discrete moving parts of the RAG system.

*   `__init__(...)`: Sets paths and initializes embedders, retrievers, generators, and models.
*   `build() -> int`: 
    *   **Flow:** Loads corpus -> chunks it -> embeds chunks -> builds FAISS index -> saves artifacts to disk.
    *   **Outputs:** Returns the total number of chunks created.
*   `load_if_needed()`: Lazy-loads FAISS index, Chunk JSON, and BM25 index from disk into memory if not already loaded.
*   `ask(query: str, method: str, k: int) -> Tuple[str, List]`: Uses the retriever to get chunks, then the generator to get an answer. Returns the `(answer_string, retrieved_chunks_list)`.

### Document Splitting (`scripts/rag/chunker.py`)
*   `__init__(chunk_size=1500, overlap=200)`: Sets the semantic length constraints.
*   `chunk_text(text: str) -> List[str]`: Uses regex `(?<=[.!?])\s+` to split text logically by sentences, accumulating them until they approach `chunk_size`, maintaining `overlap` via token backtracking.
*   `chunk_corpus(corpus: List[Dict]) -> List[Dict]`: Wraps `chunk_text`, mapping the resulting sub-strings back to their parent PDF filename and page number.

### Vector Conversion (`scripts/rag/embedder.py`)
*   `get_embeddings(texts: List[str]) -> List[List[float]]`: Calls OpenAI API to convert a list of text strings into a list of vector arrays.
*   `embed_chunks(chunks: List[Dict]) -> np.ndarray`: Loops over all dictionaries, extracts the `"text"`, fetches embeddings, and converts the whole batch into a 2D numpy array.

### Database Indexing (`scripts/rag/indexer.py`)
*   `build(vectors: np.ndarray) -> faiss.Index`: Creates a `faiss.IndexFlatIP` (Inner Product) index for cosine similarity and adds the numpy vectors.
*   `save(index, path)` / `load(path) -> faiss.Index`: Wrappers around FAISS disk I/O.

### Document Fetching (`scripts/rag/retriever.py`)
*   `retrieve(query: str, index: faiss.Index, chunks: List[Dict], bm25, method, k) -> List[Dict]`: 
    *   **Inputs:** The user query, loaded FAISS index, chunk metadata dictionary, and retrieval configuration.
    *   **Flow:** Detects if the user asked a comparison question using RegEx (e.g. "compare A and B"). If so, extracts Entities A and B, runs batched vector searches for both, and logically interleaves the top-$K$ results ensuring documents for both entities pull equally.
    *   **Outputs:** A filtered, sorted list of dictionary chunks most relevant to the query/queries.

### Final Brain (`scripts/rag/answer_generator.py`)
*   `generate(query: str, retrieved_chunks: List[Dict]) -> str`:
    *   **Inputs:** The user's query and the Top-K chunks returned by `retriever.py`.
    *   **Flow:** Formats the chunks into a context string. Sends it to OpenAI's Chat API bound by an extremely strict 5-step System Prompt that forces: Scope Checks, Ambiguity rejection, Prompt Injection defenses, and forced Comparison schemas.
    *   **Outputs:** The final, cited answer string.

### API Cost Tracker (`scripts/rag/cost.py`)
*   `track_cost(response)`: Intercepts the OpenAI `response.usage` object. Calculates cost based on whether it was an embedding call ($0.02 / 1M tokens) or a GPT-4o-mini generation call ($0.15 in / $0.60 out / 1M tokens) and saves it associatively to `data/cost.json`.
*   `get_total_cost() -> float`: Reads `cost.json` and returns the cumulative session expense.

### Extraction Pipeline (`scripts/extract.py`)
*   *(Various standalone PyMuPDF traversal functions)*: Iterates through PDFs, cleans standard ASCII control characters, identifies image-only pages that yield no text length, and dumps the finalized dataset into `corpus.json` while drafting `EXTRACTION_REPORT.md`.

### CLI Runner (`app/cli.py`)
*   *(DEPRECATED/CLI WRAPPER)* This file is a simple `argparse` wrapper that allows you to run the RAG pipeline directly from the command line (e.g. `python app/cli.py ask --query "..."`) without starting the Gradio UI. It imports and relies entirely on the functional brain inside `scripts/rag/pipeline.py`.
