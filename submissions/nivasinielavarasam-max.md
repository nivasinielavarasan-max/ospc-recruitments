# Security Assessment — nivasinielavarasan-max

**Challenge commit:** `<Finding the vulnerabilities>`  
**Date:** 2026-10-10

## Finding 1 — Predictable and Forgable Session Tokens

- **Severity:** High
- **Affected file and line(s):** `src/web.js`, lines 31–40
- **Affected route or feature:** Login and authentication for protected routes.
- **Evidence:** The `makeSessionToken()` function at lines 31–33 creates a token from the username and a fixed string, then encodes it using Base64 URL encoding. The `getUserFromRequest()` function at lines 35–40 decodes the token and trusts the extracted username if it matches a user. Base64 encoding does not provide secrecy or integrity. This was identified through source-code review; token forgery was not dynamically tested.
- **Impact:** An attacker who knows a valid username may be able to construct a matching session token and impersonate that user without knowing their password.
- **Recommended mitigation:** Use cryptographically random session identifiers backed by a server-side session store, or securely signed tokens with proper signature verification. Rotate session identifiers after login and invalidate them on logout.

## Finding 2 — Privilege Escalation Through Mass Assignment

- **Severity:** High
- **Affected file and line(s):** `src/server.js`, lines 58–65; authorization check at lines 68–72.
- **Affected route or feature:** `POST /api/profile` and `GET /admin`.
- **Evidence:** At line 64, the profile handler applies client-supplied properties directly to the user object using `Object.assign(user, changes)`. I logged in as the fictional user Alex and submitted `{"role":"admin"}` to the profile endpoint. After the update, Alex could access `/admin`, which previously returned HTTP 403.
- **Impact:** An authenticated ordinary user can change their role and gain administrative access.
- **Recommended mitigation:** Allowlist only permitted profile fields, such as `displayName`. Never accept authorization-sensitive fields such as `role` from client input. Enforce authorization checks on privileged operations.

## Finding 3 — Broken Object-Level Authorization in Notes API

- **Severity:** Medium
- **Affected file and line(s):** `src/server.js`, lines 41–47.
- **Affected route or feature:** `GET /api/notes/:id`.
- **Evidence:** The endpoint checks authentication at lines 42–43 and looks up a note by ID at line 45, but does not verify that the note belongs to the authenticated user. While logged in as Alex, I requested `/api/notes/202` and received the note belonging to the fictional user Sam.
- **Impact:** An authenticated user can retrieve another user's private note by supplying its ID.
- **Recommended mitigation:** Verify that the requested note belongs to the authenticated user before returning its contents, unless access has been explicitly authorized.

## Finding 4 — Unescaped User Input Rendered as HTML

- **Severity:** Medium
- **Affected file and line(s):** `src/web.js`, lines 10–13; `src/server.js`, lines 75–79 and 33–38.
- **Affected route or feature:** Search page and dashboard profile display.
- **Evidence:** The `page()` function at `src/web.js` lines 10–13 inserts the supplied title and body directly into an HTML document without escaping. The search handler at `src/server.js` lines 76–79 inserts the query into the HTML input value and results heading. The dashboard at lines 33–38 inserts the user's `displayName` into the page. I tested harmless `<b>...</b>` input in search and in the profile display name; the browser rendered the text in bold, confirming HTML injection. JavaScript execution was not tested.
- **Impact:** User-controlled markup can alter page content and presentation. Depending on the injection context, this may also enable cross-site scripting (XSS), which could allow scripts to run in another user's browser.
- **Recommended mitigation:** Contextually escape untrusted values before inserting them into HTML. Prefer safe templating or DOM text APIs, and avoid constructing HTML through direct string interpolation. Consider a Content Security Policy as an additional defense.

## Review notes

- **How I approached the review:** I reviewed the supplied source code, traced authentication, profile updates, note retrieval, and HTML rendering, then tested the application locally using the fictional accounts and data provided by the challenge. I used harmless input to verify the HTML-rendering behavior.
- **Assumptions or limitations:** Testing was limited to the local training application and its fictional data. Session-token forgery was identified through source review but was not dynamically tested. HTML injection was confirmed, but JavaScript execution was not tested. Severity ratings are preliminary and may depend on the challenge rubric and deployment context.
