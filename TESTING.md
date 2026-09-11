# Testing record (Joshua8-AI fork)

How the Revit 2027 from-source path is verified, and the last recorded results.

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
