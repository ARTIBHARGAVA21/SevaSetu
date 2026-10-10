# Sewa Setu Assistant

Sewa Setu Assistant connects conversational AI assistants to the Sewa Setu Old Age Pension portal through two role-separated Model Context Protocol (MCP) servers. It provides citizen-facing tools for scheme information and application tasks, and officer-facing tools for reviewing applications. The portal remains the system of record: application validation and state changes are performed by the existing portal rather than duplicated in the assistant layer.

## Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Technology stack](#technology-stack)
- [Repository structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Run with Docker Compose](#run-with-docker-compose)
- [Run locally without Docker](#run-locally-without-docker)
- [Application URLs](#application-urls)
- [MCP endpoints and tools](#mcp-endpoints-and-tools)
- [OAuth 2.1 authentication flow](#oauth-21-authentication-flow)
- [Evaluation framework](#evaluation-framework)
- [Configuration](#configuration)
- [Database and persistence](#database-and-persistence)
- [Security notes](#security-notes)
- [Troubleshooting](#troubleshooting)

## Overview

The project exposes two separate MCP endpoints over Streamable HTTP:

| Endpoint | Intended user | Purpose |
|---|---|---|
| `/mcp/citizen` | Citizen/applicant | Scheme information, eligibility guidance, application submission, application tracking and withdrawal |
| `/mcp/officer` | Departmental officer | Queue statistics, application listing and inspection, and application approval or rejection |

The endpoints expose different tool sets and enforce role- and scope-based access. A citizen token must not be accepted for officer operations. Requests are authorized using OAuth access tokens and the requested resource audience.

> **Important:** This repository is an assessment/development project. Review configuration, access controls, logging, and deployment settings before using it with real citizen data.

## Architecture

```text
Citizen assistant / Officer assistant
                 |
                 | OAuth 2.1 access token
                 v
        Sewa Setu Flask portal
          /               \
         v                 v
  Citizen MCP          Officer MCP
  /mcp/citizen          /mcp/officer
         \                 /
          \               /
           Existing portal business logic
                      |
                      v
                PostgreSQL database

Portal ----> Simulated SMS gateway (development OTP delivery)
Portal ----> Evaluation dashboard (/eval-dashboard)
```

The portal performs the authoritative validation and writes application changes to its database. The MCP layer is an interface to existing portal capabilities, not a separate source of pension eligibility rules.

## Features

- **Separate citizen and officer MCP servers:** each endpoint advertises only its own tools.
- **Citizen workflows:** scheme information, eligibility checks, application submission, status tracking and withdrawal.
- **Officer workflows:** queue statistics, application listing and inspection, approval and rejection with a reason.
- **OAuth 2.1 authorization:** authorization-code flow with PKCE S256, resource-specific access tokens and token discovery endpoints.
- **JWT validation:** signed access tokens, issuer and audience checks, plus scope, role and subject validation.
- **Client registration:** dynamic client registration and support for client metadata URLs.
- **Program tokens:** administrator-issued tokens for approved unattended officer workflows.
- **Auditability:** application changes are recorded through the portal's existing business logic and audit mechanisms.
- **Evaluation dashboard:** displays evaluation personas, conversations, checks and results.
- **Automated evaluation harness:** uses MCP clients and checks the resulting database state rather than relying only on conversation transcripts.
- **Docker Compose deployment:** portal, PostgreSQL, simulated SMS gateway and scheduled task run as separate services.
- **SQLite development mode:** run the complete stack locally without Docker.

## Technology stack

- Python and Flask
- Model Context Protocol (MCP), Streamable HTTP transport
- OAuth 2.1 concepts, PKCE, JWT (RS256) and JWKS
- PostgreSQL for the Docker Compose environment
- SQLite for the local development runner
- HTML templates for the portal and evaluation dashboard
- Docker and Docker Compose
- Simulated SMS gateway for development OTP delivery

Python dependencies are listed in `app/requirements.txt`.

## Repository structure

```text
sewasetu-assistant/
├── app/
│   ├── app.py                 # Flask portal and web routes
│   ├── mcp.py                 # Citizen and officer MCP servers/tools
│   ├── oauth.py               # OAuth authorization server and token handling
│   ├── eval.py                # Evaluation dashboard routes
│   ├── schema.sql             # Database schema
│   ├── requirements.txt       # Python dependencies
│   ├── Dockerfile
│   ├── config/app.ini         # Application configuration
│   ├── deploy/crontab         # Scheduled jobs
│   ├── scripts/               # Maintenance/scheduled scripts
│   └── templates/             # Portal, OAuth and admin HTML templates
├── evals/
│   ├── run_evals.py           # Evaluation runner
│   ├── assistant.py           # Assistant scenarios and conversation logic
│   ├── personas.py            # Test personas
│   ├── checks.py              # Assertions and checks
│   ├── mcp_client.py          # MCP client helper
│   ├── oauth_client.py        # OAuth test client helper
│   └── ...                    # Database and PDF utilities
├── smsgw/
│   ├── app.py                 # Simulated SMS gateway
│   └── Dockerfile
├── dev/
│   ├── run_local.py           # Local SQLite development runner
│   ├── shim_db.py             # Local database compatibility helpers
│   ├── db/                    # Local development database (created/used locally)
│   └── uploads/               # Local uploaded files
├── seed/
│   └── seed.sql               # Seed database snapshot
├── docs/handout/
│   ├── BRIEF.md
│   └── PORTAL-README.md
├── docker-compose.yml
└── README.md
```

## Prerequisites

### Docker setup

- Docker Engine or Docker Desktop
- Docker Compose v2 (`docker compose`)
- A terminal (PowerShell, Command Prompt, macOS Terminal or Linux shell)

### Local setup without Docker

- Python compatible with the project's dependencies
- `pip`

Use a virtual environment when installing dependencies locally. The Docker setup is the most representative way to run the full multi-service stack.

## Run with Docker Compose

Run these commands from the repository root (`sewasetu-assistant/`).

### 1. Configure environment variables

For a local development run, set an administrator password before starting the stack.

**PowerShell:**

```powershell
$env:ADMIN_USERNAME = "admin"
$env:ADMIN_PASSWORD = "replace-with-a-strong-local-password"
$env:PUBLIC_BASE_URL = "http://localhost:8000"
docker compose up --build -d
```

**macOS/Linux:**

```bash
export ADMIN_USERNAME="admin"
export ADMIN_PASSWORD="replace-with-a-strong-local-password"
export PUBLIC_BASE_URL="http://localhost:8000"
docker compose up --build -d
```

You can also create a `.env` file in the repository root:

```dotenv
ADMIN_USERNAME=admin
ADMIN_PASSWORD=replace-with-a-strong-local-password
PUBLIC_BASE_URL=http://localhost:8000
```

Do not commit secrets or production credentials to version control.

### 2. Check service status

```bash
docker compose ps
```

### 3. View logs

```bash
docker compose logs -f app
docker compose logs -f db
docker compose logs -f smsgw
```

### 4. Stop the stack

```bash
docker compose down
```

To remove the database and uploaded-file volumes as well, use `docker compose down -v`. **This permanently deletes persisted local data**, so only do this when you intend to reset the environment.

## Run locally without Docker

The repository includes a local runner that runs the portal, simulated SMS gateway and MCP endpoints using SQLite for development.

From the repository root:

```bash
python dev/run_local.py
```

Open `http://localhost:8000` in your browser.

To run the evaluation suite against the local environment, use a second terminal:

```bash
python dev/run_local.py --evals
```

Use the local runner for development and debugging. Verify the configuration and database behavior before using it as a production deployment method.

## Application URLs

When running locally with Docker Compose or the local runner:

| Page or service | URL |
|---|---|
| Main portal | `http://localhost:8000/` |
| About page | `http://localhost:8000/about` |
| Citizen MCP endpoint | `http://localhost:8000/mcp/citizen` |
| Officer MCP endpoint | `http://localhost:8000/mcp/officer` |
| Evaluation dashboard | `http://localhost:8000/eval-dashboard` |
| Simulated SMS gateway interface | `http://localhost:8000/__gateway/` |
| OAuth authorization server metadata | `http://localhost:8000/.well-known/oauth-authorization-server` |
| Protected resource metadata | `http://localhost:8000/.well-known/oauth-protected-resource` |
| JSON Web Key Set (JWKS) | `http://localhost:8000/.well-known/jwks.json` |
| OAuth client registration | `POST http://localhost:8000/oauth/register` |
| OAuth authorization | `GET http://localhost:8000/oauth/authorize` |
| OAuth token endpoint | `POST http://localhost:8000/oauth/token` |
| Admin token management | `http://localhost:8000/admin/tokens` |

Some routes require authentication or particular roles/scopes. The SMS gateway is simulated; it is intended for local testing, not real SMS delivery.

## MCP endpoints and tools

Both MCP endpoints use Streamable HTTP. Connect an MCP-compatible client to the relevant endpoint and complete the OAuth authorization flow before invoking protected tools.

### Citizen endpoint

`POST /mcp/citizen`

The citizen-facing tool set supports these task categories:

- Get information about the pension scheme.
- Check eligibility using the portal's rules.
- Submit an application after required details are collected and validated.
- Track an existing application.
- Withdraw an application where the portal permits it.

### Officer endpoint

`POST /mcp/officer`

The officer-facing tool set supports these task categories:

- View queue statistics.
- List applications in the officer's authorized queue.
- Inspect application details.
- Approve an application.
- Reject an application with a reason.

The exact tool names and input schemas are defined by the MCP implementation in `app/mcp.py`. Use the endpoint's MCP tool-listing mechanism to discover the current names and schemas rather than assuming that the categories above are literal tool names.

## OAuth 2.1 authentication flow

The portal provides its own authorization server. A client should redirect the person to the portal for sign-in and consent; it should not collect a citizen's OTP or an officer's portal password itself.

1. **Discover metadata.** Fetch `/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource`.
2. **Register the client.** Use `POST /oauth/register` or a supported client metadata URL.
3. **Start authorization.** Open `/oauth/authorize` with the required OAuth parameters, PKCE S256 challenge, requested scope and resource audience.
4. **Sign in and consent.** Citizens authenticate with their mobile number and OTP. Officers use the departmental sign-in flow.
5. **Exchange the code.** Send the authorization code and PKCE verifier to `POST /oauth/token`.
6. **Validate tokens.** Use the JWKS at `/.well-known/jwks.json` to validate the signature. Validate issuer (`iss`), audience (`aud`), expiration, scope, role and subject (`sub`).
7. **Call the MCP endpoint.** Send the access token as a Bearer token in the HTTP `Authorization` header.

The expected token subject distinguishes citizen and officer identities. Tokens are resource-specific: a token intended for one MCP endpoint must not authorize access to the other endpoint. Access tokens are short-lived and refresh tokens rotate within an absolute session lifetime; refer to `app/oauth.py` for the implementation details and configured lifetimes.

Administrators can manage long-lived program tokens for approved unattended officer use at `/admin/tokens`. Treat these tokens as secrets, limit their distribution, and revoke them when no longer required.

## Evaluation framework

The evaluation harness exercises the assistant through an MCP client, authenticates using the relevant user flow, and then checks the resulting portal database state. This helps verify that the assistant's actions—not merely its wording—match the intended outcome.

### Run evaluations with Docker Compose

Run from the repository root:

```bash
docker compose run --rm app python /evals/run_evals.py
```

Run one persona:

```bash
docker compose run --rm app python /evals/run_evals.py --only sewali
```

Where configured and supported by the evaluation environment, run the language-model-backed mode:

```bash
docker compose run --rm app python /evals/run_evals.py --llm
```

For a local Compose stack, set the base URL to the app service if required by the runner:

```bash
docker compose run --rm app python /evals/run_evals.py --base-url http://app:8000
```

Check the runner's available flags with:

```bash
docker compose run --rm app python /evals/run_evals.py --help
```

### View results

Open `http://localhost:8000/eval-dashboard`. Evaluation runs are stored in the database and displayed with the personas, conversations, checks and pass/fail outcomes.

### Evaluation scenarios

The included scenarios cover outcomes such as:

- An eligible citizen submitting an application with the details they supplied.
- An ineligible applicant receiving guidance without an invalid application being recorded.
- A citizen asking only for status information without changing an existing application.
- A citizen correcting information by withdrawing an old pending application and submitting a replacement, where supported by the portal workflow.
- An officer processing applications in an authorized queue, with the decision and reason attributed to the officer.

The scenarios and assertions live in `evals/`. Review those files before running evaluations against any non-disposable database: test runs can create or modify records.

## Configuration

The Docker Compose environment supports these key variables:

| Variable | Purpose | Example/default |
|---|---|---|
| `ADMIN_USERNAME` | Administrator login name | `admin` |
| `ADMIN_PASSWORD` | Administrator password | Set a strong value; do not use the development default in production |
| `PUBLIC_BASE_URL` | Public base URL used in OAuth metadata and tokens | `http://localhost:8000` locally; use the deployment's HTTPS URL in production |

Other database, gateway and application settings are defined in `docker-compose.yml`, `app/config/app.ini` and the application code. Review these files before deployment.

### Deployment URL

For a deployment, set `PUBLIC_BASE_URL` to the externally reachable **HTTPS** origin of the portal, for example:

```dotenv
PUBLIC_BASE_URL=https://your-domain.example
```

The value must match the address users and MCP clients actually use because OAuth discovery metadata and token validation depend on it. Configure TLS termination, allowed network access, backups and monitoring for your hosting environment.

## Database and persistence

- Docker Compose uses PostgreSQL.
- The database schema is defined in `app/schema.sql`.
- Initial seed data is loaded from `seed/seed.sql` when the PostgreSQL data volume is first initialized.
- Docker volumes preserve database data, uploaded files and application logs across container restarts.
- The local development runner uses SQLite and its local development data directory.

PostgreSQL's initialization scripts run when the database data directory is first created. If you change the schema or seed file after the volume has already been initialized, those changes are not automatically re-applied to an existing database. Back up data and use a deliberate migration/reset process when changing database structure.

## Security notes

Before any public deployment:

- Replace all example and default passwords with strong, unique secrets.
- Set `PUBLIC_BASE_URL` to the public HTTPS origin.
- Do not commit `.env` files, private keys, access tokens, program tokens, or production database credentials.
- Keep PostgreSQL and the simulated SMS gateway private; expose only the necessary public application endpoint.
- Use least-privilege officer accounts and restrict administrator access.
- Keep citizen and officer MCP endpoints, scopes and audiences separate.
- Validate JWT signature, issuer, audience, expiry, role, scope and subject for every protected request.
- Protect and rotate program tokens; revoke them when no longer needed.
- Review file-upload handling, logs, audit records, session settings, CORS/network rules and dependency versions.
- Use backups and a documented retention policy for personal data and uploaded documents.
- Run evaluation scenarios against a disposable or explicitly approved test database. Evaluation scenarios may create or change records.
- Do not expose the simulated SMS gateway as a real messaging service; replace it with an appropriately secured provider integration if required.

## Troubleshooting

### The portal does not start

```bash
docker compose ps
docker compose logs --tail=200 app
```

Check that port `8000` is available, the images build successfully, and the database and SMS gateway containers are healthy enough for the app to connect.

### Database initialization or connection errors

```bash
docker compose logs --tail=200 db
```

Confirm the database service is running. Remember that the schema and seed scripts are applied automatically only when the PostgreSQL data directory is first initialized. Do not delete volumes unless you are willing to lose their stored data.

### OAuth issuer or audience errors

Check that `PUBLIC_BASE_URL` matches the URL used by the browser and MCP client. For a deployed environment, use the public HTTPS URL rather than an internal Docker service name or localhost. Ensure the requested `resource` matches the MCP endpoint the client intends to call.

### `insufficient_scope` or access denied

Confirm that the client completed the correct authorization flow, requested the required scope, and is using a token for the correct resource endpoint. Citizen tokens are not intended for officer operations.

### OTP not received

This project uses a simulated SMS gateway in development. Open `http://localhost:8000/__gateway/` to inspect the test messages. This is not delivery to a real mobile network.

### Evaluation results are missing

Confirm that the app is running, that the evaluation harness points at the intended base URL, and that the database used by the harness is the same database read by `/eval-dashboard`. Review the runner output and the app logs for authentication, network or database errors.

## Development notes

- Keep the portal's business rules as the source of truth.
- Add citizen tools only to the citizen MCP endpoint and officer tools only to the officer endpoint.
- Add evaluation scenarios for new workflows, including both the expected successful state and cases where no state change should occur.
- Verify database state and audit history when testing actions that change applications.
- Update this README when routes, configuration variables, setup steps or evaluation behavior change.

## License

No license is specified in this repository. Contact the project owner before redistributing or using the project outside its intended assessment or development context.
