---
name: fc-symbol-lookup
description: Trigger when FC code needs any referenced symbol verified, including APIs, imports, types, functions, events, enums, components, declare graphs, EditorGen symbols, or package symbols.
---

# FC Symbol Lookup

Use this skill before any `.fcg` / `.fcc` code depends on a symbol that FC code can reference. This includes official standard library symbols and project-generated symbols.

## Goal

Do not write FC symbols from memory. Every API, type, function, event, enum value, component name, declare graph, public property, generated project symbol, and import source must come from a real source.

This skill is the FC script handbook lookup path. It answers "what symbol exists, where is it declared, how should it be imported, and what exact signature or type should code use?"

It does not prove that a Craftland Studio asset or entity currently exists in the editor. Use `fc-asset-check` for assetId, entityId, UI object, scene entity, script attachment, button binding, and current editor state validation.

## Lookup Order

1. Search likely `.fcc` files first, based on the user's wording and the domain implied by the task:
   - Prefer `Temp/UGCLanguage/fcconfig.json.CorePath`.
   - Read editor setting `ffugclanguage.editor.CorePath` when available.
   - Otherwise fall back to the plugin `library/release/` directory.
2. Read current project generated libraries:
   - `Temp/UGCLanguage/editorGen/EditorGenLib.fcc`
3. Read referenced package generated libraries and package source declarations:
   - `Temp/UGCLanguage/editorGen/<libId>.fcc`
   - `Temp/UGCLanguage/packages/<libId>/*.fcc`
4. If likely files do not contain the symbol, search all available `.fcc` files under the same roots.
5. Query `FF_UGC_Docs` MCP when available for explanations, examples, and behavior details. Do not rely on docs alone when local `.fcc` declarations are available.

## Symbol Scope

This skill covers both official and project-defined FC symbols:

- Official APIs, types, functions, imports, events, enums, components, and function signatures.
- Project-generated custom events, custom enums, custom components, declare graphs, graph public properties, UI control component types, and generated project symbol names.
- Asset registration symbols in `EditorGenLib.fcc`: `ERes*` / `EResKey*` enums and the `Res` declare graph. This skill verifies whether they exist; if a needed one is missing, route to `fc-asset-registration` to create it instead of concluding the asset is unusable.
- Referenced package symbols from:
   - `Temp/UGCLanguage/editorGen/<libId>.fcc`
   - `Temp/UGCLanguage/packages/<libId>/*.fcc`

If the user asks whether an asset or entity exists in the editor, do not answer from this skill alone. Route to `fc-asset-check`.

## Search Strategy

Treat user-provided candidate symbol names or concepts as first-class search keys. If the user mentions a possible API, event, enum, component, graph, property, or domain concept, verify that direction before settling on a manual implementation.

Use semantic file targeting before broad searches. Guess likely `.fcc` files from the task wording, nearby imports, and domain terms, then search only those files first. For example, UI tasks should start from UI or widget-related `.fcc` files; animation, playable, movement, physics, audio, or math tasks should start from their likely domain files before searching everything.

Use narrow searches before broad searches:

```powershell
rg -n "<Candidate>|Create<Candidate>|Create.*<Candidate>" <LikelyFccFileOrDirectory> -g "*.fcc"
rg -n "<Candidate>|Create<Candidate>|Create.*<Candidate>" <ProjectRoot>\Temp\UGCLanguage\editorGen -g "*.fcc"
rg -n "<Candidate>|Create<Candidate>|Create.*<Candidate>" <ProjectRoot>\Temp\UGCLanguage\packages -g "*.fcc"
```

Then broaden only if the likely `.fcc` files do not contain a usable symbol. Search all `.fcc` files under `CorePath`, `editorGen`, and referenced packages before concluding that the symbol does not exist. Expand to neighboring domain terms, likely verbs, property names, or lower-level primitives only after the user-provided candidate has been checked directly.

When search output is large, do not manually scan a noisy result set and stop early. Filter again with the strongest candidate term, exact symbol fragment, type name, event name, enum name, component name, graph name, or likely function prefix.

For FC symbols, treat `.fcc` declarations as the primary source. Confirm import, namespace, signature, fields, platform, and deprecation status from the `.fcc` file that owns the declaration. If a declaration has `[deprecated]` and its preceding `//@deprecated` comment says `Use X instead`, generate `X`, not the deprecated symbol.

Do not assume library names from general programming concepts. The owning `.fcc` may be named after a project-specific domain rather than the generic concept the user used.

Before concluding that a candidate symbol does not exist, record the exact keywords and file ranges checked. The "not found" result is not valid if a user-provided candidate term was never searched directly.

## Plugin Fallback Paths

When project config is missing, look for an installed FC plugin:

- VS Code: `%USERPROFILE%\.vscode\extensions\craftlandstudio.ffugclanguage-*`
- Cursor: `%USERPROFILE%\.cursor\extensions\craftlandstudio.ffugclanguage-*`

Unless project config provides an explicit path, use the latest matching plugin directory. Standard `.fcc` files and `fccompile.exe` should be under `library/release/`.

## Information To Extract

For every symbol you will use, record:

- The source file or MCP document that confirms the symbol.
- The import statement, if one is required.
- Whether a namespace alias is needed to resolve an actual ambiguity.
- The exact function signature, type declaration, event parameters, enum members, component fields, graph name, or public property declaration.
- Any platform or type constraints.

## Import Rules

- Add an import only after confirming the owning `.fcc` file.
- Keep aliases consistent with confirmed examples or local project style.
- Do not assume common libraries are already imported.
- Prefer unqualified API, event, and type symbols after importing their owning `.fcc`; use `Alias.Symbol` only to resolve an actual ambiguous identifier. Keep required static graph and cross-script member qualification.
- Referenced package `.fcc` files under `Temp/UGCLanguage/packages/<libId>/` are only lookup sources. Do not use that physical path in code imports. Use the `.fcc` filename or an existing relative import style; the compiler resolves it through `fcconfig.json` directory protocols.

## If Lookup Fails

Do not generate code that uses unconfirmed symbols. Tell the user which symbol could not be verified and which source is missing, such as `CorePath`, standard library `.fcc` files, `EditorGenLib.fcc`, referenced package `.fcc` files, `FF_UGC_Docs`, or plugin `library/release/`.

## Output Check

Before editing or replying, make sure you can state:

- The confirmed `.fcc` file or MCP document.
- Required import lines.
- Function, type, event, enum, component, declare graph, or public property declarations.
- Any unresolved symbols that must not be used.
