# Aster Learning

Aster is a full-stack learning and assessment foundation with a Next.js interface and a Python/FastAPI API. It implements real account registration, private question banks, question snapshots, timed assessments, persisted answers, server-authoritative grading, attempt history, topic analytics, document text extraction, and an optional provider-backed question-generation pipeline.

This repository is a working product foundation, **not a claim that every feature in the original broad product brief is production-complete**. See [Current scope and limitations](#current-scope-and-limitations).

## Architecture

- `frontend/`: Next.js 16, React 19, TypeScript, responsive interface connected to the API.
- `backend/app/`: FastAPI REST API, SQLAlchemy data model, JWT authentication, ownership checks, scoring, analytics, document parsing, and AI provider adapter.
- PostgreSQL is used by Docker Compose; SQLite is the local API default.
- Alembic migrations live in `backend/migrations/`.
- Uploaded content is parsed in memory and only extracted text is persisted; the raw upload is not served back or stored.
- Each account starts with 18 curated, answer-checked starter questions across core subjects. These are stored in that user's private question bank and can be used in real exams without configuring an AI provider.

The attempt stores an immutable JSON snapshot of the questions and answer keys at exam creation. The browser never receives answer keys during an exam. The server persists answers, enforces the expiry time, and computes results on submission. Repeated submissions return the persisted result.

## Run locally

Requirements: Python 3.11+, Node.js 20+, and npm.

1. Start the API:

   ```powershell
   cd backend
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   pip install -r requirements.txt
   Copy-Item .env.example .env
   uvicorn app.main:app --reload
   ```

   The API is at `http://localhost:8000`; interactive OpenAPI docs are at `http://localhost:8000/api/v1/docs`.

2. Start the web app in a second terminal:

   ```powershell
   cd frontend
   npm install
   $env:NEXT_PUBLIC_API_URL = "http://localhost:8000"
   npm run dev
   ```

   Open `http://localhost:3000`.

SQLite tables are initialized on API start to simplify local development. For a database initialized through migrations, run `alembic upgrade head` from `backend/`. Set `DATABASE_URL` to a PostgreSQL SQLAlchemy URL to use PostgreSQL.

## Docker Compose

From the project root:

```powershell
$env:POSTGRES_PASSWORD = "set-a-unique-database-password"
$env:JWT_SECRET = "set-a-different-unique-secret-of-at-least-32-characters"
docker compose up --build
```

Both secrets are required; Compose refuses to start if either is unset. Use URL-safe alphanumeric database passwords because the Compose connection URL embeds the password. The API container applies Alembic migrations before starting. Visit `http://localhost:3000` and `http://localhost:8000/api/v1/docs`. Put production secrets in an appropriate secret manager and place the services behind HTTPS before deployment.

## Core API

All versioned endpoints use `/api/v1`. Send `Authorization: Bearer <access_token>` after registration/login.

- `POST /auth/register`, `POST /auth/login`, `GET /auth/me`
- `POST /questions`, `GET /questions`, `PATCH /questions/{id}`, `DELETE /questions/{id}`
- `POST /exams`, `GET /exams`, `POST /exams/{exam_id}/start`
- `GET /attempts/{attempt_id}`, `POST /attempts/{attempt_id}/answers`, `POST /attempts/{attempt_id}/submit`
- `GET /analytics/overview`, `GET /analytics/attempts/{attempt_id}`
- `POST /documents/upload`, `GET /documents`
- `POST /ai/generate-questions`

## AI provider setup

For OpenAI, copy `backend/.env.example` to `backend/.env`, add your API key to `AI_API_KEY`, and restart the API process:

```powershell
cd backend
Copy-Item .env.example .env
notepad .env
```

The template uses `https://api.openai.com/v1` and `gpt-4o-mini`. Keep the key in this backend-only `.env` file; never put it in frontend settings, source control, or chat. `.env` is ignored by git. The generation API validates the response schema, rejects exact duplicates already in the user's bank, and, for document-grounded output, requires a source quotation that occurs in extracted document text. Until all three settings are configured, generation returns an explicit HTTP 503; no placeholder/mock generation is used.

## Tests

```powershell
cd backend
pytest
```

Frontend type/build verification:

```powershell
cd frontend
npm install
npm run check
npm run build
```

## Current scope and limitations

- Included question formats: single-choice, multiple-select, true/false, short answer, and numerical schema/grading primitives. The initial UI authors single-choice questions; the exam runner currently renders choice options.
- Short-answer grading currently uses normalized exact matching; numerical grading uses a small tolerance. Rubric-based human review, partial credit, and AI grading are not implemented.
- Supported uploads are PDF, DOCX, and UTF-8 TXT, limited by `MAX_UPLOAD_BYTES`. Text-based PDFs are extracted. Scanned PDFs return `ocr_required`; an OCR service is not configured. Images, PPTX, audio, and video are not supported.
- AI output receives schema, exact-duplicate, and source-quote checks; this is not a guarantee of factual correctness. No independent answer verifier or human-review workflow is included.
- Adaptive tests, spaced repetition, study plans, teacher/admin operations, email verification, password reset, refresh-token rotation, OAuth, billing, report exports, notifications, Redis, background jobs, semantic search, audit logs, and comparative percentile/rank are not implemented.
- The UI stores a short-lived access token in session storage. Use HTTPS, a managed secret store, production-grade rate limiting, monitoring, backups, upload malware scanning, email lifecycle flows, and security review before a public deployment.
- The Docker Compose secrets are local-development defaults and must be overridden. The API's development startup creates metadata tables; production startup runs Alembic migrations.
