# pg-connection-pool — Query Gateway

A small HTTP + MCP service that runs SQL against Postgres **on a caller's behalf**, under
guardrails. Callers send a **datasource name + SQL** — never a connection string, password,
host, or engine type. The gateway owns the connection pool; you get a guarded "run this
query" capability instead of a raw connection.

Built with Fastify + `pg` + `zod` + `pino`. TypeScript ESM.

```
   caller  ──(datasource name + SQL + bearer token)──▶  gateway  ──▶  Postgres
            ◀──────────( rows / columns / error )──────           (pool it owns)
```

**Read-only by default.** Out of the box it can only run `SELECT`-style queries. Writes
require three separate opt-ins (see [Writes](#writes)).

---

## Quick start

```bash
npm install
npm run build

cp .env.example .env          # then edit — see Configuration below
docker compose up -d          # optional: a local Postgres 18 on 127.0.0.1

npm start                     # HTTP gateway (default http://127.0.0.1:3200)
```

Make a query (the default token id in `.env.example` is `agent_ro`):

```bash
curl -s http://127.0.0.1:3200/query \
  -H "Authorization: Bearer change-me-agent-ro" \
  -H "Content-Type: application/json" \
  -d '{ "datasource": "main", "sql": "SELECT 1 AS n" }'
# → { "columns":[{"name":"n",...}], "rows":[{"n":1}], "rowCount":1, ... }
```

On boot, each datasource is pinged and its read-only posture is logged as
`read-only posture OK | WEAK | UNVERIFIED`. `WEAK` means only app code is stopping writes —
the DB role can still write. See [security details](docs/security-and-operations.md#the-read-only-guarantee-lives-in-the-database).

---

## Configuration

All config is env vars, normally in `.env`. **`.env.example` is the annotated source of
truth** — copy it and fill in the blanks. The essentials:

```dotenv
# One or more logical datasource names (callers only ever see the name)
DATASOURCES=main
DS_MAIN_HOST=localhost
DS_MAIN_PORT=5432
DS_MAIN_USER=agent_ro_pg          # point at a READ-ONLY Postgres role (the real write barrier)
DS_MAIN_PASSWORD=...
DS_MAIN_DATABASE=appdb
DS_MAIN_DEFAULT_SCHEMA=public
DS_MAIN_STATEMENT_TIMEOUT_MS=10000

# Access tokens (callers authenticate with the secret; the id is what gets logged)
TOKENS=agent_ro
TOKEN_AGENT_RO_SECRET=...
TOKEN_AGENT_RO_DATASOURCES=main   # which datasources this token may reach (or *)
TOKEN_AGENT_RO_MODE=read          # read | write
TOKEN_AGENT_RO_SCHEMAS=*          # which schemas (or * for any non-system schema)
```

Good to know:

- **Zero-config fallback:** if no `DS_*` datasource is set, a single `main` is seeded from
  the standard `DATABASE_HOST/PORT/USERNAME/PASSWORD/NAME` vars.
- **Add a datasource without touching `.env`:** drop a `datasources.d/<name>.env` file
  (mode `600`, bare keys). The filename is the datasource name.
- **Reload without a restart:** `kill -HUP <pid>` re-reads `.env` and `datasources.d/`.
- **Binding:** only `127.0.0.1` binds unless you set `ALLOW_PUBLIC_BIND=true`.

Full details for every knob — reload semantics, `datasources.d/`, denied tables, the
read-only DB role — are in **[docs/security-and-operations.md](docs/security-and-operations.md)**.

---

## HTTP API

Auth is `Authorization: Bearer <secret>` on every route except `/health`.

| Method + path               | Auth | Purpose |
|-----------------------------|------|---------|
| `GET  /health`              | no   | `{ status, datasources:[{name, ok, poolSize}] }` (503 if degraded) |
| `GET  /datasources`         | yes  | datasources this token may use: `[{name, defaultSchema, writable}]` |
| `POST /query`               | yes  | run one statement |
| `POST /introspect/schemas`  | yes  | schemas visible to the token |
| `POST /introspect/tables`   | yes  | `{ datasource, schema }` → tables + views |
| `POST /introspect/describe` | yes  | `{ datasource, schema, table }` → columns |

`POST /query` body (only `datasource` and `sql` are required):

```jsonc
{
  "datasource": "main",
  "schema": "<account-uuid>",   // optional; defaults to the datasource's default schema
  "sql": "SELECT * FROM orders WHERE id = $1",
  "params": [42],               // ALWAYS use params — never inline values into sql
  "readOnly": true,             // default true
  "maxRows": 1000,              // clamped to MAX_ROWS_CEILING (default 10000)
  "timeoutMs": 10000            // clamped to the datasource's statement timeout
}
```

Response: `{ columns:[{name,dataType}], rows:[...], rowCount, truncated, elapsedMs, rowsAffected? }`.

> **One statement per request.** `a; b` is rejected. **Pass values via `params` (`$1…`)**,
> not string-concatenated into the SQL — params are safer and are never logged.

---

## MCP adapter (for AI agents)

The same capability is exposed over MCP with five tools:
`run_query`, `list_datasources`, `list_schemas`, `list_tables`, `describe_table`. They are
thin wrappers over the *same* services as the HTTP API, so identical guardrails, auth, and
audit apply.

Identity is process-level: set `MCP_TOKEN=<a configured token secret>` and the process runs
with that token's capabilities.

```bash
npm run start:mcp        # stdio (for a local agent client)
npm run start:mcp:http   # streamable HTTP (loopback only)
npm run inspect          # MCP inspector against the built server
```

### Install into a project (recommended)

`install-mcp.sh` registers this gateway as a Claude Code MCP server pointing at **this**
shared install (one `.env`, so credentials never fork). It builds if needed and is
idempotent.

```bash
./install-mcp.sh              # user scope → available in every project
./install-mcp.sh --project    # write ./.mcp.json (committable — teammates inherit it)
./install-mcp.sh --local      # this project only, private
./install-mcp.sh --print      # just print the JSON block
./install-mcp.sh --help       # all flags
```

Then run `/mcp` in Claude Code to connect.

> **Hand-writing the MCP config?** An MCP client spawns the process in *its own* working
> directory, so `datasources.d/` (resolved relative to cwd) will not be found — you must set
> `DATASOURCES_DIR` to an **absolute path** in the entry's `env`. `install-mcp.sh` does this
> for you. And the client runs `dist/`, so **rebuild after editing `src/`**. See
> [security-and-operations.md](docs/security-and-operations.md#adding-a-datasource-without-editing-env-datasourcesd).

---

## Writes

The gateway is read-only unless **all three** of these are turned on (none is, by default):

1. a write-mode token (`TOKEN_*_MODE=write`)
2. the datasource is writable (`DS_<NAME>_WRITABLE=true`)
3. the request opts in (`readOnly: false`)

Any one missing → **403 before any DB contact**. Even with all three, the real guarantee is
pointing the datasource at a **Postgres role that has no write grants** — app-layer gates
are what this process controls; DB grants are what actually stop a write. See
[The read-only guarantee lives in the database](docs/security-and-operations.md#the-read-only-guarantee-lives-in-the-database).

---

## How it stays safe (the short version)

Every request funnels through one choke point (`QueryService.run()`) that applies, **before
touching the DB**:

- **One statement only** — enforced by the Postgres wire protocol, not just a text scan.
- **Statement guard** — a leading-keyword allowlist (`SELECT/WITH/EXPLAIN/VALUES/TABLE`, plus
  `INSERT/UPDATE/DELETE/MERGE` for write tokens) and a ban on side-effecting functions
  (`COPY`, `pg_read_file`, `dblink`, …).
- **Relation guard** — parses the SQL with the real Postgres parser and blocks catalog /
  `information_schema` reads, cross-schema access outside the token's caps, denied tables,
  known secret/credential tables, and connection-identity functions.
- **Read-only transaction** + **layered timeouts** + **tenant isolation** via
  `SET LOCAL search_path` per query, with `DISCARD ALL` to return the connection pristine.

This is defense-in-depth, **not** the guarantee. The durable barrier is a read-only DB role.
The full reasoning for each control — and the exploits each one closes — lives in
**[docs/security-and-operations.md](docs/security-and-operations.md)**.

---

## Local Postgres (`docker-compose.yml`)

A PostgreSQL 18 container for local development, published on `127.0.0.1` only:

```bash
docker compose up -d      # start (credentials come from .env — none are duplicated)
docker compose logs -f pg # watch
docker compose down       # stop (data survives in the pgcp-pgdata volume)
docker compose down -v    # stop AND delete the data
```

It reuses `DS_MAIN_USER/_PASSWORD/_DATABASE` from `.env`, so pointing the gateway at it is a
one-line change: `DS_MAIN_HOST=localhost`. Note it starts **empty** — a tenant-UUID default
schema and the `agent_ro_pg` role won't exist until you create them (see
[security-and-operations.md](docs/security-and-operations.md#the-read-only-guarantee-lives-in-the-database)).

---

## Scripts

```bash
npm run dev            # tsx watch (HTTP server, live reload)
npm run build          # tsc → dist/
npm start              # node dist/server.js (HTTP)
npm run start:mcp      # MCP over stdio
npm run check:config   # validate config without starting the server
npm run typecheck:all  # the static gate (src + test) — there is no linter
npm test               # node --test via tsx
```

## Testing

- **Unit + route tests** run with **no database** (a stub driver exercises the full
  HTTP/MCP path): `npm test`.
- **Integration tests** (`test/integration/*`) hit a **real** Postgres and are **skipped**
  unless configured. They prove tenant isolation, engine read-only enforcement, timeouts,
  write commit/rollback, and the security regressions a stub can't. Run them with:

  ```bash
  PGCP_TEST_HOST=localhost PGCP_TEST_USER=postgres PGCP_TEST_PASSWORD=postgres \
  PGCP_TEST_DATABASE=postgres npm test
  ```

  (falls back to `DATABASE_*`; creates/drops throwaway `pgcp_test_a` / `pgcp_test_b` schemas.)

---

## Learn more

- **[docs/security-and-operations.md](docs/security-and-operations.md)** — the authoritative
  rationale for every guardrail, reload semantics, and operator procedures.
- **`CLAUDE.md`** — the code map (choke point, guard pipeline, where each control lives).
- **`docs/runbooks/agent-ro-pg-role.md`** — creating the read-only DB role.
- **`docs/design-notes/`, `docs/journals/`, `docs/risks/`** — design decisions and the
  bypasses that shaped the guards.
