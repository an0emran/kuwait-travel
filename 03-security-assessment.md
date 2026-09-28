# 3 - Security Assessment

[← Return to Chapter 2: System Architecture](02-system-architecture.md) | [Next: System Hardening →](04-system-hardening.md)

---

## 3.1 Overview

Prior to commencing hardening, a comprehensive read-only security assessment and static code audit was executed against the **Kuwait Travel & Tourism Platform** codebase. The objective was to catalog operational vulnerabilities, architectural defects, and business logic flaws without introducing premature or breaking code modifications.

The assessment evaluated the platform against the **OWASP Top 10 Web Application Security Risks** (specifically Broken Access Control, Identification and Authentication Failures, Security Misconfiguration, and Cryptographic Failures) alongside runtime stability criteria.

---

## 3.2 Comprehensive Findings Matrix

The audit identified eleven discrete findings, partitioned into reliability engineering defects and application security vulnerabilities:

### Engineering & Reliability Defects

| ID | Area | Finding | Severity | Initial Status | Remediated Status |
|---|---|---|:---:|:---:|:---:|
| **BUG-001** | Data Persistence | Pilgrim records dropped on server startup | High | Verified Vulnerable | **REMEDIATED** |
| **BUG-002** | Schema Integrity | Missing `alerts` table causing runtime 500 errors | Medium | Verified Vulnerable | **REMEDIATED** |
| **BUG-003** | Database Paths | Inconsistent SQLite database paths from relative paths | Medium | Verified Vulnerable | **REMEDIATED** |
| **BUG-004** | Module Lifecycle | ES module import hoisting breaking AI environment config | Medium | Verified Vulnerable | **REMEDIATED** |
| **BUG-005** | Credential Hygiene | Third-party Google API keys committed to source files | High | Verified Vulnerable | **REMEDIATED** |

### Application Security Vulnerabilities

| ID | Area | Finding | Severity | Initial Status | Remediated Status |
|---|---|---|:---:|:---:|:---:|
| **SEC-01** | Credential Storage | Plaintext password storage and comparison | **CRITICAL** | Verified Vulnerable | **REMEDIATED** |
| **SEC-02** | Access Control | Complete lack of server-side authorization on API routes | **CRITICAL** | Verified Vulnerable | **REMEDIATED** |
| **SEC-03** | Authorization (BOLA) | Broken object-level authorization via client query parameters | **HIGH** | Verified Vulnerable | **REMEDIATED** |
| **SEC-04** | File Upload Security | Unrestricted arbitrary file upload to publicly served folder | **HIGH** | Verified Vulnerable | **REMEDIATED** |
| **SEC-06** | Availability / DoS | Global rate limit conflicts with client notification polling | **MEDIUM** | Verified Present | *Pending Refactor* |
| **SEC-07** | Network Policy | Permissive wildcard CORS policy across API routes | **LOW** | Verified Present | *Pending Refactor* |

---

## 3.3 Detailed Security Findings Analysis

---

### [SEC-01] Stored Plaintext Passwords

- **Severity:** **CRITICAL** (CVSS: 9.8)
- **Affected Endpoints:** `POST /api/register`, `POST /api/login`, `server/seedAgents.js`, `server/create_admin.js`

#### Problem
User credentials across all accounts (customers, travel agents, and administrators) were stored as unhashed, plaintext strings directly in the `users` table within `auth.sqlite`. During authentication, the login endpoint performed a literal string equality check (`user.password !== password`).

#### Root Cause
The initial prototype omitted a cryptographic password hashing pipeline. Registration queries directly interpolated the incoming password string into the database:
```javascript
// Vulnerable baseline pattern
await authDb.run(
    "INSERT INTO users (name, email, password, phone, account_type) VALUES (?, ?, ?, ?, ?)",
    [name, email, password, phone, account_type]
);
```

#### Impact
A compromise of `auth.sqlite` via directory traversal, inadvertent repository commit, physical backup exposure, or operational misconfiguration resulted in the instant, total disclosure of all user, agent, and administrator passwords across the entire system.

#### Remediation Direction
Migrate all existing database credentials to salted `bcrypt` hashes using cost factor 10. Update registration and administrative seed scripts to hash passwords prior to storage, and alter the login endpoint to verify credentials via `bcrypt.compare`.

---

### [SEC-02] Complete Lack of Server-Side Authorization

- **Severity:** **CRITICAL** (CVSS: 9.1)
- **Affected Endpoints:** All administrative routes (`/api/admin/*`), operational routes (`/api/pilgrims/*`), and AI controllers

#### Problem
Access control was enforced exclusively on the client side by conditionally rendering UI buttons based on in-memory state. The backend Express API server exposed all administrative and operational endpoints with zero authentication checks or role verification middleware.

#### Root Cause
No authentication middleware existed in the Express request pipeline. Sensitive administrative routes directly processed unauthenticated incoming requests:
```javascript
// Vulnerable baseline pattern: unauthenticated admin route
app.get('/api/admin/users', async (req, res) => {
    const users = await authDb.all("SELECT id, name, email, phone, role FROM users");
    res.json(users);
});
```
Sending an unauthenticated `GET http://localhost:5000/api/admin/users` via `curl` immediately dumped every registered user, phone number, email, and role.

#### Impact
Any unauthenticated network actor could enumerate confidential customer records, manipulate or cancel active bookings, delete customer feedback messages, and promote arbitrary accounts to platform administrators (`PUT /api/admin/users/:id/role`).

#### Remediation Direction
Implement stateless JSON Web Token (JWT) authentication issued upon successful login and stored in secure `HttpOnly`, `SameSite=Lax` cookies. Construct reusable middleware layers (`verifyToken`, `requireAdmin`, `requireAgent`, `verifyCsrf`) and bind them explicitly across every private API endpoint.

---

### [SEC-03] Broken Object-Level Authorization (BOLA / IDOR)

- **Severity:** **HIGH** (CVSS: 8.5)
- **Affected Endpoints:** `GET /api/my-bookings`, `GET /api/notifications`, `POST /api/notifications/read`, `GET /api/pilgrims`, `GET /api/pilgrims/export-pdf`

#### Problem
Data retrieval endpoints relied on caller-supplied identifiers passed as query parameters (such as `phone` or `agent_id`) rather than extracting identity from the verified user session.

#### Root Cause
The application backend trusted client input to determine data ownership:
```javascript
// Vulnerable baseline pattern: trusting caller phone query
app.get('/api/my-bookings', async (req, res) => {
    const { phone } = req.query;
    const rows = await dataDb.all("SELECT * FROM bookings WHERE phone = ?", [phone]);
    res.json(rows);
});
```

#### Impact
Any authenticated or unauthenticated attacker who knew or guessed a customer's phone number could retrieve full booking histories, scheduled travel dates, and passport attachment paths. Furthermore, any travel agent could query or export pilgrim manifests managed by competing travel agents simply by altering the `agent_id` query parameter.

#### Remediation Direction
Eliminate client-controlled identity parameters. Bind all queries to the cryptographically verified `req.user.id` extracted from the server-validated JWT. Require administrative or owning-agent privileges before permitting access to scoped records.

---

### [SEC-04] Unrestricted Arbitrary File Upload

- **Severity:** **HIGH** (CVSS: 8.2)
- **Affected Endpoints:** `POST /api/bookings` (Passport photo upload)

#### Problem
The file upload pipeline accepted files of arbitrary extension, MIME type, and size. Uploaded files were written directly to `./uploads/` and made publicly accessible to anyone on the internet via an unauthenticated static route (`app.use('/uploads', express.static('uploads'))`).

#### Root Cause
Multer was instantiated with default storage configuration lacking limits and file filters:
```javascript
// Vulnerable baseline pattern
const storage = multer.diskStorage({
    destination: (req, file, cb) => { cb(null, 'uploads/'); },
    filename: (req, file, cb) => { cb(null, Date.now() + path.extname(file.originalname)); }
});
const upload = multer({ storage: storage });
```

#### Impact
Malicious users could upload executable scripts, HTML/SVG files containing malicious JavaScript (Stored XSS), or massive multi-gigabyte files causing server disk exhaustion (Denial of Service). Furthermore, sensitive passport identity scans could be harvested by unauthenticated web crawlers.

#### Remediation Direction
1. Enforce a strict 5 MB file size limit.
2. Whitelist permitted extensions (`.jpg`, `.jpeg`, `.png`, `.webp`, `.pdf`).
3. Validate client-declared MIME types.
4. Verify file signatures (magic bytes) to ensure the physical binary payload matches its declared image or document header.
5. Store files using randomized cryptographic filenames.
6. Protect the `/uploads` directory with `verifyToken` middleware.

---

### [SEC-06] Denial of Service Risk via Rate Limiter Misconfiguration

- **Severity:** **MEDIUM** (CVSS: 5.3)
- **Affected Area:** Global rate limiting middleware vs. frontend notification poller

#### Problem
A global rate limiter was configured in `server/index.js` capping all requests at 100 requests per 15 minutes per IP. However, the client component `App.jsx` instituted a 10-second polling interval against `GET /api/notifications`.

#### Root Cause
Ten-second polling generates 90 requests per 15 minutes for notifications alone. A user actively interacting with the platform for more than 10 minutes exceeded the 100-request quota and was locked out with HTTP `429 Too Many Requests`.

#### Current Status
Documented as a known operational limitation. The immediate mitigation is adjusting polling frequency in the client; future architectural remediation will implement WebSockets or Server-Sent Events (SSE).

---

### [SEC-07] Permissive Wildcard CORS Configuration

- **Severity:** **LOW** (CVSS: 3.7)
- **Affected Area:** Global CORS configuration in `server/index.js`

#### Problem
The Express API applies unconstrained CORS: `app.use(cors())`, allowing cross-origin requests from any origin.

#### Root Cause
Configured as an open wildcard during early prototype development to ease local frontend-backend proxying.

#### Current Status
Documented as a known configuration limitation. Because authentication is now anchored to `SameSite=Lax` cookies with CSRF header enforcement, cross-origin request forgery is mitigated; however, CORS should be explicitly restricted to the production client origin in future releases.

---

## 3.4 Next Chapter

Review the resolution of system configuration and reliability defects:  
👉 **[4 - System Hardening](04-system-hardening.md)**
