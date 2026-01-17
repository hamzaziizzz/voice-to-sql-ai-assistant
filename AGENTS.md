# AGENTS.md (Codex)

This repository is a production-ready template for a **Voice/Text → SQL → ERP/DB AI Assistant**.
Codex should prioritize **safety, determinism, and clarity** over cleverness.

---

## 1) Goals

- Convert user **voice/text** queries into **validated SQL** against a configured schema.
- Prevent hallucinations by **schema grounding** + **SQL validation** + **read-only by default**.
- Provide clean, auditable outputs: generated SQL, execution metadata, and a natural-language answer.

---

## 2) Non-Goals (Do NOT do these)

- Do not generate or execute destructive SQL (DROP/DELETE/UPDATE/INSERT/ALTER) unless explicitly enabled.
- Do not bypass validation rules or permissions to “make it work.”
- Do not hardcode any secrets (API keys, DB creds). Use `.env` only.

---

## 3) High-Level Architecture

Pipeline:

1. **STT** (optional): audio → text
2. **NLU**: intent + entities + module routing
3. **Disambiguation**: resolve ambiguous entities/values (vector search / synonym maps)
4. **SQL Generation**: LLM produces SQL grounded in schema + constraints
5. **SQL Validation**: allowlist tables/columns, enforce SELECT-only, cap rows
6. **DB Execution**: execute safely, timeouts, row limits
7. **Response**: summarize results + return structured metadata

Key modules (suggested):
- `services/stt_service.py`
- `services/nlu_service.py`
- `services/disambiguation_service.py`
- `services/sql_generator.py`
- `services/sql_validator.py`
- `services/db_service.py`
- `services/response_service.py`
- `adapters/llm_gemini.py`, `adapters/vector_milvus.py`, `adapters/db_postgres.py`

---

## 4) Commands (Local)

If `docker-compose.yml` exists:

- Start:
  - `docker compose up --build`
- Stop:
  - `docker compose down -v`

If running locally (no Docker):
- `python -m venv .venv && source .venv/bin/activate`
- `pip install -r requirements.txt`
- `uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload`

Tests:
- `pytest -q`

Lint/format (if configured):
- `ruff check .`
- `black .`

---

## 5) Safety Rules (SQL)

Codex MUST enforce these by default:

- **SELECT-only mode** (default ON)
- Allowlist **tables & columns**
- Block:
  - `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `TRUNCATE`, `GRANT`, `REVOKE`
  - multi-statements `;` chaining
  - dangerous functions (configurable)
- Add:
  - `LIMIT <MAX_ROWS>` if missing
- Timeouts + retry rules in `db_service`
- Log:
  - query text, generated SQL, validation outcome, execution time, row count
  - never log secrets

Environment toggles:
- `READ_ONLY_MODE=true|false`
- `MAX_ROWS=...`
- Optional: `ALLOWED_TABLES=table1,table2`

---

## 6) LLM Prompting Guidelines

When editing prompts:
- Always include:
  - Database dialect (Postgres/MySQL/etc.)
  - Allowlisted tables/columns
  - Output format constraints (SQL only; no markdown)
  - Mandatory LIMIT / safe filters
- Prefer deterministic outputs:
  - Lower temperature
  - Strict templates
- Add a “reasoning” section internally if needed, but never return it in API response.

---

## 7) Coding Conventions

- Python 3.10+.
- Prefer **type hints**.
- Keep functions small and testable.
- Errors:
  - Raise domain-specific exceptions (e.g., `ValidationError`, `LLMError`, `DBError`)
  - Return structured error payloads in API.

Folder responsibility:
- `services/` = business logic
- `adapters/` = external dependencies (LLM/DB/vector)
- `api/` = FastAPI routes & schemas
- `prompts/` = prompt templates

---

## 8) API Contract (Recommended)

Endpoints:
- `POST /query/text`
- `POST /query/voice`

Response should include:
- `query` (original user query)
- `sql` (final SQL executed)
- `answer` (natural language)
- `metadata` (module, confidence, time_ms, row_count, warnings)

---

## 9) Adding a New ERP Module

Steps:
1. Add schema mapping in `docs/` (or `config/`).
2. Add intent(s) to NLU:
   - module routing
   - entity extraction rules
3. Add disambiguation rules:
   - synonyms
   - vector collection (if used)
4. Add prompt grounding:
   - include module-specific tables & columns
5. Add tests:
   - “happy path” + “ambiguous” + “invalid SQL attempt” cases

---

## 10) What Codex Should Do First When Asked to Implement Something

1. Identify where the change belongs (`api` vs `services` vs `adapters`).
2. Add/modify unit tests first (when feasible).
3. Implement minimal safe version.
4. Ensure SQL validation rules still hold.
5. Update docs if behavior changes.

---

## 11) Secrets & Compliance

- Never commit `.env`.
- Do not store user audio or PII unless explicitly requested and documented.
- If adding logs, ensure PII redaction is possible.

---

## 12) Quick “Definition of Done”

A change is complete when:
- Tests pass
- Validation rules remain enforced
- API responses are stable and structured
- Docs reflect the new behavior (if relevant)
