# Architecture: how the repos fit together

```
                         ┌──────────────────────────── aerosend-core (low churn) ──────────────────────────┐
  web  ──REST /api/v1──► │  Spring Boot API ──► PostgreSQL 16 + TimescaleDB                                  │
  admin ─REST (writes)─► │                         ├─ domains, mailboxes, ip_pools, server_batches           │
                         │                         ├─ deliverability_events  (hypertable, 180d retention)    │
                         │                         ├─ domain_daily_stats     (continuous aggregate)          │
                         │                         └─ outbox_events ──┐                                      │
                         └────────────────────────────────────────────┼─────────────────────────────────────┘
                                                                      │ CDC (Debezium, pgoutput)
                                                                      ▼
                                                       RabbitMQ topic exchange "aerosend.events"
                                     ┌────────────────────────────────┼───────────────────────────┐
                                     ▼                                ▼                           ▼
                        ancillary: mailbox-sync          ancillary: burn-alerts         ancillary: webhooks
                        → Smartlead/Instantly/…          (5 signals → AT_RISK/BURNED)   → customer URLs

  admin, ancillary ──read-only──► Postgres read replica
  utils (Python)   ──writes─────► deliverability_events (placement tests, backfills)
  internal-tools   ──► core API (IP rotation), RabbitMQ mgmt API (queue depth), DNS (SPF/DKIM/DMARC checks)
```

## Contracts (changing these is a cross-repo change)

| Contract | Owner | Consumers |
|---|---|---|
| REST `/api/v1/*` | core | web (`CoreApiClient`), admin (`CoreAdminClient`), internal-tools (`aeroctl`) |
| Event types on `aerosend.events` (`domain.provisioned`, `mailbox.created`, `deliverability.daily_rollup`, …) | core (outbox) | ancillary listeners |
| Table schema (Flyway, append-only) | core | admin + ancillary (replica SQL), utils (direct SQL) |
| `burn_status` values `HEALTHY / AT_RISK / BURNED` | core | web badge, admin, ancillary evaluator |

## Conventions shared across Java repos

- Java 21, Spring Boot 3.3, Maven, constructor injection, records for DTOs
- Thymeleaf pages with a `layout :: page(title, content)` fragment
- Writes to core data only through core's API (so outbox events fire). Never write to the replica.
- Every admin mutation is audit-logged.

## Change-frequency profile

| Churn | Repos | What it means for tooling |
|---|---|---|
| low | core | Changes are rare and high-blast-radius: extra review, contract checks, migration safety |
| medium | web, internal-tools, utils | Regular feature work |
| high | ancillary, admin | Many small PRs. Optimise for fast, safe, repeatable changes (templates, commands, tests) |
