# Codex Traffic Light

A small macOS menu bar app that shows Codex local session status as a floating traffic light.

- Red count: sessions blocked waiting for approval or the next user action.
- Yellow count: sessions currently running.
- Green count: sessions idle or finished.

Counts are based on stable Codex thread/session identifiers, not individual turns. The emphasized lamp shows the main action state: blocked sessions win first, finished sessions win over running sessions when nothing is blocked, and running wins when only running sessions exist. Sessions inactive for more than 24 hours are dropped from the display. Lamp numbers are capped at `99`.

The app also considers lightweight runtime health:

- If the Codex desktop app exits, stale app-origin `running` or `blocked` sessions stop affecting the red/yellow lamps after a short grace period.
- If `~/.codex/config.toml` explicitly sets `cli_auth_credentials_store = "file"` and `auth.json` is missing, the app treats authentication as a global red state.
- Diagnostics show Codex app process state, auth cache state, hook freshness, ignored stale app sessions, and recent hook events.

## Interaction

- Click the red lamp to open the latest blocked Codex session.
- Click the yellow lamp to open the latest running Codex session.
- Click the green lamp to open the latest completed Codex session.
- The menu bar item also exposes **Open Latest Blocked Session**, **Open Latest Running Session**, and **Open Latest Completed Session** actions.

Session opening uses local Codex deep links in the form `codex://threads/<thread-id>`. Only UUID-shaped stable thread/session keys can be opened this way. Fallback keys are still counted, but Diagnostics marks them as `not linkable` and clicking the lamp will do nothing for those sessions.

Sound alerts are enabled by default and can be toggled from **Sound Alerts: On/Off** in the menu. The app plays a macOS built-in alert sound when a new blocked session appears and a completion sound when a session reaches `idle/stop`. Running events do not play sounds.

The app does not play sounds for historical state loaded at launch, and repeated polling of the same event does not replay the same sound.

The app uses Codex lifecycle hooks to update:

- `UserPromptSubmit`, `PreToolUse` -> running
- `PermissionRequest` -> blocked
- `Stop` -> idle

## Build

```bash
swift test
scripts/build_app.sh
open dist/CodexTrafficLight.app
```

To create an unsigned Apple Silicon beta ZIP for internal testing:

```bash
sh scripts/package_beta.sh
```

After opening the app, use the menu bar item to run **Install / Repair Codex Hooks**. Then open Codex, type `/hooks`, and trust the `CodexTrafficLight` hooks.

If the app detects that trusted hooks already point to an older helper binary, it replaces only the helper on launch without rewriting `hooks.json`. When the old helper contained the turn-level parser, the app backs up and resets `state.json` so stale turn-count entries stop affecting the lights.

## State File

The shared state lives at:

```text
~/Library/Application Support/CodexTrafficLight/state.json
```

The helper binary installed into Application Support updates this file under a file lock so concurrent hook invocations do not corrupt state.

For isolated testing, set `CODEX_TRAFFIC_LIGHT_SUPPORT_DIR` to point the app or helper at a temporary state directory.

If you used an older build that counted turn IDs, remove `state.json` once after upgrading so old turn-level entries do not remain visible until they expire.

You can also use **Reset Session Counts** from the menu bar item. The app writes a `state.json.codex-traffic-light.*.bak` backup before clearing the current counts.
[README.md](https://github.com/user-attachments/files/28786921/README.md)
