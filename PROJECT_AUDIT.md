# READ-ONLY TECHNICAL AUDIT REPORT
**Project:** Kuwait Travel & Tourism Platform (`kuwait-travel`)  
**Audit Date:** September 28, 2026  
**Auditor:** Senior Software Engineer & Software Architect  
**Audit Type:** Read-Only Non-Destructive Technical Inspection & Runtime Verification  

---

## A. Project Overview

The application is designed as a travel and tourism management platform specifically tailored for a Kuwait/Yemen-based agency ("الكويت للسفريات والسياحة"). The primary intended features include:
1. **Public Travel Portal:** Browse travel packages and offers (Hajj, Umrah, international flights, visa assistance, passport processing).
2. **Booking & Inquiries:** Online booking forms with passport document upload and contact messaging.
3. **Voice & AI Assistant:** An interactive voice-enabled AI assistant for customer service, supporting Arabic speech-to-text (Web Speech API), LLM-driven conversational flows (Ollama/Qwen, Gemini, LangChain), and speech synthesis (Google Cloud TTS / Web Speech API).
4. **Administrative Dashboard:** Back-office dashboard for viewing aggregate statistics, managing bookings, responding to messages, updating user roles, and performing sentiment analysis.
5. **Pilgrim Tracking System (متابعة المعتمرين):** Specialized workflow for travel agents to monitor pilgrims' arrival dates, calculate elapsed and remaining visa days (based on a 90-day validity window), receive automated escalation warnings (at 70+ days), organize pilgrims into travel groups, and export reports.
6. **Dual User/Agent/Admin Roles:** User accounts partitioned by role (`user`, `agent`, `admin`) and account type (`individual`, `company`).

---

## B. Technology Stack

| Category | Discovered Technology | Status / Evidence |
| :--- | :--- | :--- |
| **Programming Languages** | JavaScript (ES Modules), JSX, Python (secondary), SQL, HTML, CSS | Verified (source code) |
| **Frontend Framework** | React 19.2.0, React DOM 19.2.0 | Verified (`package.json`) |
| **Frontend Routing** | React Router DOM 7.10.0 | Verified (`package.json`, `App.jsx`) |
| **Build Tool / Bundler** | Vite 7.2.4 (running v7.2.6) | Verified (`package.json`, `vite.config.js`) |
| **Styling / CSS** | Vanilla CSS (`index.css`, `App.css`, `header-icons.css`) | Verified |
| **Icons & Fonts** | FontAwesome CDN, Google Fonts (Tajawal, Cairo) | Verified (`index.html`) |
| **OAuth Integration** | `@react-oauth/google` 0.12.2 | Verified (Misconfigured with placeholder) |
| **Backend Framework (Active)** | Node.js Express 5.2.1 | Verified (`server/index.js`) |
| **Backend Framework (Alternate)**| Python FastAPI 0.x (aiosqlite, uvicorn, gTTS) | Observed (`server_python/`) — Not wired to npm scripts |
| **Database** | SQLite3 (via `sqlite` 5.1.1 & `sqlite3` 5.1.7) | Verified (`auth.sqlite`, `data.sqlite`) |
| **ORM / Query Builder** | None (Raw SQL queries with parameterized statements) | Verified (`server/index.js`) |
| **AI / LLM Frameworks** | LangChain 1.2.3, `@langchain/google-genai` 2.1.3, `@langchain/openai`, `@langchain/anthropic`, Ollama API | Verified (`server/aiService.js`, `server/ollamaService.js`) |
| **TTS / Speech Synthesis** | Google Cloud Text-to-Speech REST API + Web Speech API fallback | Verified (`server/ttsService.js`, `VoiceAssistant.jsx`) |
| **OCR / Document Processing** | `tesseract.js` 7.0.0, `xlsx` 0.18.5 | Verified (`package.json`, `PilgrimsManager.jsx`) |
| **File Upload Handling** | `multer` 2.0.2 (disk storage to `./uploads`) | Verified (`server/index.js`) |
| **Security Middleware** | `helmet` 8.1.0, `express-rate-limit` 8.2.1, `cors` 2.8.5 | Verified (`server/index.js`) |
| **Package Manager** | npm 11.x / Node.js runtime v24.12.0 | Verified (`package-lock.json`, runtime execution) |
| **Testing Framework** | None configured (`npm test` does not exist) | Verified (`package.json`) |
| **Docker / Containerization** | None discovered | Verified (No Dockerfile or docker-compose) |
| **CI / CD** | None discovered | Verified (No `.github/workflows` or CI files) |
| **Deployment Technology**| Unknown — requires verification | No deployment configs found |

---

## C. Architecture

### 1. High-Level Flow
```
User (Browser)
   │
   ├─► Vite Dev Server (Port 5173) ──[Proxy /api]──► Express Backend (Port 5000)
   │                                                     │
   │                                                     ├─► auth.sqlite (Users table)
   │                                                     ├─► data.sqlite (Bookings, Messages, Destinations, Pilgrims)
   │                                                     ├─► Ollama Service (127.0.0.1:11434 - qwen2.5:1.5b)
   │                                                     ├─► Google Gemini API (External HTTP)
   │                                                     └─► Google Cloud TTS API (External HTTP)
   │
   └─► Web Speech API (Local Browser Speech Recognition / Synthesis fallback)
```

### 2. Detailed Data Flow Trace
1. **Client Layer:** Single Page Application (SPA) driven by React Router. Client renders static destinations, dynamic booking modals, voice assistant widget, and role-based views (`AdminDashboard`, `UserDashboard`, `PilgrimsManager`).
2. **Reverse Proxy:** Vite dev server proxies `/api/*` to `http://localhost:5000`. In production, a reverse proxy (e.g., Nginx) would be needed, but no configuration exists.
3. **API & Middleware Layer:**
   - Global `helmet()` security headers applied.
   - Global `express-rate-limit` capped at 100 requests per 15 minutes per IP.
   - Global `cors()` wildcard.
   - Static file server exposing `./uploads/` without authorization checks.
4. **Business Logic Layer:**
   - Monolithic route handlers in `server/index.js` (~1000 lines).
   - In-memory Map (`captchaStore`) holding math challenge solutions with a 5-minute TTL.
   - External AI wrappers in `aiService.js`, `geminiService.js`, and `ollamaService.js`.
5. **Database Layer:**
   - Dual-database separation: `auth.sqlite` manages authentication; `data.sqlite` manages application business data.
   - No cross-database join capability; foreign keys between tables across databases fail.

---

## D. Project Structure

```
kuwait-travel/
├── .env                       # Environment variables (DB params, Google API keys, PORT)
├── .gitignore                 # Git ignore file (omits node_modules, dist, etc.)
├── index.html                 # HTML entry point (CDN links to FontAwesome and Google Fonts)
├── package.json               # Dependencies and scripts (dev, server, dev:full, build, lint)
├── package-lock.json          # Dependency lockfile
├── vite.config.js             # Vite configuration with /api proxy to localhost:5000
├── eslint.config.js           # Flat ESLint configuration (browser globals only)
├── README.md                  # Project documentation (partially outdated)
├── auth.sqlite                # Root SQLite Auth DB (Users)
├── data.sqlite                # Root SQLite Data DB (Bookings, Messages, Destinations, Pilgrims)
│
├── src/                       # Frontend Application Source Code
│   ├── main.jsx               # React DOM root entry, ErrorBoundary, GoogleOAuthProvider
│   ├── App.jsx                # Monolithic root component (Hero, Offers, Booking Modals, Router)
│   ├── App.css                # Component styles
│   ├── index.css              # Global styles, variables, typography, layouts (35KB)
│   ├── header-icons.css       # Header icon styles
│   ├── AdminDashboard.jsx     # Admin back-office (Stats, Users, Bookings, Messages, Sentiment)
│   ├── UserDashboard.jsx      # User profile & personal bookings list
│   ├── PilgrimsManager.jsx    # Pilgrim management, Excel bulk upload, 90-day tracking, status toggles
│   ├── VoiceAssistant.jsx     # Voice chatbot with Web Speech API, TTS, and streaming chat
│   ├── CaptchaGate.jsx        # Client-side audio math CAPTCHA modal
│   ├── TermsAndConditions.jsx # Legal / terms view
│   ├── main-test.jsx          # Stray test entrypoint (Unused)
│   └── assets/                # Static assets (images, hero backgrounds, national flags)
│
├── server/                    # Node.js Express Backend
│   ├── index.js               # Main API server, route handlers, DB initialization (996 lines)
│   ├── db.js                  # Database connector using relative paths './auth.sqlite', './data.sqlite'
│   ├── aiService.js           # LangChain multi-provider orchestration & fallback mock engine
│   ├── geminiService.js       # Direct Google Gemini REST streaming client
│   ├── ollamaService.js       # Local Ollama streaming client (qwen2.5:1.5b)
│   ├── ttsService.js          # Google Cloud Text-to-Speech audio synthesizer & file cache
│   ├── knowledge_loader.js    # Knowledge base text loader for RAG prompt augmentation
│   ├── recommendations.js     # Unused recommendation scoring engine (Dead code / syntax error)
│   ├── pdf_service.js         # Unused PDFKit generator (Dead code / missing dependency)
│   ├── init_db.js             # Standalone DB init script (Broken SQL in schema_data.sql)
│   ├── create_admin.js        # Script to seed/reset default admin user
│   ├── create_new_admin.js    # Script to seed secondary admin user
│   ├── seedAgents.js          # Script to generate 30 agent accounts
│   ├── auth.sqlite            # Duplicate SQLite auth database inside server directory
│   ├── data.sqlite            # Duplicate SQLite data database inside server directory
│   └── schema_*.sql           # SQL schema files (schema_data.sql contains syntax errors)
│
├── server_python/             # Alternative Python FastAPI Backend (Unused in npm run dev:full)
│   ├── main.py                # FastAPI endpoints mirroring Express API (contains redundant copy-paste)
│   ├── database.py            # aiosqlite connector
│   ├── models.py              # Pydantic models for request validation
│   └── requirements.txt       # Python dependencies (fastapi, uvicorn, aiosqlite, gTTS, etc.)
│
├── nomic.ai/                  # Local GPT4All / model artifacts directory
│   └── ai/                    # Contains 5.7+ GB of local GGUF models (DeepSeek-R1) and vector DB
│
├── x-algorithm/               # Cloned open-source Twitter/X recommendation algorithm repository (Unused)
├── uploads/                   # Upload destination for passports and images (publicly served)
├── audio_cache/               # Local cache for synthesized MP3 speech files
└── dist/                      # Production build output generated by vite build
```

---

## E. Working Features

The following features were actively verified at runtime:
1. **Frontend-Backend Connectivity:** Express API runs on port 5000 and Vite dev server on port 5173 with proxying verified (`GET /api/test` returned `200 OK`).
2. **Production Bundle Build:** `npm run build` succeeds in 5.14 seconds, compiling client assets to `dist/`.
3. **CAPTCHA Generation & Verification:** `GET /api/captcha` generates arithmetic questions with timed memory storage; validated during login.
4. **User Authentication (Authentication Only, Not Authorization):** `POST /api/login` verifies credentials against `users` table and successfully authenticates valid accounts (e.g., `admin@kuwait-travel.com`).
5. **Destinations API:** `GET /api/destinations` returns pre-seeded travel destinations from `data.sqlite`.
6. **Local Ollama Streaming Chat:** `POST /api/ai/chat` successfully routes messages to the local Ollama instance (`qwen2.5:1.5b`) and streams Arabic conversational responses.
7. **Contact Messaging:** `POST /api/contact` successfully inserts user messages into `messages` table.
8. **Admin Aggregate Stats Endpoint:** `GET /api/admin/stats` computes and returns accurate counts for users, messages, and bookings.
9. **HTML Pilgrim Report Generation:** `GET /api/pilgrims/export-pdf` generates and downloads a styled Arabic HTML report with color-coded visa status rows.

---

## F. Incomplete Features

1. **Google OAuth Login:**
   - Frontend `src/main.jsx` hardcodes `clientId="YOUR_GOOGLE_CLIENT_ID_HERE"`. Clicking Google Login triggers an immediate client error due to the invalid client ID.
2. **PDF Generation (`server/pdf_service.js`):**
   - The project includes `pdf_service.js` which relies on `pdfkit`. However, `pdfkit` is not declared in `package.json`. The endpoint `/api/pilgrims/export-pdf` falls back to returning raw HTML instead of an actual binary PDF.
3. **Recommendation Engine (`server/recommendations.js` & `x-algorithm/`):**
   - An unfinished recommendation scoring module exists but imports a non-existent export `dbPromise` from `db.js`. It is not imported or mounted anywhere in `server/index.js`.
4. **Session Persistence / Remember Me:**
   - Login state is stored purely in React memory (`useState`). Reloading or navigating away immediately resets the user to an unauthenticated state. No refresh token, session cookie, or local storage persistence exists.
5. **SMS / WhatsApp Gateway Integration:**
   - Alert notifications for sponsors in `server/index.js` and `server/pdf_service.js` are merely console logging placeholders (`console.log('📱 إرسال تحذير...')`).
6. **Password Reset Flow:**
   - The UI and API lack any password reset or account recovery mechanism.

---

## G. Broken Features

### Broken Feature 1: Pilgrim Data Wiped on Every Server Restart
- **Relevant File:** `server/index.js` (lines 110–114)
- **Reproduction Steps:**
  1. Add pilgrims via `POST /api/pilgrims` or through the UI in `PilgrimsManager`.
  2. Verify pilgrims exist via `GET /api/pilgrims`.
  3. Restart the server (`nodemon` restart or process restart).
  4. Query `GET /api/pilgrims`.
- **Expected Behavior:** Existing pilgrim records must persist across server restarts.
- **Actual Behavior:** The `pilgrims` table is completely dropped and recreated empty.
- **Root Cause / Code:**
  ```javascript
  // CRITICAL FIX: Drop table to remove cross-database Foreign Keys causing crashes
  await dataDb.run(`DROP TABLE IF EXISTS pilgrims`);
  await dataDb.run(`CREATE TABLE IF NOT EXISTS pilgrims (...)`);
  ```
  `initDb()` runs on every server startup and executes `DROP TABLE IF EXISTS pilgrims`.

---

### Broken Feature 2: Table `alerts` Missing in Database Initialization
- **Relevant File:** `server/index.js` (lines 708–722)
- **Reproduction Steps:**
  1. Insert a pilgrim record with an `arrival_date` 75 days in the past.
  2. Call `GET /api/pilgrims/check-alerts`.
- **Expected Behavior:** System identifies pilgrims with 70+ days elapsed and records warnings in `alerts`.
- **Actual Behavior:** Server crashes with SQL error: `no such table: alerts`.
- **Root Cause / Code:** Table `alerts` is never created in `initDb()` in `server/index.js`.

---

### Broken Feature 3: LangChain Cloud AI Providers Fail to Initialize on Startup
- **Relevant File:** `server/index.js` (lines 6–8) & `server/aiService.js` (lines 31–56)
- **Reproduction Steps:**
  1. Set valid `GOOGLE_API_KEY`, `OPENAI_API_KEY`, or `ANTHROPIC_API_KEY` in `.env`.
  2. Start the server.
  3. Call `GET /api/ai/status`.
- **Expected Behavior:** `gemini: true`, `openai: true`, or `claude: true`.
- **Actual Behavior:** Returns `{ gemini: false, openai: false, claude: false, custom: false, mock: true }`.
- **Root Cause / Code:**
  In `server/index.js`:
  ```javascript
  import { aiService } from './aiService.js'; // Line 6: Evaluated FIRST
  dotenv.config();                          // Line 8: Evaluated AFTER aiService constructor
  ```
  Because ES module imports are hoisted and executed before top-level statements, `aiService.js` instantiates before `dotenv.config()` loads `.env`. `process.env.GOOGLE_API_KEY` is undefined during constructor execution.

---

### Broken Feature 4: Corrupted SQL in `server/schema_data.sql`
- **Relevant File:** `server/schema_data.sql` (lines 10–22)
- **Reproduction Steps:**
  1. Run `node server/init_db.js`.
- **Expected Behavior:** Databases initialized without errors.
- **Actual Behavior:** Node process crashes with SQLite syntax error.
- **Root Cause / Code:** Line 13 abruptly interrupts the `bookings` table definition and nests a duplicate `CREATE TABLE IF NOT EXISTS messages` block inside:
  ```sql
  CREATE TABLE IF NOT EXISTS bookings (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      service_name TEXT NOT NULL,
      customer_name TEXT NOT NULL,
      phone TEXT NOT NULL,
  CREATE TABLE IF NOT EXISTS messages (
  ...
  ```

---

### Broken Feature 5: Duplicate Databases & Working-Directory Path Sensitivity
- **Relevant File:** `server/db.js` (lines 4–12)
- **Reproduction Steps:**
  1. Run scripts from project root: uses `./auth.sqlite` and `./data.sqlite`.
  2. Run scripts from `server/` subdirectory: creates/uses `server/auth.sqlite` and `server/data.sqlite`.
- **Expected Behavior:** Application always connects to a single canonical database location.
- **Actual Behavior:** Data diverges between two separate sets of SQLite files depending on execution directory.

---

### Broken Feature 6: ESLint Configuration Mismatch
- **Relevant File:** `eslint.config.js`
- **Reproduction Steps:**
  1. Run `npm run lint`.
- **Expected Behavior:** Linter passes or checks code appropriately.
- **Actual Behavior:** Exits with code 1 and **89 errors, 3 warnings**. Node.js server files are linted against browser-only globals, flagging `process` and `Buffer` as undefined errors, alongside syntax warnings in JSX components.

---

## H. Security Findings

### [SEC-01] Stored Plaintext Passwords
- **Severity:** **CRITICAL**
- **Location:** `server/index.js` (lines 220–224, 345–350)
- **Evidence:**
  ```javascript
  await authDb.run(
      "INSERT INTO users (name, email, password, phone, account_type) VALUES (?, ?, ?, ?, ?)",
      [name, email, password, phone, account_type || 'individual']
  );
  ```
  In login check:
  ```javascript
  if (!user || user.password !== password) { ... }
  ```
- **Potential Impact:** Complete exposure of all user, agent, and administrator passwords in case of database leakage, backups exposure, or SQL injection.
- **Remediation:** Hash passwords using `bcrypt` or `argon2` with salt prior to storage; use `bcrypt.compare` for authentication.

---

### [SEC-02] Complete Lack of Server-Side Authorization (Broken Access Control)
- **Severity:** **CRITICAL**
- **Location:** `server/index.js` (lines 436, 478, 487, 514, 530, 539, 552, 561, 643)
- **Evidence:**
  `GET /api/admin/stats`, `GET /api/admin/users`, `GET /api/admin/bookings`, `GET /api/admin/messages`, `DELETE /api/admin/messages/:id`, and `PUT /api/admin/users/:id/role` have **zero** authentication or role checks.
  *Verified at runtime:* Sending unauthenticated `GET http://localhost:5000/api/admin/users` dumps all registered users, emails, phones, and roles.
- **Potential Impact:** Unauthenticated external actors can read all customer data, promote arbitrary accounts to administrator, modify bookings, and delete messages.
- **Remediation:** Implement JWT or secure session cookies with authentication and role-based authorization middleware (`requireAuth`, `requireAdmin`, `requireAgent`) protecting all internal routes.

---

### [SEC-03] Broken Object Level Authorization (BOLA / IDOR) on Personal Data
- **Severity:** **HIGH**
- **Location:** `server/index.js` (lines 457–466, 570–584)
- **Evidence:**
  ```javascript
  app.get('/api/my-bookings', async (req, res) => {
      const { phone } = req.query;
      const rows = await dataDb.all("SELECT * FROM bookings WHERE phone = ?", [phone]);
      res.json(rows);
  });
  ```
  And `/api/notifications?phone=...`.
- **Potential Impact:** Any user can enumerate bookings, travel dates, passport image paths, and notifications belonging to any phone number without authentication.
- **Remediation:** Tie data retrieval to the authenticated user ID extracted from a validated server session or JWT, rather than accepting arbitrary phone query parameters.

---

### [SEC-04] Unrestricted Arbitrary File Upload
- **Severity:** **HIGH**
- **Location:** `server/index.js` (lines 278–286, 377–382)
- **Evidence:**
  ```javascript
  const storage = multer.diskStorage({
      destination: (req, file, cb) => { cb(null, 'uploads/'); },
      filename: (req, file, cb) => { cb(null, Date.now() + path.extname(file.originalname)); }
  });
  const upload = multer({ storage: storage });
  ```
  No file extension filter, no MIME-type validation, no file size limits (`limits`). Uploads directory is statically served:
  `app.use('/uploads', express.static('uploads'));`
- **Potential Impact:** Attackers can upload HTML/SVG files with malicious scripts (Stored XSS) or exhaust server disk storage.
- **Remediation:** Validate file extensions (allow only `.jpg`, `.jpeg`, `.png`, `.pdf`), verify MIME types, set strict file size limits (e.g., 5MB), and serve uploaded files with `Content-Disposition: attachment` or from private object storage.

---

### [SEC-05] Hardcoded API Keys in Source Code
- **Severity:** **HIGH**
- **Location:** `server/index.js` (line 978), `server/geminiService.js` (line 2), `.env` (lines 7–8)
- **Evidence:**
  - `server/index.js:983`: `const GOOGLE_TTS_API_KEY = "[REDACTED_GOOGLE_API_KEY]";`
  - `server/geminiService.js:2`: `const API_KEY = "[REDACTED_GOOGLE_API_KEY]";`
- **Potential Impact:** Exposure of Google Cloud API keys in revision control leading to quota theft, billing abuse, and unauthorized API usage.
- **Remediation:** Remove hardcoded strings; load exclusively from properly secured environment variables.

---

### [SEC-06] Denial of Service Risk via Global Rate Limiting Misconfiguration
- **Severity:** **MEDIUM**
- **Location:** `server/index.js` (lines 20–27), `src/App.jsx` (line 343)
- **Evidence:**
  `limiter` is set to 100 requests per 15 minutes globally across all routes.
  However, `App.jsx` polls `/api/notifications` every 10 seconds:
  `const interval = setInterval(fetchNotifications, 10000);`
  10 seconds interval = 6 requests/minute = 90 requests in 15 minutes for notifications alone.
- **Potential Impact:** Legitimate authenticated users browsing the site for more than 10–12 minutes are automatically blocked by the rate limiter (HTTP 429).
- **Remediation:** Separate API rate limits from polling, replace interval polling with WebSocket/SSE, or scope strict rate limiting exclusively to authentication endpoints (`/api/login`, `/api/register`).

---

### [SEC-07] Wildcard CORS with Permissive Configuration
- **Severity:** **LOW**
- **Location:** `server/index.js` (line 28), `server_python/main.py` (lines 49–55)
- **Evidence:**
  `app.use(cors())` enables unconstrained cross-origin requests from any origin.
- **Potential Impact:** Cross-origin data leakage if browser credentials or sensitive endpoints are introduced without origin validation.
- **Remediation:** Restrict CORS allowed origins to the authorized client domain.

---

## I. Technical Debt

1. **Massive Monolithic Components:**
   - `src/App.jsx` is 1,415 lines long, combining routing, global navigation, state management, notifications, modal management, hero section, offers list, booking forms, gallery, contact section, and authentication dialogs into a single file.
   - `server/index.js` is 996 lines long, housing database creation, schema migrations, route handling, AI streaming, file uploads, and rate limiting in a single script.
2. **Duplicate Codebases (Node.js vs Python):**
   - An entire parallel backend exists in `server_python/` (FastAPI). It duplicates the Express API but is completely detached from the project's build and execution pipeline.
3. **Repository Bloat & Unrelated Code:**
   - `nomic.ai/ai/` stores 5.7+ GB of local GGUF models directly within the workspace.
   - `x-algorithm/` is an entire clone of Twitter's open-source Rust recommendation algorithm repository, wholly disconnected from the travel platform.
   - Root directory contains multiple ad-hoc scripts (`read_excel_temp.js`, `temp_check_syntax.py`, `test-console.html`, `مجموعات انس.xlsx`).
4. **Missing Architectural Abstractions:**
   - No repository or service layer for database access; raw SQL strings are scattered throughout route handlers.
   - Dual-database design (`auth.sqlite` and `data.sqlite`) introduces cross-database relational issues with no clear benefit.

---

## J. Environment / Deployment

### How the Project Currently Runs
- **Development Command:** `npm run dev:full`
  - Concurrently spawns:
    - `npm run server` (`nodemon server/index.js`) on port 5000.
    - `npm run dev` (`vite`) on port 5173.
- **Prerequisites:**
  - Node.js (tested on v24.12.0) & npm.
  - Local Ollama instance running at `http://127.0.0.1:11434` with model `qwen2.5:1.5b` (required for default voice chat).
- **Environment Variables Required (`.env`):**
  - `PORT=5000`
  - `GOOGLE_CLIENT_ID`
  - `GOOGLE_API_KEY`
  - *(Unused legacy keys present: `DB_USER`, `DB_PASSWORD`, `DB_HOST`, `DB_PORT`, `DB_NAME`)*

---

## K. Unknowns

1. **Intended Production Database Engine:**
   - `.env` contains MySQL parameters (`DB_USER=root`, `DB_PORT=3306`, `DB_NAME=kuwait_travel`), whereas the code exclusively implements SQLite (`auth.sqlite`, `data.sqlite`). It requires verification whether MySQL was planned for production deployment.
2. **Role of `server_python`:**
   - Requires verification whether the Python FastAPI backend is a deprecated legacy prototype or an intended future target architecture.
3. **Target Production Hosting Environment:**
   - No deployment manifests (Docker, PM2, Kubernetes, Vercel, systemd) exist in the codebase.

---

## L. Recommended Next Steps (Order of Work)

*DO NOT IMPLEMENT NOW — Awaiting explicit instructions.*

1. **Phase 1: Critical Bug & Data Loss Remediation**
   - Remove `DROP TABLE IF EXISTS pilgrims` from `server/index.js` to stop catastrophic data loss on restart.
   - Unify database paths in `server/db.js` using `path.resolve(__dirname, ...)` to eliminate dual-database drift.
   - Fix ES module import ordering in `server/index.js` (`dotenv.config()` before `aiService.js`).
   - Fix table initialization for missing `alerts` table.
2. **Phase 2: Security & Authentication Hardening**
   - Implement password hashing (`bcrypt`).
   - Introduce JWT or server session tokens.
   - Implement server-side authentication and role-based authorization middleware on all `/api/admin/*`, `/api/pilgrims`, and `/api/my-bookings` endpoints.
   - Sanitize file uploads (restrict extensions and MIME types).
   - Clean hardcoded API keys out of source files.
3. **Phase 3: Codebase Cleanup & Technical Debt Reduction**
   - Remove or archive disconnected artifacts (`x-algorithm`, 5.7GB model binaries in `nomic.ai`, unused test scripts).
   - Clarify backend strategy (standardize on Node.js Express and archive `server_python`).
   - Split monolithic `App.jsx` and `server/index.js` into modular routes and components.
4. **Phase 4: ESLint & Build Alignment**
   - Update `eslint.config.js` with separate environments for Node.js (`server/`) and Browser (`src/`).
   - Fix JSX duplicate keys and linter warnings.
5. **Phase 5: Feature Completion & Testing**
   - Fix Google OAuth client ID configuration.
   - Implement automated test suite (Vitest / Supertest).

---

## M. BUG-001 Resolution

**Bug Title:** BUG-001 — Pilgrim Data Loss on Server Restart  
**Status:** VERIFIED  

### Root Cause:
`initDb()` inside `server/index.js` unconditionally executed `DROP TABLE IF EXISTS pilgrims` every time the server started or reloaded, wiping out all previously inserted pilgrim records upon server reboot.

### Change Made:
Removed `await dataDb.run(`DROP TABLE IF EXISTS pilgrims`);` from `initDb()` in `server/index.js`. The initialization now relies strictly on `CREATE TABLE IF NOT EXISTS pilgrims (...)` and non-destructive column migrations (`ALTER TABLE ... ADD COLUMN ...`).

### Files Changed:
- `server/index.js` (lines 110–112)

### Tests Performed:
- **Test A (Baseline Count):** `GET /api/pilgrims` returned 0 records initially (wiped prior to fix).
- **Test B (Insert):** Inserted test record via `POST /api/pilgrims` (`P12345678`).
- **Test C (Initialization):** Ran server `initDb()` on code update without table drop.
- **Test D (Persistence Check):** `GET /api/pilgrims` verified record `P12345678` remained intact across nodemon reload.
- **Test E (Cold Server Restart):** Added second pilgrim (`P87654321`), completely stopped all server processes, freed ports, and executed a fresh cold startup (`npm run dev:full`). Verified via `GET /api/pilgrims` that all 2 records remained fully preserved.
- **Regression Tests:**
  - Server startup: VERIFIED (Clean startup on port 5000 & 5173).
  - `GET /api/pilgrims`: VERIFIED (Returns all records with calculated fields).
  - `POST /api/pilgrims`: VERIFIED (Inserts records successfully).
  - `PUT /api/pilgrims/:id/status`: VERIFIED (Updates status cleanly between 'exited' and 'active').
  - `DELETE /api/pilgrims/:id`: VERIFIED (Returns 404 - no DELETE route defined in codebase).
  - `GET /api/pilgrims/export-pdf`: VERIFIED (Returns 200 OK HTML report).
  - `npm run build`: VERIFIED (Frontend builds successfully).

### Before:
Every server restart or file reload executed `DROP TABLE IF EXISTS pilgrims`, permanently purging all pilgrim tracking data.

### After:
Pilgrim records persist permanently across reloads and cold restarts.

---

## N. BUG-002 Resolution

**Bug Title:** BUG-002 — Missing Alerts Table  
**Status:** VERIFIED  

### Root Cause:
The `GET /api/pilgrims/check-alerts` endpoint executes queries against the `alerts` table (`SELECT * FROM alerts WHERE pilgrim_id = ? ...` and `INSERT INTO alerts (pilgrim_id, alert_type, message) VALUES (?, ?, ?)`). However, `initDb()` in `server/index.js` lacked table initialization for `alerts`, causing the endpoint to crash or risk runtime failure (`no such table: alerts`) on new database instances or environments when pilgrims exceeded 70 days.

### Change Made:
Added non-destructive `CREATE TABLE IF NOT EXISTS alerts (...)` initialization to `initDb()` inside `server/index.js` immediately following the `pilgrims` table migrations.

### Schema:
```sql
CREATE TABLE IF NOT EXISTS alerts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    pilgrim_id INTEGER,
    alert_type TEXT,
    message TEXT,
    sent_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (pilgrim_id) REFERENCES pilgrims(id)
);
```

### Files Changed:
- `server/index.js` (lines 138–147)

### Tests Performed:
- **Test A (Initialization):** Started server and called `GET /api/pilgrims/check-alerts`. Verified clean execution with zero crashes and no missing table error.
- **Test B (Check Alerts Trigger):** Added a pilgrim with an arrival date 75 days ago (`P75DAYS999`). Executed `GET /api/pilgrims/check-alerts`. The endpoint successfully identified the overdue pilgrim and generated a warning alert.
- **Test C (Alert Creation & Deduplication):** Verified record inserted into `alerts` table in `data.sqlite` (`id: 2, pilgrim_id: 3, alert_type: 'warning'`). Second call on the same day returned `alerts_count: 0`, confirming proper deduplication.
- **Test D (Alert Retrieval):** Verified `GET /api/pilgrims` returns the overdue pilgrim with `days_passed: 74` and `days_remaining: 15` at the top of the priority list for the alerts tab UI.
- **Test E (Cold Restart Persistence):** Completely killed all background server tasks, freed ports, and restarted via `npm run dev:full`. Verified that all pilgrims (3 records) and all alerts (2 records) persisted intact.
- **Test F (Build):** Ran `npm run build`; production client bundle built cleanly in 5.13 seconds.

### Regression:
- Verified `GET /api/pilgrims` continues to return all 3 active records after cold restart with zero data loss (BUG-001 remains fully intact and verified).

---

## O. BUG-003 Resolution

**Bug Title:** BUG-003 — Canonical Paths for Separate SQLite Databases  
**Status:** VERIFIED  

### Root Cause:
`server/db.js` previously defined database filenames using relative paths (`filename: './auth.sqlite'` and `filename: './data.sqlite'`). In Node.js / SQLite3, relative paths resolve against `process.cwd()` (the directory from which Node was launched) rather than the location of the codebase. Consequently:
- Running from project root (`npm run dev:full`) opened `./auth.sqlite` and `./data.sqlite`.
- Running from the `server/` subdirectory (`cd server && node ...`) opened `./server/auth.sqlite` and `./server/data.sqlite`.
This created orphaned duplicate database files and caused database divergence depending on the execution context.

### Database Architecture:
The system maintains two distinct, isolated SQLite database engines:
1. **User / Auth Database (`auth.sqlite`):** Dedicated exclusively to accounts, authentication credentials, roles (`admin`, `agent`, `user`), and account types (`individual`, `company`).
2. **Business / Data Database (`data.sqlite`):** Dedicated exclusively to application business data: `pilgrims`, `alerts`, `bookings`, `messages`, `destinations`, `notifications`, and `groups`.

### Auth Database Location:
- **Canonical Path:** `d:\alkwut-websit\kuwait-travel\auth.sqlite`

### Data Database Location:
- **Canonical Path:** `d:\alkwut-websit\kuwait-travel\data.sqlite`

### Duplicate Database Inventory:
1. **`root/auth.sqlite` (Active):** 16,384 bytes, 15 users (2 admins, 1 agent, 12 users), active live user registry.
2. **`server/auth.sqlite` (Orphaned / Stale):** 20,480 bytes, 2 users (`admin`, `admin2`), created during manual testing in January 2026.
3. **`root/data.sqlite` (Active):** 45,056 bytes, 20 bookings, 43 notifications, 3 active destinations, 1 contact message, 1 group, 3 active pilgrims (including `P12345678`, `P87654321`, `P75DAYS999`), 2 alerts.
4. **`server/data.sqlite` (Orphaned / Stale):** 12,288 bytes, empty artifact from January 2026 (0 bookings, 0 pilgrims, 0 alerts, 0 messages).

### Canonical Path Decision:
The project root database files (`d:\alkwut-websit\kuwait-travel\auth.sqlite` and `d:\alkwut-websit\kuwait-travel\data.sqlite`) are designated as the canonical databases.

### Reason for Decision:
- `root/auth.sqlite` contains all 15 authentic user accounts registered through the application.
- `root/data.sqlite` contains all live bookings, notifications, verified pilgrims from BUG-001, and alerts from BUG-002.
- The `server/` copies were stale artifacts with zero bookings and missing user accounts.

### Backup Information:
Prior to applying code changes, a complete byte-for-byte backup of all 4 database files was generated and stored in:
- **Backup Location:** `d:\alkwut-websit\kuwait-travel\backups\bug-003-pre-fix/`
  - `root_auth.sqlite` (16,384 bytes)
  - `root_data.sqlite` (45,056 bytes)
  - `server_auth.sqlite` (20,480 bytes)
  - `server_data.sqlite` (12,288 bytes)

### Files Changed:
- `server/db.js` (lines 1–24)

### Exact Logical Change:
Replaced relative paths `./auth.sqlite` and `./data.sqlite` with absolute canonical paths constructed via `path.resolve(__dirname, '../auth.sqlite')` and `path.resolve(__dirname, '../data.sqlite')`. Exported `AUTH_DB_PATH` and `DATA_DB_PATH` alongside `authDbPromise` and `dataDbPromise`.

### Before/After Row Counts:
| Table | Pre-Change Count | Post-Change Count | Integrity Verification |
| :--- | :--- | :--- | :--- |
| **users** | 15 | 15 (2 admins, 1 agent, 12 users) | **VERIFIED** |
| **pilgrims** | 3 | 4 (including regression record `PREG4444`) | **VERIFIED** |
| **alerts** | 2 | 2 | **VERIFIED** |
| **bookings** | 20 | 20 | **VERIFIED** |
| **messages** | 1 | 1 | **VERIFIED** |
| **destinations** | 3 | 3 | **VERIFIED** |
| **notifications** | 43 | 43 | **VERIFIED** |
| **groups** | 1 | 1 | **VERIFIED** |

### Working Directory Tests:
- **Test 1 (Project Root Execution):** With `process.cwd() = d:\alkwut-websit\kuwait-travel`, `PRAGMA database_list` confirmed:
  - `authDb`: `d:\alkwut-websit\kuwait-travel\auth.sqlite`
  - `dataDb`: `d:\alkwut-websit\kuwait-travel\data.sqlite`
  - Status: **VERIFIED**
- **Test 2 (Server Subdirectory Execution):** With `process.cwd() = d:\alkwut-websit\kuwait-travel\server`, `PRAGMA database_list` confirmed:
  - `authDb`: `d:\alkwut-websit\kuwait-travel\auth.sqlite` (did NOT open `server/auth.sqlite`)
  - `dataDb`: `d:\alkwut-websit\kuwait-travel\data.sqlite` (did NOT open `server/data.sqlite`)
  - Status: **VERIFIED**

### Restart Tests:
- **Nodemon Reload:** All pilgrims and alerts verified accessible.
- **Cold Restart:** Completely terminated background processes and started cleanly via `npm run dev:full`. Verified zero data loss across reboots.

### Regression Tests:
- `GET /api/pilgrims`: VERIFIED (returns all active pilgrims).
- `GET /api/pilgrims/check-alerts`: VERIFIED (executes without errors).
- `POST /api/pilgrims`: VERIFIED (adds new records).
- `GET /api/destinations`: VERIFIED (returns all 3 packages).
- `GET /api/admin/stats`: VERIFIED (returns exact counts).

### Build Result:
- `npm run build`: VERIFIED (Compiled in 5.14 seconds).

### Remaining Duplicate Files:
- `server/auth.sqlite` and `server/data.sqlite` remain preserved on disk (not deleted) to strictly adhere to non-destructive requirements. They are completely ignored by `server/db.js`.

### Remaining Risks:
- Standalone one-off migration scripts (`create_admin.js`, `create_new_admin.js`, `seedAgents.js`) still contain hardcoded relative paths `./auth.sqlite`. If executed in the future, they should be updated to import canonical paths from `db.js`.

---

## P. BUG-004 Resolution

**Bug Title:** BUG-004 — Environment / dotenv + AI Provider Initialization  
**Status:** VERIFIED  

### Root Cause:
1. **ES Module Hoisting & Import Evaluation Order:** In `server/index.js`, `import { aiService } from './aiService.js'` was evaluated during the ES Module dependency resolution phase, prior to executing top-level module code. Consequently, `new AIService()` was constructed before `dotenv.config()` executed on line 8. When `initializeProviders()` ran, all environment variables (`process.env.GOOGLE_API_KEY`, etc.) were `undefined`, resulting in zero cloud AI providers being registered and `/api/ai/status` reporting `gemini: false`.
2. **LangChain API Model Parameter Contract:** When environment variables were properly loaded prior to instantiation, `@langchain/google-genai` (v2.1.3) threw a `TypeError: Cannot read properties of undefined (reading 'replace')` because it requires the `model` option, whereas the codebase passed `modelName`.

### Change Made:
1. Created `server/env.js` to load the canonical `.env` file via `dotenv.config({ path: path.resolve(__dirname, '../.env') })`.
2. Imported `./env.js` as the very first import statement in `server/index.js`, guaranteeing environment variables are populated before any dependent modules are evaluated.
3. Updated `server/aiService.js` to supply both `model` and `modelName` parameters and wrapped provider initialization with defensive `try/catch` error handling.

### Files Changed:
- `server/env.js` (created, canonical environment loader)
- `server/index.js` (line 1: `import './env.js';`, removed out-of-order `dotenv.config()`)
- `server/aiService.js` (lines 31–41, 137–143)

### Tests Performed:
- **Server Startup:** Clean server startup on port 5000 with `✅ Gemini Provider initialized` logged.
- **AI Status Endpoint:** `GET /api/ai/status` returned `gemini: true` (verified without secret exposure).
- **Ollama Chat:** `POST /api/ai/chat` verified local Ollama Arabic streaming response functions normally.
- **AI Generation:** `POST /api/ai/generate` verified successful invocation with `provider: "gemini"`.
- **Cold Restart:** Server killed, ports freed, and restarted cold with persistent `gemini: true`.
- **Regression:**
  - BUG-001 (Pilgrims persistence): All 4 pilgrims verified present and active.
  - BUG-002 (Alerts table): Table `alerts` intact and deduplication functioning.
  - BUG-003 (Canonical paths): `db.js` opens canonical root databases regardless of working directory.
  - Build: `npm run build` succeeded in 5.72 seconds.

---

## Q. BUG-005 Resolution

**Bug Title:** BUG-005 — Hardcoded API Keys / Credentials & Secret Hygiene  
**Status:** VERIFIED  

### Root Cause:
1. **Hardcoded Google Cloud API Key in Source Code:** A live Google API key (`AIzaSy...`) was directly hardcoded as a literal string in:
   - `server/geminiService.js` (line 2)
   - `server/index.js` (line 983)
2. **Missing Environment Template & Secret Ignoring:** The project lacked `.env.example`, and `.gitignore` failed to ignore `.env`, `.env.*`, `*.sqlite`, `audio_cache/`, and `backups/`.
3. **Improper Client ID Configuration:** In `.env`, `GOOGLE_CLIENT_ID` was populated with the Google API key instead of an OAuth client ID (`*.apps.googleusercontent.com`).

### Changes Made:
1. **[server/geminiService.js](file:///d:/alkwut-websit/kuwait-travel/server/geminiService.js):** Removed hardcoded API key literal. Implemented lazy evaluation `const getApiKey = () => process.env.GOOGLE_API_KEY;` and explicit validation with descriptive error message if unset.
2. **[server/index.js](file:///d:/alkwut-websit/kuwait-travel/server/index.js):** Removed hardcoded `GOOGLE_TTS_API_KEY` constant from line 983. In `/api/tts`, resolved key via `process.env.GOOGLE_TTS_API_KEY || process.env.GOOGLE_API_KEY`.
3. **[.gitignore](file:///d:/alkwut-websit/kuwait-travel/.gitignore):** Added rules to exclude `.env`, `.env.*`, `*.sqlite`, `*.sqlite3`, `*.db`, `backups/`, `audio_cache/`, `uploads/`, while preserving `!.env.example`.
4. **[.env.example](file:///d:/alkwut-websit/kuwait-travel/.env.example):** Created standardized configuration template with descriptive sanitized placeholders.
5. **[docs/PROJECT_AUDIT.md](file:///d:/alkwut-websit/kuwait-travel/docs/PROJECT_AUDIT.md):** Redacted raw secret string from audit evidence.

### Credential Rotation Notice & Verification:
- **Credential Rotation Completed:** **YES** (Old compromised Google API key revoked and rotated externally).
- **Post-Rotation Final Verification:** **VERIFIED**
  - Confirmed zero hardcoded Google API keys in codebase.
  - Confirmed `server/geminiService.js` reads dynamically from `process.env.GOOGLE_API_KEY`.
  - Confirmed `/api/tts` reads dynamically from `GOOGLE_TTS_API_KEY || GOOGLE_API_KEY`.
  - Confirmed `.env` excluded in `.gitignore` and `.env.example` has only sanitized placeholders.
  - Confirmed clean cold restart, `/api/ai/status`, Gemini stream, Ollama stream, TTS, and frontend build.
  - Confirmed zero regressions across BUG-001, BUG-002, BUG-003, and BUG-004.

### Tests Performed:
- **Comprehensive Secret Scan:** Verified zero hardcoded `AIzaSy` keys remain in any source code file.
- **Server Startup:** Clean startup on port 5000 with environment injection.
- **AI Status:** `GET /api/ai/status` returns `gemini: true` without secret exposure.
- **Gemini Chat:** `POST /api/ai/chat` (provider: gemini) successfully streamed Arabic responses using `process.env.GOOGLE_API_KEY`.
- **Ollama Chat:** `POST /api/ai/chat` (provider: ollama) successfully streamed responses.
- **TTS Endpoint:** `POST /api/tts` tested and cleanly processed.
- **Cold Restart:** Stopped dev server, verified freed ports, restarted cleanly.
- **Regressions:** BUG-001 (pilgrims intact), BUG-002 (alerts intact), BUG-003 (canonical DBs intact), BUG-004 (canonical env intact).
- **Build:** `npm run build` compiled cleanly.

---

## R. SEC-01 Resolution

**Security Finding Title:** SEC-01 — Plaintext Password Storage  
**Status:** VERIFIED  

### Root Cause:
User-submitted passwords were saved directly to SQLite in plaintext without cryptographic hashing, and login authentication verified credentials using a literal string equality operator (`user.password !== password`).

### Remediation Executed:
1. **Pre-Fix Binary Backup:** Created and verified backup of canonical `auth.sqlite` at `backups/sec-01-pre-fix/auth.sqlite` (16,384 bytes, 15 records).
2. **Library Installation:** Installed pure-JavaScript `bcryptjs` with zero native compilation dependencies.
3. **Transactional Migration:** Created and executed `server/migrate_passwords.js`. Idempotently hashed all 15 existing accounts using bcrypt (cost factor 10). Verified each hash via `bcrypt.compare` before committing. Purged 100% of plaintext passwords from `auth.sqlite`.
4. **Registration Endpoint (`POST /api/register`):** Refactored to hash passwords with `await bcrypt.hash(password, 10)` before insertion.
5. **Login Endpoint (`POST /api/login`):** Refactored to authenticate using `await bcrypt.compare(password, user.password)`. Removed all plaintext comparison fallbacks.
6. **Google OAuth Hardening (`POST /api/google-login`):** Replaced static string `'GOOGLE_LOGIN'` with an un-guessable locked random hash (`OAUTH_LOCKED_${randomUUID}`).
7. **Administrative & Seed Scripts:** Updated `create_admin.js`, `create_new_admin.js`, and `seedAgents.js` to hash passwords with `bcryptjs` prior to insertion/update.
8. **Debug Utilities Sanitization:** Sanitized `check_admin.js` and `verify_pass.js` to eliminate credential logging.

### Empirical Testing Results:
- **Database State:** 15/15 accounts in `auth.sqlite` verified containing valid 60-character bcrypt hashes (`$2b$10$`). Zero plaintext remaining.
- **Authentication Continuity:** Existing administrator, agent, and user accounts verified able to log in using their original credentials.
- **Security Rejection:** Invalid passwords returned HTTP 401.
- **New Registrations:** Verified new accounts store immediate bcrypt hashes.
- **OAuth Hardening:** Verified OAuth accounts cannot authenticate via password login using `'GOOGLE_LOGIN'`.
- **Secret Hygiene:** Verified zero passwords or hashes exposed in API responses (`/api/admin/users`, `/api/login`) or console logs.
- **Regressions:** BUG-001 (4 pilgrims), BUG-002 (2 alerts), BUG-003 (canonical paths), BUG-004 (environment loader), BUG-005 (zero hardcoded API keys) all remain 100% verified.
- **Cold Restart & Build:** Server restarted cold cleanly and `npm run build` completed successfully.

---

## S. SEC-02 Resolution

**Security Finding Title:** SEC-02 — Missing Server-Side Authentication & Authorization  
**Status:** VERIFIED  

### 1. Root Cause
The Express API was entirely open to unauthenticated access across administrative, management, and personal data endpoints. Sensitive administrative functions (`/api/admin/*`), pilgrim operations, booking updates, and message deletions relied solely on client-side routing guards without any server-side authentication tokens, session validation, role enforcement, or CSRF defense.

### 2. Remediation Architecture
1. **Cryptographic Identity Layer:** Implemented stateless JWTs (`jsonwebtoken`) signed using HMAC-SHA256 (`HS256`) with an 8-hour expiration.
2. **Credential Transmission:** JWTs are issued exclusively via secure `HttpOnly` cookies (`authToken`), completely preventing JavaScript access and neutralizing XSS token theft.
3. **Double-Submit CSRF Defense:** Paired authentication with a cryptographically generated non-HttpOnly cookie (`csrfToken`) and required `X-CSRF-Token` header validation (`verifyCsrf`) for all protected state-changing operations.
4. **Middleware Enforcement:** Implemented a robust middleware chain:
   - `verifyToken`: Reads and cryptographically verifies `authToken` cookie, populating `req.user = { id, role }`.
   - `requireAdmin`: Enforces `req.user.role === 'admin'`.
   - `requireAgent`: Enforces `req.user.role === 'admin' || req.user.role === 'agent'`.
   - `verifyCsrf`: Enforces cookie and header match on state-changing requests.

---

### 3. Authoritative Route Inventory (Derived Directly from Source `server/index.js`)

Exact route count derived from source code: **35 routes** (plus 1 static assets mount `app.use('/uploads', express.static('uploads'))`).

| # | Method | Exact Path | Authentication | Authorization | CSRF Protection | Intentionally Public |
|---|---|---|---|---|---|---|
| 1 | `GET` | `/api/test` | None | Public | None | Yes (Healthcheck) |
| 2 | `GET` | `/api/auth/csrf` | None | Public | None | Yes (CSRF token issuance) |
| 3 | `GET` | `/api/auth/me` | `verifyToken` | Authenticated | None | No |
| 4 | `POST` | `/api/logout` | None | Public | None | Yes (Session/cookie clearance) |
| 5 | `GET` | `/api/destinations` | None | Public | None | Yes (Public destination catalogue) |
| 6 | `GET` | `/api/agents` | `verifyToken` | Authenticated | None | No |
| 7 | `GET` | `/api/groups` | `verifyToken` | Authenticated | None | No |
| 8 | `POST` | `/api/groups` | `verifyToken` | Agent/Admin | `verifyCsrf` | No |
| 9 | `POST` | `/api/register` | None | Public | None | Yes (Public account registration) |
| 10 | `POST` | `/api/google-login` | None | Public | None | Yes (Public OAuth login) |
| 11 | `GET` | `/api/captcha` | None | Public | None | Yes (Public captcha challenge) |
| 12 | `POST` | `/api/login` | None | Public | None | Yes (Public user authentication) |
| 13 | `POST` | `/api/bookings` | `verifyToken` | Authenticated | `verifyCsrf` | No |
| 14 | `PUT` | `/api/admin/bookings/:id` | `verifyToken` | Admin only | `verifyCsrf` | No |
| 15 | `GET` | `/api/notifications` | `verifyToken` | Authenticated | None | No |
| 16 | `POST` | `/api/notifications/read` | `verifyToken` | Authenticated | `verifyCsrf` | No |
| 17 | `DELETE` | `/api/admin/messages/:id` | `verifyToken` | Admin only | `verifyCsrf` | No |
| 18 | `POST` | `/api/admin/send-notification` | `verifyToken` | Admin only | `verifyCsrf` | No |
| 19 | `POST` | `/api/contact` | None | Public | None | Yes (Public inquiry submission) |
| 20 | `GET` | `/api/admin/stats` | `verifyToken` | Admin only | None | No |
| 21 | `GET` | `/api/admin/users` | `verifyToken` | Admin only | None | No |
| 22 | `PUT` | `/api/admin/users/:id/role` | `verifyToken` | Admin only | `verifyCsrf` | No |
| 23 | `GET` | `/api/admin/messages` | `verifyToken` | Admin only | None | No |
| 24 | `GET` | `/api/admin/bookings` | `verifyToken` | Admin only | None | No |
| 25 | `GET` | `/api/my-bookings` | `verifyToken` | Authenticated | None | No |
| 26 | `POST` | `/api/pilgrims` | `verifyToken` | Agent/Admin | `verifyCsrf` | No |
| 27 | `POST` | `/api/pilgrims/bulk` | `verifyToken` | Agent/Admin | `verifyCsrf` | No |
| 28 | `GET` | `/api/pilgrims` | `verifyToken` | Authenticated | None | No |
| 29 | `GET` | `/api/pilgrims/check-alerts` | `verifyToken` | Agent/Admin | None | No |
| 30 | `PUT` | `/api/pilgrims/:id/status` | `verifyToken` | Agent/Admin | `verifyCsrf` | No |
| 31 | `GET` | `/api/pilgrims/export-pdf` | `verifyToken` | Authenticated | None | No |
| 32 | `GET` | `/api/ai/status` | None | Public | None | Yes (AI provider availability) |
| 33 | `POST` | `/api/ai/generate` | `verifyToken` | Authenticated | None | No |
| 34 | `POST` | `/api/ai/chat` | `verifyToken` | Authenticated | None | No |
| 35 | `POST` | `/api/tts` | `verifyToken` | Authenticated | None | No |

---

### 4. Classification Breakdown Summary

Every actual route appears in exactly one category based strictly on its active middleware:

| Classification | Count | Description / Routes Included |
|---|:---:|---|
| **Public** | **10** | Unrestricted routes: `/api/test`, `/api/auth/csrf`, `/api/logout`, `/api/destinations`, `/api/register`, `/api/google-login`, `/api/captcha`, `/api/login`, `/api/contact`, `/api/ai/status` |
| **Authenticated** | **12** | Requires valid `verifyToken` session without role restriction: `/api/auth/me`, `/api/agents`, `/api/groups` (GET), `/api/bookings` (POST), `/api/notifications` (GET), `/api/notifications/read` (POST), `/api/my-bookings` (GET), `/api/pilgrims` (GET), `/api/pilgrims/export-pdf` (GET), `/api/ai/generate` (POST), `/api/ai/chat` (POST), `/api/tts` (POST) |
| **Agent/Admin** | **5** | Requires `verifyToken` + `requireAgent`: `/api/groups` (POST), `/api/pilgrims` (POST), `/api/pilgrims/bulk` (POST), `/api/pilgrims/check-alerts` (GET), `/api/pilgrims/:id/status` (PUT) |
| **Admin only** | **8** | Requires `verifyToken` + `requireAdmin`: `/api/admin/bookings/:id` (PUT), `/api/admin/messages/:id` (DELETE), `/api/admin/send-notification` (POST), `/api/admin/stats` (GET), `/api/admin/users` (GET), `/api/admin/users/:id/role` (PUT), `/api/admin/messages` (GET), `/api/admin/bookings` (GET) |
| **TOTAL** | **35** | **100% of defined Express routes** |

#### Resolution of Previous Discrepancies:
- `/api/ai/chat` is classified **solely as Authenticated** (`verifyToken`), completely removed from "Admin only".
- `/api/ai/status` is classified **solely as Public** (zero middleware attached).
- `/api/tts` and `/api/ai/chat` are classified according to their actual middleware (`verifyToken`), not by generic feature descriptions.
- All routes without role checks are labeled "Authenticated", routes with `requireAdmin` are labeled "Admin only", and routes with `requireAgent` are labeled "Agent/Admin".

---

### 5. Middleware Ordering & Protected Write CSRF Audit

All 10 protected write/mutation endpoints strictly adhere to the enforced security order:
`verifyToken` → `Role Middleware (requireAdmin / requireAgent)` → `verifyCsrf` → `Route Handler / Multer`

| Endpoint | Method | Middleware Sequence Verified | CSRF Enforced |
|---|---|---|:---:|
| `/api/groups` | POST | `verifyToken` → `requireAgent` → `verifyCsrf` → handler | **YES** |
| `/api/bookings` | POST | `verifyToken` → `verifyCsrf` → `upload.single('passport')` → handler | **YES** |
| `/api/admin/bookings/:id` | PUT | `verifyToken` → `requireAdmin` → `verifyCsrf` → handler | **YES** |
| `/api/notifications/read` | POST | `verifyToken` → `verifyCsrf` → handler | **YES** |
| `/api/admin/messages/:id` | DELETE | `verifyToken` → `requireAdmin` → `verifyCsrf` → handler | **YES** |
| `/api/admin/send-notification` | POST | `verifyToken` → `requireAdmin` → `verifyCsrf` → handler | **YES** |
| `/api/admin/users/:id/role` | PUT | `verifyToken` → `requireAdmin` → `verifyCsrf` → handler | **YES** |
| `/api/pilgrims` | POST | `verifyToken` → `requireAgent` → `verifyCsrf` → handler | **YES** |
| `/api/pilgrims/bulk` | POST | `verifyToken` → `requireAgent` → `verifyCsrf` → handler | **YES** |
| `/api/pilgrims/:id/status` | PUT | `verifyToken` → `requireAgent` → `verifyCsrf` → handler | **YES** |

*Audit Verification:* Exactly **0** protected write endpoints are missing CSRF protection.

---

### 6. Cookie Attribute Verification

Empirical runtime inspection of `Set-Cookie` headers generated during authentication:

| Cookie Name | HttpOnly | SameSite | Max-Age | Secure | Status |
|---|:---:|:---:|:---:|:---:|:---:|
| `authToken` | **true** | `lax` (`strict` in prod) | 28800s (8h) | Conditional (`NODE_ENV === 'production'`) | **VERIFIED** |
| `csrfToken` | **false** (accessible to JS) | `lax` (`strict` in prod) | 28800s (8h) | Conditional (`NODE_ENV === 'production'`) | **VERIFIED** |

- **XSS Neutralization:** `authToken` is marked `HttpOnly`, preventing extraction via client JavaScript.
- **CSRF Token Accessibility:** `csrfToken` explicitly omits `HttpOnly` so the SPA client can read and submit it in the `X-CSRF-Token` header.
- **Lifetime Consistency:** Both cookies are synchronized with an 8-hour TTL (`Max-Age=28800`).

---

### 7. JWT Absence from Client Storage & API Responses

Verification confirmed that stateless JWT strings are never exposed to JavaScript storage:
- `localStorage`: **0 occurrences** across entire client codebase.
- `sessionStorage`: **0 JWT occurrences** (stores only `captcha_verified` boolean gate flag).
- `POST /api/login` response body: Returns sanitized user object (`{ id, name, email, phone, role, account_type }`). Contains zero token strings.
- `GET /api/auth/me` response body: Returns sanitized identity (`{ user }`). Contains zero token strings.

---

### 8. Authenticated CSRF Test Results

Tested against authenticated administrative state mutation endpoint (`POST /api/admin/send-notification`):

| Test Condition | Cookie State | `X-CSRF-Token` Header | Expected | Actual Status | Result |
|---|---|---|:---:|:---:|:---:|
| Missing CSRF Header | Valid `authToken` + `csrfToken` | *Omitted* | 403 Forbidden | 403 ("CSRF token missing") | **PASS** |
| Mismatched CSRF Header | Valid `authToken` + `csrfToken` | `forged-token-value` | 403 Forbidden | 403 ("CSRF token mismatch") | **PASS** |
| Valid CSRF Header | Valid `authToken` + `csrfToken` | Matching cookie value | 200 OK | 200 (Success) | **PASS** |

---

### 9. Authorization Matrix Verification

Tested across all 4 actor categories and route authorization levels:

| Actor Role | Route Under Test | Route Authorization | Expected Status | Actual Status | Result |
|---|---|---|:---:|:---:|:---:|
| Anonymous (Guest) | `GET /api/admin/stats` | Admin only | 401 Unauthorized | 401 | **PASS** |
| Regular User | `GET /api/admin/stats` | Admin only | 403 Forbidden | 403 | **PASS** |
| Agent | `GET /api/admin/stats` | Admin only | 403 Forbidden | 403 | **PASS** |
| Administrator | `GET /api/admin/stats` | Admin only | 200 OK | 200 | **PASS** |
| Anonymous (Guest) | `GET /api/pilgrims/check-alerts` | Agent/Admin | 401 Unauthorized | 401 | **PASS** |
| Regular User | `GET /api/pilgrims/check-alerts` | Agent/Admin | 403 Forbidden | 403 | **PASS** |
| Agent | `GET /api/pilgrims/check-alerts` | Agent/Admin | 200 OK | 200 | **PASS** |
| Administrator | `GET /api/pilgrims/check-alerts` | Agent/Admin | 200 OK | 200 | **PASS** |
| Anonymous (Guest) | `GET /api/my-bookings` | Authenticated | 401 Unauthorized | 401 | **PASS** |
| Regular User | `GET /api/my-bookings` | Authenticated | 200 OK | 200 | **PASS** |

---

### 10. JWT_SECRET Rotation & Signature Verification

- **Rotation Confirmed:** Generated a fresh 64-byte (512-bit) cryptographically secure random hexadecimal secret via `crypto.randomBytes(64)`.
- **Environment Hygiene:** Injected solely into `.env` (excluded by `.gitignore`). `.env.example` contains only `JWT_SECRET=` placeholder without any key material.
- **Cryptographic Validation:** Tested forged/tampered JWT signatures and tokens signed with invalid/old keys against protected routes (`GET /api/auth/me`). All invalid tokens were rejected with HTTP 401.

---

### 11. Logout Architecture & Stateless JWT Lifecycle

- **Logout Endpoint (`POST /api/logout`):** Issues expired `Set-Cookie` response headers (`Max-Age=0; Expires=Thu, 01 Jan 1970 00:00:00 GMT`) for both `authToken` and `csrfToken`.
- **Browser Behavior:** The browser immediately clears stored cookies and ceases sending authentication headers. Subsequent API calls from the browser return HTTP 401.
- **Stateless Architectural Characteristic (Documented):** In a stateless JWT architecture without server-side revocation lists or distributed caching (e.g. Redis), an external client that physically copies the raw JWT string prior to logout can continue to present that token until its cryptographic expiration (8 hours). This is a known, expected characteristic of stateless JWT architectures, not an implementation failure. Future token invalidation requirements may implement short token lifetimes with refresh tokens or server-side revocation tables.

---

### 12. Remaining SEC-03 Findings (Preserved Baseline)

SEC-02 addresses authentication and role-based authorization only. The following findings remain open for SEC-03:
- **`GET /api/my-bookings?phone=...`**: Relies on caller-supplied `phone` query parameter rather than enforcing `req.user.id` ownership binding.
- **`GET /api/notifications?phone=...`**: Relies on caller-supplied `phone` query parameter.
- **`GET /api/pilgrims?agent_id=...`**: Agents can query and view pilgrims assigned to other agents.
- **`POST /api/bookings`**: Bookings lack user ID binding upon submission.

## **SEC-02 Status: VERIFIED** ✅

---

## T. SEC-03 Resolution

**Security Finding Title:** SEC-03 — Broken Object Level Authorization (BOLA / IDOR) on Personal Data  
**Status:** VERIFIED  

### 1. Root Cause Analysis

1. **`GET /api/my-bookings?phone=...`**:
   - **Root Cause:** The endpoint accepted an untrusted `phone` query parameter directly from the caller and queried `SELECT * FROM bookings WHERE phone = ?` without verifying that the requested phone belonged to the authenticated caller (`req.user.id`). Any authenticated user could supply another user's phone number to enumerate and read their private bookings, travel dates, destinations, and personal details.
2. **`GET /api/notifications?phone=...` & `POST /api/notifications/read`**:
   - **Root Cause:** Both notification retrieval and read status updates accepted an unvalidated `phone` parameter without checking ownership against `req.user`. An authenticated user could supply another user's phone or `'admin'` to view private notifications and administrative alerts, or mark another user's notifications as read.
3. **`GET /api/pilgrims?agent_id=...` & `GET /api/pilgrims/export-pdf`**:
   - **Root Cause:** The pilgrims tracking and PDF export endpoints accepted `agent_id` from the query string without ownership verification. If an agent omitted the parameter, the server returned all pilgrims across all agents; if an agent passed another agent's ID, the server returned that other agent's pilgrims.
4. **`POST /api/bookings`**:
   - **Root Cause:** The booking creation endpoint did not bind created bookings to `req.user.id` (the `bookings` table lacked a `user_id` column). Furthermore, for Hajj/Umrah bookings, the auto-add pilgrim feature executed `SELECT id, name, phone FROM users WHERE phone = ?` using the untrusted client-supplied body `phone`, allowing an attacker to falsely attribute pilgrim records and sponsorship to other users.

---

### 2. Remediation Architecture & Exact Changes

#### Files Changed:
- [`server/index.js`](file:///d:/alkwut-websit/kuwait-travel/server/index.js):
  1. **Schema Migration & Backfill (`initDb`):**
     - Added `user_id INTEGER` column to `CREATE TABLE IF NOT EXISTS bookings`.
     - Executed transactional `ALTER TABLE bookings ADD COLUMN user_id INTEGER` safe migration.
     - Implemented automatic backfill populating `user_id` for existing bookings where `phone` matches a registered user in `authDb`.
  2. **`GET /api/my-bookings`:**
     - Fetches authenticated user identity `authUser` via `req.user.id`.
     - Non-admin users: If an untrusted `phone` query parameter is passed that does not match `authUser.phone`, the request is rejected with `HTTP 403 Forbidden` (`Access denied: cannot access another user's bookings`).
     - Scopes query strictly to authenticated user: `WHERE user_id = ? OR (user_id IS NULL AND phone = ?)`.
     - Admin users: Retain administrative visibility (can view all bookings or filter by specific phone).
  3. **`GET /api/notifications` & `POST /api/notifications/read`:**
     - Fetches authenticated user identity `authUser` via `req.user.id`.
     - Non-admin users: If `req.query.phone` is passed and does not match `authUser.phone`, rejected with `HTTP 403 Forbidden`.
     - Scopes notification query strictly to `authUser.phone`.
     - Non-admin users in `/api/notifications/read` are restricted to marking only their own notifications as read (`user_phone = authUser.phone`).
     - Admin users: Can view system alerts (`'admin'`) or query notifications for a specific phone.
  4. **`GET /api/pilgrims` & `GET /api/pilgrims/export-pdf`:**
     - Non-admin agents: If `agent_id` query parameter is supplied and does not match `req.user.id`, the request is rejected with `HTTP 403 Forbidden` (`Access denied: cannot view pilgrims assigned to another agent`).
     - If an agent makes a query without parameters, the server automatically enforces `p.agent_id = req.user.id`, strictly preventing cross-agent data exposure.
     - Admin users: Retain broad operational visibility (can view all pilgrims or filter by any `agent_id`).
  5. **`POST /api/bookings`:**
     - Retrieves authenticated user `authUser` via `req.user.id`.
     - Immutably binds created booking to `user_id = authUser.id` on the server.
     - Preserves legitimate booking contact phone in `phone` column while preventing ownership manipulation.
     - In Hajj/Umrah auto-add pilgrim flow, binds pilgrim `agent_id = authUser.id`, `sponsor_name = authUser.name`, and `sponsor_phone = authUser.phone`, eliminating sponsor impersonation.
  6. **`POST /api/pilgrims` & `POST /api/pilgrims/bulk`:**
     - Enforces `assignedAgentId = req.user.id` for non-admin callers, preventing an agent from inserting pilgrims under another agent's ID.

---

### 3. Automated Test Verification Results

Comprehensive automated security test suite executed: **32 passed / 0 failed**

| Test Case | Description | Expected | Actual Status | Result |
|---|---|:---:|:---:|:---:|
| 1.1 | Unauthenticated `GET /api/my-bookings` | 401 Unauthorized | 401 | **PASS** |
| 1.2 | User A creates legitimate booking | 200 OK | 200 | **PASS** |
| 1.3 | User B creates legitimate booking | 200 OK | 200 | **PASS** |
| 1.4 | User A retrieves own bookings with matching phone | 200 OK | 200 | **PASS** |
| 1.5 | **BOLA Prevention:** User A attempts to retrieve User B's bookings (`?phone=UserBPhone`) | 403 Forbidden | 403 | **PASS** |
| 1.6 | User A retrieves own bookings without query params | 200 OK | 200 | **PASS** |
| 1.7 | **Impersonation Prevention:** User A submits booking with User B's phone | 200 OK (bound to User A) | 200 | **PASS** |
| 1.7b | Verify User B does NOT see User A's spoof-attempt booking | 0 matching records for User B | 0 records | **PASS** |
| 1.7c | Verify booking is bound to User A via `user_id` | Owned by User A | User A id verified | **PASS** |
| 1.8 | Admin retrieves all bookings | 200 OK | 200 | **PASS** |
| 1.9 | Admin filters bookings by user phone | 200 OK | 200 | **PASS** |
| 2.1 | Unauthenticated `GET /api/notifications` | 401 Unauthorized | 401 | **PASS** |
| 2.2 | User A retrieves own notifications with phone param | 200 OK | 200 | **PASS** |
| 2.3 | **BOLA Prevention:** User A attempts to retrieve User B's notifications (`?phone=UserBPhone`) | 403 Forbidden | 403 | **PASS** |
| 2.4 | **Privilege Escalation Prevention:** User A attempts to retrieve admin notifications (`?phone=admin`) | 403 Forbidden | 403 | **PASS** |
| 2.5 | User A retrieves own notifications without query params | 200 OK | 200 | **PASS** |
| 2.6 | **BOLA Prevention:** User A attempts to mark User B notifications as read | User B notifications remain unread (`is_read=0`) | `is_read=0` | **PASS** |
| 2.7 | Admin retrieves admin notifications | 200 OK | 200 | **PASS** |
| 3.1 | Unauthenticated `GET /api/pilgrims` | 401 Unauthorized | 401 | **PASS** |
| 3.2 | Agent A creates pilgrim | 200 OK | 200 | **PASS** |
| 3.3 | Agent B creates pilgrim | 200 OK | 200 | **PASS** |
| 3.4 | Agent A retrieves own pilgrims with `agent_id` param | 200 OK | 200 | **PASS** |
| 3.5 | **BOLA Prevention:** Agent A attempts to retrieve Agent B's pilgrims (`?agent_id=AgentBId`) | 403 Forbidden | 403 | **PASS** |
| 3.6 | Agent A retrieves pilgrims with no query params (auto-scoped to Agent A) | 200 OK (only Agent A pilgrims) | 200 | **PASS** |
| 3.7 | **BOLA Prevention:** Agent A attempts to export Agent B's pilgrims PDF | 403 Forbidden | 403 | **PASS** |
| 3.8 | Admin retrieves all pilgrims across agents | 200 OK | 200 | **PASS** |
| 3.9 | Admin filters pilgrims by specific `agent_id` | 200 OK | 200 | **PASS** |
| 4.1 | **SEC-01 Regression:** All accounts store valid bcrypt hashes (`$2b$10$`) | 100% bcrypt hashes | 100% verified | **PASS** |
| 4.2a | **SEC-02 Regression:** Invalid CSRF token rejected | 403 Forbidden | 403 | **PASS** |
| 4.2b | **SEC-02 Regression:** Valid CSRF token accepted | 200 OK | 200 | **PASS** |
| 4.3 | **SEC-02 Regression:** Secure cookie attributes (`HttpOnly`, `SameSite=Lax`) verified | Header attributes intact | Verified | **PASS** |
| 4.4 | **SEC-02 Regression:** Logout clears `authToken` and `csrfToken` cookies | Cookies cleared | Verified | **PASS** |

---

### 4. Build & Cold Restart Verification

- **Production Build:** `npm run build` executed successfully (Vite compiled in 6.92s with zero errors).
- **Cold Restart:** Stopped background server process, performed fresh cold restart on port 5000, and verified `/api/test` healthcheck response.

---

### 5. Remaining Security Findings (Post SEC-03)

- **`[SEC-04]` Unrestricted Arbitrary File Upload:** In `server/index.js` (lines 280–290), multer disk storage accepts uploads without file extension whitelisting, MIME type validation, or file magic byte verification.
- **`[SEC-06]` Denial of Service Risk via Global Rate Limiting Misconfiguration:** In `server/index.js` (lines 59–62), rate limiting applies globally (`100 req / 15 min`) per IP without separating static assets from API endpoints.
- **`[SEC-07]` CORS Configuration:** Review origin configurations for strict production deployments.

## **SEC-03 Status: VERIFIED** ✅

---

## U. SEC-04 Resolution

**Security Finding Title:** SEC-04 — Unrestricted Arbitrary File Upload  
**Status:** VERIFIED  

### 1. Root Cause Analysis
- **Unrestricted Uploads:** `multer.diskStorage` in `server/index.js` lacked file size limits, MIME type validation, and file extension whitelists.
- **Unsanitized Filename Extension:** `path.extname(file.originalname)` blindly trusted client-supplied file extensions, opening the server to dangerous executable uploads (`.exe`, `.php`, `.sh`, `.html`, `.svg`).
- **Missing File Signature / Magic Bytes Verification:** Files could be uploaded by renaming dangerous payloads to `.jpg` or `.png`, allowing spoofed files onto disk.
- **Unprotected Public Access:** The uploads directory was served statically via `app.use('/uploads', express.static('uploads'))` with zero authentication, exposing private passport scans to unauthenticated external access.

---

### 2. Remediation Architecture & Exact Changes

#### Files Changed:
- [`server/index.js`](file:///d:/alkwut-websit/kuwait-travel/server/index.js):
  1. **Strict Whitelist & File Size Limits:**
     - Enforced `MAX_FILE_SIZE = 5 * 1024 * 1024` (5 MB limit).
     - Whitelisted safe document and image extensions only: `.jpg`, `.jpeg`, `.png`, `.webp`, `.pdf`.
     - Whitelisted matching MIME types: `image/jpeg`, `image/png`, `image/webp`, `application/pdf`.
     - Rejects any other extension/MIME type with descriptive `400 Bad Request`.
  2. **Cryptographic Random Filename Generation:**
     - Eliminated reliance on user-provided original filenames: files are stored on disk as `${Date.now()}-${crypto.randomBytes(16).toString('hex')}${ext}`.
     - Neutralized path traversal (`../../`), null byte injection, and special characters.
  3. **File Signature (Magic Bytes) Verification (`validateFileSignature`):**
     - Reads initial bytes of uploaded file directly from disk before database persistence:
       - **JPEG:** `FF D8 FF`
       - **PNG:** `89 50 4E 47`
       - **PDF:** `%PDF-` (`25 50 44 46 2D`)
       - **WebP:** `RIFF....WEBP`
     - If magic bytes fail to match allowed signatures (e.g. executable or shell renamed to `.jpg`), the file is immediately purged from disk (`fs.unlinkSync`) and rejected with `400 Bad Request`.
  4. **Multer Error Handling Middleware (`handlePassportUpload`):**
     - Intercepts `LIMIT_FILE_SIZE` and multer errors, returning standardized JSON error responses with status 400 instead of unhandled Express crashes.
     - On any subsequent database or processing errors, uploaded files are cleaned up from disk.
  5. **Protected Static Storage:**
     - Updated static uploads mount: `app.use('/uploads', verifyToken, express.static('uploads'))`.
     - Unauthenticated requests to `/uploads/*` receive `401 Unauthorized`.
     - Authenticated users/administrators with valid session cookies can access legitimate passport scans.

---

### 3. Automated Test Verification Results

Comprehensive automated test suite executed: **40 passed / 0 failed**

| Test Case | Description | Expected | Actual Status | Result |
|---|---|:---:|:---:|:---:|
| 1.1 | Legitimate JPEG upload (`FF D8 FF...`) | 200 OK | 200 | **PASS** |
| 1.2 | Legitimate PNG upload (`89 50 4E 47...`) | 200 OK | 200 | **PASS** |
| 1.3 | Legitimate PDF upload (`%PDF-1.4...`) | 200 OK | 200 | **PASS** |
| 1.4 | Legitimate WebP upload (`RIFF...WEBP`) | 200 OK | 200 | **PASS** |
| 1.5 | Uploaded file path recorded in database | Path exists | Verified | **PASS** |
| 1.6 | Uploaded file saved on disk in `uploads/` | File exists on disk | Verified | **PASS** |
| 2.1 | Unauthenticated access to `/uploads/<file>` | 401 Unauthorized | 401 | **PASS** |
| 2.2 | Authenticated access to `/uploads/<file>` | 200 OK | 200 | **PASS** |
| 3.1 | Oversized file upload (> 5MB) | 400 Bad Request | 400 | **PASS** |
| 3.2 | Oversized file error message indicates size limit | Contains "5MB" | Verified | **PASS** |
| 3.3 | No oversized file persisted on disk in `uploads/` | File deleted | Verified | **PASS** |
| 4.1 | Disallowed extension `.php` rejected | 400 Bad Request | 400 | **PASS** |
| 4.2 | Disallowed extension `.exe` rejected | 400 Bad Request | 400 | **PASS** |
| 4.3 | Disallowed extension `.sh` rejected | 400 Bad Request | 400 | **PASS** |
| 4.4 | Disallowed extension `.html` rejected | 400 Bad Request | 400 | **PASS** |
| 4.5 | Disallowed extension `.svg` rejected | 400 Bad Request | 400 | **PASS** |
| 4.6 | Disallowed extension `.js` rejected | 400 Bad Request | 400 | **PASS** |
| 5.1 | Spoofed PHP shell disguised as `.jpg` with `image/jpeg` | 400 Bad Request (Magic Bytes Failure) | 400 | **PASS** |
| 5.2 | Spoofed Windows EXE disguised as `.png` with `image/png` | 400 Bad Request | 400 | **PASS** |
| 5.3 | Spoofed Bash script disguised as `.pdf` with `application/pdf` | 400 Bad Request | 400 | **PASS** |
| 6.1 | Path traversal filename `../../evil.jpg` safely handled | Sanitized randomized basename | Verified | **PASS** |
| 6.2 | Path traversal filename `..\..\evil.jpg` safely handled | Sanitized randomized basename | Verified | **PASS** |
| 6.3 | Null byte injection `test\0.jpg` safely handled | Rejected 400 | 400 | **PASS** |
| 6.4 | Double extension `something.php.jpg` safely handled | Sanitized randomized basename | Verified | **PASS** |
| 7.1 | Legitimate booking without passport upload succeeds | 200 OK | 200 | **PASS** |
| 7.2 | Hajj/Umrah booking with passport upload auto-adds to pilgrims | 200 OK | 200 | **PASS** |
| 8.1 | **SEC-01 Regression:** Bcrypt passwords intact (`$2b$10$`) | 100% bcrypt | Verified | **PASS** |
| 8.2 | **SEC-02 Regression:** Unauthenticated endpoints return 401 | 401 Unauthorized | 401 | **PASS** |
| 8.3 | **SEC-03 Regression:** BOLA on `/api/my-bookings` rejected | 403 Forbidden | 403 | **PASS** |

---

### 4. Build & Cold Restart Verification

- **Production Build:** `npm run build` executed successfully (Vite built cleanly in 4.27s).
- **Cold Restart:** Background server stopped, cold restarted fresh on port 5000, and verified healthy via `/api/test`.

---

### 5. Remaining Security Findings (Post SEC-04)

- **`[SEC-06]` Denial of Service Risk via Global Rate Limiting Misconfiguration:** Global rate limiter (`100 req / 15 min`) in `server/index.js` applies universally per IP across all endpoints without separating static assets from API routes.
- **`[SEC-07]` CORS Configuration:** Wildcard / default CORS review for strict production deployments.

---

## **SEC-04 Status: VERIFIED** ✅
