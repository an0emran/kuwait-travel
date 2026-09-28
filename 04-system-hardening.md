# 4 - System Hardening

[← Return to Chapter 3: Security Assessment](03-security-assessment.md) | [Next: Authentication & Authorization →](05-authentication-and-authorization.md)

---

## 4.1 Overview

Before tackling complex access control and cryptographic challenges, the platform required foundational stabilization. Baseline runtime analysis identified five architectural and configuration defects (BUG-001 through BUG-005) that caused data loss across server restarts, triggered 500 runtime exceptions, fragmented database storage across working directories, broke AI provider initialization, and exposed API credentials in source files.

Each defect was resolved following the strict engineering paradigm:

$$\text{Problem} \longrightarrow \text{Root Cause} \longrightarrow \text{Implementation} \longrightarrow \text{Verification} \longrightarrow \text{Result}$$

---

## 4.2 BUG-001: Pilgrim Data Loss Across Server Restarts

### Problem
Whenever the Node.js server restarted (via development file watchers, deployment reboots, or crashes), all previously registered pilgrims and Excel batch imports vanished. Agents were forced to re-enter traveler records repeatedly.

### Root Cause
During server boot, `initDb()` in `server/index.js` executed an unconditional destructive DDL statement:
```javascript
// Defective baseline code in server/index.js
await dataDb.run(`DROP TABLE IF EXISTS pilgrims`);
await dataDb.run(`CREATE TABLE IF NOT EXISTS pilgrims (...)`);
```
This query had been added during early prototyping to bypass a foreign key conflict, but inadvertently wiped production data on every restart.

### Implementation
The destructive `DROP TABLE` statement was permanently removed. An additive migration strategy was implemented using `CREATE TABLE IF NOT EXISTS`, complemented by isolated `ALTER TABLE` blocks inside safe `try/catch` handlers to migrate schema fields without dropping existing rows:
```javascript
// Remediated code in server/index.js
await dataDb.run(`CREATE TABLE IF NOT EXISTS pilgrims (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    passport_number TEXT,
    arrival_date DATE,
    agent_id INTEGER,
    sponsor_name TEXT,
    sponsor_phone TEXT,
    group_id INTEGER,
    status TEXT DEFAULT 'active',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
)`);

// Safe column migrations for backward compatibility
try {
    await dataDb.run("ALTER TABLE pilgrims ADD COLUMN status TEXT DEFAULT 'active'");
} catch (e) { /* Column already exists */ }

try {
    await dataDb.run("ALTER TABLE pilgrims ADD COLUMN group_id INTEGER");
} catch (e) { /* Column already exists */ }
```

### Verification
1. Inserted multiple pilgrim test records with valid passport numbers and arrival dates into `data.sqlite`.
2. Queried `GET /api/pilgrims` to confirm row existence.
3. Terminated the server process and executed a clean cold start (`node server/index.js`).
4. Re-queried `GET /api/pilgrims` and checked the database row count directly via SQLite CLI.

### Result
**VERIFIED:** Zero pilgrim data loss on server restarts. Schema upgrades execute idempotently without truncating operational data.

---

## 4.3 BUG-002: Missing Schema Definition for Alerts Table

### Problem
Whenever an agent visited the pilgrim tracking dashboard or the system executed an automated visa duration audit via `GET /api/pilgrims/check-alerts`, the Express server threw an unhandled SQL exception and crashed or returned `HTTP 500 Internal Server Error`:
```text
SQLITE_ERROR: no such table: alerts
```

### Root Cause
The route handler for `/api/pilgrims/check-alerts` attempted to query and insert into `alerts`:
```javascript
await dataDb.run(
    "INSERT INTO alerts (pilgrim_id, alert_type, message) VALUES (?, ?, ?)",
    [pilgrim.id, 'overstay_warning', warningMessage]
);
```
However, the `alerts` table was never defined or initialized in `initDb()`.

### Implementation
Added an explicit, structured schema definition for `alerts` inside `initDb()` in `server/index.js`, including relational integrity with the `pilgrims` table:
```javascript
// Remediated code in server/index.js
await dataDb.run(`CREATE TABLE IF NOT EXISTS alerts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    pilgrim_id INTEGER,
    alert_type TEXT,
    message TEXT,
    sent_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (pilgrim_id) REFERENCES pilgrims(id)
)`);
```

### Verification
1. Created a pilgrim record with an `arrival_date` 75 days in the past (exceeding the 70-day warning threshold).
2. Invoked `GET /api/pilgrims/check-alerts`.
3. Checked that the endpoint responded with `HTTP 200 OK` and returned the generated alert payload.
4. Inspected `data.sqlite` to verify that the corresponding warning record was successfully inserted into `alerts`.

### Result
**VERIFIED:** `alerts` table is reliably initialized on application launch; visa duration audits and escalation alerts execute cleanly without runtime SQL failures.

---

## 4.4 BUG-003: Inconsistent Working-Directory Database Paths

### Problem
When the server or maintenance utilities (`create_admin.js`, `seedAgents.js`) were executed from different working directories (e.g., executing `node server/index.js` from the repository root versus executing `node index.js` from within `server/`), duplicate SQLite files were spawned. Changes made in one directory were invisible in the other.

### Root Cause
In `server/db.js`, database paths were declared using relative string paths:
```javascript
// Defective baseline code in server/db.js
const authDbPromise = open({ filename: './auth.sqlite', driver: sqlite3.Database });
const dataDbPromise = open({ filename: './data.sqlite', driver: sqlite3.Database });
```
In Node.js, `./` resolves relative to `process.cwd()` (the directory from which the command was invoked), not the script directory.

### Implementation
Refactored `server/db.js` to anchor database locations to the canonical project root using Node.js URL and path resolution primitives:
```javascript
// Remediated code in server/db.js
import path from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// Canonical database paths (resolves to project root regardless of process.cwd())
const AUTH_DB_PATH = path.resolve(__dirname, '../auth.sqlite');
const DATA_DB_PATH = path.resolve(__dirname, '../data.sqlite');

const authDbPromise = open({ filename: AUTH_DB_PATH, driver: sqlite3.Database });
const dataDbPromise = open({ filename: DATA_DB_PATH, driver: sqlite3.Database });

export { authDbPromise, dataDbPromise, AUTH_DB_PATH, DATA_DB_PATH };
```

### Verification
1. Ran an inspection script from the project root (`d:\alkwut-websit\kuwait-travel`) that wrote a test record to `data.sqlite`.
2. Ran a second script from inside `server/` that queried the same test record.
3. Verified that both processes read from and wrote to identical canonical files (`d:\alkwut-websit\kuwait-travel\data.sqlite` and `d:\alkwut-websit\kuwait-travel\auth.sqlite`).

### Result
**VERIFIED:** Single source of truth for both databases across all execution contexts. Zero duplicate database fragmentation.

---

## 4.5 BUG-004: ES Module Initialization Race in AI Services

### Problem
When starting the server with valid Google Gemini or LangChain API keys configured in `.env`, invoking `GET /api/ai/status` returned all providers as disabled:
```json
{ "gemini": false, "openai": false, "claude": false, "custom": false, "mock": true }
```

### Root Cause
In ECMAScript Modules (ESM), static `import` declarations are hoisted and evaluated before any top-level executable statements. In `server/index.js`, the code was ordered as:
```javascript
// Defective baseline code in server/index.js
import { aiService } from './aiService.js'; // 1. Hoisted: constructor runs NOW
import dotenv from 'dotenv';
dotenv.config();                          // 2. Evaluated AFTER constructor!
```
Because `aiService.js` instantiated its client in its constructor, `process.env.GOOGLE_API_KEY` was still `undefined` at initialization time, forcing the service into mock mode.

### Implementation
Created a dedicated environment initialization module (`server/env.js`) that immediately executes `dotenv.config()` using the canonical root path, and imported it as the very first line of `server/index.js`:
```javascript
// server/env.js
import dotenv from 'dotenv';
import path from 'path';
import { fileURLToPath } from 'url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

// Load .env from canonical project root path
dotenv.config({ path: path.resolve(__dirname, '../.env') });
```
```javascript
// server/index.js (Line 1)
import './env.js'; // Evaluated first, guaranteeing environment availability
import express from 'express';
import { aiService } from './aiService.js';
```

### Verification
1. Set a valid environment key in `.env`.
2. Started the server and queried `GET /api/ai/status`.
3. Verified that the response returned `gemini: true` and that conversational completions streamed successfully from the cloud provider without falling back to mock responses.

### Result
**VERIFIED:** Deterministic environment variable loading on startup across all imported modules and service constructors.

---

## 4.6 BUG-005: Credential Hygiene and Secret Externalization

### Problem
Third-party Google Cloud and Gemini API keys were hardcoded directly in application source files (`server/geminiService.js` and `server/index.js`), risking accidental credential leak during version control pushes.

### Root Cause
Development shortcuts during early prototype integration where credentials were pasted directly into service files instead of referencing `process.env`.

### Implementation
1. **Secret Eviction:** Removed all hardcoded API key strings from JavaScript source code.
2. **Environment Variable Binding:** Bound service initializers exclusively to `process.env.GOOGLE_API_KEY` and `process.env.GOOGLE_TTS_API_KEY`.
3. **Template Standardization:** Created a sanitized `.env.example` in the repository root documenting required configuration variable names without disclosing actual values.
4. **Credential Rotation:** Revoked and regenerated all previously exposed development keys.

### Verification
Executed an automated regex scan across all source files (`*.js`, `*.jsx`, `*.json`, `*.html`) checking for Google API key patterns (`AIzaSy...`) and verifying zero hardcoded matches outside of git-ignored local `.env` files.

### Result
**VERIFIED:** Zero credentials stored in application source code. Clean separation of secrets from codebase logic.

---

## 4.7 Hardening Summary Table

| Finding ID | Component | Core Change | Impact |
|---|---|---|---|
| **BUG-001** | `server/index.js` | Removed destructive `DROP TABLE pilgrims` | Persistent pilgrim manifests |
| **BUG-002** | `server/index.js` | Defined `alerts` table in `initDb()` | Error-free 70+ day visa alerts |
| **BUG-003** | `server/db.js` | Absolute path resolution for SQLite handles | Single source of truth for databases |
| **BUG-004** | `server/env.js` | Pre-import environment loader module | Reliable cloud AI initialization |
| **BUG-005** | Source Code & `.env` | Externalized API keys; added `.env.example` | Secure credential hygiene |

---

## 4.8 Next Chapter

Review the implementation of cryptographic password hashing and server-side authorization:  
👉 **[5 - Authentication & Authorization](05-authentication-and-authorization.md)**
