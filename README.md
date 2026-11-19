# production-secure-deploy

Production security and deployment rules for AI-generated code. Enforces OWASP-aligned practices for secrets, input validation, authentication, uploads, and headers, and includes a step-by-step cPanel shared hosting guide for deploying without a terminal, using only File Manager uploads.

## Contents

```
production-secure-deploy/
  SKILL.md
  references/
    cpanel-no-terminal.md
```

| File | Purpose |
| --- | --- |
| `SKILL.md` | Core rules Claude must follow when writing, reviewing, or deploying production code. Kept short on purpose to save tokens. |
| `references/cpanel-no-terminal.md` | Deployment guide for cPanel shared hosting without terminal access. Loaded only when the task involves cPanel, shared hosting, or File Manager uploads. |

## What it enforces

- Secrets and configuration: no hardcoded credentials, `.env` outside the web root, debug mode off in production, rotation after any leak.
- Input and output: prepared statements, server-side validation, context-aware output escaping, explicit mass assignment rules.
- Authentication and authorization: Argon2id or bcrypt, server-side ownership checks (IDOR), rate limiting, secure session cookies, CSRF protection.
- File uploads: allowlists, random server-side names, no script execution in upload folders.
- Transport and headers: HTTPS, HSTS after verification, baseline security headers, strict CORS, blocked dotfiles and backup files.
- Dependencies and operations: production-only dependencies, lockfiles, audit before release, tested backups, rollback plan.
- A pre-deploy gate of five checks before anything is declared ready.

## Efficiency rules

- Only the relevant section is applied to each task.
- Final code is written directly, without teaching basics unless asked.
- Changes to existing code are sent as diffs, not full files.
- Each answer ends with at most five risks specific to the task, not a generic checklist.
- Claude never claims a system is "100% secure" and states what remains unverified.

## Installation

1. Download `production-secure-deploy.zip`.
2. Open the Skills settings in Claude and upload the zip.
3. Enable the skill.

The skill triggers on requests about production readiness, security reviews, deployment, and any mention of cPanel, shared hosting, or deploying without a terminal.

## cPanel deployment overview

Assumes a PHP/Laravel application where build steps run locally and the server only receives the finished result.

1. Build locally: match the host PHP version, `composer install --no-dev --optimize-autoloader`, build frontend assets, create a zip without `.env`, `.git`, and `node_modules`.
2. Upload the zip through File Manager and extract it. Keep application code outside `public_html`, and place only the contents of `public/` inside it.
3. Create the database with the MySQL wizard and import the schema through phpMyAdmin. Do not import development data or default seeders.
4. Create a fresh `.env` on the server with `APP_DEBUG=false`, a new `APP_KEY`, and permission 640.
5. Run one-off artisan commands and `storage:link` through cPanel Cron Jobs, then delete those jobs.
6. Enable AutoSSL, force HTTPS, apply the `.htaccess` hardening rules, and verify that `/.env` and `/storage/logs` are not reachable.

Full details, including update and rollback steps, are in `references/cpanel-no-terminal.md`.

## Limitations

- The skill reduces common mistakes. It does not replace human code review, penetration testing, or an independent audit.
- Business logic flaws specific to your application can still be missed.
- cPanel details differ per host (PHP binary path, document root changes, symlink support). Test on a staging subdomain first.
- Shared hosting offers limited isolation and control. For payment data or large volumes of personal data, a VPS or managed platform is the more defensible choice.
- Cron-based workarounds are not a substitute for a real queue worker or CI pipeline.

## Customizing

Edit `SKILL.md` to add project-specific rules, such as your framework conventions or compliance requirements (for example Indonesian PDP law obligations). Keep the file short. Move long, situational material into `references/` so it is only loaded when needed.

