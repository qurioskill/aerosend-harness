# Aerosend harness

The top-level repo engineers open in **Cursor**. It holds no product code. It holds:

- a **manifest** of every Aerosend repo (`.gitmodules`), with category, change frequency and stack for each
- the **`harness` CLI** (`bin/harness`), which attaches a repo as a git submodule while you work on it and detaches it afterwards
- shared, cross-repo context for people and agents (`docs/`, and later the Cursor rules, commands and skills in `.cursor/`)

> This is a mentoring sandbox. The six repos are realistic dummies that mirror Aerosend's multi-repo layout. Their "remotes" are bare repos in `../aerosend-remotes/`. Run `harness set-remote-base <url>` to point everything at a real GitHub org.

## Repos

| Repo | Category | Churn | Stack | What it is |
|---|---|---|---|---|
| `core` | core | **low** | Spring Boot, Postgres + TimescaleDB, Flyway, outbox → Debezium CDC → RabbitMQ | System of record: domains, mailboxes, IPs, deliverability telemetry |
| `web` | frontend | medium | Spring Boot, Thymeleaf, Vue 3 islands | Customer app (app.aerosend.io) |
| `ancillary` | ancillary | **high** | Spring Boot, Spring AMQP | Sequencer integrations, burn alerts, warmup sync, webhooks, free tools |
| `admin` | admin | **high** | Spring Boot, Thymeleaf, Spring Security | Back office: customers, IP pools, resellers |
| `internal-tools` | internal tools | medium | Go (`aeroctl`), Vue 3 | Ops CLI and ops dashboard |
| `utils` | utils | medium | Python 3.12, SQL | Data scripts, backfills, deliverability analysis |

See [docs/architecture.md](docs/architecture.md) for how they talk to each other.

## Quick start

```bash
git clone <harness-url> aerosend-harness && cd aerosend-harness
export PATH="$PWD/bin:$PATH"      # or: alias harness=./bin/harness

harness list                       # what exists, what's attached
harness attach ancillary core      # check out just what this task touches
cursor aerosend.code-workspace     # multi-root workspace: harness + attached repos
```

When you're done:

```bash
harness status                     # anything unpushed?
harness detach ancillary core      # refuses if you'd lose uncommitted/unpushed work
```

## How the submodules are set up (and why)

Every repo is registered in `.gitmodules`, but with:

| Setting | Effect |
|---|---|
| `update = none` | `git clone --recurse-submodules` / `git submodule update --init` check out **nothing**. A fresh harness clone is a few KB. |
| `branch = main` + `harness attach` uses `--remote` | You always start from the tip of the repo's `main`, never from a stale SHA the harness happened to record. |
| `ignore = all` | The harness's own `git status` stays clean while you commit inside the repos. The harness isn't a release manifest, so it doesn't pin versions. |
| custom keys `category`, `churn`, `stack`, `description` | Git ignores them. `harness list` (and later, Cursor rules) read them. `.gitmodules` is the only manifest. |

Inside `repos/<name>` you're in a normal git repo on a normal branch. Branch, commit, push and open PRs there exactly as before. Every repo keeps its own PRs, CI and deploys.

`harness detach` runs `git submodule deinit`. That removes the working tree but keeps the clone in `.git/modules/<name>`, so re-attaching is instant and your local branches survive.

## `harness` commands

```
harness list                         all repos, category, churn, attached?
harness attach <repo...>             check out repos under repos/ on their default branch
harness attach --category <cat>      e.g. --category ancillary
harness attach --all
harness detach <repo...> [--force]   remove working tree (refuses if work would be lost)
harness status                       branch / ahead-behind / dirty for attached repos
harness sync                         fetch all attached; fast-forward clean default branches
harness exec <cmd...>                run a command in every attached repo
harness workspace                    regenerate aerosend.code-workspace for Cursor
harness set-remote-base <url>        point every submodule at a new host/org
```

## Next: the Cursor harness

Not built yet. That's the next mentoring session. Planned layers on top of this repo:

- `.cursor/rules/`: always-on cross-repo context, plus rules scoped by repo or category (for example, stricter review rules for `core` because it changes rarely and every other repo depends on it)
- `AGENTS.md`: orientation for agents working from the harness root
- `.cursor/commands/` and skills: repeatable workflows (cross-repo contract change, new sequencer integration, admin screen, Flyway migration)
- MCP: Postgres read-only, RabbitMQ management, CI
