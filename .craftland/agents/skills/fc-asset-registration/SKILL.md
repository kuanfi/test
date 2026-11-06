---
name: fc-asset-registration
description: Trigger when FC code needs to use a project asset, scene, scene entity, or UI widget — registration is how FC code obtains them — or when creating or removing asset registrations through Craftland Studio MCP.
---

# FC Asset Registration

Asset registration exposes project assets (Prefab, CSV, UI, Scene, Texture, Sound, and other supported types) to FC code. It is the default way for FC code to reference any asset known at edit time, and it also makes the build collect the registered asset into the shipped package.

## Registration-First Policy

Whenever FC code is about to use an asset, scene, scene entity, or UI widget known at edit time, first check `fc-asset-registration-list` to see whether its type is registrable (`supportedTypes`); if it is, register it and use the generated symbols.

Bypasses to reject (they compile, but they are wrong):

- Passing an asset/scene/entity/widget name or id as a string literal to an API parameter that identifies an asset, e.g. `SwitchScene("Fishing Break Scene")` — a display name is not even a valid asset id.
- Locating an edit-time-known entity or widget through a find-by-name API instead of registering it.
- Casting hardcoded id strings to asset id types, or writing enum-like constants for assets.

Name-based lookup APIs and raw string parameters are acceptable only when the type is not registrable or the target is genuinely decided at runtime.

## Generated Symbols

Each registration generates symbols in `EditorGenLib.fcc`, named after the registration key:

- `ERes<Type>` element-typed enums: a member IS a value of the enum's `[element <T>]` type (static access, the default). The `[element ...]` attribute on the declaration in `EditorGenLib.fcc` is the authoritative answer to "what is this member" — read it there instead of assuming. E.g. `[element PrefabID] enum EResPrefab { Mini_Kelly }` members are `PrefabID`s, `[element string]` `EResScene` members are scene id strings, `[element entity]` scene-child members are usable as `entity` directly. An enum without `[element]` is a plain string-key enum (the `EResKey*` family).
- `Res` declare static graph plus `EResKey<Type>` key enums: a runtime map from registration keys to asset ids (dynamic access). E.g. `Res.Sprite Map<EResKeySprite, SpriteID>`. `EResKey*` members are the key strings; they exist for autocomplete and have no other meaning.

Most types are flat `type + key + asset`. UI and Scene are nested — register the parent UI/Scene asset first, then widget/entity children under it. Child symbols are named after the parent key:

- Scene `IsleLand` with entity child `Cone001` generates `EResSceneIsleLand.Cone001` (`[element entity]`, the member is the entity itself) and `Res.IsleLand Map<EResKeyIsleLand, entity>`.
- UI `NewUI` with widget child `MyPic` generates `EResUINewUI.MyPic` (`[element CustomUIWidgetID]`) and `Res.NewUI Map<EResKeyNewUI, CustomUIWidgetID>`.

CSV registration is asset-level only — it produces a `CsvID` for the file, not keys for rows or cells.

## Usage In FC Code (compile-verified)

```
import "EditorGenLib.fcc" as EditorLib
import Res from "EditorGenLib.fcc"   // only needed for dynamic access

// Static access (default): the enum member is the asset id
SwitchScene(EResScene.IsleLand)          // scene id string
var petPrefab = EResPrefab.Mini_Kelly    // PrefabID

// Registered UI asset -> create UI; registered widget -> locate it via API
CreateCustomUI(out var ui, player, EResUI.NewUI)                 // CustomUIAssetID
var pic = GetWidgetFromCustomUI(player, ui, EResUINewUI.MyPic)   // entity<UIWidget>

// Registered scene entity: the enum member is the entity itself
var cone = EResSceneIsleLand.Cone001

// Dynamic access: runtime-computed key indexes the Res map
var spriteID = Res.Sprite[csvKey as EResKeySprite]
```

Usage rules:

- `ERes*` / `EResKey*` enums are referenced unqualified after `import "EditorGenLib.fcc" as EditorLib`.
- The `Res` graph requires its own `import Res from "EditorGenLib.fcc"` and is only for keys computed at runtime (for example read from CSV). A runtime string key must be cast to the `EResKey*` enum: `Res.Sprite[key as EResKeySprite]`. This key cast is the sanctioned dynamic pattern and is different from casting strings to asset id types, which is forbidden.
- Child members follow their `[element ...]` type, and the two nested kinds differ: scene children are `[element entity]` and usable as entities directly; UI widget children are `[element CustomUIWidgetID]` — an id, not an entity. Consume widget ids through `GetWidgetFromCustomUI(owner, uiEntity, widgetID)`, which returns `entity<UIWidget>`; the UI entity comes from `CreateCustomUI` / `CreateCustomUIClient` with the registered `CustomUIAssetID`.

## Registering Through MCP

When a needed registration does not exist yet and Craftland Studio MCP is available:

1. `fc-asset-registration-list` to see supported registration types and existing keys. If `editorSessionActive=true`, ask the user to close the Resource Registration editor; writes are blocked while it is open.
2. Locate the target asset with `asset-list` / `asset-get`, and UI widget / Scene entity children with `asset-entity-search` / `asset-entity-get`.
3. `fc-asset-registration-upsert` with `registrationType`, the asset (`assetId` or `assetPath`), and an explicit `key` (plus `childId` / `childKey` for nested registration). Register directly in one call for the normal case; use `dryRun=true` only when the outcome is uncertain — the key may be invalid or duplicated, or the same asset already has multiple registrations. A failed upsert reports the reason and rolls back, so a routine registration does not need a dryRun round-trip.
4. A successful upsert saves the registration and refreshes EditorGen in one operation (rollback on `editorgen_failed` / `recovery_failed`). Re-verify the new symbol in `EditorGenLib.fcc` through `fc-symbol-lookup` before writing FC code that uses it.

`fc-asset-registration-remove` deletes a registration. Removing a UI/Scene parent is blocked while child registrations exist — remove children first. Prefer `dryRun=true` before removal when existing FC code may reference the symbol.

If Craftland Studio MCP is unavailable, or the request is ambiguous about which asset or key to register, ask the user whether to register manually in Craftland Studio or provide the missing information.

## Constraints

- Registration keys must be valid FC identifiers: letters, digits, underscore, no leading digit, no FC keyword, unique within the type or parent.
- Craftland Studio is the authority for registration state. Do not hand-edit `ProjectSettings/ResourceRegisteration.asset`, `EditorGenLib.fcc`, or anything under `Temp/UGCLanguage/editorGen/`.
- Do not write FC code that references a registration symbol before `fc-symbol-lookup` confirms it exists in `EditorGenLib.fcc`.

## Output Check

Before editing or replying, make sure you can state:

- Which assets the FC code references, and which of them are covered by existing registrations.
- Which registrations were created or removed through MCP, with which keys.
- Which generated symbols (`ERes*`, `EResKey*`, `Res` members) were verified through `fc-symbol-lookup` after registration.
- Whether any asset reference intentionally stayed dynamic (runtime key or non-registrable type) and why.
