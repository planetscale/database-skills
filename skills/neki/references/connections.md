---
title: Connecting to Neki
description: Postgres clients, roles, router groups, TLS, and replica routing
tags: neki, connections, router, roles, tls, replica-routing
---

# Connecting to Neki

Docs: https://planetscale.com/docs/neki/connecting · https://planetscale.com/docs/neki/replicas

Applications connect to a **Neki router** (not to individual shards) using the Postgres wire protocol, so standard Postgres clients, drivers, and ORMs work. The router plans each statement and sends it to the required shards. A router is more than a query relay or a proxy: it has a full Postgres parser, planner, buffering, and health-aware routing.

**All connections use port `5432`.** There is no separate pooler port — pooling happens in the sidecars beside each Postgres instance.

## Connection details

Get credentials from the dashboard (Connect) or `pscale`. The generated `psql` command looks like:

```bash
psql 'host=<HOST> port=5432 user=<USERNAME> password=<PASSWORD> dbname=postgres sslnegotiation=direct sslmode=verify-full sslrootcert=system'
```

- **Host and port** — from the Connect page; port `5432`.
- **Username** — the full generated username, including any suffix for a non-default **router group**.
- **Database** — logical database, `postgres` by default.
- **TLS** — required; verify the certificate chain and hostname (`sslmode=verify-full`, system CA). Other drivers use equivalent settings shown on the Connect page.

Create a **role** per application or permission boundary (`pscale role`). `pscale shell <db> <branch> --org <org>` opens a session with temporary credentials; add `--router <group>` for a non-default router group.

## Router groups and private connectivity

Use additional **router groups** when a workload needs independent router sizing or autoscaling. Choose the group on the Connect page; the username carries a suffix for non-default groups. Private connectivity is available via AWS PrivateLink and GCP Private Service Connect (same roles, router groups, and TLS).

## Primary vs replica routing

Queries go to the shard **primary** by default (required for writes and queries needing a read-after-write guarantee). Route reads to replicas with `__neki.target`:

```sql
SET __neki.target = 'replica';        -- before a transaction
```
```bash
PGOPTIONS='-c __neki.target=REPLICA' psql 'host=<HOST> port=5432 ...'
```

For a connection URI, add `options=-c%20__neki.target%3DREPLICA`. For `pscale shell`, add `--replica`. Selecting "Route queries to a replica" on the Connect page adds this option; it does not change the username.

Replica connections are **read-only** — writes fail rather than falling back to a primary, and if no eligible replica exists for a shard the query fails. Replica selection and lag behavior are in [replication.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/replication.md).

## Pooling and session behavior

Connection pooling is handled by the **sidecars**, configured per configuration profile (`pool-capacity`, `pool-min-conns`, `pool-max-lifetime`, `pool-max-wait-time`, `pool-idle-timeout`, `tx-idle-timeout`, `external-conn-reservation`). Routers also enforce `idle-session-timeout` (default 30 minutes) and `idle-in-transaction-session-timeout` (default 5 minutes). Prefer scaling application connections to routers over raising Postgres `max_connections`.

## Application guidance

- Include the shard key in queries so they resolve to one shard (see [query-serving.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/query-serving.md)).
- Retry transient connection and query errors — a router or shard failover briefly interrupts connections; routers buffer where possible so it usually looks like elevated latency.
- Keep transactions single-shard (see [transactions.md](https://raw.githubusercontent.com/planetscale/database-skills/main/skills/neki/references/transactions.md)).
- Framework guides exist for node-postgres, Drizzle, Kysely, Prisma, Rails, Django, Laravel, Bun, and Cloudflare Workers (Hyperdrive).
