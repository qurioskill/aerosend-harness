# repos

## Overview

Mount point for the six Aerosend repos. Each one is a git submodule with its own history, PRs, CI and deploys, attached on demand by [`bin/harness`](../bin/AGENTS.md). A link below resolves only while that repo is attached and its `AGENTS.md` exists on the checked-out branch.

## Subdirectories & Boundaries

- core: [./core/AGENTS.md](./core/AGENTS.md) - System of record. Spring Boot REST API on port 8081, Flyway schema on PostgreSQL with TimescaleDB, and the transactional outbox that Debezium publishes to RabbitMQ.
- web: [./web/AGENTS.md](./web/AGENTS.md) - Customer app on port 8080. Thymeleaf pages with Vue 3 islands. No database: all data comes from core through `CoreApiClient`.
- ancillary: [./ancillary/AGENTS.md](./ancillary/AGENTS.md) - Event consumers and free tools on port 8082. Sequencer mailbox sync, burn alerts, a webhook stub and `GET /tools/domain-variations`.
- admin: [./admin/AGENTS.md](./admin/AGENTS.md) - Internal back office on port 8083. Customers, IP pools and resellers, with role-based access and audit logging. Writes to core data go through `CoreAdminClient`.
- internal-tools: [./internal-tools/AGENTS.md](./internal-tools/AGENTS.md) - The `aeroctl` Go CLI and the read-only `ops-dashboard` Vue app.
- utils: [./utils/AGENTS.md](./utils/AGENTS.md) - Python data jobs and SQL snippets. Writes directly to core's `deliverability_events` table.
