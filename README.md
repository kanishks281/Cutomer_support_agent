## Customer Support Copilot – Project Overview

An AI‑powered **customer support agent** that helps human agents reply faster by:

- **Understanding tickets** (subject, description, priority, customer info).
- **Calling tools** (e.g. plan / billing / account checks) when needed.
- **Using customer memory** (past resolutions and signals) and a **knowledge base** so agents don’t need to juggle between many systems.
- Returning a **draft reply** plus rich context, inside a **Streamlit dashboard**.

The backend is built with **FastAPI + LangChain + Mem0 + ChromaDB**, and the UI is a **Streamlit app**.

---

## Problem Statement

Support agents often:

- Switch between many tools (CRM, billing, docs, past tickets).
- Re‑answer similar questions from the same customer or company.
- Spend time copying context into every new reply.

This project’s goal is to build a **copilot** that:

- Reads the **ticket + customer info**.
- Pulls in **relevant memories** (customer history) and **KB snippets**.
- Optionally calls **support tools** (e.g. account / plan / risk checks).
- Produces a **draft email reply** the agent can edit and accept.

---

## High‑Level Architecture

- **FastAPI backend** (`main.py`, `customer_support_agent/api`):
  - REST API for tickets, drafts, health, knowledge ingest, and memory search.
  - Uses a `SupportCopilot` service for LLM + tools + memory + RAG.

- **Copilot service** (`customer_support_agent/services/copilot_service.py`):
  - Wraps a **Groq LLM** via `langchain-groq`.
  - Orchestrates:
    - **Memory** via `CustomerMemoryStore` (Mem0) – optional, can be disabled.
    - **Knowledge base search** via `KnowledgeBaseService` (ChromaDB).
    - **Support tools** via `get_support_tools` (e.g. plan / billing checks).
  - Generates:
    - A **draft reply**.
    - A `context_used` object (signals, memory hits, KB hits, tool calls, errors).

- **Memory & knowledge integrations**:
  - `customer_support_agent/integrations/memory/mem0_store.py`
  - `customer_support_agent/integrations/rag/chroma_kb.py`

- **Streamlit UI** (`app.py`):
  - “Support Copilot Dashboard” for agents.
  - Create tickets, trigger draft generation, edit/accept/discard drafts.
  - Run **knowledge ingest** and **memory probe** on demand.
  - Visualizes `context_used` (memory hits, KB hits, tool calls, errors).

---

## Key Technologies & Tools

- **Language & runtime**: Python **3.11**
- **Backend**: FastAPI, Uvicorn
- **LLM orchestration**: LangChain, `langchain-groq`, `google-genai`
- **Memory**: Mem0 (`CustomerMemoryStore`)
- **RAG / Knowledge base**: ChromaDB
- **UI**: Streamlit
- **Dependency & env management**: `uv` + `.venv`, `.env`

All dependencies are declared in `pyproject.toml` and synced via **uv**.

---

## API Surface (summary)

Backend routes are wired in `customer_support_agent/api/app_factory.py`:

- `GET /health` – health check.
- `POST /api/tickets` – create a ticket (with optional auto‑draft generation).
- `GET /api/tickets` – list tickets.
- `GET /api/tickets/{ticket_id}` – get a single ticket.
- `POST /api/tickets/{ticket_id}/generate-draft` – trigger draft generation.
- `GET /api/drafts/{ticket_id}` – get latest draft for a ticket.
- `POST /api/knowledge/ingest` – ingest local docs into the knowledge base.
- `GET /api/customers/{customer_id}/memories` – list stored memories.
- `GET /api/customers/{customer_id}/memory-search` – search customer memory.

Interactive docs are available at:

- **`http://127.0.0.1:8000/docs`** (Swagger UI)

---

## Environment Variables (.env)

Create a file named `.env` in the project root (`C:\cutomer_support_agent_live-main` on Windows) with at least:

```env
# Required for Groq LLM (draft generation)
GROQ_API_KEY=your_groq_api_key

# Choose ONE embedding provider for Mem0 (recommended Google, or OpenAI),
# or enable local embeddings instead.

## Option A: Google embeddings (recommended)
GOOGLE_API_KEY=your_google_ai_studio_key

## Option B: OpenAI embeddings
# OPENAI_API_KEY=your_openai_key

## Option C: Local embeddings (no external API, heavier)
# ENABLE_LOCAL_EMBEDDINGS=true
```

There are additional tuning settings in `customer_support_agent/core/settings.py`
(e.g. `groq_model`, `rag_top_k`, `mem0_top_k`, host/port), but the above is the
minimum to get a local dev instance working.

---

## Installing Dependencies (Windows, with uv)

1. **Install uv** (if not already installed):

   Open **PowerShell** and run:

   ```powershell
   powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
   ```

   Ensure `C:\Users\<YourUser>\.local\bin` is on your `PATH`. For a single session:

   ```powershell
   $env:Path = "C:\Users\<YourUser>\.local\bin;$env:Path"
   ```

2. **Sync the project environment**:

   In the project root:

   ```powershell
   cd C:\cutomer_support_agent_live-main
   uv sync
   ```

   This will create `.venv` and install everything from `pyproject.toml`.

---

## How to Run Locally

You will typically run **two processes**:

1. **FastAPI backend (API + copilot)**
2. **Streamlit frontend (agent dashboard)**

### 1. Start the backend API

In a terminal (preferably PowerShell or cmd) from the project root:

```powershell
cd C:\cutomer_support_agent_live-main
.\.venv\Scripts\python.exe main.py
```

This launches Uvicorn using the settings from `customer_support_agent/core/settings.py`
(default host/port are typically `0.0.0.0:8000` / `127.0.0.1:8000`).

Verify it’s running:

- `http://127.0.0.1:8000/health` – should return `{"status": "ok"}` (or similar).
- `http://127.0.0.1:8000/docs` – should show FastAPI Swagger UI.

### 2. Start the Streamlit dashboard

In a **second** terminal:

```powershell
cd C:\cutomer_support_agent_live-main
.\.venv\Scripts\streamlit.exe run app.py
```

Streamlit will print a URL like `http://localhost:8501`. Open that in your browser.

The UI will:

- Show an **API Settings** sidebar (with the current `API_BASE_URL`).
- Allow you to **ingest knowledge**.
- Let you **create tickets**, **generate drafts**, and **probe memory**.

---

## Typical Workflow in the UI

1. **Ingest knowledge base (optional but recommended)**
   - Use the sidebar button **“Ingest Knowledge Base”**.
   - Backend will index local docs into ChromaDB for RAG.

2. **Create a new ticket**
   - Fill in customer email, name, company, priority, subject, description.
   - Optionally keep “Auto‑generate draft” checked.
   - Submit the form to create a ticket.

3. **Generate a draft**
   - Select a ticket from the **Tickets** dropdown.
   - Click **“Generate Draft”**.
   - The copilot will:
     - Search memory (if configured).
     - Search the knowledge base.
     - Optionally call support tools.
     - Produce a draft reply plus a `context_used` object.

4. **Review & edit**
   - Edit the draft text in the text area.
   - Optionally inspect the **Context used** section
     (memory hits, KB hits, tool calls, errors).

5. **Accept or discard**
   - **Accept Draft**: saves resolution + updates customer memory.
   - **Discard Draft**: marks it as discarded (no memory update).

6. **Probe customer memory**
   - Use the **Memory Probe** input under the ticket.
   - This queries the customer’s memory store based on your query and ticket context.

---

## Notes on Memory & Knowledge Errors

The system is designed to **fail soft** for optional components:

- If **Mem0 (memory)** cannot be initialized (e.g. missing embedding provider),
  `SupportCopilot` sets an internal `_memory_error` and:
  - Draft generation still works.
  - Memory‑related endpoints return empty results.
  - `context_used["errors"]` includes a message like
    **“Memory disabled: No embedding provider configured…”**.

- If **knowledge search** fails, RAG will be skipped and the LLM will rely on
  ticket + memory + tools only (plus a fallback if needed).

You can see these issues surfaced in the **Context used** section of the UI.

---

## Troubleshooting (Common Issues)

- **`uv is not recognized`**  
  Make sure `uv` is installed and `C:\Users\<YourUser>\.local\bin` is on your `PATH`,
  or call it via full path, e.g.:

  ```powershell
  "C:\Users\<YourUser>\.local\bin\uv.exe" sync
  ```

- **`ModuleNotFoundError: No module named 'uvicorn'`**  
  You are using a Python without project deps. Always run via the venv:

  ```powershell
  .\.venv\Scripts\python.exe main.py
  ```

- **`GROQ_API_KEY is missing` / “Copilot unavailable”**  
  Ensure `.env` contains `GROQ_API_KEY` and restart `main.py`.

- **Mem0 / embeddings error (memory disabled)**  
  Add `GOOGLE_API_KEY` (recommended) or `OPENAI_API_KEY`, or set
  `ENABLE_LOCAL_EMBEDDINGS=true` to use local embeddings, then restart the backend.

- **Swagger UI / root URL**  
  `/` may return `{"detail": "Not Found"}` – this is expected. Use:
  - `http://127.0.0.1:8000/docs` for API docs.
  - `http://127.0.0.1:8000/health` for health.

---

## Future Improvements (Ideas)

- Add authentication and per‑agent views.
- Integrate with real ticketing systems (e.g. Zendesk, Intercom).
- Add richer analytics on memory/tool usage and draft quality.
- Support multiple LLM providers and routing logic.

