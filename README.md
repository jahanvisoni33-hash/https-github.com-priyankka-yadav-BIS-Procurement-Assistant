# BIS Assist — React + Vite

This existing frontend uses React and Vite with Oxlint. Backend setup, supported workflows and current service limits are documented below.

Currently, two official plugins are available:

- [@vitejs/plugin-react](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react) uses [Oxc](https://oxc.rs)
- [@vitejs/plugin-react-swc](https://github.com/vitejs/vite-plugin-react/blob/main/packages/plugin-react-swc) uses [SWC](https://swc.rs/)

## React Compiler

The React Compiler is not enabled on this template because of its impact on dev & build performances. To add it, see [this documentation](https://react.dev/learn/react-compiler/installation).

## Expanding the Oxlint configuration

If you are developing a production application, we recommend using TypeScript with type-aware lint rules enabled. Check out the [TS template](https://github.com/vitejs/vite/tree/main/packages/create-vite/template-react-ts) for information on how to integrate TypeScript and Oxlint's TypeScript related rules in your project.

## BIS Assist backend and run instructions

This project keeps the existing React/Vite frontend and adds a FastAPI backend. Open two terminals in the project folder.

Terminal 1 (Windows):

```powershell
py -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8000
```

Terminal 2:

```powershell
npm install
npm run dev -- --host 127.0.0.1 --port 5173
```

Open `http://localhost:5173`, create an account, then submit a tender. Vite forwards `/api` to the backend. SQLite is created at `backend/data/bis_assist.sqlite3` and excluded from Git.

Run checks with `npm run build`, `npm run lint`, and `python -m pytest backend/tests -q`.

### Implemented

- PBKDF2 password hashing, bearer sessions and account-scoped analysis history.
- Manual requirement submission and text-based PDF extraction with a 20 MB size limit.
- Deterministic field extraction with supporting snippets, BM25 keyword ranking, and optional local SBERT.
- Up to three evidence-supported recommendations from records loaded in the local catalogue.
- Generated PDF reports with source/provenance, evidence, ranking indicators and a disclaimer.
- Live keyword search against the public BIS Know Your Standards service. Search results are cached locally as metadata and ranked against the submitted tender text.

### BIS data and verification limits

The live connector uses BIS's public `searchKnowStandards` service for keyword search and caches the returned standard number, title, publication/validity dates and withdrawal indicator. It does not download standard PDFs or clause text. The recommendation score compares tender text with those returned metadata fields, so it is a discovery aid rather than a full clause-by-clause compliance check. Reconfirm the current record on the official [Know Your Standards search](https://www.services.bis.gov.in/php/BIS_2.0/bisconnect/knowyourstandards/indian_standards/isdetails). BIS publishes an official portal guide for navigating published-standard listings: [Standards Portal guide](https://standards.bis.gov.in/website/how-to-use).

The latest local cache can be exported as a readable JSON snapshot with `python backend/export_standards.py`; it writes `backend/data/bis_standards_cache.json`. This is the set of genuine records fetched by searches run in this app, not a complete BIS catalogue export.

Set `BIS_LIVE_SEARCH_ENABLED=false` to disable calls to BIS. The live connector searches up to five high-signal terms per analysis. BIS's full catalogue Excel export is permission-controlled; the app does not bypass that permission or copy copyrighted specification documents. To import a separately authorized full catalogue, use the reviewed-record importer below.

Only import records from a reviewed, permitted BIS source. Follow `backend/data/standards_catalogue.example.json` and include the source and retrieval date in each `provenance` field. Import with:

```powershell
python backend/import_standards.py path\to\reviewed-bis-catalogue.json
```

The importer accepts HTTPS BIS hosts and `cached` or `unverified` states; it cannot mark records live-verified. A match score does not determine legal applicability or a certification requirement.

### Production deployment (Vercel + Render)

The frontend is a static React/Vite site; `vercel.json` supplies the SPA route fallback. The FastAPI backend is configured in `render.yaml` with a persistent disk mounted at `/var/data`, SQLite at `/var/data/bis_assist.sqlite3`, one service instance, and `/api/health` as its health check. Keep one backend instance while using SQLite. The frontend and backend use separate origins, so configure CORS and the public API base as described below. No secrets are needed in the frontend.

The existing local database is intentionally excluded from Git. It contains application records, including account and session data, so do not commit or publish it. Preserve it by transferring `backend/data/bis_assist.sqlite3` securely to the Render disk before directing users to the hosted API. Back up the file first. On the first deploy, allow the service to initialize, stop/restart it after placing the existing file at `/var/data/bis_assist.sqlite3`, and verify `/api/health` before use. Use your deployment provider's authenticated file-transfer method; do not use a public download link. This keeps the existing SQLite schema and rows intact. The health endpoint reports connectivity and standard count, not a full data comparison.

1. Push this project to a Git repository. In Vercel, import it and set the project root to this directory (`BIS Project` if the repository contains the outer folder). Use `npm install` as the install command and `npm run build` as the build command; the output directory is `dist`.
2. Deploy the Render Blueprint from `render.yaml`. Set the prompted `BIS_CORS_ORIGINS` value to the exact HTTPS origin of the Vercel site, with no trailing slash (for example, `https://your-site.vercel.app`). Keep the service at one instance because SQLite is stored on its persistent disk.
3. Securely transfer the backed-up local `backend/data/bis_assist.sqlite3` to the mounted Render disk as `/var/data/bis_assist.sqlite3`. Preserve the file as-is; do not run a data import or change its schema. Restart the service after transfer, then check `https://<your-api-host>/api/health` and confirm `standards_loaded` is `1763` for this included database snapshot.
4. In Vercel project settings, add `VITE_API_BASE` with value `https://<your-api-host>/api` (no trailing slash), then redeploy. `VITE_API_BASE` is a public build-time URL, not a secret. Keep all database credentials and other secrets in backend hosting settings only.
5. Visit the Vercel URL and exercise sign-in, analysis history, manual submission, PDF upload, and report download. Configure a custom frontend domain by updating `BIS_CORS_ORIGINS` to that exact origin and redeploying the backend.

The committed Render configuration uses the existing SQLite database engine and does not provision a replacement database. Render's persistent disk is a paid resource. If switching to managed PostgreSQL later, migrate and verify a copy of the existing data first; do not point the current app at an empty database if the existing rows must remain available. Keep independent backups of the persistent database file.

### PDF, OCR, SBERT and deployment

Text-based PDFs use PyMuPDF and are processed in memory; extracted text and analysis metadata are stored in the database. Scanned PDFs need OCR, which is not configured; enter their requirements manually. BM25 works locally. SBERT is disabled unless `BIS_SBERT_MODEL_PATH` points to a pre-downloaded model; configure local `sentence-transformers` and `scikit-learn` packages to enable it. No model download or paid AI service is used automatically.

For this deployment, set `BIS_CORS_ORIGINS` on the backend and `VITE_API_BASE` on the frontend as described above. Keep the backend on HTTPS and maintain backups of its persistent disk. OCR, password reset, rate limiting, database migrations and direct live BIS verification remain unconfigured.
