# Codex Prompt — Drone Box Camera + JUMP Button (Choose Drop Point)

> Paste everything below the line into Codex. Craftland Studio must be OPEN so the MCP (`craftland_studio`, see `.codex/config.toml`) is connected.

---

## Task

Implement a **"spectate the drop box, then JUMP"** pre-drop phase for the BR map:

1. When the match starts, a **box (moving entity)** travels around **inside the safe zone**.
2. A **camera follows the box** (attached/following, not a fixed shot).
3. Every player's view is switched to that box camera, and the player may **rotate the camera** to look around freely.
4. At the same time each player sees a HUD with a **JUMP button**.
5. When a player presses **JUMP**: teleport that player to the **box's current position**, restore their normal player camera, hide the HUD, and make them **skydive/parachute exactly like the current flow** (reuse the existing drop/`StartParachute` logic — do not write a second parachute flow).

## Mandatory workflow (read first)

- Read `AGENTS.md`, then `.craftland/agents/rules/fc-workflow.mdc`. Follow it, including the skills it routes to (`fc-symbol-lookup`, `fc-asset-check`, `fc-asset-registration`, `fc-lint-fix`).
- Game logic lives in **`Assets/ECA/BermudaBRRules.eca`** (visual-block JSON, server). `.eca`, `.ui`, scene entities and prefabs are editor-owned: **do NOT hand-edit them.** Use the Craftland Studio MCP. Read `instructions://index` first, then `asset-list` / `asset-get` / `asset-entity-*` / `entity-eca-script-update`.
- If MCP is not connected, **stop and tell me to open Craftland Studio.** Never guess serialized formats.
- Do not invent API / event / enum / component names. Verify every one from the stdlib `.fcc` (`fcconfig.json` `CorePath`), `Temp/UGCLanguage/editorGen/EditorGenLib.fcc`, or the `ugc-mcp` docs (`search_docs`, `get_doc`).
- Anything the code names (box entity, UI, JUMP button, camera entity, scenes) must be **registered** via `fc-asset-registration-*` and referenced via generated `ERes*`/`Res` symbols, not name strings.
- After any `.fcg/.fcc` edit run the lint loop in `.craftland/agents/skills/fc-lint-fix/SKILL.md`. After MCP edits finish with MCP `game-build` + `game-console-get-logs`.

## Step 0 — Discovery (do this before designing; report briefly)

1. Read the **current** `BermudaBRRules.eca` match-start flow. Determine what is *actually* there now: how players are placed (Waiting Island → main area), where `safeZoneCenter` / `OpenSafeZone(...)` are set, and whether the sky-drop from `.codex/prompts/CODEX_PROMPT_SkyDrop.md` (`SkyDropPlayer`, `SkyDropHeight`, `StartParachute`) is already implemented. **Reuse that drop path for step 5**; state which function/block you reuse.
2. Find (via stdlib/ugc-mcp docs) the supported APIs for: moving an entity along a path/between random points, a follow camera, switching a player's camera to another camera entity, **letting the player rotate that camera by input**, and showing/hiding a UI per player + button click callback. If a required capability (especially player-controlled rotation of a non-player camera) does **not** exist, **stop and ask me** with the closest alternatives instead of faking it.
3. Check whether a box entity, camera entity, or JUMP UI already exist in `Assets/Scene/*.scene` / `Assets/UI` via MCP. Create only what's missing, through MCP.

## Requirements

**R1 — Box**
- One box entity (server-owned, visible to all). Spawn at match start at a random point inside the safe-zone disc at height `BoxHeight` (named constant, default = `safeZoneCenter.Y + 150`, tunable).
- Keeps moving **only inside the current safe-zone radius**: repeatedly pick a new random target inside the disc (uniform: `r = R·sqrt(u)·0.9`), move toward it at `BoxSpeed` (named constant, start ~20 m/s), then pick the next. The disc shrinks over time — compute the live radius the same way the SkyDrop prompt does (`safeZoneStartMs`, `R0`, `wait`, `duration`, `R_min` floor); do not hard-code duplicated numbers.
- Starts moving only after the safe zone is opened (the center must exist). During the Waiting Island phase nothing of this runs.

**R2 — Follow camera**
- One camera that follows the box with a fixed offset (e.g. slightly behind/above, looking at/around the box). Configurable offset constants.

**R3 — Camera switch + look-around**
- When the pre-drop phase starts, switch **each human player** to the box camera. The player can rotate the camera (yaw/pitch, pitch clamped) to look around while it keeps following the box position.
- Per-player state: keep a `Dictionary`/list keyed by player UID (e.g. `hasJumped`) so one player's jump doesn't affect others' camera/UI.
- Bots do not use the camera/UI; they keep whatever drop flow exists today (or drop at the same moment as the box phase ends, see R6).

**R4 — HUD**
- A HUD with a single **JUMP** button, shown to each human player at the same moment as the camera switch (bottom-center, thumb-reachable on mobile, localized text if the project uses `Assets/Localization`). Hidden after the player jumps.

**R5 — JUMP behaviour** (`func OnClick(button entity<UIWidgetButton>, player entity<Player>)`)
- Ignore if that player already jumped (debounce / double-tap safe).
- Read the box's **current** world position at the click moment; teleport the player there (server-side), face along the box's movement direction or toward `safeZoneCenter`.
- Restore the player's normal camera, hide the JUMP HUD, mark `hasJumped`.
- Start the **existing skydive flow** (`StartParachute` with the current parameters/helper). Do not duplicate parameters; call the existing helper.

**R6 — Edge cases**
- Pre-drop phase end: if a player never presses JUMP, auto-jump them after `AutoJumpSeconds` (named constant, default 30 s) with the same JUMP routine, so nobody is stuck. When **all** humans have jumped (or timer ended) stop and **remove/hide the box and its camera** (don't delete scene entities permanently unless they were created for this feature).
- Player disconnect / death / revive during the phase: clean their state; a revived player must not be re-put into the box camera (the revive sky-drop flow from the SkyDrop prompt stays as is).
- `SetMatchCanMove` / Waiting Island logic and `freeReviveUsed` rule must keep working. Don't change the 9 safe-zone center candidates or `OpenSafeZone` parameters.
- Everything gameplay-affecting runs server-side (graph platform = 2); only use client-only APIs where the stdlib says they are required (camera/UI per-player), and say so.
- Don't leave the aircraft + `CustomCurvePath` blocks as the delivery vehicle; if you remove their usage, say exactly what you removed and ask me first if another system depends on them.

## Deliverables

1. Changes made **through MCP** (no hand-edited `.eca`/`.ui`/`.scene`).
2. Summary: discovery findings (what already existed), new entities/UI/variables/constants (`BoxHeight`, `BoxSpeed`, camera offset, `AutoJumpSeconds`, ...), which existing drop function was reused, anything removed.
3. MCP `game-build` + `game-console-get-logs` results (pass/fail; list anything unverified honestly).
4. Manual test checklist: start match → box flies inside the zone and stays in the shrinking disc; every player sees box camera and can look around; JUMP button visible; press JUMP → teleported to the box's current position and parachutes like before, camera/HUD restored; other players unaffected; AFK player auto-jumps at the timeout; box/camera disappear after the last jump; revive flow still works.

Ask me before choosing anything not covered above.
