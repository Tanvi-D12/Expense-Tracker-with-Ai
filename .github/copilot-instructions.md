## Copilot / AI agent instructions for Expense Tracker

Short, focused guidance so an AI assistant can be immediately productive in this repo.

1) Big-picture architecture
   - Monorepo-like layout with two top-level apps: `Backend/` (Express + SQLite) and `Frontend/` (React + Vite).
   - Backend exposes REST endpoints under `/api` and serves uploaded receipt images from `/uploads`.
   - OCR is performed by a dedicated service calling OCR.space (see `Backend/src/services/ocrService.js`).

2) Key files to read first (quick tour)
   - `Backend/server.js` — app entry, middleware, and route mounting (`/api/expenses`, `/api/ocr`).
   - `Backend/src/routes/ocr.js` — multer upload config (field name `receipt`, 5MB limit), upload lifecycle and error cleanup.
   - `Backend/src/services/ocrService.js` — OCR api integration, parsing strategy and amount-extraction heuristics.
   - `Backend/src/routes/expenses.js` and `Backend/src/services/expenseService.js` — validation rules, DB access, analytics queries.
   - `Frontend/src/services/api.js` — axios instance and global base URL (currently points to a deployed backend); also contains error-handling interceptors.
   - `Frontend/src/components/ReceiptScanner.js` — upload flow and how frontend expects OCR responses (fields: amount, merchant, date, rawText).

3) Environment & runtime
   - Backend uses `dotenv`. Required envs discovered in code:
     - `OCR_SPACE_API_KEY` — used by `ocrService`.
     - `PORT` (optional) — default 5000.
   - Database: SQLite file created automatically at repo root `database.sqlite` (resolved from `Backend/src/services/expenseService.js` using `../../../database.sqlite`).
   - Uploaded images are saved to `Backend/uploads/` and served statically at `/uploads`.

4) How to run locally (developer workflow)
   - Backend (from `Backend/`):
     - Install: `npm install`
     - Dev: `npm run dev` (uses `nodemon server.js`)
     - Prod: `npm start` (runs `node server.js`)
     - Health check: `GET http://localhost:5000/api/health` (server prints health URL on start).
   - Frontend (from `Frontend/`):
     - Install: `npm install`
     - Dev: `npm run dev` (Vite server on port 3000)
     - Build: `npm run build`
     - Note: `Frontend/vite.config.js` currently proxies `/api` to a deployed backend URL. For local end-to-end testing update `Frontend/src/services/api.js`'s `API_BASE_URL` to `http://localhost:5000/api` or change the proxy target.

5) API surface & expectations (concrete)
   - POST /api/ocr/scan-receipt
     - multipart form field: `receipt` (see `ocr.js` and `ReceiptScanner.js`)
     - Response `data` shape: { amount, date, merchant, rawText, category, ... }
     - Note: OCR route enforces keeping only the TOTAL amount (server-side code comments).
   - GET /api/expenses
   - POST /api/expenses  — expects { amount, merchant, date } (amount must be positive, see validation in `expenses.js`)
   - DELETE /api/expenses/:id
   - GET /api/expenses/analytics

6) Project-specific patterns & gotchas
   - Lots of debug logs use emoji prefixes (e.g., `🔍`, `✅`). These are reliable anchors for grepping logs during debugging.
   - SQLite usage: callbacks wrapped in Promises inside `expenseService`. Prefer to preserve existing callback-to-Promise pattern when editing.
   - OCR extraction uses multiple heuristics (look for "total" line, inspect last lines, fallback to largest amount). Changes to this logic affect many flows — keep tests or manual receipts when iterating.
   - Frontend's `api.js` hardcodes a deployed backend URL; this is intentional for the demo but needs to be changed for local testing.
   - File uploads: multer config accepts only certain image mimetypes and enforces 5MB limits. Keep the `fileFilter` & `limits` intact unless deliberate.

7) Quick debugging checklist (what to check first)
   - Is backend running? Check console for `Server running on port` and then `http://localhost:5000/api/health`.
   - Is `database.sqlite` present at repo root? If not, the server will create it on startup (look for `✅ Database initialized successfully`).
   - Are uploads appearing in `Backend/uploads/` and served at `http://localhost:5000/uploads/<filename>`?
   - If OCR fails, inspect `Backend/src/services/ocrService.js` logs — it prints RAW OCR text and the extraction steps.
   - Frontend network errors: `Frontend/src/services/api.js` shows user-facing alerts for connection issues; check `API_BASE_URL`.

8) Minimal examples (what changes look like)
   - To change upload limit: edit `Backend/src/routes/ocr.js` `limits: { fileSize: 5 * 1024 * 1024 }`.
   - To force local frontend to talk to local backend: set `API_BASE_URL = 'http://localhost:5000/api'` in `Frontend/src/services/api.js`.
   - To add an OCR test endpoint call: `GET /api/ocr/test` (exists in `ocr.js`).

If anything is missing or you'd like me to expand specific sections (e.g., add example curl requests, or a small smoke test script), tell me which parts to expand. I'll update this file accordingly.
