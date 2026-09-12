# Evaluation

## Current state: no automated tests

There is no test suite in this repository. No test files, no test framework in `requirements.txt`, no CI configuration, and no metrics recorded anywhere in code or docs. Any quality claim about the agent, retrieval, or OCR paths is currently unverified.

What follows is (a) an inventory of the defensive behavior that does exist in `app.py`, and (b) a proposed evaluation harness.

## Edge cases the code visibly handles

Enumerated from `app.py` — each item names the function that implements it:

**Startup and dependencies**

- Missing `OPENAI_API_KEY` stops the app with an explicit error banner before any model call (module-level check, `st.stop()`); `run.sh` refuses to launch without it.
- `pytesseract` and `pdf2image` are optional: imports are wrapped in `try/except ImportError`, and both `ocr_image` / `ocr_from_upload` and the OCR UI check for `None` and name the missing package instead of crashing.
- Schema drift: `init_db` detects a missing `ocr_confidence` column via `PRAGMA table_info` and adds it, so databases created before that column existed still open.
- Seeding is idempotent: `init_db` inserts the JSON seed only when the `patients` table is empty.

**Agent and retrieval**

- `enforce_hospital_number` rejects tool calls when no active profile is set or when the requested hospital number differs from the selected one, returning a structured error the agent is instructed to relay.
- `search_patient_history` rejects empty queries and reports a missing vector store rather than raising.
- `get_patient_profile` returns a "Patient not found" error for unknown hospital numbers.

**Writes and imports**

- Duplicate patient creation (`patient_id` primary key or `hospital_number` UNIQUE violation) is caught as `sqlite3.IntegrityError` and shown as a UI error.
- `add_doctor_notes` / `add_documents` return an `(inserted, skipped)` tuple; rows referencing an unknown hospital number are skipped and counted, and the UI reports both totals.
- Upload parsers (`parse_notes_upload`, `parse_documents_upload`) drop rows without a hospital number, ignore unsupported file extensions (returning `[]`), and the UI additionally filters out notes without an entry and documents without content.
- `normalize_note_type` collapses any type other than `digital`/`handwritten` to `digital`.

**OCR**

- All OCR exceptions are caught at the UI call site and rendered as an error banner; `read_handwritten_with_vision` wraps the vision call and re-raises with context.
- Empty extraction results produce a warning ("OCR produced no readable text") instead of writing an empty document.
- `ocr_image` filters out Tesseract's sentinel negative confidences before averaging; `image_to_base64` converts RGBA/LA/P images to RGB before JPEG encoding.
- Multi-page PDFs are processed page-by-page with per-page confidences averaged and pages joined with explicit break markers.

## Known gaps (visible in the code, not handled)

- No timeouts, retries, or rate-limit handling on any OpenAI call (embeddings, agent run, vision OCR) — a hung or throttled call blocks the UI.
- `ACTIVE_HOSPITAL_NUMBER` is a module global; concurrent sessions in one Streamlit process could interleave (see ARCHITECTURE.md).
- The vision OCR confidence is a fixed 95.0 placeholder, not a measurement; it is honest in a code comment but indistinguishable from a real score in the database.
- `update_patient` performs delete-and-reinsert of child rows without a transaction boundary distinct from its single implicit connection commit; a mid-function failure after the deletes would lose notes/history.

## Proposed evaluation harness

Nothing below exists yet.

**1. Golden Q&A dataset (agent grounding).**
A checked-in JSONL of ~50 cases over the synthetic patients: `{hospital_number, question, must_contain: [...], must_not_contain: [...]}`. `must_not_contain` should include facts belonging to *other* patients (e.g., asking HN-40218's console about Metformin, which belongs to HN-91024) to test isolation, and questions with no answer in the record to test refusal. Run with the real agent against the seed database; assert substring/regex presence.

**2. Scope-enforcement unit tests (no model needed).**
Direct tests of `enforce_hospital_number`, the tool functions with a mismatched hospital number, and the parsers (`parse_notes`, `parse_documents`, upload variants) — these are pure or near-pure functions and cover the security-relevant logic deterministically.

**3. Retrieval quality checks.**
For each seeded history/note entry, query its own paraphrase and assert the source chunk appears in the top-4 (`search_patient_history` k=4). Track recall@4 per patient; gate on 100% for the seed corpus, which is small enough to demand it.

**4. OCR fixture suite.**
A folder of fixture images/PDFs (printed text, handwritten samples, blank page, RGBA PNG, multi-page PDF) with expected-substring assertions for the Tesseract path, and a recorded-response (cassette) test for the vision path so CI does not require live GPT-4o calls.

**5. Persistence round-trip tests.**
Against a temp SQLite file: create → fetch → update (verify full-replace semantics) → import with unknown hospital numbers (verify skip counts) → duplicate insert (verify IntegrityError).

**Gates.** A reasonable CI gate for this codebase: parsers and scope tests 100% pass, retrieval recall@4 = 1.0 on seed data, golden Q&A pass rate ≥ 90% with zero cross-patient leaks (any leak fails the build outright).
