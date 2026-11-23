# Secure Patient Medical Record Management API (Projet 10)

IUC · BTECH 3 CSE · API Design and Integration. Built from the course Practical Pack (REST baseline, FastAPI).
**Synthetic data only. Educational, not production hardened.**

## Stack
React + Vite (5173) · FastAPI + SQLAlchemy Core (8000) · PostgreSQL 17 (Docker) · JWT · Argon2 · Fernet field encryption.

## Security design in one glance
- Roles: `patient` (self-registered), `doctor`, `admin`. Privileged roles are never self-assigned; an admin (or SQL for the first admin) sets them.
- A patient **grants** a doctor access; revoking removes access at once. Role is re-read from the DB on every request.
- Doctors read/write only records of patients who granted them access; only the **author** may update/delete.
- Admins manage roles and see the audit log but **cannot read clinical data**.
- Clinical summaries are **encrypted at rest** (Fernet). List endpoints never return summaries.
- Every read/write/denial/grant/role change is written to an **audit log** (identifiers only, no clinical text).
- Unauthorized access returns **404** (no existence leak); wrong role returns **403**.
- Login throttling (5 failures / 15 min / email), 30-minute tokens, `Cache-Control: no-store`, strict input validation.

## Setup (Ubuntu)
1. `docker compose up -d db`
2. `cd backend && python3 -m venv .venv && source .venv/bin/activate`
3. `python -m pip install -r requirements-dev.txt`
4. `cp .env.example .env` then set `JWT_SECRET` (`openssl rand -hex 32`) and
   `FIELD_ENCRYPTION_KEY` (`python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`). Never commit `.env`.
5. `uvicorn app.main:app --reload --host 127.0.0.1 --port 8000` → http://127.0.0.1:8000/docs
6. New terminal: `cd frontend && cp .env.example .env && npm install && npm run dev`
7. Tests: `cd backend && python -m pytest -q` (offline, SQLite, providers mocked).

## Create the first admin and a doctor
Register two accounts in the UI (or via curl), then promote them:
```bash
docker compose exec db psql -U medrec -d medrecords -c "UPDATE app_users SET role='admin' WHERE email='admin@example.edu';"
docker compose exec db psql -U medrec -d medrecords -c "UPDATE app_users SET role='doctor' WHERE email='doctor@example.edu';"
```
(After that, an admin can change roles through `PUT /api/admin/users/{id}/role`.)

## curl smoke test
```bash
curl -i http://127.0.0.1:8000/api/health
curl -X POST http://127.0.0.1:8000/api/auth/register -H 'Content-Type: application/json' \
  -d '{"fullName":"Pat Patient","email":"pat@example.edu","password":"strong-demo-password"}'
# copy data.accessToken into $TOKEN
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8000/api/records
curl -G -H "Authorization: Bearer $TOKEN" --data-urlencode 'term=bronchitis' http://127.0.0.1:8000/api/external/diagnoses
```

## Verification status (be honest in your report)
- Written and reviewed for this pack. In the authoring sandbox FastAPI could not be installed (no network), so **`pytest` was NOT executed there**.
- Done there: Python compile check of all backend and test files; every SQL statement shape run against real SQLite; JSX parse check of `App.jsx`.
- **You must run** `python -m pytest -q`, the manual tests in `docs/manual-test-cases.md`, and one real call to each provider, then fill `labs/acceptance_matrix.csv` with your evidence.

## Documents
`docs/api-contract.md` · `docs/manual-test-cases.md` · `docs/privacy-and-ethics.md` · `docs/integration-log-openfda.md` · `docs/integration-log-nlm-icd10.md` · `labs/acceptance_matrix.csv`

## Limitations
No HTTPS/reverse proxy, token revocation, MFA, email verification, distributed rate limiting, key rotation, DB migrations or deployment automation. Throttling is per-process memory. Not suitable for real patient data.
