# Expense Tracker

Smart Expense Tracker — a compact demo app with a Node/Express backend (SQLite DB) and a React + Vite frontend. The app supports uploading receipt images, extracts key fields (amount, merchant, date) using OCR.space, and stores expenses in a lightweight SQLite database.

## Quick summary
- Backend: `Backend/` — Express, routes under `/api`, file uploads saved to `/uploads`, OCR integration in `Backend/src/services/ocrService.js`.
- Frontend: `Frontend/` — React + Vite, uses an axios wrapper in `Frontend/src/services/api.js` to call the backend.

This README explains how the project is organized, how to run it locally, key implementation notes, and helpful troubleshooting tips.

## Table of contents
- What you'll find where
- Prerequisites
- Run locally (backend and frontend)
- Environment variables
- API endpoints (examples)
- File structure & key files
- Development notes & gotchas
- Troubleshooting
- Suggested improvements

## What you'll find where

- `Backend/` — Express server entrypoint `server.js`, route definitions in `Backend/src/routes/`, and services in `Backend/src/services/`.
- `Frontend/` — Vite + React app; important files: `src/services/api.js`, `src/components/ReceiptScanner.js`.
- `database.sqlite` — created automatically at repo root when backend runs.
- `Backend/uploads/` — saved receipt images (served statically at `/uploads`).

## Prerequisites

- Node.js (LTS recommended) and npm. On Windows PowerShell you may need to bypass the `npm.ps1` policy (see Troubleshooting section).
- Optional: OCR.space API key (required for OCR functionality).

## Run locally — Backend

Open a terminal and run these commands from the repository root or the `Backend/` folder.

PowerShell (recommended immediate fix without changing policies):

```powershell
cd 'c:\Users\tanvi\Desktop\Expense Tracker\Backend'
npm.cmd install
npm.cmd run dev
```

Notes:
- `npm run dev` uses `nodemon server.js`. The server listens on `process.env.PORT || 5000`.
- Health check URL (after server starts): `http://localhost:5000/api/health`.
- On startup the backend will create `database.sqlite` at the repo root and initialize the `expenses` table.

Run in production mode:

```powershell
cd 'c:\Users\tanvi\Desktop\Expense Tracker\Backend'
npm.cmd install
npm.cmd start
```

## Run locally — Frontend

Open a new terminal (keep backend running in its own terminal) and run:

```powershell
cd 'c:\Users\tanvi\Desktop\Expense Tracker\Frontend'
npm.cmd install
npm.cmd run dev
```

Notes:
- Vite dev server defaults to port 3000.
- `Frontend/vite.config.js` currently proxies `/api` to the deployed backend: `https://expensetracker-backend-cuvv.onrender.com/`.
- For local end-to-end testing either update the proxy in `vite.config.js` or edit `Frontend/src/services/api.js` to point to your local backend. Example quick change in `api.js`:

```js
// change this line
const API_BASE_URL = "https://expensetracker-backend-cuvv.onrender.com/api";
// to this for local testing
const API_BASE_URL = "http://localhost:5000/api";
```

Or use a PowerShell one-liner to replace the URL in-place:

```powershell
(Get-Content .\src\services\api.js) -replace 'https://expensetracker-backend-cuvv.onrender.com/api','http://localhost:5000/api' | Set-Content .\src\services\api.js
```

## Environment variables

Backend expects (in `Backend/.env` or environment):

- `OCR_SPACE_API_KEY` — required if you want OCR requests to succeed. If missing, the server will run but OCR requests will fail.
- `PORT` — optional (defaults to 5000).

Create a `.env` in `Backend/` with:

```
OCR_SPACE_API_KEY=your_api_key_here
PORT=5000
```

No special frontend environment variables are required by default.

## API endpoints (selected)

- GET /api/health — health check.
- POST /api/ocr/scan-receipt — multipart form (`receipt` field) to upload a receipt image. Returns parsed fields: `amount`, `merchant`, `date`, `rawText`, `category`, and `imageUrl` (path under `/uploads`). See `Backend/src/routes/ocr.js` and `Backend/src/services/ocrService.js`.
- GET /api/expenses — list expenses.
- POST /api/expenses — add an expense. Required fields: `amount`, `merchant`, `date`. See validation in `Backend/src/routes/expenses.js`.
- DELETE /api/expenses/:id — delete expense.
- GET /api/expenses/analytics — spending analytics (total, byCategory, monthly, recent).

Examples (curl):

```bash
# Health
curl http://localhost:5000/api/health

# OCR test
curl http://localhost:5000/api/ocr/test

# Upload receipt (multipart)
curl -F "receipt=@/full/path/to/receipt.jpg" http://localhost:5000/api/ocr/scan-receipt

# List expenses
curl http://localhost:5000/api/expenses
```

## File structure & key files

- Backend/
  - `server.js` — app entry, mounts routes, serves `/uploads` statically.
  - `src/routes/ocr.js` — multer config, POST `/scan-receipt`, `/test` endpoint.
  - `src/routes/expenses.js` — CRUD endpoints and analytics.
  - `src/services/ocrService.js` — OCR.space integration, response parsing and amount extraction heuristics.
  - `src/services/expenseService.js` — SQLite access with callback-to-Promise wrappers; DB file located at `../../../database.sqlite` relative to this file.

- Frontend/
  - `src/services/api.js` — axios instance and interceptors; `API_BASE_URL` lives here.
  - `src/components/ReceiptScanner.js` — upload flow and how the frontend expects OCR response fields.

## Development notes & gotchas

- Logging: code uses emoji-prefixed logs (e.g., `🔍`, `✅`) — these are helpful anchors when grepping logs.
- SQLite: `expenseService` uses the `sqlite3` package and initializes the DB on server start. The DB file is `database.sqlite` at the repo root.
- Multer upload config: enforced file size limit `5 * 1024 * 1024` (5MB) and allowed image mimetypes only. Keep this intact when changing upload behavior.
- OCR heuristics: `ocrService.extractTotalAmount` uses a multi-step strategy — search for lines containing `total`/`amount`, check last lines, then fallback to the largest amount found. Changing this logic can affect many flows (frontend auto-fill, analytics).
- Frontend `api.js` sets a default `API_BASE_URL` to a deployed backend. For local testing change it to `http://localhost:5000/api` or update the Vite proxy.
- Validation: `POST /api/expenses` enforces `amount`, `merchant`, and `date` (amount must be positive). See `Backend/src/routes/expenses.js`.

## Troubleshooting

- PowerShell `npm.ps1` execution policy error: On Windows PowerShell you may see `npm.ps1 cannot be loaded because running scripts is disabled on this system.` Use `npm.cmd` instead (examples above) or run one of these options:
  - One-off (temporary):
    ```powershell
    powershell -ExecutionPolicy Bypass -NoProfile -Command "npm.cmd install"
    ```
  - Persistent (current user):
    ```powershell
    Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned -Force
    ```
  - Or run commands inside `cmd.exe` instead of PowerShell.

- OCR fails: ensure `OCR_SPACE_API_KEY` is set in `Backend/.env` or environment. The OCR service logs raw OCR text and extraction steps — check console logs from the backend for detailed debugging.

- Uploads missing: uploaded files are saved to `Backend/uploads/`. If images are not found, confirm multer accepted the file and check server logs for file path and any deletion on error.

- Database issues: `database.sqlite` is created if missing. If migrations or schema changes are required, edit `initDatabase()` in `Backend/src/services/expenseService.js`.

## Security & maintenance notes

- Dependencies: `npm install` may show deprecation or vulnerability warnings. Audit and upgrade dependencies as needed (e.g., `multer` has had historical CVE notices). Use `npm audit` and `npm audit fix` when appropriate.
- Rate limits & API keys: OCR.space may have rate limits or quota — do not commit your API key to source control.

## Suggested improvements (low-risk suggestions)

- Add unit tests for `ocrService` parsing functions (amount extraction and merchant parsing).
- Add E2E integration test that runs the backend and a headless browser to validate the upload → OCR → create expense flow.
- Make the front-end API base URL configurable via environment variables to avoid editing source files for local testing.

## Deployed Link
https://expense-tracker-frontend-kudf.vercel.app/

## Contributing

1. Fork the repo and create a branch for your feature/fix.
2. Run backend and frontend locally and verify behavior.
3. Keep API changes backwards-compatible or document breaking changes in this README.

---

If you'd like, I can also:
- add a small smoke-test script that uploads a sample image to the OCR endpoint and prints results,
- add a short CONTRIBUTING.md or create a `.env.example` file,
- or update `Frontend/src/services/api.js` to prefer `process.env` for `API_BASE_URL`.

Tell me which follow-up you'd like and I will make it.
