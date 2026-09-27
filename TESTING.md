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

### Site / coordinates / MEP / IFC tools (branch `feature/mep-topo-coords-ifc`, 2026-09-26)

| check | command | result |
|---|---|---|
| Command set compiles for every Revit version | `dotnet build commandset\RevitMCPCommandSet.csproj -c "Release R23"` … `"Release R27"` with `-p:PublishAddinFiles=false` | 0 errors on R23, R24, R25, R26, R27 |
| Solution compiles | `dotnet build mcp-servers-for-revit.sln -c "Release R27" -p:PublishAddinFiles=false` | 0 errors |
| Revit API tests (all) | `tests\commandset\bin\Release.R27\RevitMCPCommandSet.Tests.exe` | 155 passed / 0 failed (~25 s) |
| New tests only | same exe with `--treenode-filter "/*/*/SiteMepCoreTests/*"` | 15 passed |
| Server typecheck / bundle / schemas | `npm run build:check`, `node esbuild.config.mjs`, `node generate-tool-schemas.mjs` | OK, 147 tools |

`SiteMepCoreTests` differ from the older tests: they compile the production
`commandset/Services/SiteMep/SiteMepCore.cs` (and its request models) into the test
project, so they do exercise this repo's logic for `get_project_location`,
`set_shared_coordinates`, `create_toposolid`, `get_toposolids`, `create_pipe`,
`create_duct`, `get_mep_systems`, `get_mep_elements` and `export_ifc` against a document
created from `English\Default-Multi-Discipline_Metric.rte`. What they establish:

- `ActiveProjectLocation.GetTotalTransform()` maps shared → internal (its inverse maps
  internal → shared); after `set_shared_coordinates` the chosen internal point reports
  exactly the requested E/N/elevation, the angle round-trips, and the internal→shared
  rotation has the same sign as `angleToTrueNorth_deg`.
- `Toposolid.Create` (points overload) takes absolute internal Z; shared-coordinate input
  lands where expected; dryRun leaves no element.
- Pipes/ducts get the requested size, a piping/mechanical system is created for them, a
  failing item rolls back the whole batch.
- The IFC exporter honours the `SitePlacement` option: with shared coordinates at
  E 500 000 / N 4 000 000 / Z 100 000 mm the IfcSite placement point is
  `(500000., 4000000., 100000.)` for `useSharedCoordinates: true` and `(0., 0., 0.)` for
  `false`; IFC4 and IFC2X3 FILE_SCHEMA headers are as requested.

Live verification on Revit 2027.2 is recorded under "Live" below. After the
`FailureCollector` fix the SiteMep suite is **16 passed** (adds the overlapping-floor
warning test), and the Node suite is **1540 passed** with the nine tools' fixtures.

Gotcha found while writing them: a generic test helper constrained to a Revit type
(`static string FirstName<T>() where T : Element`) makes Nice3point's injector fail for the
whole session with "Attempted to write protected memory"; use a `Type` parameter instead.

Two things to know about the 140 original tests:

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
| Site / coordinates / MEP / IFC tools | see the 2026-09-26 table below | passed after one fix (modal warning dialog) |
| Port slide | hold `127.0.0.1:8080` with a dummy listener, toggle the switch off/on | plugin bound 8081, `mcp-port.txt`=8081, server followed it |

The official Autodesk read-only server stalls (>120 s) while Revit has a dialog open
and recovers when it is dismissed — documented behaviour, not a fault in this repo.

### Site / coordinates / MEP / IFC tools, live on Revit 2027.2 (2026-09-26)

Add-in built from `joshua8-main` (Release R27) and copied into
`%APPDATA%\Autodesk\Revit\Addins\2027`; commands sent straight to the plugin on 8080
(newline-delimited JSON-RPC, the same wire path the Node server uses; the Node layer is
covered by the contract suite). Models: scratch copies of the 2027 samples
`Snowdon Towers Sample Site.rvt` and `Snowdon Towers Sample Plumbing.rvt`.

| tool | result |
|---|---|
| `get_project_location` (Site) | Default Site, 40.0224 N / 79.8867 W, PBP 26.0156° to true north |
| `set_shared_coordinates` | dryRun rolled back; +1000 mm E / 30° committed and read back exactly; restore read back exactly |
| `get_toposolids` / `get_mep_systems` / `get_mep_elements` | 11 toposolids (Site); 71 systems / 3,051 pipes (Plumbing), counts match the official server |
| `create_toposolid` dryRun (Site, shared coords) | 5 points, rolled back, < 1 s |
| `create_toposolid` real (Site) | **bug:** "Highlighted toposolid and floor overlap" opened a modal dialog at commit; the external event could not finish, and every later command (including `get_project_info`) timed out until the dialog was clicked. Fixed in `f6c3058` (`FailureCollector`: warnings returned in `warnings`, errors roll back); Revit test `CreateToposolid_OverlappingFloor_ReturnsWarningInsteadOfDialog` |
| `set_shared_coordinates` after the fix (via the bridge) | Revit's "survey point is outside the current coordinate reference system" came back in `warnings`, no dialog |
| `create_toposolid` after the fix (via the bridge, 403 points) | committed in ~0.2 s; "Highlighted toposolids overlap" returned in `warnings`, no dialog |
| `create_pipe` dryRun / real (Plumbing) | 2" Domestic Cold Water, new system; 5,000 mm |
| `create_duct` real (Plumbing) | 10" round Exhaust Air |
| `export_ifc` IFC4, `useSharedCoordinates: true` (Site) | 7.4 MB in 13.6 s; IfcSite placement (1370151.795, 258247.976, 780.5) ft = the model's shared coordinates, rotation 26.0156°, `IFCPROJECTEDCRS` EPSG:2272 |
| `delete_element` of the test elements | 4 deleted (note: `dryRun` defaults to true) |

Official Autodesk read-only server, same session: `get_running_revit_instances` and
`query_model` fine (now also list the 4 `OST_RvtLinks` instances);
`get_element_data` with `parametersOutputType: "SpecificParameters"` always fails with
"SpecificParameterNames must be provided…" whether the names are sent camelCase or
PascalCase — an Autodesk preview bug; `KeyParameters` works.

## Known upstream noise

`npx knip` in `server/` reports 138 "unused exports" — the `register*Tool` functions
are listed as entries in `knip.json` so their exports are flagged; inherited, not
introduced here.
