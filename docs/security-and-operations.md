# Security & Operations Reference

> This is the **authoritative rationale for every security control** in the gateway.
> The [README](../README.md) is the quick start; this document is the reasoning.
> `CLAUDE.md` is the code map.

## Core invariants

- **Server owns the pool**, exposes a QUERY capability — never a raw connection.
- **One `pg.Pool` per datasource**, shared across tenants. Tenant isolation is per-query:
  every query runs inside a transaction with `SET LOCAL search_path TO "<schema>", pg_temp`,
  which auto-resets on COMMIT/ROLLBACK so a pooled connection can't leak one tenant's
  `search_path` to the next borrower. **This is a P0 correctness invariant.**
- **Connections are returned pristine**: `SET LOCAL` only auto-resets the settings *we*
  issue — a caller's own plain `SET` survives COMMIT on that pooled connection. Every
  query therefore ends with `DISCARD ALL`, which also drops temp tables and prepared
  statements. Without it, `SET statement_timeout = 0` in one request silently disabled
  the timeout guardrail for every later borrower of that connection.
- **One statement per request, enforced by the wire protocol.** All SQL runs through the
  extended protocol (`queryMode:'extended'`), under which *the server* rejects
  multi-statement text. This is structural; the text scanner is only a fast 400.
- **Read-only by default**, enforced at the engine (`BEGIN TRANSACTION READ ONLY`) with a
  DB-role backstop; an app-layer **statement guard** (leading-keyword allowlist + banned
  side-effecting-function scan) adds defense-in-depth for what a read-only txn does *not*
  stop (`COPY`, `pg_read_file`, …), and a **relation guard** (real-parser tree walk) bounds
  *what/where* a query may read (catalog block + schema caps + denied tables) — belt to the
  braces, never the sole barrier. Writes are opt-in (see [Writes](#writes-off-by-default)).
- **Layered timeouts**: `connectionTimeoutMillis` (acquire), `statement_timeout` (query),
  `idle_in_transaction_session_timeout` (stalled txn).
- `acquire → use → release-in-finally`; a mandatory `pool.on('error')` handler on every
  pool; graceful shutdown drains pools; fail-fast boot (one `SELECT 1` per datasource).

## Configuration (`.env`)

See `.env.example`. Logical datasource names decouple callers from real endpoints:

```
DATASOURCES=main
DS_MAIN_HOST=localhost
DS_MAIN_USER=postgres
DS_MAIN_PASSWORD=postgres
DS_MAIN_DATABASE=appdb
DS_MAIN_DEFAULT_SCHEMA=public
DS_MAIN_STATEMENT_TIMEOUT_MS=10000

TOKENS=agent_ro,svc_rw
TOKEN_AGENT_RO_SECRET=...   TOKEN_AGENT_RO_DATASOURCES=main  TOKEN_AGENT_RO_MODE=read   TOKEN_AGENT_RO_SCHEMAS=*
TOKEN_SVC_RW_SECRET=...     TOKEN_SVC_RW_DATASOURCES=main    TOKEN_SVC_RW_MODE=write    TOKEN_SVC_RW_SCHEMAS=public
```

- **Fallback:** if NO `DS_*` datasource is configured, a single `main` is seeded from
  the canonical `DATABASE_HOST/PORT/USERNAME/PASSWORD/NAME/SSL`.
- **Hard caps:** `MAX_ROWS_CEILING` (default `10000`) is the absolute row cap; a request's
  `maxRows` is clamped to it. A request's `timeoutMs` is clamped to the datasource's
  `STATEMENT_TIMEOUT_MS`.
- **Binding:** `HOST` defaults to `127.0.0.1`. Only loopback binds without
  `ALLOW_PUBLIC_BIND=true` — see [Network binding](#network-binding).
- **Denied tables:** `DS_<NAME>_DENIED_TABLES` is a comma list of relations the
  [relation guard](#relation-guard-schema-boundary--metadata-block--denylist) rejects
  (400) before DB contact. Entries are `table` (matches in ANY schema) or `schema.table`
  (exact); case-insensitive. **Code default is EMPTY** — the gateway is generic, so a
  deployment must declare its own list in `.env` (see `.env.example`); boot logs the count.
- **Writable:** `DS_<NAME>_WRITABLE` is the **third** write gate, after `TOKEN_*_MODE=write`
  and an explicit `readOnly:false` on the request. A write reaching a non-writable
  datasource is a 403 before any DB contact, whatever the token says. **Defaults to
  `false` from every config source** — `.env`, `datasources.d/`, and the `DATABASE_*`
  fallback alike. Opt-**in** semantics: only an exact `true` grants it, so a typo leaves
  the datasource read-only rather than silently writable. Boot logs a WARN naming every
  datasource this is enabled for. See [Writes](#writes-off-by-default).
- **DB role:** point each datasource at a **read-only Postgres role**; that, not the
  token mode, is the real write barrier. See [The read-only guarantee](#the-read-only-guarantee-lives-in-the-database).

## Adding a datasource without editing `.env` (`datasources.d/`)

A datasource may live in its own file, so adding one does not mean editing a file that
also holds every token secret:

```
datasources.d/warehouse.env        (mode 600)   →  datasource "warehouse"
```

**The filename is the datasource name** — there is no `NAME` key. The stem must match
`/^[a-z0-9_-]+$/`. Only `*.env` is read, which is why the shipped template is called
`example.env.disabled`: it documents the format in place without ever being loaded.

Keys are **bare** — no `DS_<NAME>_` prefix. They are normalized to exactly the
`DS_<NAME>_*` keys `.env` uses, so every zod default, guard default and the
`bool()`/`boolSecureDefault()` asymmetry behave identically regardless of which file a
datasource came from. There is one config model, not two.

```
HOST=warehouse.internal
PORT=5432
USER=agent_ro_pg
PASSWORD=<secret>
DATABASE=analytics
DEFAULT_SCHEMA=public
DENIED_TABLES=billing_accounts
# WRITABLE=true      # omitted ⇒ READ-ONLY (as from every config source)
```

Three rules that are enforced, not advisory:

- **Mode `600` or stricter.** The loader refuses a group- or other-readable file and names
  the `chmod 600 <path>` fix. There is no CLI to set the mode for you, so this check is the
  only thing between a default-umask file and a world-readable password.
- **Unknown keys are rejected.** A stray `TOKENS=` in a datasource file cannot widen
  anyone's grants. Token grants live in `.env` only.
- **A name defined in both `.env` and `datasources.d/` refuses the whole reload.** A
  credential is never silently overridden.

**The directory is read at boot AND on reload**, through the same function, so a datasource
defined only by a file survives a restart and means the same thing either way. (An earlier
revision read it only on `SIGHUP`; a directory datasource then evaporated on the next
restart, or failed boot outright when a token named it.)

`datasources.d/` is git-ignored (`datasources.d/*` plus a negation for the template).
The directory is resolved CWD-relative, overridable with `DATASOURCES_DIR` — which an MCP
client entry **must** set to an absolute path, since the client chooses the cwd (see
[Registering it in an MCP client](#registering-it-in-an-mcp-client)). A **missing
directory is not an error** — the feature is opt-in and every pre-existing config keeps
booting untouched.

**The `DATABASE_*` fallback survives this.** A zero-config deployment that adds its first
datasource file keeps its `main` datasource; the fallback is materialized into the merge
rather than being suppressed by the directory's names.

## Runtime reload (SIGHUP)

Both entrypoints re-read `.env` **and** `datasources.d/` on `SIGHUP` and apply the diff to
the running process — no restart, no dropped request, no cancelled in-flight query:

```bash
kill -HUP $(pgrep -f dist/server.js)        # HTTP gateway
kill -HUP $(pgrep -f dist/mcp/mcp-server.js) # MCP (stdio)
```

An agent already connected over MCP picks up a new datasource by calling
`list_datasources` again — no reconnecting `/mcp`.

**Two stages, with deliberately different failure semantics:**

| Stage | Scope | On failure |
|---|---|---|
| 1 — validation | permissions, parse, zod, name collision, process identity | **all-or-nothing**: refuses, running config bit-for-bit unchanged |
| 2 — probes | one `ping` + one posture probe per added/changed datasource | **degrades per datasource**: that one is withheld and logged; siblings still publish |

The only sanctioned difference between boot and reload: an unreachable datasource is
**fatal at boot** (nothing is serving yet) and **withheld on reload** (something is). The
server never dies in stage 2.

**Stage 2 has three distinct failure outcomes, and they mean different things about
availability.** Conflating them is how an operator draws the opposite conclusion:

| Outcome | Serving? | Meaning |
|---|---|---|
| `withheld` | **no** | Built but never published — construction, `ping`, or the pre-publish probe failed |
| `retainedPrevious` | **yes, on the OLD config** | The change could not be applied; the previous pool keeps serving. Retried on the next reload |
| `postureUnverified` | **yes, on the NEW config** | Swapped and live, but its read-only posture could not be established |

`retainedPrevious` is the one to watch. Change detection is a full config compare, but the
gate before a swap is a network `SELECT 1` — so a **policy-only tightening** (adding a
`DENIED_TABLES` entry, revoking `WRITABLE`) can fail on a transient blip, and the failure
direction is that the **looser previous policy stays live**. It is retried automatically on
the next reload, and it appears in the summary line, per-datasource warnings, and the
security event.

A reload is logged as a security event naming what was added/removed/changed/withheld/
retained — **names only, never values**. The *reasons* are logged as separate
per-datasource lines, because a `pg` connect error can name a host or user and the security
event must stay free of them.

Note `ok: true` on a reload means stage 1 passed and the diff was applied — **not** that
every datasource succeeded. Anything automating against a reload must read the outcome
lists above, not just the overall result.

**Boot-only keys** cannot be applied to a running process (bound sockets,
constructor-injected values). They are **compared and warned about** — never applied
silently, never ignored silently: `PORT`, `HOST`, `LOG_LEVEL`, `MAX_ROWS_CEILING`,
`HEALTH_CACHE_TTL_MS`, `ALLOW_PUBLIC_BIND`, `MCP_TRANSPORT`, `MCP_HTTP_HOST`,
`MCP_HTTP_PORT`, `DATASOURCES_DIR`. Restart to change one.

**`MCP_TOKEN` is special.** A reload that would remove the token the MCP process runs as —
**or merely rotate its secret** — is refused outright, because the process would be left
unable to authorize its own tool calls. The identity is re-authenticated against the
candidate token set rather than matched by id, which is what catches the rotation case.

**Windows:** `SIGHUP` is not deliverable. Restart the process instead; everything else
about the directory format works unchanged.

### Rolling back

Reverting this feature is **not** purely a code revert, because operators may have created
state the old code cannot read. Do these first:

1. **Inline every `datasources.d/` datasource into `.env`** as `DS_<NAME>_*`, and remove the
   files. The old code never reads the directory, so a datasource defined only by a file
   disappears — and if any `TOKEN_<ID>_DATASOURCES` still names it, **boot fails** with
   `Token "X" references unknown datasource "…"`, an error that points at the token rather
   than at the directory.
2. **Keep the `.gitignore` entry**, or delete the credential files before reverting it.
   Reverting `datasources.d/*` un-ignores files that are still on disk holding live database
   passwords, so a later `git add -A` can commit them. This is the one genuinely dangerous
   one-way step.

Everything else rolls back cleanly. `WRITABLE` rolls back *looser* — the old schema has no
such field, so writes resume and leftover `DS_<NAME>_WRITABLE` keys are simply unread. No
database migrations are involved; all merged config state is derived, nothing is persisted.

## Writes (off by default)

**This gateway is read-only unless three independent things are all turned on**, and none
of them is on in a stock install:

| # | Gate | Default | Where |
|---|---|---|---|
| 1 | a write-mode token | `TOKEN_*_MODE=read`; `.env.example` ships **no** write token | `TokenAuth.authorize` |
| 2 | the datasource is writable | `DS_<NAME>_WRITABLE` unset ⇒ **false**, from every config source | `QueryService.run` |
| 3 | the request opts in | `readOnly` defaults **true** | `TokenAuth.authorize` |

Any one of them missing is a **403 before any DB contact**. All three are opt-**in**: each
uses exact-`true` matching, so a typo (`1`, `yes`, `on`) leaves you read-only rather than
silently writable — the deliberate inverse of the secure-by-default toggles, where a typo
keeps the safety net *on*. Both conventions converge on "a mistake leaves you safer".

Boot logs a **WARN naming every writable datasource**, so a gateway that can write says so
on every start rather than only in a `.env` nobody re-reads.

Write queries add `write:true`, `command`, and `rowsAffected` to the audit line.

None of this replaces the real barrier: **point the datasource at a read-only DB role.**
App-layer gates are what this process controls; DB grants are what actually stops a write.
See [The read-only guarantee lives in the database](#the-read-only-guarantee-lives-in-the-database).

## The read-only guarantee lives in the database

A token's `MODE=read` and the `BEGIN TRANSACTION READ ONLY` wrapper are **app-layer**
controls. They are correct, but they are app logic: one gateway bug, or one
`MODE=read`→`write` edit, removes them. The guarantee that survives that is a Postgres
login role with **no write grants**:

```sql
-- USER ACTION (privilege changes are never run by an agent). Run as superuser/owner.
CREATE ROLE agent_ro_pg WITH LOGIN PASSWORD '<strong-password>';
-- pg_read_all_data (PG14+): SELECT on all current AND FUTURE relations + schema USAGE.
-- Exactly right for schema-per-tenant, where new schemas appear at runtime.
GRANT pg_read_all_data TO agent_ro_pg;
-- Second lock: a plain BEGIN (the write path) is read-only too, so a mode misconfig
-- still cannot write. Advisory only — it is USERSET, so a caller can turn it off.
ALTER ROLE agent_ro_pg SET default_transaction_read_only = on;

-- PG14 only: PUBLIC still holds TEMP on the database, so any role can CREATE TEMP
-- TABLE and write to it. Session-local and cleared by DISCARD ALL, so it cannot touch
-- tenant data — revoke it anyway if you want the "no writes at all" claim to be literal.
-- REVOKE TEMP ON DATABASE <db> FROM PUBLIC;
-- Also PG14: PUBLIC holds CREATE on schema `public` unless revoked (PG15 changed the
-- default). Check with:
--   SELECT has_schema_privilege('public','public','CREATE');
-- REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

Then set `DS_<NAME>_USER=agent_ro_pg` + its password. After this the role can write **no
existing relation** regardless of token mode, transaction mode, a smuggled statement, or a
gateway bug — it holds no privilege to do so.

Do **not** substitute `GRANT pg_write_all_data` for explicit grants on a write datasource:
its privileges are implicit in the ACL check and leave no per-table grant row, so naive
audit queries report it as harmless. (The boot probe below handles it explicitly.)

**Every boot reports the posture** for each datasource (`read-only posture OK` / `WEAK` /
`UNVERIFIED`). It asks whether the role *can* write — `has_table_privilege` /
`has_any_column_privilege` over every non-catalog relation, plus explicit superuser and
`pg_write_all_data` checks — rather than counting `information_schema.table_privileges`
rows, which are built from `relacl` and therefore blind to column-level grants, predefined
roles and superuser. It **fails closed**: a probe error or an unexpected result shape is
reported `UNVERIFIED`, never `OK`. It is a warning, never a hard failure — a legitimately
write-capable datasource may exist — so the point is that a misconfiguration is *loud*
rather than found in a later audit.

One caveat the probe cannot cover: if `dblink` or `postgres_fdw` is installed, `EXECUTE`
defaults to `PUBLIC` and `dblink('…','INSERT …')` writes over a *separate* connection —
outside the read-only transaction and outside any grant the role holds. Revoke `EXECUTE`
on those functions, or don't install them on a database this gateway reaches.

Runbook for creating the role: [`runbooks/agent-ro-pg-role.md`](runbooks/agent-ro-pg-role.md).

## Statement guard (defense-in-depth)

A Postgres `READ ONLY` transaction blocks DB *data/catalog* writes but **not** `COPY … TO
PROGRAM/'file'`, `pg_read_file`/`pg_ls_dir`, backend signals (`pg_terminate_backend`), or
WAL messages (`pg_logical_emit_message`) — all catastrophic under a superuser role (see
[the risk report](risks/2026-07-29-mcp-run_query-write-and-rce-bypass.md)). Every call
therefore passes an app-layer guard at the single `QueryService.run()` choke point (so
HTTP `/query`, MCP `run_query`, and introspection are all covered), **before any DB
contact**:

- **Leading-keyword allowlist.** Read mode permits `SELECT`, `WITH`, `EXPLAIN`,
  `VALUES`, `TABLE`; write mode adds `INSERT`, `UPDATE`, `DELETE`, `MERGE`. Anything else
  (`COPY`, `CALL`, `DO`, `SET`, `ALTER`, `CREATE`, …) is rejected — an allowlist needs no
  per-keyword maintenance. **`SHOW` was removed** — it leaks server settings (`SHOW all` →
  `data_directory`, `config_file`, connection strings) and exposes no relation for the
  relation guard to police; read a specific setting with `current_setting('name')` or
  `SELECT version()` instead.
- **Banned-function scan.** Side-effecting/host-access functions (`pg_read_file`,
  `pg_ls_*`, `lo_export`, `dblink*`, `pg_reload_conf`, `pg_terminate_backend`,
  `pg_logical_emit_message`, `pg_stat_reset*`, …) are rejected in **both** modes, even
  inside an allowed statement (`SELECT pg_read_file(…)`).

Both run over the shared `stripToCode` view (`src/query/sql-lexer.ts`) that
blanks comments/strings/dollar-bodies, so the checks can't be comment- or string-bypassed
(`SELECT 'pg_read_file(' AS note` is allowed; `SELECT/**/pg_read_file(…)` is not). The
function scan reads an ident-revealing variant so a **quoted** call `SELECT
"pg_read_file"(…)` — which Postgres resolves to the same function — is caught too. A
rejection is a `400` and is **audited** (blocked attempts, including `;`-smuggle, show up
in the security stream). The multi-statement scan and the read-only transaction are
separate and always-on.

The `U&"\0070g_read_file"` unicode-escape identifier a text scanner cannot decode is now
closed by the **relation guard** below, which runs the same banned-function list against
the real parser's decoded names. Both scans share one list
(`src/query/banned-functions.ts`) so they cannot drift; this text
scan stays as belt-and-braces for SQL the parser accepts but shapes differently.

This is **defense-in-depth, not the guarantee** — it ships *with* the non-superuser DB
role above, never instead of it (denylists drift; the role holds no privilege to begin
with).

**Escape hatch.** A datasource whose DB role is trusted and which legitimately needs
admin/`COPY`/file statements can opt out with `DS_<NAME>_ALLOW_UNSAFE_STATEMENTS=true`
(default **false**, fail-closed — any non-`true` value keeps the guard on). It relaxes the
statement guard **and** the relation guard below; the multi-statement scan and the
read-only transaction stay enforced. Enabling it removes two security layers, so **every
boot logs a WARN** naming the datasource.

## Relation guard (schema boundary + metadata block + denylist)

The statement guard answers *whether* a statement may run; it says nothing about *what* it
reads. Read-only is a write control, and the DB role is `pg_read_all_data` — SELECT on
every relation, forever — so on its own the gateway would expose `information_schema`, the
`pg_catalog` views (`pg_tables`, `pg_roles`, `pg_settings`, …), and every identity/secrets
table to any read token. Postgres cannot fix this at the grant layer: `pg_catalog` access
cannot be revoked, and grants are additive so tables cannot be carved out of
`pg_read_all_data`. So the boundary is enforced at the app layer, at the same
`QueryService.run()` choke point, **before any DB contact**:

- **Parse, don't pattern-match.** Every statement is parsed with the *real* Postgres parser
  (`libpg-query`, `src/query/relation-guard.ts`); the guard walks the parse tree for the
  relations it references (CTE-aware — a `WITH` name is not a table) and the functions it
  calls. This includes **write/create targets** on the write path — the `INSERT`/`UPDATE`/
  `DELETE`/`MERGE` target and `SELECT … INTO` — so a write-mode token is confined to its
  caps and the denylist just as reads are (a target named like an enclosing CTE is still the
  real table). Decoded identifiers close the `U&"\0070g_read_file"` / `U&"\0075ser"`
  unicode-escape gap a lexer cannot. **Unparseable SQL is rejected (400)** — SQL the gateway
  cannot understand is SQL it cannot police (fail-closed).
- **Five relation rules:** a relation qualified with a **system schema** (`pg_catalog`,
  `pg_toast`, `pg_temp*`, `information_schema`) → **400**; qualified with **another schema
  outside the token's `SCHEMAS` caps** → **403**; an **unqualified `pg_%`** name (which
  resolves to `pg_catalog`, implicitly first on `search_path`) → **400**; the effective
  schema + relation on the datasource's **`DS_<NAME>_DENIED_TABLES`** list → **400**; and a
  relation whose **name matches the built-in sensitive-relation denylist** → **400**.
- **Built-in sensitive-relation denylist (secure-by-default).** Because `DS_<NAME>_DENIED_TABLES`
  ships empty, a name-pattern safety net (`src/query/sensitive-relations.ts`)
  blocks obvious credential/secret/token/key stores (`secrets`, `api_keys`, `oauth_tokens`,
  `private_keys`, `recovery_codes`, `mfa_secrets`, `vault`, …) even when no denylist is
  configured. It is a token-boundary match (`user_secrets`, `oauth_access_tokens` are caught),
  with a small curated set of zero-false-friend roots (`credential`, `apikey`, `password`)
  also matched as a substring so glued compounds (`credentialstore`) are caught, and it
  deliberately excludes generic identity tables (`users`, `accounts`, `sessions`) to avoid
  over-blocking ordinary reads. Default on, fail-closed — only an explicit
  `DS_<NAME>_SENSITIVE_RELATION_DENYLIST=false`/`0`/`no`/`off` turns it off (boot WARNs when
  it does); skipped by `ALLOW_UNSAFE_STATEMENTS`.
- **Connection-identity functions blocked (secure-by-default, always on).** `run_query`
  returns table *data*; it must not answer "who / where am I?". Identity functions
  (`src/query/connection-info-functions.ts`) that would read
  back the datasource's real **user / database / host** — `current_user`, `session_user`,
  `current_role`, `user`, `current_database`, `current_catalog`, `current_schema(s)`,
  `inet_server_*` / `inet_client_*` — are rejected (**400**). Both parse shapes are covered:
  the keyword forms are `SQLValueFunction` nodes, the call forms `FuncCall`. `current_setting`
  is allowed **except** for the identity GUCs `session_authorization` / `role` (and a
  non-literal argument fails closed) — so `current_setting('session_authorization')` cannot be
  used as a back door to the session user. Date/time value functions (`now()`, `current_date`,
  …) carry no identity and pass. This closes the `SELECT current_user, current_database()`
  path an agent could otherwise use to disclose the values `.env` holds.
- **Metadata path.** `list_schemas` / `list_tables` / `describe_table` are the sanctioned,
  caps-filtered way to see structure. They run fixed, parameterized `information_schema`
  reads on an **internal trusted route** that a caller cannot request — the trust flag is a
  second positional argument to `run()`, never a request field, so no HTTP body or MCP args
  object can set it. (The boot posture probe uses the same internal route.)
- **Escape hatch.** `DS_<NAME>_ALLOW_UNSAFE_STATEMENTS=true` skips the statement guard **and**
  this relation guard; boot WARNs. Every boot also logs the guard state and denied-table
  count per datasource.

Rejections (both 400 and 403) are **audited** — a qualified cross-schema 403 is the
tenant-probe signal worth alerting on.

**Error redaction.** Error text returned to a caller (MCP tool result / HTTP body) is run
through `src/query/redact-error.ts`: connection URIs, `password=`/`user=…`
params, pg auth-failure user names, and `getaddrinfo`/`connect` host/address details are
masked, so a connection failure can never echo connection metadata to the LLM. The full
detail still goes to the server-side audit log. Useful SQL errors (syntax, undefined column)
pass through unchanged.

**Residual risks (accepted).** A **view** in an allowed schema over a denied table still
reads it (views run with owner privileges; no such views are known — the DB-grants runbook
is the durable fix). Connection-identity functions (`current_user`, `current_database()`,
`inet_server_*`, `current_setting('session_authorization')`) are now blocked (above), but
non-identity scalar metadata such as `version()` and other `current_setting('<guc>')` reads
stay readable — server fingerprint, not connection identity or table data. And this is
app-layer enforcement over a shared `pg_read_all_data` role: it holds only as long as the
guard does. The belt exists;
explicit per-token grants ([runbook](runbooks/agent-ro-pg-role.md)) are the braces.

## Network binding

`HOST` defaults to `127.0.0.1`, and **only loopback binds without an explicit opt-in** —
anything else needs `ALLOW_PUBLIC_BIND=true`. This gateway holds live DB credentials and,
over HTTP, the only thing in front of them is a plaintext bearer secret, so binding beyond
this machine must be a deliberate decision made with TLS termination and a rotated secret.

The check is an **allowlist, not a denylist**, because the set of hosts that bind every
interface cannot be enumerated: `0.0.0.0`, `::`, `0`, `0.0`, `::0`, `0x0` and the *empty
string* all do it (an empty or bare-integer host is resolved rather than rejected, and
`net.isIP('0')` is `0`, so IP parsing does not catch it either). Requiring loopback is the
one rule that cannot be out-spelled. Note this means binding a specific private address
also needs the flag — deliberate either way.

The guard covers both `listen()` sites: the Fastify gateway (`HOST`) and the MCP
streamable-HTTP transport (`MCP_HTTP_HOST`), whose env vars bypass zod and so are
normalised for the empty-string case at the call site.

## Rotating the bearer secret

Secrets are plaintext in `.env` and compared as constant-time SHA-256 digests — never
logged. Rotation is therefore the only exposure control:

1. Generate: `openssl rand -base64 24`
2. Update **both** `TOKEN_<ID>_SECRET` and `MCP_TOKEN` in `.env` — the MCP process
   authenticates as one of the configured tokens, so a mismatch fails boot with
   `MCP_TOKEN does not match any configured token.`
3. Restart the gateway (`npm start`) and/or reconnect the MCP client (`/mcp` in Claude Code).
4. Rotate the **DB** password separately (`ALTER ROLE … PASSWORD …`, a user action) and
   update `DS_<NAME>_PASSWORD`; the bearer and the DB credential are independent secrets.

## Trust boundary — schema caps, and how far they reach

Since the relation guard, a token's `SCHEMAS` capability **is** enforced against the
relations a query actually references — not just the declared `schema` / `search_path`. A
token scoped to `public` now gets a **403** on `SELECT * FROM "other_tenant".t`, because the
guard parses the statement and checks every qualified relation against the caps (the same
`capabilityAllows` used for the declared schema). `SET LOCAL search_path` still guarantees
tenant isolation *between pooled borrowers* (the P0 invariant); the guard adds confinement
of the SQL body on top of it.

Two limits remain, so this is a strong app-layer boundary, **not** a database-enforced one:

- It holds only as long as the guard does — every query still runs as one shared
  `pg_read_all_data` role, and a **view** in an allowed schema can read across into a schema
  the token could not name directly.
- For a *hard* per-token boundary, enforce it in Postgres: a DB role per token with `USAGE`
  revoked on other schemas, or explicit per-schema grants
  ([`runbooks/agent-ro-pg-role.md`](runbooks/agent-ro-pg-role.md)). That is a
  deliberate follow-up, incompatible with the current single shared-pool topology.

## TLS note

When `SSL=true`, the client uses `rejectUnauthorized:false` (a typical TypeORM
config for managed Postgres with a self-signed chain) — this encrypts the connection but
does **not** authenticate the server. Provide a CA / set stricter TLS if MITM is in scope.

## search_path note

Only the target schema is put on `search_path` (not `public`), for isolation — so
references to shared `public` objects must be **fully qualified** (`public.foo`).
`pg_temp` is appended explicitly: when it is *absent* Postgres still searches it, and
searches it **first**, ahead of the tenant schema, for relation names — so a temp table
left on a pooled connection would shadow the next borrower's real table. Naming it
demotes it to last place.

## Why one statement per request is enforced twice

`assertSingleStatement` is a text scanner and text scanners lose. It previously missed
`E'…'` escape strings, where `\'` is an escaped quote rather than a terminator: the
scanner believed the literal was still open, swallowed the rest of the input, and let
`SELECT E'\''; COMMIT; <anything>` through. That smuggled `COMMIT` ended
`BEGIN TRANSACTION READ ONLY` and handed a **read-scoped token a read-write session** —
the read-only guarantee and the "defence in depth" guard were never independent, because
read-only lives on a transaction the caller could simply commit away.

The fix moves enforcement into the wire protocol. `pg` selects the protocol by whether
params were supplied — with none it uses the *simple* protocol, which executes `a; b`
from one call. `PostgresDriver.exec` now pins `queryMode:'extended'`, so Postgres answers
`cannot insert multiple commands into a prepared statement` regardless of what the
scanner thinks. The scanner is kept purely to fail such input earlier, as a 400, before
any DB contact. **Do not revert `exec` to the two-argument `client.query(sql, params)`
form** — that silently restores the simple protocol and the whole exploit chain.

## Security notes (misc)

- Bearer secrets are compared in constant time and never logged (audit logs the token
  *id*, not the secret).
- The schema is applied as a **quoted identifier** (validated, double-quote-escaped); all
  values must be passed via `params` (`$1…`). The audit line records SQL text and error
  text (each capped at 2,000 chars with a `…[+N chars]` marker; a truncated statement also
  records `sqlLength` so it stays attributable) while `params` are **never** logged — so
  **never inline secret values as SQL literals**, use `params`.
- Not an ORM/migration runner (never generates or runs migrations), not a general SQL
  console — it's an infra utility for trusted services/agents.
- **MCP over HTTP** requires the caller to present the process's own token as a bearer and
  enables DNS-rebinding protection (Host allow-list); it still binds loopback by default.
- **`/health` is unauthenticated** but its DB pings are cached (`HEALTH_CACHE_TTL_MS`,
  default 5000ms) and de-duplicated, so probe floods can't exhaust the pool.
- Query error messages are returned to the caller verbatim (a debugging affordance);
  combined with the boundary above, treat error text as revealing structure.

## Known limitations

The row cap is enforced by fetching the result and slicing to `maxRows` (`truncated:true`
if exceeded), rather than a server-side cursor. For this trusted-caller utility that is
bounded in practice by `statement_timeout` + the row ceiling; a cursor-based bounded fetch
is a possible future optimization.

**Reload is signal-driven, not watched.** There is deliberately no `fs.watch`: watchers are
inconsistent across platforms and fire mid-write on partially-written files — which for a
credential file means loading half a password. A signal is explicit, atomic and scriptable.
A watcher stays cheap to add later precisely because reload is already atomic-or-nothing.

**`SIGHUP` is undeliverable on Windows**, so reload there means restarting the process.

**A key deleted from `.env` since boot is not unset by a reload.** The merged candidate is
built over the live environment, and node loaded `.env` into it at boot, so the old value
survives. Changing a value works; removing one entirely needs a restart.
