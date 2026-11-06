---
description: Craftland Studio asset handling rules for ECA and non-FC assets
globs: *
applyTo: "**/*"
alwaysApply: true
---

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
