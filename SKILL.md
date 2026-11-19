---
name: production-secure-deploy
description: Mandatory rules when writing, reviewing, or deploying code for production so it is secure and follows industry standards (OWASP Top 10, ASVS, 12-factor). Use for web apps and APIs (PHP, Laravel, Node, etc.), server configuration, file uploads, authentication, databases, and hosting deployment. Also use when the user mentions cPanel, shared hosting, File Manager, or having no terminal access.
---

# Production Secure Deploy

Reference standards: OWASP Top 10, OWASP ASVS level 1 as the minimum, 12-factor app for configuration. The rules below are mandatory unless the user explicitly states the project is a local-only prototype.

## Working rules and efficiency

1. Read only the parts relevant to the task. Load `references/cpanel-no-terminal.md` only when deploying to cPanel, shared hosting, or without terminal access.
2. Write the final code directly. Do not explain basic concepts unless asked.
3. If the stack, PHP/Node version, or data type (personal data, payments, health) is unknown and changes the recommendation, ask once at the start. If it does not change the recommendation, state the assumption in one line and continue.
4. When modifying existing code, send only the changed parts, not the whole file.
5. End with at most 5 remaining risks specific to this task. Do not output a generic checklist that does not apply.
6. Never claim something is "secure" or "100% safe". State what has been addressed and what has not been verified.
7. If the user's request violates the rules below (for example `APP_DEBUG=true` in production, chmod 777, hardcoded passwords), decline that part, explain the risk in one or two sentences, and provide a working alternative.

## Secrets and configuration

- Never hardcode secrets (DB passwords, API keys, APP_KEY, tokens) in code, HTML, frontend JS, or the repository. Use environment variables or `.env`.
- Keep `.env` and sensitive config outside the web root. If that is impossible, block access through server configuration.
- `.env.example` contains variable names only, with empty or fake values.
- Use different credentials for dev, staging, and production. Never reuse them.
- If a secret was ever committed or uploaded publicly, treat it as leaked and rotate it. Deleting it is not enough.
- Debug mode and stack traces must be off in production. Users see generic error messages, details go to logs.

## Input, output, and database

- All queries use prepared statements, parameter binding, or an ORM. String concatenation into SQL is forbidden, including for `ORDER BY` and column names (use an allowlist).
- Validate input on the server (type, length, format, allowlist). Frontend validation is for UX only.
- Escape output for its context (HTML, attribute, JS, URL). In Blade use `{{ }}`, not `{!! !!}`, for user data. In React avoid `dangerouslySetInnerHTML`.
- Mass assignment: define `$fillable` explicitly, never `$guarded = []`.
- The application database user has only the privileges it needs (SELECT/INSERT/UPDATE/DELETE on that database only), never root or global ALL PRIVILEGES.
- Redirect only to internal URLs or an allowlist (prevent open redirect). Restrict server-side requests to user-supplied URLs (prevent SSRF).

## Authentication and authorization

- Hash passwords with Argon2id or bcrypt. MD5, SHA1, plain SHA256, or reversible encryption are forbidden.
- Check authorization on the server for every resource request, including data ownership (prevent IDOR). Hiding a button in the UI is not authorization.
- Rate limit login, password reset, OTP, and expensive endpoints. Consider progressive lockout.
- Sessions: regenerate the ID after login, cookies with `Secure`, `HttpOnly`, `SameSite=Lax` or `Strict`, and a timeout.
- CSRF protection on every state-changing request. API tokens are stored hashed, revocable, and scoped.
- Login and password reset messages must not reveal whether an account exists.

## File uploads

- Validate extension and MIME type with an allowlist, limit size, and rename files with a random server-generated name.
- Store outside the web root, or make sure script execution is disabled in the upload folder. Never allow `.php`, `.phtml`, `.phar`, or `.htaccess` to be uploaded.
- Re-encode images where possible. Serve private downloads through a controller that checks authorization.

## Transport and headers

- HTTPS is mandatory, redirect HTTP to HTTPS. Enable HSTS only after HTTPS is verified stable.
- Minimum headers: `X-Content-Type-Options: nosniff`, `X-Frame-Options: SAMEORIGIN` (or CSP `frame-ancestors`), `Referrer-Policy`, `Permissions-Policy`. Build CSP for the specific application and test it. Do not copy a generic CSP that breaks pages or is too permissive (`unsafe-inline` plus `*`).
- CORS: list explicit origins. `*` together with credentials is forbidden.
- Disable directory listing. Block web access to dotfiles (`.env`, `.git`), backup files (`.sql`, `.bak`, `.zip`), and `composer.json/lock`, `package.json`.

## Dependencies and build

- Production contains production dependencies only (`composer install --no-dev --optimize-autoloader`, `npm ci` then build).
- Pin versions with lockfiles. Run `composer audit` / `npm audit` before release and record ignored findings with a reason.
- Do not load scripts from a CDN without a clear need. If required, pin the version and use SRI.
- Never include `.git`, `tests`, `node_modules`, internal documentation, or database dumps in the uploaded package.

## Logging, backup, and operations

- Log security events (failed logins, permission changes, 5xx errors). Never log passwords, tokens, card numbers, or full personal data.
- Schedule backups of the database and files, store them off the same server, and test a restore at least once. A backup that has never been tested counts as no backup.
- Prepare a rollback plan before deploying: keep a copy of the previous version.
- Apply least privilege to hosting, FTP, and database accounts. Enable 2FA on hosting and domain registrar accounts.
- Personal data: collect the minimum, and mention the Indonesian PDP Law (UU PDP) implications when relevant to the data processed.

## Pre-deploy gate

Do not declare the project ready to deploy until all of these are answered:

1. Debug is off, and no secrets exist in the code or the uploaded package.
2. Every endpoint that changes data has authentication, authorization, and CSRF protection.
3. No query or output is vulnerable to injection.
4. HTTPS is active and sensitive files cannot be reached by URL.
5. Backup and rollback are available.

## Limits to acknowledge

This skill reduces common mistakes but does not replace human code review, security testing, or an independent audit. For applications handling payments, health data, or large volumes of personal data, state that shared hosting has limited isolation and control, and recommend a VPS or managed platform.
