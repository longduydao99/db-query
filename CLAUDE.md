# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **query-execution gateway** (package name `pg-connection-pool`; the directory is `db-query` — the mismatch is expected). Fastify + `pg` + `zod` + `pino`, TypeScript ESM (`"type": "module"`, `NodeNext`). It owns one `pg.Pool` per *logical datasource* and runs SQL on a caller's behalf under guardrails. Callers know a datasource **name** + SQL — never a connection, credentials, host, or engine. Two transports expose the same capability: HTTP (`src/server.ts`) and MCP (`src/mcp/mcp-server.ts`).

`docs/security-and-operations.md` is the authoritative rationale for every security control; `README.md` is the concise quick start that links into it. This file is the code map. Prefer linking to the security doc over restating it.

## Commands

```bash
npm run dev            # tsx watch src/server.ts (HTTP)
npm run build          # tsc → dist/
npm start              # node dist/server.js (HTTP; --env-file-if-exists=.env)
npm run start:mcp      # MCP over stdio (needs MCP_TOKEN)
npm run start:mcp:http # MCP over streamable HTTP (MCP_TRANSPORT=http)
npm run inspect        # build + @modelcontextprotocol/inspector against dist/
npm run typecheck      # tsc --noEmit (src only — what the build uses)
npm run typecheck:test # tsc --noEmit -p tsconfig.test.json (test/**)
npm run typecheck:all  # both — the static gate to run before calling work done
npm test               # node --import tsx --test "test/**/*.test.ts"
```

Single file / single test:

```bash
node --import tsx --test test/relation-guard.test.ts
node --import tsx --test --test-name-pattern="DISCARD ALL" test/query-service.test.ts
```

- **There is no linter or formatter configured.** `npm run typecheck:all` is the only static gate.
- **`typecheck` covers `src/` only; `typecheck:test` covers `test/`.** `tsconfig.json` scopes `include`/`rootDir` to `src/**` because it also drives the `dist/` build — tests must never be emitted. `tsconfig.test.json` extends it with `noEmit`, `rootDir: "."` and `test/**` in scope. Before it existed, `tsx` stripped test types without checking them and type errors in `test/` surfaced only at runtime, or not at all. **Run `typecheck:all`, not `typecheck`** — the narrow one still cannot see `test/`.
- **Integration tests skip by default** (currently 20 of 378; configured, the suite is 378/378). `npm test` loads no env file, so export the vars in the shell:
  ```bash
  PGCP_TEST_HOST=localhost PGCP_TEST_USER=postgres PGCP_TEST_PASSWORD=postgres \
  PGCP_TEST_DATABASE=postgres npm test
  ```
  They fall back to `DATABASE_*`, and create/drop throwaway `pgcp_test_a` / `pgcp_test_b` schemas. `docker compose up -d` starts a local PG 18 that reads its credentials from the same `.env` keys.

## Architecture

### One choke point

`QueryService.run()` (`src/query/query-service.ts`) is where **every** guardrail and the audit line live. HTTP `/query`, MCP `run_query`, introspection, and the boot posture probe all funnel through it. This is why routes and MCP tools contain no guard logic — they only authenticate, authorize, and map errors to a status. When adding a control, add it here, not per-transport.

Both entrypoints build identical services through `buildServices()` in `src/services.ts`, so guardrails/auth/audit cannot diverge between HTTP and MCP. The driver is injectable there — that is what lets tests drive the full path against `StubDriver` with no live Postgres.

### Guard pipeline (all pre-DB-contact, in this order)

1. `assertSingleStatement` (`query/single-statement.ts`) — **always on**, never relaxed.
1b. **Datasource write gate** — `input.write && !dsCfg.writable` → 403. The *third* write gate, after the write-mode token and explicit `readOnly:false` (both in `TokenAuth.authorize`). Deliberately **outside** the `allowUnsafeStatements` blocks: that escape hatch relaxes SQL *shape* guards and must never become a way to earn writes on a read-only datasource. Also precedes the `InternalTrust` bypass, so it covers introspection and the boot probe — inert there, since every internal caller passes `write: false`.
2. `assertStatementAllowed` (`query/statement-guard.ts`) — leading-keyword allowlist (read: `SELECT WITH EXPLAIN VALUES TABLE`; write adds `INSERT UPDATE DELETE MERGE`) + banned-function text scan.
3. `assertRelationsAllowed` (`query/relation-guard.ts`) — real Postgres parser (`libpg-query`) tree walk: system-schema block, token schema caps, unqualified `pg_%`, denied tables, built-in sensitive-relation denylist, connection-identity functions. **Fails closed** — unparseable SQL is a 400.
4. Clamps (`timeoutMs` ≤ datasource `statementTimeoutMs`, `maxRows` ≤ `maxRowsCeiling`) — a request may only *lower* them.

`DS_<NAME>_ALLOW_UNSAFE_STATEMENTS=true` skips **2 and 3 only**; the single-statement scan and the read-only transaction stay on. Guard 3 is also skipped for the gateway's own fixed catalog SQL via `InternalTrust`. Every rejection is audited before it throws — blocked attempts must appear in the security stream.

### Transaction wrap (asserted verbatim by unit tests)

```
BEGIN [TRANSACTION READ ONLY]
SET LOCAL statement_timeout
SET LOCAL idle_in_transaction_session_timeout    (statement_timeout + 5s)
SET LOCAL search_path TO "<schema>", pg_temp
<caller sql>
COMMIT                       (ROLLBACK on error)
DISCARD ALL                  (skipped only if ROLLBACK itself failed)
release() in finally
```

`SET LOCAL search_path` is the **P0 tenant-isolation invariant** — a pooled connection is a long-lived session, so a plain `SET` would leak to the next borrower. `DISCARD ALL` is what actually returns the connection pristine: `SET LOCAL` only auto-resets settings *we* issue, and a caller's own plain `SET` survives COMMIT. `pg_temp` is named last on purpose (unnamed, Postgres searches it *first*).

### Engine-neutral boundary

Only `driver/postgres-driver.ts` and `pool/pool-manager.ts` import `pg`. Everything above talks to the `QueryDriver` interface (`connect` → `exec`/`release`, `ping`). Keep it that way — it is what makes `StubDriver` (`test/helpers.ts`) able to record the exact `exec()` sequence.

### Shared definitions that must not drift

Several policies are deliberately defined once and consumed twice; duplicating them is how a bypass gets reintroduced:

- `query/banned-functions.ts` → the text scan (`statement-guard`) **and** the parse-tree scan (`relation-guard`). The text scan catches quoted calls; the parser scan catches unicode-escape identifiers.
- `isSystemSchema` — exported from `relation-guard.ts`, reused by `IntrospectService.visibleSchema`.
- `capabilityAllows` — exported from `auth/token-auth.ts`, reused by the relation guard's cross-schema check so `authorize()` and the guard agree on `'*'`.
- `stripToCode` (`query/sql-lexer.ts`) — the one tokenizer behind the single-statement scan and the statement guard; same-length blanking keeps offsets stable. `revealQuotedIdents` flips quoted-identifier handling per consumer (blank for `;` scanning, reveal for function scanning).
- `parseEnvFile` (`config/env-file.ts`) — the one env parser behind `.env` (on reload) and `datasources.d/*.env`. Its two dialects share the quoted-value *span* step and differ only on whether to accept it; that shared span is what stops text inside a multi-line value becoming a real top-level key.
- `list` (`config/load-config.ts`) — the one comma-splitter behind `DATASOURCES`/`TOKENS` expansion **and** the cross-source collision check. If those two disagreed about which names exist, the check would guard a different set than the one that gets credentials.

### Config

**Entry point is `config/config-sources.ts`, not `load-config.ts`.** `resolveConfig()` is what `server.ts` and `mcp/mcp-server.ts` call; `gatherConfigSources()` merges every source into one env-shaped map, and `loadConfig()` only validates that map. Reload calls `gatherConfigSources` directly (it must *refuse* rather than throw). Editing `load-config.ts` alone will miss `.env` parsing, the `datasources.d/` directory, the `DATABASE_*` fallback materialization, and the cross-source collision check.

Sources merge lowest-precedence first: `process.env` → `.env` → `DATABASE_*` fallback (only when nothing else defines a datasource) → `datasources.d/*.env`.

- `config/load-config.ts` — expands `DATASOURCES` / `TOKENS` id lists into `DS_<NAME>_*` / `TOKEN_<ID>_*` keys and validates with zod (`config/config.schema.ts`); invalid config is a boot failure. Token→datasource cross-references are checked at load.
- `config/datasources-dir.ts` — reads `datasources.d/*.env`, enforces mode `600`, derives the name from the filename, normalizes bare keys to `DS_<NAME>_*`, rejects unknown keys.
- `config/env-file.ts` — the one env-file parser, with **two dialects**. `.env` gets the `node` dialect because `node --env-file` also parses it at boot and any divergence means boot and reload disagree; `datasources.d/` gets the `credential` dialect (`#` literal, no escape expansion) because nothing else reads those files and truncating a password at a `#` is worse than keeping it.
- `config/reload.ts` — the SIGHUP orchestrator. Stage 1 validates all-or-nothing; stage 2 probes and degrades per datasource. See README → Runtime reload.

**Boot and reload MUST share one rule set** (design decision 6). They read the same sources through the same function; the only sanctioned divergence is that an unreachable datasource is fatal at boot and withheld on reload.

Booleans are converted in the loader, never by `z.coerce.boolean()` (`Boolean("false") === true`). Two converters with different failure directions:

- `bool()` — `"true"` ⇒ true, anything else false. For opt-*in* dangerous flags (`ALLOW_UNSAFE_STATEMENTS`).
- `boolSecureDefault()` — only an explicit `false`/`0`/`no`/`off` turns it off. For secure-by-default safety nets (`SENSITIVE_RELATION_DENYLIST`), so a typo keeps the net on.

Boot (`boot/assert-readonly-posture.ts`) probes each datasource's *DB role* privileges and logs `read-only posture OK / WEAK / UNVERIFIED`. It is WARN-level, never fatal, and fails closed — an unexpected result shape reports UNVERIFIED, never OK. It also warms the `libpg-query` WASM parser so a broken install is loud at boot rather than as a 400 on someone's query.

## Do not break these

Each has a documented exploit history; reverting one silently reopens it.

- **`PostgresDriver.exec` pins `queryMode:'extended'`.** `pg` selects the wire protocol by whether params were supplied — the two-arg `client.query(sql, params)` form uses the *simple* protocol, which runs `a; b` from one call. That let a smuggled `COMMIT` end `BEGIN TRANSACTION READ ONLY` and hand a read token a read-write session. The text scanner is only a fast 400; the protocol is the structural defense.
- **`InternalTrust` is the second positional argument to `run()`, never a `RunInput` field.** An HTTP body or MCP args object can be spread into `RunInput`, but it can never become argument #2. Never widen it to carry caller-supplied SQL.
- **Caller-facing error text goes through `redactErrorMessage`** (`query/redact-error.ts`) on both transports. Full detail belongs in the server-side audit log only.
- **Audit logs SQL text and errors but never `params`** — so values must be passed as `$1…`, never inlined as literals.
- **Authorization order**: datasource membership (403) before existence (400), so an unauthorized token cannot enumerate datasources.

## Repo conventions

- Comments in `src/` are unusually dense and explain *why* — mostly the attack a line prevents. Match that register when touching guard code; a change that removes a comment removes the reason someone won't undo it.
- **Three documentation schemes now coexist. They are not redundant — pick by what you are writing.**
  - `docs/ai/<phase>/<YYYY-MM-DD>-feature-<slug>.md` — the **ai-devkit lifecycle** docs (`requirements`, `design`, `planning`, `implementation`, `testing`, `deployment`, `monitoring`), one file per phase per feature, validated by `npx ai-devkit@latest lint --feature <slug>`. Use this for anything driven through the `dev-*` skills. The `design` doc is the intent and the `implementation` doc is what shipped — when they disagree, **amend the design doc rather than silently deviating**, or phase-7 review reads the difference as an undocumented deviation.
  - `plans/<YYYYMMDD-HHMM>-<slug>/plan.md` + `phase-NN-*.md` — multi-phase implementation plans predating the above. Still valid; don't migrate them.
  - `docs/design-notes/`, `docs/journals/`, `docs/runbooks/` (dated filenames, flat folders) — standalone notes, incident/bypass writeups, and operator procedures. Not feature-scoped.
- This repo predates the dated-**folder** scheme in the global instructions (`docs/{YYYY_MM_DD}/tasks/…`); follow what is here rather than introducing it.
- `.claude/agent-memory/code-reviewer/` holds writeups of three real bypasses found in review (quoted-identifier scan bypass, untagged-`RangeVar` write-target hole, advisory schema caps). Read them before changing the guards — each describes a hole that looked closed.
- `docs/runbooks/agent-ro-pg-role.md` — the read-only DB role. Privilege changes are a **user action**; never run them.
