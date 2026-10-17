# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file Streamlit RAG (Retrieval-Augmented Generation) Q&A app (`app.py`). Users upload documents (PDF, DOCX, CSV, XLSX, HTML, TXT/MD) or paste text, the app chunks and embeds them with the OpenAI API, indexes the embeddings in an in-memory FAISS index, and answers chat questions by retrieving relevant chunks and passing them to a chat model.

## Running the app

```bash
pip install -r requirements.txt
streamlit run app.py
```

The app runs at `http://localhost:8501`. In the devcontainer/Codespaces environment, `postAttachCommand` runs this automatically (with CORS/XSRF disabled for the forwarded port).

There are no build steps, tests, or lint configs in this repo — it's a single Python file with no test suite.

## Configuration

Config values are resolved by `get_secret()` (app.py): `st.secrets[name]` from `.streamlit/secrets.toml` (gitignored) first, then the environment variable of the same name. It catches `StreamlitSecretNotFoundError`, since `st.secrets` raises when no secrets.toml exists.

- `OPENAI_API_KEY` (required) — without it, the UI shows setup instructions and disables document processing.
- `APP_PASSWORD` (optional) — when set, `check_password()` gates the whole app behind a password prompt. Unset locally = no gate.

## Deployment (Railway)

Live at https://blue-rag-production.up.railway.app (Railway project `blue-rag`, service `blue-rag`, environment `production`).

- **Auto-deploys from GitHub**: the service is connected to `cmurray4492/blue_rag` branch `main`; every push to `main` rebuilds and redeploys. Don't use `railway up` — it deploys local files instead of what's on GitHub.
- **Builder**: Railpack, installing from `requirements.txt`. Python version is pinned by `.python-version` (3.13).
- **Start command**: set via the Railway variable `RAILPACK_START_CMD` (`streamlit run app.py --server.port $PORT --server.address 0.0.0.0 --server.headless true`). Railpack doesn't detect Streamlit and falls back to `python app.py`; a `railway.json` `startCommand` was ignored when deploying via `railway up`, so the variable is the source of truth.
- **Secrets**: `OPENAI_API_KEY` and `APP_PASSWORD` are Railway service variables (`railway variable list` / `railway variable set KEY --stdin`), never committed.
- **Checking a deploy**: `railway deployment list --json` (status + commit hash), `railway logs`, and `curl https://blue-rag-production.up.railway.app/_stcore/health`.

## Architecture

Everything lives in `app.py`, organized into three sections:

- **Document Readers** (`read_txt`, `read_pdf`, `read_docx`, `read_csv`, `read_excel`, `read_html`, dispatched by `read_file`) — each returns a `(content, file_type)`-style plain-text extraction from its format. CSV/Excel are converted to markdown tables via `df.to_markdown()`; HTML strips script/style/nav/footer/header before extracting text.
- **Core RAG functions**:
  - `chunk_text` — naive word-count-based chunking with overlap (defaults: 500 words, 50 overlap; both are user-adjustable in the sidebar).
  - `get_embeddings` — batches text through the OpenAI embeddings API (`text-embedding-3-small`), 100 texts per request.
  - `create_faiss_index` — wraps embeddings in a flat L2 `faiss.IndexFlatL2`.
  - `search_similar` — embeds the query and does a top-k FAISS search to retrieve chunks.
  - `generate_answer` — sends retrieved chunks + query to the chat model (`gpt-4o-mini`) with a system prompt that constrains answers to the provided context.
- **Streamlit UI** (`main()`) — all state (chunks, FAISS index, chat history, doc stats, top_k) lives in `st.session_state`; there is no persistence layer, so everything resets on server restart or "Clear All". The sidebar handles config/upload/processing; the main pane is a `st.chat_input`/`st.chat_message` loop that calls `search_similar` then `generate_answer` per turn and shows retrieved chunks in a "Sources" expander.

Model names and chunk defaults are top-of-file constants (`EMBEDDING_MODEL`, `CHAT_MODEL`, `CHUNK_SIZE`, `CHUNK_OVERLAP`, `SUPPORTED_TYPES`) — change behavior there rather than inline.

Since the FAISS index and chunks are rebuilt from scratch on every "Process Documents" click (not incrementally updated), adding a new document type means: write a `read_<type>` function returning extracted text, add a branch in `read_file`, and add the extension to `SUPPORTED_TYPES`.
