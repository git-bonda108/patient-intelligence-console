# Architecture

Everything runs in one process from one file, `app.py` (~1,300 lines). Streamlit executes the whole script top to bottom on every user interaction, which shapes most of the design decisions below.

## Component map

| Component | Where | Responsibility |
|---|---|---|
| Persistence layer | `app.py` — `init_db`, `fetch_patients`, `fetch_patient_history`, `fetch_doctor_notes`, `fetch_patient_documents`, `add_doctor_notes`, `add_documents`, `create_patient`, `update_patient` | SQLite schema creation, seeding from JSON, and all CRUD. Every function opens and closes its own `sqlite3` connection; every statement is parameterized. |
| Seed data | `data/patients.json` | 10 synthetic patients used only when the `patients` table is empty. |
| Input parsing | `parse_history`, `parse_notes`, `parse_documents`, `parse_notes_upload`, `parse_documents_upload`, `normalize_note_type` | Turn the Admin panel's line-oriented text formats (`type|note`, `type|title|content`) and CSV/JSON uploads into row dicts; unknown note types collapse to `digital`. |
| OCR pipeline | `ocr_image`, `image_to_base64`, `read_handwritten_with_vision`, `ocr_from_upload` | Two extraction paths selected by the user: Tesseract for printed text (with a real word-level confidence average), GPT-4o vision for handwritten text (a fixed 95.0 placeholder confidence, since the API returns none). PDFs are rasterized page-by-page via `pdf2image` and joined with `---PAGE BREAK---` markers. |
| Retrieval layer | `build_patient_documents`, `build_vectorstores` | Converts each patient row into typed LangChain `Document`s (profile, doctor_note, document, history — each carrying `hospital_number` metadata), splits at 350 chars / 40 overlap, and builds **one FAISS index per patient**. |
| Agent layer | `enforce_hospital_number`, `get_patient_profile`, `search_patient_history`, `build_agent`, `ask_agent` | OpenAI Agents SDK `Agent` with two `@function_tool`s, run synchronously per question. |
| UI | module-level Streamlit code (bottom ~400 lines) | Two-column layout: patient profile browser plus Admin panel (add/edit, CSV/JSON import, OCR import) on the left; agent Q&A chat on the right. |
| Launcher | `run.sh` | Activates `venv`, refuses to start without `OPENAI_API_KEY`, runs Streamlit on port 8501. |

Dependency note: `requirements.txt` pins several packages `app.py` never imports (`chromadb`, `plotly`, `pandas`, `langchain` core). Installs work but are heavier than the code requires.

## Data flow, end to end

1. **Startup (every script run).** `load_dotenv()` → hard stop if `OPENAI_API_KEY` is missing → `init_db` creates four tables (`patients`, `patient_history`, `doctor_notes`, `patient_documents`) if absent, applies a one-off column migration (`ocr_confidence` added via `PRAGMA table_info` check), and seeds from JSON only when the patients table is empty.
2. **Load and index.** `fetch_patients` reads every patient with all child rows; `build_vectorstores` embeds every chunk and builds a FAISS index per hospital number. This happens at module level with no `st.cache_resource`, so **each Streamlit rerun re-reads the database and re-embeds the full corpus** — correct-by-reconstruction, paid for in embedding calls and latency (see trade-offs).
3. **Question path.** The user selects a patient and submits a question. `ask_agent` sets the module global `ACTIVE_HOSPITAL_NUMBER`, builds a fresh `Agent` whose instructions embed the selected hospital number, and calls `Runner.run_sync`. The model may call `get_patient_profile` (dict from the in-memory `PATIENT_MAP`) and/or `search_patient_history` (top-4 similarity search against that patient's FAISS index). The final output is appended to `st.session_state.qa_history` and rendered as chat.
4. **Write paths.** Admin add/edit, CSV/JSON imports, and OCR imports write to SQLite, then call `st.rerun()` — which re-executes the script and therefore rebuilds the indexes, so new data is immediately retrievable.

## Orchestration analysis

- **Everything is sequential and synchronous.** One agent, one question at a time, `Runner.run_sync` blocking the Streamlit script. Tool calls happen inside the SDK's internal loop; the application code has no async, threads, or queues. For a single-operator console this is the simplest correct choice — the cost is that the UI blocks for the duration of a model round-trip (and per-page vision calls during handwritten OCR of multi-page PDFs, which run in a sequential `for page in pages` loop).
- **Agent construction is per-question.** `build_agent` is cheap (instructions string plus two tool references), so rebuilding per question keeps the hospital-number constraint fresh with zero state carried between questions. The agent has no conversation memory: each question stands alone; prior Q&A shown in the chat panel is display-only.
- **Scoping is enforced twice.** The instructions tell the model to answer only for the selected hospital number, and — independently — every tool call passes through `enforce_hospital_number`, which rejects any `hospital_number` argument that does not match `ACTIVE_HOSPITAL_NUMBER`. Prompt-level restriction is advisory; the tool guard is the actual control.

## State and context engineering

- **Durable state:** SQLite at `data/patients.db` (gitignored). Update semantics in `update_patient` are full-replace: child rows (history, notes, documents) are deleted and re-inserted from the form contents.
- **Session state:** only `st.session_state.qa_history` (the visible transcript). Nothing else survives a rerun except through the database.
- **Process state:** `ACTIVE_HOSPITAL_NUMBER` is a module global, set before `Runner.run_sync` and cleared after. Because Streamlit serves all browser sessions from one Python process, two users asking questions concurrently could interleave on this global. With one user (the intended use) this cannot occur, but it is the first thing to fix before multi-user deployment (see below and HARDENING.md).
- **Context assembly:** retrieval context is bounded by construction — chunks of ≤350 characters, k=4, drawn only from the selected patient's index. Each document is prefixed with its kind ("Patient Profile", "Doctor Note (handwritten):", "Patient History (HN-…):"), so the chunks the model sees are self-describing. The profile is also available whole via `get_patient_profile`, so the model can combine exact vitals with semantically retrieved history.

## Design decisions and trade-offs visible in the code

1. **One FAISS index per patient, not one index with metadata filtering.** Isolation is structural: a similarity search physically cannot return another patient's chunks, which is the right bias for medical records. The cost is N indexes rebuilt on every rerun and no cross-patient queries (which the product deliberately does not offer).
2. **Rebuild-everything on rerun instead of caching/incremental updates.** No cache invalidation bugs, and writes are trivially consistent with retrieval — at the price of full re-embedding per interaction. Fine at 10 patients; the scaling knob is obvious and local (`build_vectorstores`).
3. **Dual OCR paths keyed by the user's declaration, not auto-detection.** The user says "handwritten" or "digital"; the code trusts that and routes to GPT-4o vision or Tesseract respectively. Simpler and more predictable than classification, and the two paths have honest, different confidence semantics (measured average vs. fixed placeholder).
4. **Per-call SQLite connections.** Every persistence function does `sqlite3.connect` / `finally: close`. No pooling, no shared cursor state, no cross-request leakage — appropriate for SQLite's model and a single-process app.
5. **Defensive optional dependencies.** `pytesseract` and `pdf2image` import inside `try/except ImportError`; the UI checks for `None` and tells the user exactly what to install rather than crashing at import time.
6. **Errors surface to the operator.** `sqlite3.IntegrityError` on duplicate patient creation and any OCR exception are caught and rendered as Streamlit error banners; the agent is instructed to relay tool errors and stop rather than improvise.

## Extending this system

Grounded next steps that the current structure makes cheap:

1. **Cache and incrementally update the vector stores.** `build_vectorstores` is a pure function of the patient rows. Wrapping it in `st.cache_resource` keyed on a data version, and re-embedding only the written patient's index after each Admin write (every write path already knows its `hospital_number`), removes the per-rerun embedding cost without touching retrieval semantics.
2. **Replace the `ACTIVE_HOSPITAL_NUMBER` global with per-run context.** The Agents SDK supports passing a context object through `Runner` into tools; alternatively, `build_agent` can close the tools over the selected hospital number. Either change deletes the only shared-mutable state in the request path and makes the app safe for concurrent sessions.
3. **Stream agent answers.** The right panel already renders `st.chat_message` blocks; switching `Runner.run_sync` to the SDK's streamed runner would turn the current blocking wait into progressive output with no data-model changes.
4. **Gate low-confidence OCR behind review.** `ocr_confidence` is already persisted per document and displayed in the UI. A threshold check at the `add_documents` call site could route low-confidence extractions into a pending state for human confirmation before they are embedded and become retrievable "truth".
5. **Lift persistence behind one interface.** The nine persistence functions share a `db_path`-first signature and contain all SQL. Consolidating them into a repository class is a mechanical refactor that opens the way to Postgres (and to encryption at rest and audit logging, which HARDENING.md treats as prerequisites for any real deployment).
