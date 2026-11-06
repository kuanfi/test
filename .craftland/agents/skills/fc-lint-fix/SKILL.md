---
name: fc-lint-fix
description: Trigger after any .fcg or .fcc edit, or when the user asks to compile, validate, lint, check, or fix FC compiler errors.
---

# FC Lint Fix

Use this skill after every `.fcg` / `.fcc` edit. Also use it when the user asks to compile, validate, lint, check, or fix FC errors. This is the highest-priority FC skill.

## Completion Rule

Pure FC text edits are not complete until full `fccompile.exe -i Assets` validation has run. `-m` is only a fast metadata-only precheck and must not be treated as final delivery validation. If the task changed editor-owned assets, UI, scene entities, or script attachments through Craftland Studio MCP, final validation must use `game-build` + `game-console-get-logs`. If validation cannot run, the final response must state which required file, path, or MCP tool is missing.

## Locate Project Config

Read `Temp/UGCLanguage/fcconfig.json` under the project root when available. Use these fields:

- `CompilerExecutable`
- `CorePath`
- `EditorGenLibPath`
- `MinimumExtensionVersion`

The editor symbol table is normally located at:

```powershell
<ProjectRoot>\Temp\UGCLanguage\editorGen\EditorGenSymbol.json
```

## Locate Compiler

Use the first existing compiler path in this order:

1. `Temp/UGCLanguage/fcconfig.json.CompilerExecutable`
2. `<CorePath>\fccompile_external.exe`
3. `<CorePath>\fccompile.exe`
4. `%USERPROFILE%\.vscode\extensions\craftlandstudio.ffugclanguage-*\library\release\fccompile.exe`
5. `%USERPROFILE%\.cursor\extensions\craftlandstudio.ffugclanguage-*\library\release\fccompile.exe`

If multiple plugin versions match, use the latest directory unless project config provides an explicit compiler path.

## Locate Standard And Generated Libraries

Prefer `CorePath` from `fcconfig.json`. If missing, read editor setting `ffugclanguage.editor.CorePath` when available, then fall back to plugin `library/release/`.

The compiler's `-e` option expects the editorGen directory, not the `EditorGenLib.fcc` file. By default it uses:

```powershell
<ProjectRoot>\Temp\UGCLanguage\editorGen
```

## Model And Session Markers (Required)

Every `fccompile` invocation that compiles sources (any command with `-i`) must include `-agent <model> -session <session-id>`. This enables local compile event collection. Craftland Studio may later upload these compile-result records through its EventLog service; session IDs are aggregation keys only and must contain no private content.

Build the model marker as follows:

- `-agent` identifies the **model**, not the coding-agent host product. Use the exact model identifier exposed by your system or runtime.
- Do not use product names such as `codex`, `cursor`, or `aismith`.
- Never guess the model name. If no model identity is available, use `unknown-model`.

Build the session marker as follows:

- Prefer the stable session/conversation ID exposed by your host.
- If the host exposes no ID, generate one globally unique ID once and reuse it for every compile in the current conversation.
- Never generate a new session ID per compile invocation.
- The ID is only an aggregation key. It must not contain user, project, path, prompt, or other private content.
- Use only letters, digits, `-` and `_` in both markers (replace other characters with `-`).

If the compiler rejects the flag with an error like `flag provided but not defined: -agent` (or `-session`), the compiler is an older version: retry the same command without that flag.

## Run Validation

Run from the project root. Match the VS Code extension's compiler invocation shape: pass source roots through `-i` and rely on compiler defaults for editorGen and symbol paths.

Final local compiler validation:

```powershell
<Compiler> -i Assets -agent <model> -session <session-id>
```

Optional fast metadata-only precheck:

```powershell
<Compiler> -i Assets -m -agent <model> -session <session-id>
```

`-m` means compile to metadata only. In the compiler, it still runs preprocessing and precompiler validation, but skips the final `Compiler()` phase. It can catch many syntax, import, type, and symbol issues quickly, but it is not a full compile and must not be the final validation for delivered `.fcg` / `.fcc` edits.

If the project has multiple source roots, join them with `;`, matching `fcconfig.json` / extension source roots:

```powershell
<Compiler> -i "Assets;Other/AssetsFolder" -agent <model> -session <session-id>
```

Do not pass `-e` or `-s` unless you must override compiler defaults. If overriding:

- `-e` must be the editorGen directory, for example `Temp\UGCLanguage\editorGen`.
- `-s` is joined with the current working directory by the compiler, so use a project-relative path such as `Temp\UGCLanguage\editorGen\EditorGenSymbol.json`.

`fccompile.exe -i Assets -agent <model> -session <session-id>` matches the VS Code extension full compiler invocation plus the required compile event markers. Use it as the default final validation for pure `.fcg` / `.fcc` edits.

Use Craftland Studio MCP `game-build` + `game-console-get-logs` as the final validation when the task touches UI assets, scene entities, script attachments, `button.clickHandlers`, or other editor-owned data. Without an editor process, local full compile (`<Compiler> -i Assets -agent <model> -session <session-id>`) is only a fallback and the final response must state that full editor validation was not run.

## Fix Loop

1. Run validation.
2. Parse diagnostics such as `./Assets/Script.fcg:10:5` or an equivalent `path:line:col` location.
3. Open the failing file and inspect nearby code.
4. Apply the smallest confirmed fix.
5. Run validation again.
6. Repeat up to 3 rounds.

If errors remain after 3 rounds, stop and report:

- Remaining diagnostics.
- Fixes already attempted.
- Which source or user action is needed next.

## Common Fix Checks

- Missing import: confirm the owning `.fcc` through `fc-symbol-lookup` before adding it.
- Unknown FC symbol: confirm it through `fc-symbol-lookup` before using it.
- Unknown assetId or entityId: confirm it through `fc-asset-check` before using it.
- Graph field declaration: use `isJumping bool`.
- Current attached entity: use `thisEntity<Transform>.Position`.
- Component access: use `entity<Component>.Field`.
- Vector construction: use `Vector3{x, y, z}`.
- Vector component access: use `pos.X`, `pos.Y`, `pos.Z`.
- HUD button callback: use `func OnClick(button entity<UIWidgetButton>, player entity<Player>)`.
- Asset/scene/entity/widget reference: never cast hardcoded id strings to asset id types or pass names as string literals; use registered `ERes*` symbols (see `fc-asset-registration`).
- Namespaced call: use the import alias confirmed by the source `.fcc`.
- Platform mismatch: verify `[platform_client]` or `[platform_server]`.
- Client/server event mismatch: put the platform decorator on the `graph`, not on the event listener inside the graph body.

## Final Response Requirements

Must include:

- Whether full compiler validation ran.
- The compiler path used.
- Whether full editor build validation ran when MCP assets/entities/UI were changed.
- Whether validation passed.
- If validation did not run, which config, path, or MCP tool is missing.
- If validation failed, a short summary of remaining errors.
