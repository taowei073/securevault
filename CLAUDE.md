# SecureVault

A portfolio project for job applications. It shows two skill sets:
**Python test automation** and **hands-on application security**.

SecureVault is a small multi-user notes vault (FastAPI) with roles and sharing.
The app is the stage; the test framework and security testing are the showcase.

## Tech stack

- Python **3.13** (pinned; do not use 3.14), managed with **uv**
- FastAPI + Jinja2 templates (server-rendered HTML, no JS framework)
- SQLAlchemy 2.x + Alembic; SQLite locally, PostgreSQL in Docker
- argon2 password hashing, JWT access + refresh tokens, slowapi rate limits
- Tests: pytest, httpx, Playwright for Python (Page Object Model), faker, pytest-xdist
- Quality: ruff, mypy, pre-commit
- Security tooling: bandit, pip-audit, trivy, OWASP ZAP baseline
- CI: GitHub Actions; Docker + Docker Compose

## Scope (frozen, do not add features without asking)

- Register, log in, log out, refresh token
- CRUD on your own notes; share a note read-only with another user
- Admin role: list users, lock/unlock accounts, view audit log
- HTML pages: login, note list, note editor, admin panel
- Out of scope: file uploads, email, payments, JS frontend frameworks

## Security requirements

- Ownership check on every note endpoint; return 404 (not 403) for other users' notes
- Role checks on admin endpoints (403 for non-admins); admins cannot read note contents
- Account lockout after 5 failed logins, time-based (15 minutes, not permanent); rate limits on login
- Token storage: browser sessions use HttpOnly, Secure, SameSite=Strict cookies with a CSRF
  token on every form; API clients use `Authorization: Bearer` tokens
- Separate request schemas so clients cannot set `role` or `owner_id`
- Security headers (CSP, HSTS, X-Frame-Options, X-Content-Type-Options), strict CORS
- Generic error messages, no stack traces, same response for unknown user and wrong password
- Structured JSON audit log for logins, failures, admin actions, access denials
- Secrets only from environment variables; commit `.env.example`, never `.env`
- `INSECURE_MODE=true` disables selected controls for the security-test demo.
  It must never be the default, and the app must refuse to start with it unless `ENV=test`.

## Repository layout

```
app/        FastAPI app (auth/, notes/, admin/, templates/, config.py, models.py)
migrations/ Alembic
tests/      conftest.py, api/, ui/pages/, security/, support/
```

## How we work together

The owner is learning and must be able to explain every line in interviews.

- Work in **small steps**: one task per change. Propose a plan before writing code.
- After writing code, **explain what it does and why** in plain language,
  including any security reasoning.
- The owner writes the **key tests personally** (access-control and auth security tests).
  For those, give hints and review their code. Do not write them unless asked.
- Never commit or push without the owner's approval. Use clear, conventional commit
  messages (`feat:`, `fix:`, `test:`, `docs:`, `ci:`, `chore:`).
- Work on a feature branch per task and merge through pull requests.
- Never put real secrets in code, tests or commits.
- Prefer simple, readable code over clever code. Type hints everywhere.

## Commands

- Install deps: `uv sync`
- Run app: `uv run uvicorn app.main:app --reload`
- Run tests: `uv run pytest`
- Lint and type check: `uv run ruff check . && uv run mypy app`
