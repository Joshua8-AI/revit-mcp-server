# mcp-server-for-revit

MCP server for interacting with Autodesk Revit through AI assistants like Claude.

This package is the MCP server component of [mcp-servers-for-revit](https://github.com/Joshua8-AI/revit-mcp-server). It exposes Revit operations as MCP tools that AI clients can call. The server communicates with the [Revit plugin](https://github.com/Joshua8-AI/revit-mcp-server) over TCP to execute commands inside Revit.

> [!NOTE]
> This server requires the mcp-servers-for-revit Revit plugin to be installed and running inside Revit. See the [full project README](https://github.com/Joshua8-AI/revit-mcp-server) for setup instructions.

## Setup

This server is not installed from npm (the `mcp-server-for-revit` package there is a stale 1.0.0 from the original project). It ships inside each [release ZIP](https://github.com/Joshua8-AI/revit-mcp-server/releases), together with a portable Node.js, under `%APPDATA%\Autodesk\Revit\Addins\<year>\revit_mcp_plugin\Commands\RevitMCPCommandSet\server\`.

**Claude Code** (PowerShell; replace `2026` with your Revit version)

```powershell
$srv = "$env:APPDATA\Autodesk\Revit\Addins\2026\revit_mcp_plugin\Commands\RevitMCPCommandSet\server"
claude mcp add revit-mcp -- "$srv\runtime\node.exe" "$srv\build\index.js"
```

**Claude Desktop**

Claude Desktop → Settings → Developer → Edit Config → `claude_desktop_config.json` (replace `<you>` and `2026`):

```json
{
    "mcpServers": {
        "revit-mcp": {
            "command": "C:\\Users\\<you>\\AppData\\Roaming\\Autodesk\\Revit\\Addins\\2026\\revit_mcp_plugin\\Commands\\RevitMCPCommandSet\\server\\runtime\\node.exe",
            "args": ["C:\\Users\\<you>\\AppData\\Roaming\\Autodesk\\Revit\\Addins\\2026\\revit_mcp_plugin\\Commands\\RevitMCPCommandSet\\server\\build\\index.js"]
        }
    }
}
```

Restart Claude Desktop. When you see the hammer icon, the MCP server is connected.

Built from source? Run `npm install && npm run build` here, then use `node` (Node.js 18+) as the command and this folder's `build\index.js` as the argument.

## Supported Tools (147)

See the [full tool list](https://github.com/Joshua8-AI/revit-mcp-server#supported-tools-147) in the main README for the complete catalog organized by category:

- **Project & Model Info** — project metadata, views, parameters, phases, worksets, links
- **Model Analysis** — AI element filter, health check, clash detection, measurements
- **Materials & Quantities** — material properties, takeoffs, quantities
- **Element Creation** — walls, doors, floors, grids, levels, arrays, structural framing
- **Element Modification** — move, rotate, copy, type swap, parameter writes, phase/workset
- **Views & Sheets** — create views, filters, templates, sheets, viewports, schedules
- **Annotation** — dimensions, text notes, filled regions, auto-tagging
- **Data Export** — room data, element data (JSON/CSV), schedule export, PDF/DWG/IFC
- **Batch Operations** — rename, renumber, purge, CAD cleanup, shared parameters
- **Advanced** — execute C# code, local data storage

## Development

```bash
npm install
npm run build
```

## License

[MIT](https://github.com/Joshua8-AI/revit-mcp-server/blob/joshua8-main/LICENSE)
