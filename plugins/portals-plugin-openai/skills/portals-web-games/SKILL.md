---
name: portals-web-games
description: Build, update, synchronize, configure, and publish browser games on Portals with the portals-web-games MCP server. Use for Portals game projects, portals.to/my-games, pushing or pulling game source, Portals SDK identity/saves/leaderboards and host UI safe areas, Portals.economy Coin microtransactions and product catalogs, Portals.net multiplayer, Portals.voice, Guardian avatars, local multiplayer tokens, and Portals AI Lab image, texture, 3D, speech, sound, or music assets.
---

# Portals Web Games

Use the `portals-web-games` MCP tools for remote Portals state and ordinary workspace tools for local source files.

## Load the platform rules

Before creating or changing a game, read [references/portals-web-games.md](references/portals-web-games.md) completely. Treat it as authoritative for the injected SDK, host-owned UI reserve, Coin products, sandbox restrictions, multiplayer and voice limits, Guardian avatar integration, local development, and generated-asset rules. Those instructions are bundled from this MCP project's `src/instructions.ts`.

For the full official documentation on one subsystem, use the dedicated skill instead of guessing at the API:

- `portals-sdk` — identity, saved progress, casual scores, leaderboards, host context.
- `portals-multiplayer-and-voice` — `Portals.net` sessions and channels, text chat, `Portals.voice`, dev-token local testing.
- `portals-server-scripts` — a root `server.js` referee for lobbies, ready checks, authoritative countdowns, and kicking.
- `portals-server-sim` — running the game simulation itself on the server: shared physics, snapshots, client prediction, and fallback to host authority.
- `portals-guardian-avatars` — Guardian avatars, wearables, animation, and the character controller in Three.js.
- `portals-game-economy` — selling in-game products for Coins: the product catalog, `Portals.economy` purchases, and the purchase sandbox.

## Choose the workflow

### Existing game

1. Call `list_web_games` to resolve the game and confirm its current state.
2. Pull with `pull_web_game_source` before editing unless the user explicitly wants to replace the remote source.
3. Use a dedicated game directory. A pull overwrites matching project files, so never target an unrelated repository root or a directory whose ownership is unclear.
4. Preserve the returned `revision` and pass it as `expectedRevision` to `push_web_game_source`. If the server reports `PROJECT_CHANGED`, stop and reconcile instead of overwriting newer edits.
5. Inspect and test the local build, then push the directory containing `index.html` at its root. Pass the optional `tag` when the user named a label for the push.

### New game

1. Call `create_web_game` and retain the returned game ID and editor URL.
2. Create the game in a dedicated local directory with `index.html` at its root.
3. Implement and test the game using the bundled platform rules.
4. Call `push_web_game_source` with the local game directory, plus the optional `tag` when the user named a label for the push. Relay the returned `share_url` — it lets anyone play the pushed draft immediately, without publishing. Leaderboards work there and in the editor preview, on a separate draft board, so a leaderboard can be tested before publishing.
5. Use `update_web_game_settings` for publishing metadata and media. Report any remaining publishing requirements returned by the tool.
6. Publish with `publish_web_game` only when the user asks for the game to go public. Pass the `revision` from the push as `expectedRevision`. The release inherits the draft's label, so pass a `tag` here only to name that release something different.

### Settings-only or discovery task

- Use `list_web_games` before acting when the game ID is not already known.
- Use `update_web_game_settings` without pulling source when the request only changes metadata or media.
- **Latency-sensitive multiplayer** (region-based servers for fast-paced games instead of the default one worldwide US room) is not settable through `update_web_game_settings` — direct the user to the toggle on the game's settings page at portals.to/my-games. The `portals-multiplayer-and-voice` skill explains the routing.
- Do not mutate source or settings when the user only asks to inspect or explain.

### Coin product workflow

When the user asks for Coin microtransactions, configure the draft catalog yourself as part of the build. Do not send them to My Games to duplicate ordinary setup:

1. Call `list_web_games` when the game ID is not already known, then `get_game_economy_catalog` before writing code or naming a SKU.
2. If the user did not specify every product field, form a small coherent proposal from the requested game design and report the SKU, pricing, kind, grant, and limit assumptions. A new SKU and its kind are permanent catalog identity choices, so confirm those two fields before the first upsert; price, copy, grant, limit, and icon can be revised later. The agent still performs the catalog writes after confirmation.
3. Call `update_game_economy_catalog` with `action: "upsert"` and the returned `draft_revision`. An upsert replaces every field, so preserve intended existing values and use the new revision for the next product. Use `retire` only when permanent withdrawal is explicitly intended.
4. Implement the game against those exact SKUs and build its shop display from `Portals.economy.getCatalog()` rather than hardcoding product prices.
5. Use `test_game_economy_purchase` to exercise success, cancellation, insufficient balance, retryable failure, consume, and refund/revocation handling in the simulated purchase sandbox. The tool creates a fresh operation ID per action, so verify consume idempotency separately in game code or automated tests by repeating the same gameplay event ID.

Catalog tools only change the owner's draft. Portals review and an explicit owner publication are still required before products reach players; never publish merely to finish setup.

## Authentication

Let the server reuse `PORTALS_ACCESS_KEY` or saved credentials first. Call `authenticate` only when the other tools report that authentication is required. Never print, copy, or commit access keys.

## Build rules

- Keep `_portals` SDK files platform-managed. Follow the reference for the limited local-development exception.
- Keep essential and interactive HUD elements out of the host-owned top-left reserve. The current trigger is 44×44 CSS pixels at the larger of 12px and the matching device safe-area inset. Use `padding-left: calc(max(12px, env(safe-area-inset-left)) + 56px)` for a top HUD or the equivalent `padding-top` expression for a left HUD. The playfield may remain full-bleed.
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

## Tag a build

Both `push_web_game_source` and `publish_web_game` take an optional `tag`: a one-line label of 1–64 characters that Portals shows on the game's page at portals.to/my-games. Use it to make a build identifiable — `"wip-boss-fight"`, `"boss-fight-v2"`, `"1.4.0"`, a milestone name.

- Pass a tag when the user named one or asked for the build to be labelled. Never invent a label they did not ask for, and never guess a version number.
- **A push labels the draft.** The label describes the source now in the project, so a push without a tag clears the label the previous push left. Tag every push in a labelled series, or the label disappears on the next one.
- **A publish labels the released version, and inherits the draft's label when it passes no tag of its own.** So tagging the push is enough for the label to follow the build all the way to the live release; pass a tag on the publish only to name that release something different.
- The release label belongs to one version. A later publish does not keep it — it takes its own tag, or whatever the draft is carrying then.
- Report the `tag` the tool returns, not the one requested. It answers with what Portals actually stored, which is what a retried call already holds and what inheritance resolved to.
- A rejected tag fails the whole call with `INVALID_PAYLOAD` — nothing is pushed or released. Shorten a long or multi-line label, or drop it, and call again.
- A GitHub sync applied at portals.to/my-games replaces the whole source and clears the draft label, so a publish after one is unlabelled unless it passes its own tag.
- Nothing else consumes the label: it is developer-facing bookkeeping, never shown to players and never used for discovery or versioning.
