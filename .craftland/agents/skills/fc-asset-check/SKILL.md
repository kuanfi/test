---
name: fc-asset-check
description: Trigger when a task involves Craftland Studio assets, UI, scene entities, assetIds, entityIds, script attachments, button bindings, or current editor state. For asset registration, use fc-asset-registration.
---

# Craftland Studio Asset Validity Check

Use this skill before referencing or modifying Craftland Studio assets, scene entities, UI controls, assetIds, entityIds, script attachments, button callbacks, or other editor-owned data.

## Goal

Do not invent project assets, entities, assetIds, entityIds, UI controls, script attachments, or button bindings. Craftland Studio is the authority for asset registration and current editor state.

This skill only checks Craftland Studio asset validity and current editor state. Use `fc-symbol-lookup` for FC-referenceable symbols, APIs, types, custom events, custom enums, custom components, declare graphs, public properties, and similar FC declarations.

## Decide The Scenario First

1. If you only need to verify FC-referenceable symbols, do not use this skill. Use `fc-symbol-lookup`.
2. If you need to verify assets, scene entities, UI controls, script attachments, button callbacks, or asset properties, use Craftland Studio MCP.
3. If you need to create or modify assets, entities, UI, script attachments, or button callbacks, use MCP and verify the written state afterwards.
4. If FC code references a specific asset, scene, scene entity, or UI widget known at edit time, or a registration must be created or removed, use `fc-asset-registration` (registration-first policy, generated symbol usage, MCP registration workflow). This skill only verifies editor state; it does not own the registration workflow.

## Current Editor Asset Verification

Use this section to confirm what currently exists in the editor, such as asset existence, entity existence, UI control structure, button callbacks, script attachments, dynamic properties, schemas, or asset properties.

When Craftland Studio MCP is available and the task needs current editor state:

1. Read `instructions://index` first.
2. Use `asset-list` / `asset-get` to inspect assets.
3. Use `asset-entity-get` / `asset-entity-search` to inspect entities and UI controls.
4. When verifying or modifying script attachments, use `entity-eca-script-update` and then recheck with `asset-entity-get`.
5. When verifying or modifying UI button callbacks, inspect `button.clickHandlers` and then recheck with `asset-entity-get`.

If MCP is unavailable, do not claim that current editor asset state has been verified. Stop and ask the user to open Craftland Studio or provide an available MCP connection.

## Missing Data Handling

- If an asset or entity is missing from MCP, stop and ask whether the user wants to create/register it manually in Craftland Studio or allow the agent to create/update it through MCP.
- If the asset exists but its FC registration symbol (`ERes*`, `EResKey*`, `Res` member) is missing, register it through `fc-asset-registration` instead of asking the user to do it manually, as long as the target asset and key intent are unambiguous.
- If the asset exists in MCP but the FC symbol is unavailable, tell the user to save the project or regenerate `Temp/UGCLanguage/editorGen/`, then verify the symbol with `fc-symbol-lookup`.
- Read related `.meta` files for fileId confirmation only when MCP does not expose the required fileId.

## Prohibited Actions

- Do not write FC code that assumes a missing asset, entity, assetId, or entityId exists.
- Do not modify generated files under `Temp/UGCLanguage/editorGen/`.
- Do not manually edit editor-owned serialized assets to fake registration, including `ProjectSettings/ResourceRegisteration.asset`.

## After MCP Writes

When the task creates or updates editor-owned data through MCP, verify the written state before writing or delivering FC code:

- After creating or updating an asset, use `asset-get` to confirm that the asset exists and has the expected properties.
- After creating or updating an entity, use `asset-entity-get` to confirm the entity id, type, parent, transform, and relevant dynamic properties.
- After attaching a script, verify the entity `scripts` data with `asset-entity-get`.
- After an asset registration upsert or remove, verify the generated symbols in `EditorGenLib.fcc` through `fc-symbol-lookup`.
- After wiring a UI button, verify `button.clickHandlers` with `asset-entity-get`.
- Treat `asset-refresh` as a disk re-import operation. Do not run it on newly modified MCP assets unless persistence has been confirmed, because refresh may discard editor-memory changes that have not been saved yet.

## Output Check

Before editing or replying, make sure you can state:

- Which user-provided candidate keywords were checked.
- Which MCP response confirmed the asset, entity, UI control, script attachment, or button binding.
- Which assetId, entityId, or UI control path should be used.
- Whether current editor state was verified; if not, state that MCP is unavailable.
- If assets, entities, or UI were modified, which MCP write verification was performed.
- Whether asset registration, project saving, or editorGen regeneration is still needed.
