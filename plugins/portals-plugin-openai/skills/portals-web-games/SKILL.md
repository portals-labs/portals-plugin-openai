---
name: portals-web-games
description: Build, update, synchronize, configure, and publish browser games on Portals with the portals-web-games MCP server. Use for Portals game projects, portals.to/my-games, pushing or pulling game source, Portals SDK identity/saves/leaderboards, Portals.net multiplayer, Portals.voice, Guardian avatars, local multiplayer tokens, and Portals AI Lab image, texture, 3D, speech, sound, or music assets.
---

# Portals Web Games

Use the `portals-web-games` MCP tools for remote Portals state and ordinary workspace tools for local source files.

## Load the platform rules

Before creating or changing a game, read [references/portals-web-games.md](references/portals-web-games.md) completely. Treat it as authoritative for the injected SDK, sandbox restrictions, multiplayer and voice limits, Guardian avatar integration, local development, and generated-asset rules. Those instructions are bundled from this MCP project's `src/instructions.ts`.

For the full official documentation on one subsystem, use the dedicated skill instead of guessing at the API:

- `portals-sdk` — identity, saved progress, casual scores, leaderboards, host context.
- `portals-multiplayer-and-voice` — `Portals.net` sessions and channels, text chat, `Portals.voice`, dev-token local testing.
- `portals-server-scripts` — a root `server.js` referee for lobbies, ready checks, authoritative countdowns, and kicking.
- `portals-server-sim` — running the game simulation itself on the server: shared physics, snapshots, client prediction, and fallback to host authority.
- `portals-guardian-avatars` — Guardian avatars, wearables, animation, and the character controller in Three.js.

## Choose the workflow

### Existing game

1. Call `list_web_games` to resolve the game and confirm its current state.
2. Pull with `pull_web_game_source` before editing unless the user explicitly wants to replace the remote source.
3. Use a dedicated game directory. A pull overwrites matching project files, so never target an unrelated repository root or a directory whose ownership is unclear.
4. Preserve the returned `revision` and pass it as `expectedRevision` to `push_web_game_source`. If the server reports `PROJECT_CHANGED`, stop and reconcile instead of overwriting newer edits.
5. Inspect and test the local build, then push the directory containing `index.html` at its root.

### New game

1. Call `create_web_game` and retain the returned game ID and editor URL.
2. Create the game in a dedicated local directory with `index.html` at its root.
3. Implement and test the game using the bundled platform rules.
4. Call `push_web_game_source` with the local game directory. Relay the returned `share_url` — it lets anyone play the pushed draft immediately, without publishing.
5. Use `update_web_game_settings` for publishing metadata and media. Report any remaining publishing requirements returned by the tool.
6. Publish with `publish_web_game` only when the user asks for the game to go public. Pass the `revision` from the push as `expectedRevision`.

### Settings-only or discovery task

- Use `list_web_games` before acting when the game ID is not already known.
- Use `update_web_game_settings` without pulling source when the request only changes metadata or media.
- Do not mutate source or settings when the user only asks to inspect or explain.

## Authentication

Let the server reuse `PORTALS_ACCESS_KEY` or saved credentials first. Call `authenticate` only when the other tools report that authentication is required. Never print, copy, or commit access keys.

## Build rules

- Keep `_portals` SDK files platform-managed. Follow the reference for the limited local-development exception.
- Default to proposing or adding a lightweight `Portals.net` feature when it suits the game. Respect the user's explicit choice to omit multiplayer.
- Handle rejected SDK promises and unsigned-player states.
- Never use client-reported scores or peer messages for valuable entitlements.
- Keep runtime assets inside the pushed bundle and reference them with paths relative to `index.html`; published games cannot load arbitrary CDN assets.
- Remove `window.__PORTALS_DEV__` tokens before committing or pushing.

## Generated assets

Generated assets spend account credits and cannot be undone. Generate only when the user requested the asset or approved that part of the work.

1. Call `list_generated_assets` before paying to regenerate something reusable.
2. Save outputs inside the game directory with a push-compatible `outputPath`.
3. For 3D jobs, continue with `check_3d_model_task` until the file is actually saved; a task ID is not a completed model.
4. Use `list_voices` before speech generation when the voice is not specified.
5. Push the saved local asset with the rest of the game bundle.

## Finish safely

- Run the project's relevant local checks before pushing when they are available.
- Summarize the game ID, local directory, resulting revision, and editor/share/play URLs returned by the tools. `get_web_game_share_link` re-fetches the shareable draft link when it is needed outside a push.
- Distinguish a successful source push from publication. A push only updates the private draft; `publish_web_game` is what releases the build to players and lists the game.
- Treat publishing as an explicit user decision — never publish to finish a build task. It is capped at 10 per day, it replaces what live players get, and this MCP cannot unpublish. When `publish_web_game` returns a checklist of missing metadata, fill it with `update_web_game_settings` and report what changed before retrying.
