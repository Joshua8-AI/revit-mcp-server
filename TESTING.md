# Testing record (Joshua8-AI fork)

How the Revit 2027 from-source path is verified, and the last recorded results.

## Node server suite (no Revit needed — runs on hosted CI)

```
cd server
npm ci
npm run build:check   # tsc over src/ and tests/
npm test              # vitest run
```

Last result (2026-09-26, `feature/node-contract-tests`, Node 24 on Windows 11):
**7 files, 1450 passed / 0 failed, ~7 s.** CI (`ci.yml` job `server-tests`) runs the same
three commands on `ubuntu-latest` and `windows-latest` for every PR and push.

| file | tests | what it covers |
|---|---|---|
| `tests/registration.test.ts` | 8 | real `McpServer` + `registerTools()`: no swallowed registration errors, each tool registered once, `register.ts` list == `src/tools/*.ts` == registered names, descriptions/schemas present; `plugin/tool_schemas.json` and `tool-schemas.txt` equal a live `tools/list` (server↔plugin drift) |
| `tests/tool_contracts.test.ts` | 1385 | table-driven over **every** registered tool (138): schema-generated full and required-only inputs send exactly one command with the expected name and every field passed through; invalid input (missing required / wrong type) rejected via a real MCP client and never sent; Revit error, connection failure and null/undefined/string/`{}`/array replies give a well-formed MCP result, never a throw; `command.json` ⇔ C# `CommandName`s ⇔ tools |
| `tests/socket_client.test.ts` | 11 | `RevitClientConnection` vs a fake plugin socket: JSON-RPC framing, chunked and out-of-order responses, junk lines, error replies, command timeout, disconnects |
| `tests/connection_manager.test.ts` | 17 | `mcp-port.txt` discovery (8080 → 8081–8089 slide, newest year first, invalid files, `REVIT_MCP_PORT`), end-to-end slide to the first free port, refused connection + backoff, recovery mid-retry, no retry on timeout / Revit error, concurrent calls serialised, mutex release, no leaked timer |
| `tests/database.test.ts` | 9 | sql.js store under a temp HOME: schema, deferred save to disk, project upsert, metadata, rooms batch/order/join, zero area, stats |
| `tests/utils.test.ts`, `tests/token_logger.test.ts` | 17 + 3 | errorUtils, suggestions, responseCompactor, compactTool helpers, tokenLogger |

**Adding a tool:** it is discovered automatically; the contract suite fails with the tool's
name until it has an entry in `server/tests/fixtures/toolFixtures.ts` — usually just
`my_tool: {},` (add `command`, `renamed`, `notForwarded`, `fixedParams` or `input` only if
the handler deviates). Regenerate the schema files with
`node esbuild.config.mjs && node generate-tool-schemas.mjs` (not `npm run build`, which
also deploys into the live Revit add-in folder).

Tests never touch real machine state: `tests/setup.ts` points `HOME`/`USERPROFILE`/`APPDATA`
at a temp dir, and the socket tests use fake servers on loopback.

## Offline (Revit installed but not running)

| check | command | last result (2026-09-11, `joshua8-main`) |
|---|---|---|
| Plugin + command set + test project compile | `dotnet build mcp-servers-for-revit.sln -c "Release R27"` | 0 errors (69 pre-existing nullability warnings) |
| Revit API tests | `dotnet test tests\commandset\RevitMCPCommandSet.Tests.csproj -c "Release R27"` | 140 passed / 0 failed (~48 s) |
| Server build + add-in seeding | `cd server && npm run build` | `Build complete`, deploys to the 2027 add-in |
| Workflow files parse | `npx -y js-yaml .github/workflows/ci.yml` (and `release.yml`) | OK |

Two things to know about the 140 tests:

- They **require Revit 2027 installed** (Nice3point.TUnit.Revit boots it from
  `C:\Program Files\Autodesk\Revit 2027`); they do not require it to be running. This is
  why `ci.yml` only builds on hosted runners and gates the test job on a self-hosted
  `revit-2027` runner.
- They exercise the Revit API itself, not this repo's command handlers — none reference
  `RevitMCPCommandSet.*`. Treat them as an API/SDK smoke test, not coverage of the tools.

`server/deploy-addins.mjs` cases (run against an isolated `%APPDATA%`): fresh
plugin+command set → seeds `build/index.js`, `sql-wasm.wasm`, `package.json`; no Addins →
exits 0 with "no plugin folders found"; plugin without command set → warning; missing
`server/build/index.js` → exits 1 "run `npm run build` first"; re-run → idempotent, stale
`package.json` overwritten.

## Live (Revit 2027 open, model loaded, Revit MCP Switch on)

| scenario | how | last result |
|---|---|---|
| Both servers see the instance | official `get_running_revit_instances`; `revit-write get_project_info` | same pid, Snowdon Towers Sample Architectural |
| Write round-trip | `create_level` (no plan views) → official `query_model` for it → `delete_element` → official `query_model` again | created id 3327602, found by official server, deleted, 0 found, 18 levels remain |
| Port slide | hold `127.0.0.1:8080` with a dummy listener, toggle the switch off/on | plugin bound 8081, `mcp-port.txt`=8081, server followed it |

The official Autodesk read-only server stalls (>120 s) while Revit has a dialog open
and recovers when it is dismissed — documented behaviour, not a fault in this repo.

## Known upstream noise

`npx knip` in `server/` reports 138 "unused exports" — the `register*Tool` functions
are listed as entries in `knip.json` so their exports are flagged; inherited, not
introduced here.
