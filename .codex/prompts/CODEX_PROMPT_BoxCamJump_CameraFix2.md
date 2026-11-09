# Codex Prompt — Fix #2: Player camera still does not switch to the Drone Box camera

> Paste everything below the line into Codex. Craftland Studio must be OPEN so the MCP (`craftland_studio`) is connected.

---

## Problem

After the previous fix round (`CODEX_PROMPT_BoxCamJump_Fix.md`) the feature compiles, but **the local player's camera still does not switch to the box camera.** Relevant files: `Assets/ECA/DroneJumpCameraClient.fcg` (client script attached to the touch pad `LookAroundTouchPad` of `Assets/UI/DroneJumpHUD.ui`), `Assets/ECA/DroneDropServer.fcg` (on scene entity `DroneDropBox`, creates the HUD per player and moves the box), `Assets/ECA/DroneDropDiagnostics.fcc` (`DebugLog` flag, already added).

## Mandatory workflow

- Read `AGENTS.md`, `.craftland/agents/rules/fc-workflow.mdc`; follow it (symbol lookup, asset check, registration, lint-fix).
- `.eca`, `.ui`, `.scene` are editor-owned: use Craftland Studio MCP (read `instructions://index` first), no hand-editing JSON. If MCP is not connected, stop and tell me to open Craftland Studio. `.fcg/.fcc` may be edited as text.
- Verify every API in stdlib `.fcc` (`CorePath` in `fcconfig.json`), `EditorGenLib.fcc`, or ugc-mcp docs. Don't invent names.
- Finish with lint (`fc-lint-fix`) and MCP `game-build` + `game-console-get-logs`. Don't touch unrelated files (`.codex/config.toml`, `.cursor/mcp.json`, `fcconfig.json`, `main.gproj`).

## Step 1 — Read the reference project (working camera switch)

`D:\Craftland\ink-game` has a working client camera switch. Read:
- `Assets/Scripts/Gameplay/Games/Dalgona/C_DalgonaCameraController.fcg` (`[platform_client]`)
- its caller in `C_DalgonaController.fcg` (`Events.RoundPlaying` → `thisEntity<C_DalgonaCameraController>.SetupCamera(_myCookie)`, `GameEnded`/`S2C_PlayerFinished` → `ResetCamera()`), and how both are attached in `Assets/Scenes/Dalgona.scene`.

Pattern that works there (compare it with our script, line by line):
1. Camera is created and switched from a **long-lived client script on a scene entity**, triggered by an **event**, not from `OnAwake/OnEnable` of a script on a UI widget that is created server-side a moment earlier.
2. Target position comes from an **entity reference passed in** (`cookieLocation`), not from looking up a registered scene entity via `EResScene...` inside the script.
3. Order: `Camera.CreateCustomCamera(out cam)` → set `cam<Transform>.Position` and `.Rotation` → `Camera.SwitchCustomCameraClient(cam, Camera.CameraTransitionType.Smooth, <ms>)` (non-zero transition time, `Smooth`), then moves the camera's `Transform` per frame in an `async` loop using `wait StdLib.WaitForNextFrame()`.
4. Reset is `Camera.SwitchCustomCameraClient(nil, Default, 0)` then `Destroy(cam)`, and also on `OnDestroy`. It never stores/restores a "previous camera".
5. `Player.SetLocalPlayerMoveStatus(false/true)` brackets the phase.
6. Local player is obtained with `StdLib.GetLocalPlayer()`; imports are namespaced (`import "Camera.fcc" as Camera`).

## Step 2 — Find the real cause (use evidence, not guesses)

Use the existing `[DroneCamera]` logs (client) and `[DroneDrop...]` logs (server). Run/inspect via MCP `game-console-get-logs`. Determine which stage fails and report it:
- (a) Is the client script's `OnAwake`/`OnEnable` ever called? (If not: the script is not attached to the touch pad, the HUD is not created, or UI scripts do not run on creation — check with `asset-get` on `DroneJumpHUD.ui`.)
- (b) Does `TryInitialize` get past `box == nil` / `HasComponent(box, Transform)`? (`EResSceneBRBermuda.DroneDropBox` may not resolve on the client, or the box is not replicated/AOI-visible to the client; the box is `visible:false`, `Primitive/Empty`.)
- (c) Does `CreateCustomCamera` return non-nil and does `SwitchCustomCameraClient(camera, Default, 0)` print "Camera switched" yet the view does not change? (Then compare with ink-game: transition type/time, setting position before the switch, **camera Transform not being updated because the box position on the client never changes**, etc.)
- (d) Does something switch the camera back right away (e.g. `RestoreCamera` from `OnDisable`/`OnDestroy` firing immediately, or another script/`StartParachute`/`SetMatchCanMove`/AOI proxy switch resetting the view)? Check the log order.
- (e) Is the box's client-side `Transform.Position` actually updating when the server moves it (log it once per second on the client)? If a hidden `Primitive/Empty` entity does not replicate transform updates to the client, the camera appears "stuck/not switched" at the wrong place. Check the server `AddAOISpecialFollow` / `SwitchAOIProxyObserver` calls too.

Do not skip this step; the fix must target the failing stage.

## Step 3 — Fix (choose based on Step 2, explain why)

Preferred direction (mirrors ink-game, robust against replication timing):
- Move camera logic into a **client script on a long-lived entity** (e.g. attach to the box entity or a dedicated scene entity via MCP — verify the client script attach mechanism for scene entities, as `Dalgona.scene` does), exposing `StartBoxCamera(boxEntity entity)` / `StopBoxCamera()`.
- Trigger it by an **event/signal from the server** after the HUD is created in `DroneDropEnroll` (verify the correct per-player server→client mechanism in ugc-mcp docs; ink-game uses events like `Events.RoundPlaying` / `EditorGenLib.S2C_*` — read how they are declared and sent to a specific player). Pass the box entity as a parameter instead of resolving `EResSceneBRBermuda.DroneDropBox` inside the client.
- Create the camera with `CreateCustomCamera`, set Position/Rotation **before** `SwitchCustomCameraClient(camera, CameraTransitionType.Smooth, 300)` (try `Smooth` first, as ink-game does; keep `Default,0` only if proven better).
- Follow the box: update the camera pose every frame from the box's `Transform` (client). If the client box position does not update (Step 2e), use a supported alternative and justify it: server-side `CreateCustomCamera` + `SetCameraFollow(camera, box, offset, coordinateSystem)` + `SwitchCustomCamera(player, camera, transition, time)` (these exist in `Camera.fcc`; check `[platform_server]` tags), keeping look-around by writing the camera/offset yaw+pitch from the client touch pad via a server event, or the client-side camera rotation if the camera entity is client-writable. Pick the simplest option that works and say what you rejected.
- Look-around: keep `OnDrag` on the touch pad (`[platform_client]`, owner type UIWidgetTouchPad) — it can forward yaw/pitch to the camera controller (call into the client script or send a local event, like ink-game's `C_EventAdapter.BroadcastLocal`).
- Restore: `SwitchCustomCameraClient(nil, Default, 0)` + `Destroy(camera)` on JUMP/cleanup/`OnDestroy`, plus `SetLocalPlayerMoveStatus(true)`. Remove the "previousCamera" restore unless Step 2 shows it is needed. Make sure `OnDisable`/`OnDestroy` of the touch pad script cannot cancel the camera before JUMP.
- Keep the JUMP flow (teleport to the box's current position + existing skydive via `SkyDropAt`) and the server cleanup/auto-jump behavior unchanged. Keep logs behind `DroneDropDiagnostics.DebugLog`.

## Constraints
- Minimal diff; no unrelated refactors; no hand-edited editor-owned assets.
- Register any new scene entity/script/event via the registration tools/MCP and use `ERes*` symbols, not name strings.
- Any new custom event/signal must be declared the supported way (check how `DroneDropEnroll` and ink-game's events are declared) and compile cleanly.

## Deliverables
1. The fix, via MCP for editor-owned assets.
2. A short root-cause report: which stage of Step 2 failed, with evidence (log lines / asset state), and how the new design differs from the old one and from ink-game.
3. Lint + `game-build` + `game-console-get-logs` results; state honestly what could not be run/verified.
4. Manual test checklist with the exact log lines to expect (HUD created → client controller received box → "Camera switched" → client box position moving → drag changes yaw/pitch → JUMP restores camera and parachutes).

Ask me before changing behavior not covered above.
