# KAAO Analytics API

A read-only REST API, built with FastAPI, that for fleet analytics shown at **kaao.co.ke/analytics**. The source data is an Excel workbook on OneDrive. The API validates that data, keeps a clean snapshot, and serves aggregated results to the analytics page.

> **About this README.** It describes the target structure of the project. Commands that refer to this project's own modules (for example `app.sync`) assume the layout in [Section 4](#4-project-structure). If you change the layout, update the commands to match. Placeholders appear in angle brackets, such as `<your-username>`.

---

## Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Tech stack](#3-tech-stack)
4. [Project structure](#4-project-structure)
5. [Prerequisites](#5-prerequisites)
6. [Quick start (local)](#6-quick-start-local)
7. [Configuration](#7-configuration)
8. [Running locally](#8-running-locally)
9. [Connecting to the OneDrive workbook](#9-connecting-to-the-onedrive-workbook)
10. [Testing and code quality](#10-testing-and-code-quality)
11. [API overview](#11-api-overview)
12. [Deploying to FastAPI Cloud](#12-deploying-to-fastapi-cloud)
13. [Scheduled data sync in production](#13-scheduled-data-sync-in-production)
14. [Operations](#14-operations)
15. [Version control workflow](#15-version-control-workflow)
16. [Security notes](#16-security-notes)
17. [Troubleshooting](#17-troubleshooting)
18. [Command cheat sheet](#18-command-cheat-sheet)

---

## 1. Overview

**Purpose.** Provide the data behind the KAAO analytics page: fleet counts by category, propulsion, manufacturer and operator segment, a Cessna drill-down, member fleets, and (if approved) estimated asset values. Wilson Airport flight movements are planned for a later release.

**Consumer.** The analytics page at kaao.co.ke/analytics only. This is not a general public API.

**Principles.**

- **Read-only.** Every data endpoint is `GET`. There are no write endpoints and no manual refresh endpoint.
- **Aggregated by default.** Sensitive groups (individual registrations, estimated values) are switched off until KAAO approves them.
- **Never serve raw spreadsheet data directly.** The workbook is edited by people, so it is validated on every sync. If validation fails, the API keeps serving the last good snapshot.

The full list of endpoints is in [`endpoints.md`](endpoints.md).

---

## 2. Architecture

```
  OneDrive workbook (edited by KAAO)
            |
            |  Microsoft Graph (or a manual file, see Section 9)
            v
  Sync job: read -> validate -> normalize -> classify -> store
            |
            v
  Snapshot store (SQLite locally, PostgreSQL in production)
            |
            v
  FastAPI service (read-only, /v1/...)
            |
            v
  kaao.co.ke/analytics (browser)
```

**Why a snapshot instead of reading OneDrive on every request?**

- Page views never depend on Microsoft's latency, rate limits or availability.
- A broken edit in the workbook (a renamed column, a typo) cannot take the page down. The sync fails loudly and the previous snapshot stays live.
- Counts, classifications and estimates are computed once per sync, not on every request.

**Cold starts.** Managed platforms may stop idle apps and start them again on demand. The API therefore reads from the snapshot store instead of holding the only copy of the data in memory, and keeps its startup work small.

---

## 3. Tech stack

| Area | Choice |
|---|---|
| Language | Python 3.12 or newer |
| Web framework | FastAPI, served by the FastAPI CLI (`fastapi dev`, `fastapi run`) |
| Validation and settings | Pydantic and pydantic-settings |
| Data access | SQLAlchemy 2.0 with Alembic migrations |
| Snapshot store | SQLite (local), PostgreSQL (production) |
| Workbook access | Microsoft Graph, authenticated with MSAL |
| Package manager | uv |
| Quality | pytest, Ruff, mypy, pre-commit |
| Hosting | FastAPI Cloud |
| CI | GitHub Actions |

---

## 4. Project structure

```
kaao-analytics-api/
├── README.md
├── endpoints.md                  Endpoint reference (the API contract in prose)
├── pyproject.toml                Dependencies, tool settings, FastAPI entrypoint
├── uv.lock                       Locked dependency versions (commit this)
├── .env.example                  Template for local settings (commit this)
├── .env                          Your real local settings (never commit)
├── .gitignore
├── .fastapicloudignore           Files left out of deployment uploads
├── .pre-commit-config.yaml
├── openapi/
│   └── openapi.yaml              The contract, reviewed with the front-end
├── docs/
│   ├── brief.md                  Scope brief
│   ├── data-dictionary.md
│   ├── data-contract.md          Rules the workbook must satisfy
│   ├── conventions.md
│   └── adr/                      Short decision records (sync pattern, auth, hosting)
├── app/
│   ├── main.py                   App creation, middleware, router wiring
│   ├── config.py                 Settings read from environment variables
│   ├── api/v1/                   One router per endpoint group
│   ├── schemas/                  Response models
│   ├── services/                 Business logic (breakdowns, drill-downs)
│   ├── repositories/             Reads from the snapshot store
│   ├── core/                     Errors, caching headers, rate limiting, logging
│   ├── reference/                Aircraft-type lookup, segment map, value ranges
│   └── sync/                     Graph client, workbook reader, validation, loader
├── migrations/                   Alembic migrations
├── tests/
│   ├── unit/
│   ├── integration/
│   └── contract/                 Checks responses against the OpenAPI file
├── data/
│   └── sample/                   Small anonymized sample data for local work and tests
└── .github/
    ├── workflows/                ci, sync, deploy
    └── PULL_REQUEST_TEMPLATE.md
```

---

## 5. Prerequisites

| Requirement | Notes |
|---|---|
| Python 3.12 or newer | Check with `python --version`. Confirm the version FastAPI Cloud supports before you deploy. |
| uv | Installs Python packages and runs commands. See below. |
| Git | For version control and deployment from a clean tree. |
| A GitHub account | Repository, CI and the scheduled sync. |
| A FastAPI Cloud account | Needed only for deployment. |
| Access to the workbook | Needed only to test against real data. Local work uses sample data. |

**Install uv**

- macOS or Linux: `curl -LsSf https://astral.sh/uv/install.sh | sh`
- Windows (PowerShell): `powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"`
- Any system with pip: `pip install uv`

Check with `uv --version`.

---

## 6. Quick start (local)

No OneDrive access is needed for this. The API runs on bundled sample data.

```bash
# 1. Get the code
git clone https://github.com/<your-username>/kaao-analytics-api.git
cd kaao-analytics-api

# 2. Install dependencies into a project-local virtual environment
uv sync

# 3. Create your local settings file
cp .env.example .env          # Windows (cmd): copy .env.example .env

# 4. Load the sample data into the local snapshot store
uv run python -m app.sync --source sample

# 5. Start the development server (auto-reloads on code changes)
uv run fastapi dev
```

Then open:

- `http://127.0.0.1:8000/v1/health` should return a status of `ok`.
- `http://127.0.0.1:8000/v1/summary` should return the headline KPIs.
- `http://127.0.0.1:8000/docs` shows interactive documentation (enabled locally only).

---

## 7. Configuration

All settings come from environment variables. Locally they are read from `.env`. In production they are set on FastAPI Cloud (see [Section 12](#12-deploying-to-fastapi-cloud)). Never commit `.env`.

| Variable | Example | Purpose |
|---|---|---|
| `ENVIRONMENT` | `local`, `staging`, `production` | Controls defaults such as docs and log format |
| `LOG_LEVEL` | `INFO` | Logging verbosity |
| `ALLOWED_ORIGINS` | `https://kaao.co.ke` | Comma-separated browser origins allowed by CORS. Exact scheme and host, no trailing slash. |
| `DATABASE_URL` | `sqlite:///./data/local.db` | Snapshot store. A PostgreSQL URL in production. **Secret in production.** |
| `DATA_SOURCE` | `sample`, `graph`, `file` | Where the sync reads the workbook from |
| `SAMPLE_DATA_PATH` | `data/sample/` | Used when `DATA_SOURCE` is `sample` or `file` |
| `AZURE_TENANT_ID` | a GUID | Microsoft Entra tenant (Graph only) |
| `AZURE_CLIENT_ID` | a GUID | App registration ID (Graph only) |
| `AZURE_CLIENT_SECRET` | | App secret (Graph only). **Secret.** |
| `GRAPH_SITE_ID` | | SharePoint site containing the workbook (Graph only) |
| `GRAPH_DRIVE_ID` | | Document library ID (Graph only) |
| `GRAPH_ITEM_ID` | | The workbook's item ID (Graph only) |
| `SNAPSHOT_MAX_AGE_HOURS` | `48` | Age after which data is reported as stale and then refused (`503`) |
| `CACHE_MAX_AGE_SECONDS` | `300` | `Cache-Control` max-age on data endpoints |
| `RATE_LIMIT_PER_MINUTE` | `120` | Per-IP request limit |
| `ENABLE_DOCS` | `false` | Serves `/docs` and `/openapi.json`. Keep `false` in production. |
| `ENABLE_AIRCRAFT_ENDPOINTS` | `false` | Gated: individual aircraft and registrations |
| `ENABLE_VALUATIONS` | `false` | Gated: estimated valuations summary and breakdowns |
| `ENABLE_VALUATIONS_BY_MEMBER` | `false` | Gated: per-member estimated values (most sensitive) |
| `ENABLE_MOVEMENTS` | `false` | Wilson Airport movements (later release) |

`.env.example` lists every variable with safe local defaults and no real secrets.

---

## 8. Running locally

### Development server

```bash
uv run fastapi dev            # auto-reload, http://127.0.0.1:8000
```

To check how the app behaves outside development mode, use:

```bash
uv run fastapi run            # production-style server, no auto-reload
```

If the CLI cannot find the app, set the entrypoint in `pyproject.toml`:

```toml
[tool.fastapi]
entrypoint = "app.main:app"
```

### Trying the endpoints

```bash
curl http://127.0.0.1:8000/v1/health
curl http://127.0.0.1:8000/v1/meta
curl http://127.0.0.1:8000/v1/summary
curl "http://127.0.0.1:8000/v1/fleet/breakdown/category"
curl "http://127.0.0.1:8000/v1/fleet/breakdown/manufacturer?limit=5&include_other=true"
curl "http://127.0.0.1:8000/v1/fleet/cross-tab?rows=segment&columns=category"
curl "http://127.0.0.1:8000/v1/members?sort=-aircraft_count&limit=10"
curl http://127.0.0.1:8000/v1/fleet/manufacturers/cessna
```

Gated endpoints return `404` until their `ENABLE_...` setting is `true`.

### Loading data

| Goal | Command |
|---|---|
| Use the bundled sample data | `uv run python -m app.sync --source sample` |
| Use a workbook file you downloaded | Set `DATA_SOURCE=file` and `SAMPLE_DATA_PATH`, then `uv run python -m app.sync --source file` |
| Pull from OneDrive through Graph | Complete [Section 9](#9-connecting-to-the-onedrive-workbook), then `uv run python -m app.sync --source graph` |
| Validate only, without storing | Add `--dry-run` to any of the above |

The sync prints a data-quality report (duplicates, unmapped types, invalid registrations, blank counts). If validation fails, nothing is stored and the exit code is non-zero.

### Testing from the web page

If you serve a local copy of the analytics page, add its origin to `ALLOWED_ORIGINS`, for example `http://localhost:5500`, or the browser will block the calls.

---

## 9. Connecting to the OneDrive workbook

### 9.1 First, find out which kind of OneDrive it is

- A URL like `https://<tenant>-my.sharepoint.com/...` or `https://<tenant>.sharepoint.com/...` means **OneDrive for Business or SharePoint**. The Microsoft Graph Excel features are supported there.
- A URL on `onedrive.live.com` or `1drv.ms` means a **personal** account. Microsoft does not officially support the Graph workbook endpoints there, and app-only access is not available.

**If it is a personal account**, choose one of these:

1. Move the workbook to a SharePoint site or a business OneDrive (recommended).
2. Use `DATA_SOURCE=file`: download the workbook, drop it in the configured folder, and run the sync manually.

### 9.2 Prepare the workbook

- Put each data set in an Excel **Table** (Insert, then Table) with a fixed header name.
- Use one sheet per entity: members, fleet, aircraft types.
- One header row, no merged cells, no notes typed into data cells.
- Give every member a permanent ID that never changes, even if the name is edited.
- Follow the rules in `docs/data-contract.md`. Anyone who edits the workbook should read `docs/data-dictionary.md` first.

### 9.3 Register the app in Microsoft Entra ID (business or SharePoint)

You need a tenant administrator for steps 4 and 5.

1. In the Microsoft Entra admin center, open **App registrations** and choose **New registration**. Use a single-tenant app.
2. Copy the **Application (client) ID** and **Directory (tenant) ID** into `AZURE_CLIENT_ID` and `AZURE_TENANT_ID`.
3. Under **Certificates and secrets**, create a client secret. Copy the value immediately (it is shown once) and **record its expiry date**. Put a reminder in your calendar well before it expires.
4. Under **API permissions**, add the Microsoft Graph **application** permission `Sites.Selected` and grant admin consent. On its own this permission gives access to nothing.
5. Have an administrator grant the app **read** access to the one SharePoint site that holds the workbook. This is a separate step and is what actually unlocks the site.
6. Use Microsoft Graph Explorer to find the site ID, document library (drive) ID and the workbook's item ID. Put them in `GRAPH_SITE_ID`, `GRAPH_DRIVE_ID` and `GRAPH_ITEM_ID`.
7. Run `uv run python -m app.sync --source graph --dry-run` to confirm access and validation.

**Rules to keep.** Request only the permissions above. Use a separate app registration (and secret) for each environment. Never put the secret in the repository.

---

## 10. Testing and code quality

```bash
uv run pytest                          # all tests
uv run pytest tests/unit               # fast unit tests only
uv run pytest --cov=app                # with coverage
uv run ruff check .                    # lint
uv run ruff format .                   # format
uv run mypy app                        # type check
uv run pre-commit install              # run checks automatically before each commit
```

| Test layer | What it covers |
|---|---|
| Unit | Classification rules, normalization, validation rules, count and share calculations |
| Integration | Endpoints against a throwaway database loaded with the sample data |
| Contract | Every response matches `openapi/openapi.yaml`, and breaking changes to the contract are flagged |

CI runs the same checks on every pull request. A pull request cannot be merged while any check fails.

---

## 11. API overview

- **Base path:** `/v1`
- **Methods:** `GET`, `HEAD`, `OPTIONS` only
- **Format:** JSON, `snake_case` field names
- **Errors:** RFC 9457 problem details (`application/problem+json`), each with a `request_id`
- **Caching:** `ETag` and `Cache-Control` on data endpoints
- **Reference:** [`endpoints.md`](endpoints.md) lists every endpoint, parameter and response field

| Status | Meaning |
|---|---|
| 200 | Success |
| 304 | Not modified (matching `ETag`) |
| 404 | Unknown ID, or a gated endpoint that is switched off |
| 422 | Invalid or unknown parameter or value |
| 429 | Rate limit exceeded |
| 503 | No valid snapshot yet, or the snapshot is too old to serve |

---

## 12. Deploying to FastAPI Cloud

FastAPI Cloud is the hosting platform from the team behind FastAPI. It was in **public beta** when this README was written, so features and pricing can change. Before you commit, check the current pricing and limits, and check whether **custom domains**, **scheduled jobs** and **background workers** are available on your plan.

### 12.1 What you need first

- A FastAPI Cloud account.
- `fastapi[standard]` installed in the project. It includes the FastAPI Cloud CLI, which provides the `login` and `deploy` commands. (`uv sync` already installs it if it is listed in `pyproject.toml`.)
- A clean `main` branch with passing CI.
- A managed PostgreSQL database for the snapshot store (see 12.4).
- The values for every production variable in [Section 7](#7-configuration).

### 12.2 First deployment

```bash
uv run fastapi login          # opens your browser to authorize the CLI
uv run fastapi deploy         # packages, uploads and deploys the app
```

On the first deploy the CLI asks for an app name and the location of `pyproject.toml`, creates the app, and prints its address, which looks like `https://<app-name>.fastapicloud.dev`. It stores the link between this folder and the app in a `.fastapicloud` directory. Add `.fastapicloud/` to `.gitignore`. CI passes the app ID explicitly (see 12.7).

The platform provides HTTPS automatically.

### 12.3 Set production environment variables

Set these before the first real traffic. Use `--secret` for anything sensitive.

```bash
uv run fastapi cloud env set ENVIRONMENT production
uv run fastapi cloud env set LOG_LEVEL INFO
uv run fastapi cloud env set ALLOWED_ORIGINS https://kaao.co.ke
uv run fastapi cloud env set ENABLE_DOCS false
uv run fastapi cloud env set SNAPSHOT_MAX_AGE_HOURS 48
uv run fastapi cloud env set --secret DATABASE_URL "<postgres-connection-string>"
```

Leave all the `ENABLE_...` gated switches at `false` until KAAO approves them. The Graph credentials (`AZURE_...`, `GRAPH_...`) are **not** needed on the web app if the sync runs elsewhere (see [Section 13](#13-scheduled-data-sync-in-production)). Keep them only where the sync runs.

Useful commands: `uv run fastapi cloud env list` and `uv run fastapi cloud env delete <NAME>`. You can also manage variables in the dashboard. Redeploy after changing variables so the running app picks them up.

### 12.4 Database for the snapshot store

The platform lists integrations with managed database providers (Neon, Redis Cloud and Supabase were listed when this was written). Confirm what is currently offered in the dashboard. Create a PostgreSQL database, copy its connection string into the `DATABASE_URL` secret, and apply the schema migrations once:

```bash
DATABASE_URL="<postgres-connection-string>" uv run alembic upgrade head
```

Run migrations as a deliberate step, never automatically at app startup. Make each migration backward compatible so the previous release keeps working while a new one rolls out.

### 12.5 Keep the upload small

Create a `.fastapicloudignore` file to leave out things the running app does not need: `tests/`, `docs/`, `data/sample/`, `.github/`, local databases and caches. The deploy command warns about files above 10 MB by default.

### 12.6 Verify the deployment

```bash
curl https://<app-name>.fastapicloud.dev/v1/health
curl https://<app-name>.fastapicloud.dev/v1/health/ready
curl https://<app-name>.fastapicloud.dev/v1/meta
uv run fastapi cloud logs              # recent or streaming logs
```

`/v1/health/ready` returns `503` until the first sync has stored a snapshot. Run the sync once (see [Section 13](#13-scheduled-data-sync-in-production)), then check that `/v1/meta` shows a recent `data_as_of`.

### 12.7 Staging, production and CI deploys

Use **two separate apps**: one for staging and one for production, each with its own variables, database and Graph credentials.

- Link a folder to an existing app with `uv run fastapi cloud link <APP_ID>`.
- Deploy to a specific app with `uv run fastapi deploy --app-id <APP_ID>`, or set the `FASTAPI_CLOUD_APP_ID` environment variable. CI uses this.
- For automatic deploys, either use the platform's GitHub integration (link the repository in the dashboard so pushes deploy) or set up a GitHub Actions workflow. Run `uv run fastapi cloud --help` to find the CI setup command in your CLI version (some versions call it `setup-ci`, others `ci setup`). The workflow needs a deploy token stored as a GitHub secret, never in the repository.

**Suggested flow:** merge to `main` deploys to staging. A version tag (`v1.2.0`) deploys to production after the staging checks pass.

### 12.8 Custom domain (api.kaao.co.ke)

1. In the FastAPI Cloud dashboard, open the app and add the custom domain `api.kaao.co.ke`. If you do not see the option on your plan, use the `fastapicloud.dev` address for now and ask support when it will be available.
2. The dashboard shows the DNS record(s) to create. Add them at whichever service hosts the DNS for kaao.co.ke. That may be the web host's DNS zone or a separate DNS provider, so check where the nameservers point.
3. Wait for DNS to propagate. The platform issues the HTTPS certificate once the record resolves.
4. Update `ALLOWED_ORIGINS` only if the page's own address changes. The API's address does not belong in CORS.
5. Point the analytics page at `https://api.kaao.co.ke/v1`.

If the browser shows `DNS_PROBE_FINISHED_NXDOMAIN`, the DNS record is missing, wrong or not yet propagated.

### 12.9 Updating and rolling back

- **Update:** merge through a pull request, let CI pass, tag a release, deploy.
- **Roll back:** check out the last good tag and deploy it again. If a migration was involved, it must have been backward compatible, so the older release still works with the newer schema.
- **List deployments:** `uv run fastapi cloud deployments --help` shows what your CLI version offers.

### 12.10 Pre-launch checklist

- [ ] CI is green on the commit being deployed
- [ ] `ENVIRONMENT=production`, `ENABLE_DOCS=false`, all gated switches `false` unless approved
- [ ] `ALLOWED_ORIGINS` is exactly `https://kaao.co.ke`
- [ ] `DATABASE_URL` is set as a secret, and migrations are applied
- [ ] First sync completed, and `/v1/meta` shows fresh data
- [ ] `/v1/health` and `/v1/health/ready` return 200 on the deployed address
- [ ] The analytics page loads data from the deployed API without CORS errors
- [ ] The scheduled sync is running and a failed run alerts someone
- [ ] The Graph secret's expiry date is in a calendar
- [ ] Load test run at about twice the expected peak

---

## 13. Scheduled data sync in production

The API itself only reads. Something else must run the sync on a schedule: read the workbook, validate, and write the snapshot.

When this was written, scheduled jobs and background workers were listed by FastAPI Cloud as upcoming, not available. Check the current documentation first. If they are still not available, run the sync outside the web app:

**Recommended: a scheduled GitHub Actions workflow.**

- Runs once or twice a day (match how often KAAO edits the workbook) and can also be started manually.
- Stores the Graph credentials and `DATABASE_URL` as **GitHub secrets**, so the web app never holds the Graph secret.
- Runs `uv run python -m app.sync --source graph`.
- On a validation or access failure, the run fails, nothing is stored, the old snapshot stays live, and GitHub notifies you.
- Check the run history regularly. Scheduled runs can be delayed, and `/v1/meta` shows when the data was last refreshed.

**Alternatives:** a platform cron job, once FastAPI Cloud offers one; or a small scheduler on a separate service. Avoid running the schedule inside the web app, because an app that is stopped when idle will miss runs.

**Staleness.** `SNAPSHOT_MAX_AGE_HOURS` defines when `/v1/meta` reports `stale` and when the API refuses to serve (`503`). Set it to a few multiples of the sync interval.

---

## 14. Operations

| Area | Practice |
|---|---|
| Health checks | `/v1/health` (liveness) and `/v1/health/ready` (readiness). Monitor both from an external uptime service. |
| Data freshness | Alert when `/v1/meta` reports `stale`, or when the sync workflow fails. |
| Logs | Structured logs with a `request_id` on every line. View with `uv run fastapi cloud logs`. |
| Errors | The platform's logs plus an error tracker such as Sentry, if you add one. |
| Secrets | Rotate the Graph client secret before it expires. Rotating means creating a new secret, updating GitHub secrets, running a manual sync, then deleting the old secret. |
| Domain and HTTPS | Monitor the certificate expiry of `api.kaao.co.ke` and the DNS record. |
| Backups | Enable automated backups on the PostgreSQL provider, and test a restore. The snapshot can also be rebuilt by re-running the sync. |
| Dependencies | Dependabot opens update pull requests. Review and merge them weekly. |

---

## 15. Version control workflow

- **Branches:** `main` is always deployable. Work happens on short-lived branches named `feature/<topic>` or `fix/<topic>`.
- **Pull requests:** every change goes through a pull request. `main` is protected: reviews and passing CI are required, and force-pushes are blocked.
- **Commits:** use Conventional Commits, for example `feat: add cross-tab endpoint`, `fix: handle empty fleet`, `docs: update README`.
- **Releases:** tag with semantic versions (`v1.0.0`). The URL stays `/v1` for compatible changes. A breaking change to the contract means a new `/v2`.
- **Decisions:** record important choices (sync pattern, auth model, hosting) as short files in `docs/adr/`.
- **Never commit:** `.env`, secrets, the real workbook, snapshot database files, or `.fastapicloud/`. The `data/sample/` folder holds anonymized sample data only.
- **Secret scanning:** pre-commit runs gitleaks, and GitHub push protection is turned on.
- **Contract:** `openapi/openapi.yaml` lives in the repository. CI lints it and fails pull requests that introduce breaking changes.

---

## 16. Security notes

- The API is read-only, with no write endpoints.
- A browser page cannot keep a secret, so no key may be placed in the page's JavaScript. If some data must not be public, put a server-side proxy in front of the API and have it add a key. The endpoints do not change.
- CORS restricts browsers only. It does not stop other clients, so the rate limit and the aggregated-only data rules are the real protection.
- Gated endpoints return `404` when off, which avoids revealing that they exist.
- Error responses never include stack traces, file paths or workbook contents.
- Production runs with docs disabled, because the contract is kept in the repository.
- The Graph app has read access to one site only. Keep it that way.

---

## 17. Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `fastapi: command not found` | Dependencies are not installed or the environment is not active. Run `uv sync`, then use `uv run fastapi ...`. |
| The CLI cannot find the app | Set `entrypoint` under `[tool.fastapi]` in `pyproject.toml` (see Section 8). |
| `/v1/health/ready` returns 503 | No snapshot is stored yet or it is too old. Run the sync and check `/v1/meta`. |
| Browser shows a CORS error | `ALLOWED_ORIGINS` must match the page's origin exactly (scheme and host, no trailing slash). Redeploy after changing it. |
| A request returns 422 | An unknown parameter or value. The error lists the allowed values. |
| A gated endpoint returns 404 | It is switched off. Set the matching `ENABLE_...` variable only after KAAO approves. |
| Sync fails with 401 or 403 from Graph | Admin consent is missing, the site-level grant was never applied, or the secret expired. Re-check Section 9.3. |
| Sync fails on validation | The workbook broke the data contract. Read the data-quality report, fix the workbook, and run the sync again. |
| Data on the page is out of date | The scheduled sync did not run or failed. Check the workflow history and `/v1/meta`. |
| Deployment fails | Check the CLI output and `uv run fastapi cloud logs`. Common causes are a missing dependency, a wrong entrypoint, or a very large upload. |
| `DNS_PROBE_FINISHED_NXDOMAIN` on the custom domain | The DNS record is missing or has not propagated. Check it at the DNS host. |
| The first request after idle is slow | The app was stopped and is starting again. Keep startup light, and check your plan's idle behavior. |

---

## 18. Command cheat sheet

| Task | Command |
|---|---|
| Install dependencies | `uv sync` |
| Run the dev server | `uv run fastapi dev` |
| Run production-style | `uv run fastapi run` |
| Load sample data | `uv run python -m app.sync --source sample` |
| Sync from OneDrive | `uv run python -m app.sync --source graph` |
| Validate only | `uv run python -m app.sync --source graph --dry-run` |
| Run tests | `uv run pytest` |
| Lint and format | `uv run ruff check .` and `uv run ruff format .` |
| Type check | `uv run mypy app` |
| Apply migrations | `uv run alembic upgrade head` |
| Log in to FastAPI Cloud | `uv run fastapi login` |
| Deploy | `uv run fastapi deploy` |
| Deploy to a specific app | `uv run fastapi deploy --app-id <APP_ID>` |
| Link a folder to an app | `uv run fastapi cloud link <APP_ID>` |
| Set a variable | `uv run fastapi cloud env set <NAME> <value>` |
| Set a secret | `uv run fastapi cloud env set --secret <NAME> "<value>"` |
| List variables | `uv run fastapi cloud env list` |
| View logs | `uv run fastapi cloud logs` |
| Check who you are logged in as | `uv run fastapi cloud whoami` |