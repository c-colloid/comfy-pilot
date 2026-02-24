# Comfy Pilot

[![Stars](https://img.shields.io/github/stars/ConstantineB6/Comfy-Pilot)](https://github.com/ConstantineB6/Comfy-Pilot/stargazers)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![ComfyUI Registry](https://img.shields.io/badge/ComfyUI-Registry-blue)](https://registry.comfy.org/publishers/constantine/nodes/comfy-pilot)

Talk to your ComfyUI workflows. Comfy Pilot gives Claude Code direct access to see, edit, and run your workflows — with an embedded terminal right inside ComfyUI.

![Comfy Pilot](thumbnail.jpg)

## Why?

Building ComfyUI workflows means manually searching for nodes, dragging connections, and tweaking values one at a time. With Comfy Pilot, you just describe what you want:

- *"Build me an SDXL workflow with ControlNet"* — Claude creates all the nodes, connects them, and sets the parameters
- *"Look at the output and increase the detail"* — Claude sees your generated image and adjusts the workflow
- *"Download the FLUX schnell model and set up a workflow for it"* — Claude downloads the model and builds a workflow from scratch

No copy-pasting node names. No hunting through menus. Just say what you want.

## Installation

**CLI (Recommended):**
```bash
comfy node install comfy-pilot
```

**ComfyUI Manager:**
1. Open ComfyUI
2. Click **Manager** → **Install Custom Nodes**
3. Search for "Comfy Pilot"
4. Click **Install**
5. Restart ComfyUI

**Git Clone:**
```bash
cd ~/Documents/ComfyUI/custom_nodes && git clone https://github.com/ConstantineB6/comfy-pilot.git
```

Claude Code CLI will be installed automatically if not found.

## Requirements

- ComfyUI
- Python 3.8+
- One of the following Claude clients:
  - **Claude Code CLI** (macOS / Linux) — embedded terminal experience
  - **Claude Desktop** (macOS / Windows / Linux) — use MCP tools from the desktop app

> **Note:** ComfyUI must be open in your browser for both clients. The MCP tools interact with the workflow through the browser frontend.

## Features

- **MCP Server** - Gives Claude direct access to view, edit, and run your ComfyUI workflows
- **Embedded Terminal** - Full xterm.js terminal running Claude Code right inside ComfyUI (CLI mode)
- **Claude Desktop Support** - Use MCP tools directly from Claude Desktop app
- **Image Viewing** - Claude can see outputs from Preview Image and Save Image nodes
- **Graph Editing** - Create, delete, move, and connect nodes programmatically

## Demo

https://github.com/user-attachments/assets/325b1194-2334-48a1-94c3-86effd1fef02

## Usage

### With Claude Code CLI (macOS / Linux)

1. Restart ComfyUI after installation
2. The floating Comfy Pilot terminal appears in the top-right corner
3. The MCP server is automatically configured for Claude Code
4. Ask Claude to help with your workflow

### With Claude Desktop (macOS / Windows / Linux)

1. Restart ComfyUI after installation
2. The Comfy Pilot panel shows Desktop mode status
3. Click **"Setup Desktop MCP"** to configure (or it auto-configures on startup)
4. **Restart Claude Desktop** to load the new MCP server
5. Open Claude Desktop and use the ComfyUI tools
6. Keep ComfyUI open in your browser while using Claude Desktop

**Example prompts:**
- "What nodes are in my current workflow?"
- "Add a KSampler node connected to my checkpoint loader"
- "Look at the preview image and tell me what you see"
- "Run the workflow up to node 5"

## MCP Tools

The MCP server provides these tools to Claude Code:

| Tool | Description |
|------|-------------|
| `get_workflow` | Get the current workflow from the browser |
| `summarize_workflow` | Human-readable workflow summary |
| `get_node_types` | Search available node types with filtering |
| `get_node_info` | Get detailed info about a specific node type |
| `get_status` | Queue status, system stats, and execution history |
| `run` | Run workflow (optionally up to a specific node) or interrupt |
| `edit_graph` | Batch create, delete, move, connect, and configure nodes |
| `view_image` | View images from Preview Image / Save Image nodes |
| `search_custom_nodes` | Search ComfyUI Manager registry for custom nodes |
| `install_custom_node` | Install a custom node from the registry |
| `uninstall_custom_node` | Uninstall a custom node |
| `update_custom_node` | Update a custom node to latest version |
| `download_model` | Download models from Hugging Face, CivitAI, or direct URLs |

### Example: Creating Nodes

```
Create a KSampler and connect it to my checkpoint loader
```

Claude will use `edit_graph` to:
1. Create the KSampler node
2. Connect the MODEL output from CheckpointLoader to KSampler's model input
3. Position it appropriately in the graph

### Example: Viewing Images

```
Look at the preview image and describe what you see
```

Claude will use `view_image` to fetch and analyze the image output.

### Example: Downloading Models

```
Download the FLUX.1 schnell model for me
```

Claude will use `download_model` to download from Hugging Face to your ComfyUI models folder. Supports:
- Hugging Face (including gated models with token auth)
- CivitAI
- Direct download URLs

## Terminal Controls

- **Drag** title bar to move
- **Drag** bottom-right corner to resize
- **−** Minimize
- **×** Close
- **↻** Reconnect session

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Browser (ComfyUI)                                  │
│  ┌─────────────────┐  ┌──────────────────────────┐  │
│  │  xterm.js       │  │  Workflow State          │  │
│  │  Terminal (CLI) │  │  (synced to backend)     │  │
│  │  — or —         │  │                          │  │
│  │  Desktop Panel  │  │                          │  │
│  └────────┬────────┘  └────────────┬─────────────┘  │
│           │ WebSocket              │ REST API       │
└───────────┼────────────────────────┼────────────────┘
            │                        │
            ▼                        ▼
┌─────────────────────────────────────────────────────┐
│  ComfyUI Server                                     │
│  ┌─────────────────┐  ┌──────────────────────────┐  │
│  │  PTY Process    │  │  Plugin Endpoints        │  │
│  │  (CLI mode)     │  │  /claude-code/*          │  │
│  └─────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────┘
            │                        │
            │                        ▼
            │           ┌──────────────────────────┐
            └──────────▶│  MCP Server              │
                        │  (stdio transport)       │
Claude Code CLI ────────│                          │
  — or —                │                          │
Claude Desktop ─────────│                          │
                        └──────────────────────────┘
```

## Files

- `__init__.py` - Plugin backend: WebSocket terminal, REST endpoints, Desktop config
- `js/claude-code.js` - Frontend: xterm.js terminal, Desktop status panel, workflow sync
- `mcp_server.py` - MCP server for Claude Code integration
- `CLAUDE.md` - Instructions for Claude when working with ComfyUI

## Troubleshooting

### "Command 'claude' not found"

Install Claude Code CLI:

**macOS / Linux / WSL:**
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows (PowerShell):**
```powershell
irm https://claude.ai/install.ps1 | iex
```

**Windows (CMD):**
```cmd
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

### MCP server not connecting

The plugin auto-configures MCP on startup. Check ComfyUI console for errors, or manually add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "comfyui": {
      "command": "python3",
      "args": ["/path/to/comfy-pilot/mcp_server.py"]
    }
  }
}
```

### Terminal disconnected

Click the ↻ button to reconnect, or check ComfyUI console for errors.

### Claude Desktop: MCP tools not appearing

1. Make sure ComfyUI is running and the Comfy Pilot panel shows "Configured"
2. **Restart Claude Desktop** after the MCP config is written
3. Check config file exists at:
   - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
   - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
   - Linux: `~/.config/Claude/claude_desktop_config.json`

### Claude Desktop: "Failed to connect to ComfyUI"

ComfyUI must be running and open in your browser. The MCP tools interact with the workflow through the browser frontend — the browser tab must stay open.

## License

MIT
