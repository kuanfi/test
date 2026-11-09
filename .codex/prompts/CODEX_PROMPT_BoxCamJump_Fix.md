# Codex Prompt — Fix: Drone Box Camera not switching & JUMP HUD not showing

> Paste everything below the line into Codex. Craftland Studio must be OPEN so the MCP (`craftland_studio`) is connected.

---

## Problem

The "box camera + JUMP button" feature (see `.codex/prompts/CODEX_PROMPT_BoxCamJump.md`) compiles, but in play-test:
- the player's camera is **not** switched to the box camera,
- the **JUMP HUD does not appear**, so the player cannot jump.

Files involved: `Assets/ECA/DroneDropServer.fcg` (on scene entity `DroneDropBox`, server), `Assets/ECA/DroneJumpCameraClient.fcg` (client, on the touch pad in `Assets/UI/DroneJumpHUD.ui`), `Assets/ECA/DroneJumpHUDServer.fcg` + `Assets/ECA/DroneJumpRelay.eca` (JUMP click → `DroneDropJump`), and `Assets/ECA/BermudaBRRules.eca` (sends `DroneDropEnroll` after `safeZoneOpened`, and `DroneDropCleanup` from `OnPlayerDead` / `OnPlayerRevive` / `OnPlayerDisconnect`).

## Mandatory workflow

- Read `AGENTS.md` and `.craftland/agents/rules/fc-workflow.mdc`; follow it (symbol lookup, asset check, registration, lint-fix).
- `.eca`, `.ui`, `.scene` are editor-owned: **no hand-editing JSON**. Use Craftland Studio MCP (read `instructions://index` first). If MCP is not connected, stop and tell me to open Craftland Studio. `.fcg` may be edited as text.
- Verify every API against stdlib `.fcc` / `EditorGenLib.fcc` / ugc-mcp docs; don't invent names.
- Finish with the lint loop (`fc-lint-fix`) and MCP `game-build` + `game-console-get-logs`.
- Do not touch unrelated files (`.codex/config.toml`, `.cursor/mcp.json`, `fcconfig.json`, `main.gproj`) and don't mass-change `unlockedValue` in scenes.

## Suspected root causes (from code review — verify each, don't assume)

1. **Client camera init is gated on `box<Visibility>.IsVisible` and `HasComponent(box, Transform)`** (`DroneJumpCameraClient.TryInitialize`). The box is a `Primitive/Empty` with `"visible": false`; the server flips `IsVisible` but it may never replicate to the client → `TryInitialize` returns silently every frame, camera never switches. The camera only needs the box position.
2. **`DroneDropEnroll` has several silent early returns before `CreateCustomUI`**: `!safeZoneOpened`; `hasJumped` already contains the UID; `PlayerStatus != Alive` (marks the player as jumped and skips). In particular `CleanupPlayer` writes a **tombstone** (`hasJumped[uid] = true`) for UIDs not yet in the map, so a `DroneDropCleanup` from `OnPlayerDead` / `OnPlayerRevive` / `OnPlayerDisconnect` arriving before enroll permanently blocks that player's HUD for the phase.
3. If `CreateCustomUI` returns `nil`, the code fail-opens into `Jump(player)` — player skydives immediately with no UI/camera and no log. Check whether `EResUI.DroneJumpHUD` is correctly registered and whether `CreateCustomUI(out var hud, player, ...)` is valid for this owner/timing.
4. **Registration/attachment**: `ProjectSettings/ResourceRegisteration.asset` lost the `ugcscriptclient` / `ecaclient` categories, and `DroneJumpCameraClient.fcg` / `DroneJumpHUDServer.fcg` are not registered. Check via MCP (`asset-get` on the UI, `fc-asset-registration-list`) that the touch pad really has the client script attached and the root has the server script + relay, and that registration is complete. Fix through MCP/registration tools only.
5. Side effect: `Invincible = true` is set on enroll; make sure every path (jump, death, disconnect, phase end, enroll failure) restores it.

## Required changes

**A. Add diagnostic logs first** (use the stdlib log/print API — verify its name; server and client variants). Log, with player UID:
- server: `DroneDropEnroll` entry, and each early-return reason (`safeZoneOpened=false`, `already enrolled/tombstoned`, `not alive`, `bot`), the result of `CreateCustomUI` (nil or not), `Jump` entry, `DroneDropCleanup` entry and its source.
- client: `OnAwake`/`OnEnable` of the camera script, each failed `TryInitialize` condition (throttled, e.g. once per second, not every frame), successful camera switch, `RestoreCamera`.
Keep logs behind one `DebugLog bool` constant so they can be turned off.

**B. Fix the camera (suspect 1)**
- Remove the `IsVisible` gate. Initialize as soon as the box entity exists on the client (verify `HasComponent(box, Transform)` semantics, or just wait until `EResSceneBRBermuda.DroneDropBox` resolves and its position is non-default).
- If the box entity cannot be resolved on the client at all, find the supported way (e.g. server passes the box via UI/relay, or `aoiSetting`/AOI follow, or setting the box `visible: true` with a transparent/empty look in the scene through MCP) and explain the choice. Make sure the box is inside the player's AOI (the server already calls `AddAOISpecialFollow` + `SwitchAOIProxyObserver`; verify they work together with the player's own position far below).
- Keep: look-around via touch pad `OnDrag`, `SetLocalPlayerMoveStatus(false)` while active, restore on jump/disable/destroy.

**C. Fix the HUD flow (suspects 2–3)**
- Tombstones must not block a player who has simply not been enrolled yet: separate "phase-ended/jumped" from "cleaned up before enroll", or ignore `DroneDropCleanup` for UIDs not enrolled in the current phase **unless** the player is currently dead/disconnected. A revived player must still not be put into the box camera (revive uses the existing sky-drop flow) — keep that guarantee explicitly.
- If a player is not Alive at enroll time, do not silently drop them without a log; keep the rule but log it.
- If `CreateCustomUI` fails, log an error before the fail-open `Jump`.
- Ensure `Invincible` and AOI proxy are restored on every exit path.

**D. Registration/attachment (suspect 4)**
- Via MCP, verify and (if needed) repair: UI registered (`EResUI.DroneJumpHUD`), `JumpButton` `clickHandlers` pointing to `DroneJumpHUDServer.OnClick`, touch pad has `DroneJumpCameraClient`, box entity has `DroneDropServer`, and all new scripts are registered in the right category (including client scripts). Report what was wrong.

**E. Verify**
- Lint + MCP `game-build` + `game-console-get-logs`. If you can run a play-test through MCP, do it and paste the relevant log lines per player. If you cannot run the game, say so clearly and list which behaviors are unverified.

## Deliverables
1. The fixes (via MCP for editor-owned assets), minimal diff, no unrelated changes.
2. A root-cause report: for each of the 5 suspects — confirmed / ruled out, with the evidence (code line, MCP asset state, or log).
3. Build/lint results, honestly stating anything unverified.
4. A manual test checklist with the exact log lines to expect: match start → enroll logs → HUD created → client camera switched; drag to look around; JUMP → teleport to box + parachute, camera/HUD restored, invincibility restored; AFK auto-jump at 30 s; player dies/disconnects before the phase; revive does not re-enter the box camera.

Ask me before changing behavior not covered above.
