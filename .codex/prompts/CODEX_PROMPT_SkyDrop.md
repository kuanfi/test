# Codex Prompt — Sky Drop Spawn & Respawn (Parachute into Safe Zone)

> Paste everything below the line into Codex. Craftland Studio must be OPEN so the MCP (`craftland_studio`, see `.codex/config.toml`) is connected.

---

## Task

Implement a **sky-drop spawn** for the BR map in this Craftland project:

1. When a match starts, every player spawns **in the sky** above the **chosen safe-zone center** (`safeZoneCenter` / `vgrule7eX`), at a random XZ position **inside the radius of the safe zone**, then parachutes down.
2. When a player is **revived** (currently: first death only, via `ReviveOnNextFrame` in `OnPlayerDead`), they also respawn in the sky at a random position **inside the CURRENT (already shrunk) safe-zone radius**, then parachute down.

## Mandatory workflow (read first)

- Read `AGENTS.md`, then `.craftland/agents/rules/fc-workflow.mdc`. Follow it.
- The game logic is in **`Assets/ECA/BermudaBRRules.eca`** (visual-block JSON, owner Global, server). `.eca` is an editor-owned asset: **do NOT hand-edit the JSON.** Use the Craftland Studio MCP (`craftland_studio`). Read `instructions://index` first, then use `asset-get` / `entity-eca-script-update` etc.
- If MCP is not connected, **stop and tell me to open Craftland Studio**. Do not guess the serialized format.
- Any API / event / enum name must be verified (stdlib `.fcc` under `fcconfig.json` `CorePath`, `Temp/UGCLanguage/editorGen/EditorGenLib.fcc`, or ugc-mcp docs). Don't invent names.
- If you add `.fcg/.fcc` code, finish with the lint loop in `.craftland/agents/skills/fc-lint-fix/SKILL.md`. After MCP edits, finish with MCP `game-build` + `game-console-get-logs`.
- Register (via `fc-asset-registration-*`) any scene/entity you need rather than using name strings.

## Current state (already reviewed — don't redo the discovery)

`BermudaBRRules.eca` has 3 event roots:

| Event | What it does now |
|---|---|
| `OnMatchStart` (A) | Sets var `freeReviveUsed` (`List<UUID>`) to empty. |
| `OnPlayerDead(player)` | If player UID **not** in `freeReviveUsed` AND player's `HostTeamEntity.IsACE != true` → append UID to `freeReviveUsed`, then `ReviveOnNextFrame(player)`. (One free revive per human; bots excluded.) |
| `OnMatchStart` (B, `waitingSequenceEvent2026`) | 1) For each player: `SwitchPlayerArea` to **Waiting Island** area (`qpi1yfifn2-mw4no05w-em1bv5g3eup`) at one of 32 grid points (z ∈ {-12,-4,4,12}). 2) Wait 60 s. 3) Pick `safeZoneCenter` = random of 9 hard-coded Vector3 (e.g. (-64,6.75,64) … (1024,9.84,-512)). 4) `OpenSafeZone(center, radius=200, wait=120, duration=180, damage=5)` (= `EnableSafeZone`). 5) For each player `SwitchPlayerArea` to **main area** (`eysb07aklem-mw3gx2bz-nf3naagrfcb`) at a random one of the same 9 points (**not** the safe-zone center, independent random). 6) Wait 5 s; set 4 `CustomCurvePath` waypoints (altitude Y≈220–240); `CreateAlongPathMotion` → `brAircraftMotion2026` (30 s, random of 4 paths) ; `SetMatchCanMove(true)`. 7) For each player `StartParachute(player, forcedHeight=120, surf(12,-6), dive(20,-18), parachute(8,-4))`. 8) Loop botIndex `ListLength(AllPlayers)..31`: `AddBot(Hell, team=botIndex/4)` then `TeleportPlayerPositionAndForward` to safe-zone center ± `RandomFloat(-35,35)` on X and Z (Y = center Y, so bots land on the ground, no parachute).

Problems vs the requested feature: players are switched to a random fixed point (not the safe-zone center); `forcedParachuteHeight=120` is relative, not a real sky spawn; the revived player gets no sky respawn at all (engine revive point + no parachute); bots are not dropped from the sky.

Relevant verified APIs (stdlib):
- `EnableSafeZone(center, radius, waitTime s, duration s, damage/s)` — shrinks to the center point. **There is NO getter for the current radius** → compute it (see below).
- `StartParachute(target entity<Player>, forcedParachuteHeight float, surfingMaxSpeed Vector2, divingMaxSpeed Vector2, parachutingMaxSpeed Vector2)`
- `event OnPlayerRevived(player, revivalSource PlayerReviveSourceType)` (Player.fcc) — use this to hook respawn. Also `OnPlayerRevivedInSpawnPos`.
- `TeleportPlayerPositionAndForward`, `SwitchPlayerArea`, `RandomFloat(min,max)`, `ReviveOnNextFrame`.

## Requirements

**R1 — Sky spawn at match start**
- Remove the dependency on the random fixed drop point/aircraft for placing players. After `OpenSafeZone` is called and players are in the main area, teleport each player (humans + bots) to `P = (cx + r·cosθ, SKY_Y, cz + r·sinθ)` where `(cx,cz)` = `safeZoneCenter`, θ random in [0,2π), `r = R_now · sqrt(random[0,1)) · 0.9` (uniform in disc, 10 % margin so nobody drops on the edge).
- `SKY_Y` = **`safeZoneCenter.Y + 150`** (make it a single named constant/variable `SkyDropHeight = 150`, easy to tune; 150 m gives ~10+ s of fall with the existing speeds). Face each player toward the center.
- Immediately after teleport call `StartParachute` (reuse the existing speed Vectors; set `forcedParachuteHeight` so the chute is auto-forced at a safe altitude above ground, e.g. 40 — verify semantics in `ScriptManual/API/StartParachute`; if it means "force-open height above ground", keep it < SkyDropHeight).
- Give a short invulnerability / fall-damage-safe window if a suitable API exists (verify); otherwise skip.
- Keep the Waiting Island (60 s) phase and the safe-zone start as they are. Decide with a comment whether the aircraft (`brAircraftMotion2026`, `CreateAlongPathMotion`, the 4 `CustomCurvePath` blocks) stays as visual-only or is removed; **recommended: remove the aircraft + path blocks** since the plane is no longer the delivery vehicle. Don't delete scene entities, only their usage in this graph.
- Bots (`AddBot` loop): spawn them with the same sky-drop helper instead of the current ground teleport.

**R2 — Sky respawn on revive**
- Hook `OnPlayerRevived(player, source)` (preferred) — or add the logic right after `ReviveOnNextFrame` using a short `WaitForSeconds` / next-frame — and teleport the player to a random point in the **current** safe-zone disc at `SkyDropHeight`, then `StartParachute`.
- Current radius: keep a global float variable `safeZoneStartMs` (record match time when `OpenSafeZone` is called) and compute
  `elapsed = now - start`; `R_now = (elapsed ≤ wait) ? R0 : max(R_min, R0 · (1 - (elapsed-wait)/duration))` with `R0=200, wait=120, duration=180` (store these as variables so the numbers aren't duplicated; first verify the actual end radius — docs say it shrinks to the center point, so `R_min` = a small floor like 5–10 so spawn stays valid). Verify the time API (`GetServerTime`/`GetMatchTime` etc.) from stdlib.
- If the safe zone hasn't opened yet (player revives during the 60 s waiting phase), no safe-zone center exists yet → just revive normally in the Waiting Island area (no parachute).
- Keep the existing "one free revive per human, bots excluded" rule unchanged. If a revive source other than this script exists (`revivalSource`), handle it the same way.

**R3 — Reusable helper**
- Implement ONE reusable function/graph, e.g. `SkyDropPlayer(player)`, that does: compute R_now → pick random point → teleport → face center → StartParachute. Call it from match start (humans+bots) and from `OnPlayerRevived`. No copy-pasted block trees.

**R4 — Constraints / edge cases**
- Don't change the 9 safe-zone center candidates or `OpenSafeZone` parameters unless needed for the radius variables.
- Don't break `freeReviveUsed`, bot creation count (fills to 32 players), or `SetMatchCanMove`.
- Random point must stay inside the live radius even when `R_now` is tiny; clamp at `R_min`.
- Server-side only (graph platform = 2); no client-only APIs.
- Players teleported while `SetMatchCanMove(false)` — make sure the drop happens after movement is enabled, or the parachute call still works (test).

## Deliverables
1. The updated `BermudaBRRules.eca` produced **through MCP** (not hand-edited).
2. A short change summary: which blocks were added/removed, new variables (names/types), tuned constants (`SkyDropHeight`, `forcedParachuteHeight`, `R_min`, margin).
3. Build result from MCP `game-build` + `game-console-get-logs` (state pass/fail, and anything unverified).
4. A manual test checklist: start match → confirm all players/bots appear at ~SkyDropHeight inside the zone and parachute; wait for shrink to ~half; kill a human → confirm revive in the sky inside the *shrunk* radius; kill again → no second revive; kill a bot → no revive.

Ask me before choosing anything not covered above (e.g. aircraft removal if you find another system relying on it).
