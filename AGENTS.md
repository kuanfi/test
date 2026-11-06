<!-- BEGIN craftland-studio-asset-rules -->
## Synced from craftland-studio-assets.mdc

# Craftland Studio Asset Rules

## Core Rule

Craftland Studio is the authoritative editor for game assets in this project.

- Any access to or modification of game assets must use Craftland Studio.
- Only `.fcc` and `.fcg` files may be edited directly as text and compiled through the FC toolchain.
- If a task involves non-text game assets, especially `.eca` files, the agent must first check whether Craftland Studio is available through MCP.
- If Craftland Studio is not available for the current session, the agent must tell the user to open the game editor first before attempting that asset task.
- The agent must not guess or hand-edit serialized asset formats such as `.eca` when Craftland Studio is required but unavailable.

## Directory Scope

The following directories belong to the Craftland Studio project structure:

- `Assets/` user asset files, including `.fcc` and `.fcg`
- `ProjectSettings/` user configuration asset files
- `Temp/` temporary files, records, debug output, and compilation results
- `Libraries/` library files
- `Cache/` project cache

## Working Rules

- For requests involving `.fcc` or `.fcg`, direct text editing is allowed.
- For requests involving assets under the directories above that are not `.fcc` or `.fcg`, prefer Craftland Studio and related MCP tools.
- Before handling `.eca`, entity attachment, asset registration, scene assets, workflow assets, or other editor-owned assets, first verify whether Craftland Studio MCP is connected.
- Once Craftland Studio MCP is connected, the agent must read `instructions://index` before using other Craftland Studio MCP tools or resources.
- If the MCP is unavailable, stop and ask the user to open Craftland Studio, then retry the task after the editor-side connection is ready.
<!-- END craftland-studio-asset-rules -->

<!-- BEGIN craftland-studio-fc-workflow -->
# Craftland Studio FC Agent Entry

This managed section is for any agent that reads the project root `AGENTS.md`. Craftland Studio sync must update only this section and must not overwrite the user's full `AGENTS.md`.

## Scope

This is a Craftland Studio UGC project. Any task involving `.fcg`, `.fcc`, FC APIs, FC imports, FC diagnostics, or FC generated project symbols is an FC task.

FC tasks mainly involve:

- `Assets/`: user-owned `.fcg` / `.fcc` source files.
- `Temp/UGCLanguage/`: toolchain config, generated libraries, symbol tables, and referenced package caches.

## Required Entry Point

Every FC coding task must read and strictly follow the official Craftland Studio agent workflow:

- `.craftland/agents/rules/fc-workflow.mdc`

Skills must be loaded from:

- `.craftland/agents/skills/`

If the official `.craftland/agents/` workflow is missing or cannot be read, stop and report the missing file. Do not bypass the workflow and handle FC tasks directly.

Legacy `.cursor/rules/` and `.cursor/skills/` FC rules may still exist in older projects, but Craftland Studio no longer maintains them. When `.craftland/agents/` exists, it has higher priority than any existing `.cursor` FC rules.

## Non-Negotiable Rules

- Do not invent FC APIs, types, events, enum values, component names, imports, assetIds, entityIds, or generated project symbols from memory.
- Craftland Studio is the authority for asset registration. Agents must not fake registration by writing FC code; register assets through the Craftland Studio MCP `fc-asset-registration-*` tools or ask the user to register manually.
- Generated files under `Temp/UGCLanguage/editorGen/` are read-only references and must not be edited as user source.
- Any `.fcg` / `.fcc` edit is not complete until full `fccompile.exe -i Assets` validation or an equivalent Craftland Studio MCP build validation has run.
<!-- END craftland-studio-fc-workflow -->
