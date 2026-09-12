# Hardening

## Current posture

- **Authentication / authorization:** none. The Streamlit app binds to port 8501 with no login; anyone who can reach the port can read every record and use the Admin panel. The only access control in the system is per-question patient scoping inside the agent (`enforce_hospital_number`), which constrains the model, not the human.
- **Secrets:** a single secret, `OPENAI_API_KEY`, read from the environment or a `.env` file. `.env`, `venv/`, and `data/*.db` are gitignored. No credentials are committed at HEAD; the only key-like strings in the repository are the `your-api-key-here` placeholders in `README.md` and `run.sh`.
- **Injection surface:** all SQL is parameterized throughout `app.py`; no string-built queries. Uploaded CSV/JSON is parsed with the standard library and field-filtered. Retrieved document text and OCR output are, however, placed into model context unsanitized — a document containing instruction-like text is a prompt-injection vector for the records agent.
- **Error handling:** database integrity errors and OCR failures are caught and shown in the UI; the agent is instructed to surface tool errors and stop. There are no timeouts or retries on OpenAI calls.
- **Observability:** none. No logging, no audit trail of who viewed or edited which record, no request metrics. `st.rerun()`-driven flow leaves no persistent trace of admin actions beyond the data change itself.
- **Data at rest:** a plain, unencrypted SQLite file (`data/patients.db`) on local disk.
- **Concurrency:** `ACTIVE_HOSPITAL_NUMBER` is process-global; concurrent browser sessions could interleave agent scoping (detailed in ARCHITECTURE.md). Safe only as a single-operator tool.

## PHI posture — read before pointing this at real data

This application ships with synthetic patients and, **as it stands, is not suitable for real protected health information.** Specifically:

1. Patient profiles, notes, history chunks, and full document images are sent to OpenAI's APIs (embeddings, the agent's model, and GPT-4o vision for handwritten scans). Handling real PHI this way requires a Business Associate Agreement with the API provider and a deliberate data-flow review; none of that is represented in this code.
2. There is no authentication, no per-user authorization, and no audit logging — each independently disqualifying for HIPAA-regulated use.
3. The database and any uploaded scans are stored unencrypted on the local filesystem.
4. The agent's patient-scoping guard is a correctness control for model behavior, not a security boundary: the UI freely exposes all patients to any visitor.

Treat the current system as a demonstrator of the architecture on mock data. The ladder below is the honest distance to production.

## Production readiness ladder

**Stage 1 — Identity and keys**

- Put the app behind real authentication (reverse proxy with OIDC/SSO, or Streamlit's supported auth options) and add role separation: read-only clinical view vs. the Admin panel's write/import functions.
- Move `OPENAI_API_KEY` from `.env` into a secrets manager appropriate to the deployment target; scope the key and set spend limits.
- Fix the `ACTIVE_HOSPITAL_NUMBER` global (per-run context or tool closures, per ARCHITECTURE.md) so scoping holds under concurrent users.

**Stage 2 — Monitoring and resilience**

- Add structured logging around the three external-call sites (embeddings, agent run, vision OCR) with timing, token/cost fields, and failure classification; add timeouts and bounded retries with backoff.
- Add an append-only audit log of admin actions (who, what record, before/after) — a prerequisite for any clinical setting and cheap to add at the nine persistence functions.
- Health checks: DB reachable, index build time, and a canary Q&A against a known synthetic record.

**Stage 3 — Deployment**

- Containerize (the runtime is Python + Tesseract + Poppler; `run.sh` documents the launch); run behind TLS.
- Replace per-rerun index rebuilds with cached, incrementally updated stores so a multi-user deployment does not multiply embedding spend (ARCHITECTURE.md, "Extending this system").
- Migrate SQLite to a managed database with encryption at rest and automated backups when moving beyond a single node; the persistence layer is already isolated in nine functions.

**Stage 4 — Compliance (only if real PHI is ever in scope)**

- Execute a BAA covering every model API in the data path, or swap to a provider/deployment where one exists; re-evaluate whether handwritten-scan images may leave the boundary at all.
- Encrypt data at rest and in transit end to end; define retention and deletion procedures for documents and OCR extracts.
- Add prompt-injection mitigations for document-derived context (content provenance tags, instruction filtering) and human review gating for low-confidence OCR before it becomes retrievable record content.
- Independent access review and penetration test before any clinical exposure.

## Secrets removed from HEAD

None. A scan of all tracked files at HEAD found no API keys, tokens, connection strings, committed `.env` files, or key material — only documentation placeholders. Nothing required redaction or rotation.
