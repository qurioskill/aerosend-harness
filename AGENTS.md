# Aerosend harness Blueprint

## System Overview

This repo holds no product code. It is the top-level workspace for six Aerosend repos, and it contains the repo manifest (`.gitmodules`), the `harness` CLI, and shared context in [docs/architecture.md](./docs/architecture.md). This is a mentoring sandbox: the six repos are realistic dummies, and their remotes are bare repos in `../aerosend-remotes/`.

Repos are attached on demand, so a repo may not be on disk. Run `bin/harness list` to see what is attached and `bin/harness attach <repo>` to check one out. Inside `repos/<name>` you are in a normal git repo: branch, commit, push and open PRs there.

How the services connect:

- `web`, `admin` and `internal-tools` call core's REST API under `/api/v1`.
- `core` writes events to `outbox_events`. Debezium tails that table and publishes to the RabbitMQ topic exchange `aerosend.events`.
- `ancillary` consumes that exchange on three queues: `q.ancillary.mailbox-sync`, `q.ancillary.burn-alerts` and `q.ancillary.webhooks`.
- `admin` and `ancillary` read PostgreSQL directly. The `application.yml` comments in both repos name a read replica in production.
- `utils` writes directly to `deliverability_events`.

## Subdirectories & System Boundaries

- bin: [./bin/AGENTS.md](./bin/AGENTS.md) - The `harness` CLI: attach, detach, status, sync, exec, workspace and set-remote-base.
- docs: [./docs/architecture.md](./docs/architecture.md) - Cross-repo architecture, contract ownership and churn profile. Documentation only.
- repos: [./repos/AGENTS.md](./repos/AGENTS.md) - Mount point for the six submodules, each with its own contract tree.

## Global Invariants

- `.gitmodules` is the only manifest. Add or change repo metadata there, not in a second list.
- The harness does not pin versions. `ignore = all` hides repo changes from the harness's `git status`, so use `bin/harness status` to see them.
- These are cross-repo contracts, and changing one means changing every consumer ([docs/architecture.md](./docs/architecture.md)):

| Contract | Owner | Consumers |
|---|---|---|
| REST `/api/v1/*` | core | web (`CoreApiClient`), admin (`CoreAdminClient`), internal-tools (`aeroctl`, `ops-dashboard`) |
| Event types on `aerosend.events` | core (outbox) | ancillary listeners |
| Table schema (Flyway, append-only) | core | admin and ancillary (SQL reads), utils (direct SQL) |
| `burn_status` values `HEALTHY`, `AT_RISK`, `BURNED` | core | web badge, admin, ancillary evaluator |

- Writes to core data go through core's API so that outbox events fire. Never write to the replica. The one exception is `utils`, which inserts telemetry into `deliverability_events` directly. Its local-only `aero-seed-local` also writes `server_batches`, `burn_status` and `dns_verified`, because core has no route for them.
- Every mutating admin action must call `AuditLogger.record`.
- `core` is low churn with a high blast radius: changes there need extra review, contract checks and migration safety.

### Known gaps between repos

These were checked against the code on 2026-11-05. Do not assume the behaviour exists until the gap is closed.

- **Missing core routes:** core defines only `/api/v1/domains`, `/api/v1/domains/{id}/stats` and `POST /api/v1/domains`. Nothing serves `POST /api/v1/domains/{id}/replace` or `POST /api/v1/ips/{id}/rotate` (used by admin and `aeroctl`), `GET /api/v1/ip-pools/{pool}/ips` (`aeroctl`), or `GET /api/v1/ip-pools` (`ops-dashboard`).
- **Missing event emitter:** ancillary listens for `deliverability.daily_rollup`, but nothing in `core` or `utils` emits it, so burn alerts never fire.
- **Unreachable event path:** only `DomainProvisioningService.createMailboxes` emits `mailbox.created`, and nothing calls it, so mailbox sync never runs.
- **Missing table DDL:** `customers`, `customer_integrations`, `resellers` and `admin_audit_log` are queried by admin or ancillary, but no repo has a migration for them. `customer_integrations` is also the only source of the `smtp_password` that ancillary sends to sequencers, and `mailboxes` has no password column.
- **Read-only user on a write path:** admin's only datasource uses username `aerosend_ro`, and `AuditLogger` inserts through it.

## Global Side Effects

- PostgreSQL with TimescaleDB: core owns the schema. `utils` inserts into `deliverability_events`, and admin inserts into `admin_audit_log`.
- RabbitMQ: Debezium publishes to `aerosend.events`. Ancillary declares its queues and bindings on the broker.
- Third-party HTTP from ancillary: Smartlead and Instantly. `WarmupInboxClient` exists for Warmupinbox but has no caller.
- Google OAuth2 login in admin, and DNS TXT lookups in `aeroctl`.
- `aeroctl ip rotate` and admin's rotate action are meant to rotate live IPs through core.
- `bin/harness` changes `.git/modules`, `repos/` and the gitignored `aerosend.code-workspace`. See [bin](./bin/AGENTS.md).
- An uncommitted file inside an attached repo, including a new `AGENTS.md`, makes `harness detach` refuse and stops `harness sync` from fast-forwarding that repo.

## System Verification

The harness has no test suite, and each repo has its own checks. Results are from 2026-11-05.

- Harness: `bash -n bin/harness`, then `bin/harness status`.

| Repo | Run from | Command | Result |
|---|---|---|---|
| core | `repos/core` | `mvn test` | Passes. One test class, `DnsRecordGeneratorTest` |
| web | `repos/web` | `mvn test` | Passes on Java 21, 3 tests. Fails if Maven runs on Java 25, because Mockito's Byte Buddy can't handle it |
| web | `repos/web/frontend` | `npm test` | Exits 1, no test files exist |
| ancillary | `repos/ancillary` | `mvn test` | Passes, 4 tests |
| admin | `repos/admin` | `mvn test` | Exits 0. The only test is `@Disabled` |
| internal-tools | `repos/internal-tools/aeroctl` | `go test ./...` | Not run: Go was not installed, and no test files exist |
| internal-tools | `repos/internal-tools/ops-dashboard` | `npm run build` | Fails, no `vite.config.*` registers the Vue plugin |
| utils | `repos/utils` | `pytest`, after `pip install -e '.[dev]'` | Passes, 8 tests |
