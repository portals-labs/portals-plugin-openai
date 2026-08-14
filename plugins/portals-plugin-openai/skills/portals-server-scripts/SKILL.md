---
name: portals-server-scripts
description: Add authoritative server-side game logic to a Portals multiplayer game with a root server.js that Portals runs as an invisible participant in every session. Use for lobbies and ready checks, drift-free countdowns and timers, server-owned state players cannot overwrite, kicking players, the server global, server-prefixed protected state keys, or the server sandbox limits.
---

# Portals server scripts (`server.js`)

Without one of these, there is no server-side game logic on Portals — every client is authoritative over what it sends, so a session is exactly as trustworthy as its least trustworthy player. A **server script** changes that: add a single file named `server.js` at the project root and Portals runs it on its own servers as an invisible, authoritative participant in every multiplayer session of the game.

Read [references/server-scripts.md](references/server-scripts.md) for the full API, a complete lobby example, and the limits table. It is the official documentation, copied verbatim.

## When to add one

**Every multiplayer game should ship with one.** For a real-time game the preferred form is running the whole simulation in it — the `portals-server-sim` skill covers that architecture, and it is the default choice for multiplayer on Portals. Even a game that doesn't simulate on the server needs its referee nobody controls: a lobby with ready checks that starts everyone at the same moment, countdowns and timers that must not drift or depend on one player's tab, authoritative state players can read but not overwrite (phase, verified scores, turn order), or kicking misbehaving players.

Going without is for the narrow cases where nothing is contested — purely cosmetic co-op canvases and emote walls — and even there, never let a peer's messages control anything valuable.

## How it runs

- One server per session; channels partition servers exactly like they partition players. It starts as players start joining.
- It is **invisible**: never in `net.players()`, and it does not count toward the 50-player session cap.
- Publishing swaps running servers to the new script within seconds. Editor preview runs the **draft's** `server.js` — reload the preview to swap it.
- A session empty for ~5 minutes ends its server; a fresh one starts when players return.
- If the script crashes or exceeds its budgets, **the session keeps going without it** — players are never disconnected because a script failed. A new server starts on the next join.

## Rules that are easy to get wrong

- **Keep the game playable when the server is absent or still connecting.** Gate lobby UI on a server-owned state key *appearing*, rather than assuming the server is already there, and keep a solo mode available.
- **`server.js` is a plain script**: no `import`/`require`, no DOM, no `window`, no network access, and no `Portals` global. Everything comes from one frozen `server` global whose API mirrors `Portals.net`.
- **Use the sandbox timers** — `server.setTimeout`, `server.setInterval`, `server.clearTimer` — not the browser ones.
- **`server:`-prefixed state keys are server-writable only.** A player who tries gets an `error` event with code `forbidden` and the write is dropped. Everything the server must own belongs under that prefix; clients read it like any other state.
- **`server.send(data)` broadcasts to all players**, but its `fromId` matches nobody in `net.players()`. On the client, recognize server messages by **shape**, not sender.
- **`server.js` is a public file.** It ships in the published bundle and anyone can read it. Never put secrets in it — its authority comes from *where it runs*, not from being hidden.
- Limits: ~50 ms CPU per callback, 32 MB memory, 16 live timers with intervals ≥ 50 ms, ~60 broadcasts and ~30 state writes per second, 8 KB JSON per message and state value, 64 state keys with 128-character names, 512 KB script size. An infinite loop, runaway allocation, or persistent errors end the script.

## Debugging

Iterate in the editor preview — it runs the draft script against a real session, so two tabs can exercise the whole flow before publishing. Sessions joined from a local dev server with a dev token also get a server, but it runs the **published** `server.js` fetched from the live release, never from the local machine; use local sessions to develop the client against published server behavior.

`server.log(...)` output appears in the editor's **Server logs** tab — click the bug icon in the preview toolbar and switch tabs. It shows the script's own lines (marked `[script]`) alongside the platform's — session starts, crashes, budget violations — for the last hour up to the last day, and a script failure lights the bug icon by itself. For values worth watching continuously, a `server:debug` state key still works well.

## Related

- Client-side sessions, channels, and the `Portals.net` API: the `portals-multiplayer-and-voice` skill.
- Running the game simulation itself on the server — shared physics, snapshots, prediction, graceful fallback: the `portals-server-sim` skill.
