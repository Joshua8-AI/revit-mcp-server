# Testing record (Joshua8-AI fork)

How the Revit 2027 from-source path is verified, and the last recorded results.

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

Not yet done for these tools: **live verification over MCP** (plugin loaded in a running
Revit 2027, called through the server). The C# command classes, JSON parameter binding and
the Node tool wrappers are therefore compiled but not exercised end-to-end.

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
| Site / coordinates / MEP / IFC tools | `get_project_location`; `set_shared_coordinates` dryRun then real; `create_toposolid` (shared); `get_toposolids`; `create_pipe` / `create_duct` dryRun then real; `get_mep_systems`; `get_mep_elements`; `export_ifc` dryRun then real | **pending** — not yet run live |
| Port slide | hold `127.0.0.1:8080` with a dummy listener, toggle the switch off/on | plugin bound 8081, `mcp-port.txt`=8081, server followed it |

The official Autodesk read-only server stalls (>120 s) while Revit has a dialog open
and recovers when it is dismissed — documented behaviour, not a fault in this repo.

## Known upstream noise

`npx knip` in `server/` reports 138 "unused exports" — the `register*Tool` functions
are listed as entries in `knip.json` so their exports are flagged; inherited, not
introduced here.
