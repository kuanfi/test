# Codex Prompt — Fix #3: Player cannot press JUMP to skydive

> Paste everything below the line into Codex. Craftland Studio must be OPEN so the MCP (`craftland_studio`) is connected.

---

## Problem

In play-test the JUMP HUD (`Assets/UI/DroneJumpHUD.ui`) is on screen, but **pressing JUMP does not make the player teleport to the box and skydive** (the flow `DroneJumpHUDServer.OnClick` → `DroneJumpRelay.RequestJump` → signal `DroneDropJump` → `DroneDropServer.Jump` → `BRRules.SkyDropAt`). The AFK auto-jump after 30 s may or may not work — report which.

## Mandatory workflow

- Read `AGENTS.md`, `.craftland/agents/rules/fc-workflow.mdc`; follow it (symbol lookup, asset check, registration, lint-fix).
- `.eca`, `.ui`, `.scene` are editor-owned: use Craftland Studio MCP (`instructions://index` first), no hand-editing JSON. If MCP is not connected, stop and tell me to open Craftland Studio. `.fcg/.fcc` may be edited as text.
- Verify every API in stdlib `.fcc` / `EditorGenLib.fcc` / ugc-mcp docs; no invented names.
- Finish with lint + MCP `game-build` + `game-console-get-logs`. Don't touch unrelated files (`.codex/config.toml`, `.cursor/mcp.json`, `fcconfig.json`, `main.gproj`).

## Step 1 — Instrument the whole click path (no silent returns)

Add logs (behind `DroneDropDiagnostics.DebugLog`; server `LogInfo`/`LogError`, tag `[DroneJump]`, include player UID) at **every** hop:
1. `DroneJumpHUDServer.OnClick` entry (player nil?), and right before/after `thisEntity<JumpRelay>.RequestJump(player)`.
2. `DroneJumpRelay.eca` `RequestJump` — if you can't log from the ECA, log in the receiving event.
3. `DroneDropServer` event `DroneDropJump` entry.
4. `DroneDropServer.Jump`: log each early-return reason separately — `player==nil`, `!active`, `uid not in hasJumped` (tombstone/ResetPhase), `already jumped`, `not alive`; then log the computed position and `SkyDropAt` call, and after it the player's resulting position.
5. `BermudaBRRules.eca` `SkyDropAt` (via MCP): confirm it is invoked and that its internal `StartParachute` runs.
Also log in `DroneDropServer.OnUpdate` when `ResetPhase()` is triggered by `!safeZoneOpened`, and in `FinishPhase`/`ResetPhase` (who called, `pendingCount`).

## Step 2 — Determine where the click dies (report with evidence)

Suspects from code review, check each:

1. **Touch pad is stacked over the button.** In `DroneJumpHUD.ui` the full-screen `LookAroundTouchPad` (`UITouchPadEntity`, priority 0, `allowPressedOutside: true`, alpha 20) is the **last child**, after `JumpButton` (priority 10). If hierarchy order/hit-testing puts the touch pad on top, taps never reach the button. Verify via MCP `asset-get` / `asset-entity-get` how priority and sibling order resolve hit-testing; compare with how other HUDs in this repo or `D:\Craftland\ink-game` layer a button over a touch pad. Fix by reordering (touch pad first/bottom) and/or raising button priority, and make sure `allowPressedOutside` isn't swallowing the tap. A drag that starts on the pad must still rotate the camera.
2. **Click handler binding.** `JumpButton.clickHandlers` → `fileID fpk2eqrg1yu-mw96t99d-9t8h9bt2yco` (`DroneJumpHUDServer.fcg`, `funcKey OnClick`). Confirm in the editor that the button really invokes it **on the server**, with `(button, player)` params; check whether the supported/current API is the event `OnTapped(player)` on the button rather than the deprecated `OnClick` (see `Hud.fcc` — `OnClick` is `[deprecated]`, `OnTapped` is the replacement; `OnPressed` also exists). Use whichever is supported and works; prove it with the Step 1 log.
3. **`DroneJumpRelay.eca` → signal.** `RequestJump` sends `DroneDropJump` (signal_44021385) to entity `SrYjXvmX` in scene `eysb07aklem-mw3gx2bz-nf3naagrfcb` (`DroneDropBox`). Verify the target resolves at runtime, the player param is passed, and the event handler in `DroneDropServer` receives it. If the relay is unreliable, call a different supported route (e.g. server graph looks up the box via `EResSceneBRBermuda.DroneDropBox` and sends the signal directly, or a global server entity holds the click handler) — pick the simplest that works and explain.
4. **`Jump()` silently refusing.** `!active` (phase ended or `ResetPhase()` fired when `safeZoneOpened` flipped/`OnUpdate` guard), `hasJumped` map reset by `ResetPhase()` while the HUD still exists, `PlayerStatus != Alive`, tombstones from `DroneDropCleanup`. Check the Step 1 logs and fix the state machine so a visible HUD always implies a jumpable player (e.g. never leave a HUD alive after `ResetPhase`/`FinishPhase`; destroy it and restore the player).
5. **Jump sequence side effects.** `Jump` calls `CleanupPlayer` first (destroys the HUD, broadcasts camera stop, restores `Invincible`, resets AOI proxy to the player) and then `SkyDropAt(player, position, forward)`. Verify: (a) destroying the HUD in the same frame as its click callback doesn't abort the callback chain — if it does, move the `DestroyCustomUI` after the teleport or defer by one frame; (b) the teleport + `StartParachute` actually work for a player whose AOI proxy was the box until a moment ago (order of `SwitchAOIProxyObserver(player, player)` vs teleport; consider teleport first, then restore AOI); (c) `SkyDropAt` expects the position/forward semantic the Jump code passes (world position of the box, not offset); (d) `SetMatchCanMove` state doesn't block the parachute.
6. **Client side blocking input.** `SetLocalPlayerMoveStatus(false)` is set while the box camera is active — confirm it doesn't also block UI taps, and that after the click the client restores (`StopBoxCamera` through `DroneCameraState(active=false)`) so the player can move/parachute-steer.

## Step 3 — Fix

Fix the failing stage(s) found in Step 2 with a minimal diff. Required end behavior:
- Tapping JUMP once: player is teleported to the box's **current** position, HUD destroyed, box camera stopped and normal camera/move restored, `Invincible` restored, then the **existing** `SkyDropAt`/`StartParachute` flow runs. Double-taps are harmless.
- The look-around drag still works and doesn't trigger JUMP; JUMP doesn't trigger a look gesture.
- The 30 s auto-jump uses the same `Jump()` path and still works.
- If another player jumps, this player's HUD/camera is unaffected.
- Keep revive/death/disconnect cleanup rules unchanged.

## Step 4 — Housekeeping
- Keep diagnostic logs behind `DebugLog`, but list them in the report so I can disable them. `DroneDropRuntimeProbe.fcg` is diagnostic-only; tell me whether it is still needed and leave it off the critical path.

## Deliverables
1. The fix (MCP for editor-owned assets).
2. Root-cause report: for each of suspects 1–6 — confirmed / ruled out, with evidence (log lines, asset state, doc reference).
3. Lint + `game-build` + `game-console-get-logs` results; state clearly what could not be run in-game.
4. Manual test checklist with expected log lines: `OnClick entry` → `RequestJump` → `DroneDropJump entry` → `Jump entry` → `SkyDropAt` → player position near the box → parachute starts → camera/HUD restored; also double-tap, AFK auto-jump, and two players where one jumps first.

Ask me before changing behavior not covered above.
