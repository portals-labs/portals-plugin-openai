# Server Scripts (server.js)

Source: [https://portals.to/documentation/web-games/server-scripts](https://portals.to/documentation/web-games/server-scripts) — verbatim copy of the official Portals documentation.

Without one of these, there is no server-side game logic on Portals: every client is authoritative over what it sends, and a session is exactly as trustworthy as its least trustworthy player. A **server script** changes that: add a single file named `server.js` at the root of your project and Portals runs it on its own servers as an invisible, authoritative participant in every multiplayer session of your game.

**Every multiplayer game should ship with one.** For a real-time game the preferred form is to run the whole simulation in it — [Server-Simulated Games](https://portals.to/documentation/web-games/server-sim) is that architecture and the default choice for multiplayer on Portals. Even a game that doesn't simulate on the server needs its referee — someone no player controls:

- A **lobby** with ready checks that starts the game for everyone at the same moment
- **Countdowns and timers** that must not drift or depend on any one player's tab
- **Authoritative state** players can read but not overwrite (phase, verified scores, turn order)
- **Kicking** misbehaving players from a session

Going without is for the narrow cases where nothing is contested — purely cosmetic co-op canvases and emote walls — and even there, [Multiplayer & Voice](https://portals.to/documentation/web-games/multiplayer-and-voice) applies: never let a peer's messages control anything valuable.

In the AI editor you rarely write it by hand: ask for the behavior — *"add a lobby with ready checks that starts a 3-second countdown when everyone is ready"* — and the AI writes both `server.js` and the client code that talks to it.

## How it runs

- The moment players start joining your game's multiplayer session, Portals starts your server script and joins it to that session. One server runs per session — channels partition servers exactly like they partition players.
- The server is **invisible**: it never appears in `net.players()` and does not count toward the 50-player session cap.
- When you publish a new version, running servers are swapped to the new script within a few seconds. In the editor preview, the same happens with your draft's `server.js` — edit it, reload the preview, and the new code is live moments later.
- A session that sits empty for about five minutes ends its server. When players return, a fresh one starts.
- If your script crashes or exceeds its budgets, the session keeps going **without** it — players are never disconnected because a script failed. A new server starts on the next join.

That last point is a design rule: **keep the game playable when the server is absent or still connecting.** Gate lobby UI on a server-owned state key appearing rather than assuming the server is already there.

## Writing server.js

`server.js` is a plain script — no `import` or `require`, no DOM, no `window`, no network access, and no `Portals` global. Everything it can do comes from one frozen `server` global whose API mirrors `Portals.net`:

```js
server.setState("server:phase", "lobby");

server.on("playerjoin", (player, players) => {
  server.setState("server:lobby", { players: players.length });
});

server.on("message", (data, fromId) => {
  if (data.kind === "ping") server.send({ kind: "pong", to: fromId });
});
```

Handlers receive the same shapes clients see: players are `{ id, playerId, displayName, avatarUrl }`, messages and state values are plain JSON.

## Server-only powers

Two things only the server can do:

**Protected state.** State keys starting with `server:` are writable only by the server script. A player who tries gets an `error` event with code `forbidden` and the write is dropped. Everything the server must own — game phase, countdown, verified results — belongs under this prefix; clients read it like any other state:

```js
// client
net.on("state", (key, value) => {
  if (key === "server:phase") applyPhase(value);
});
```

**Kick.** `server.kick(sessionId)` disconnects that player from the session. Pass the `id` from the roster or a message's `fromId`.

## Messages from the server

`server.send(data)` broadcasts to **all** players. On the client it arrives like any other message, but the sender's `fromId` will not match anyone in `net.players()` — the server is not in the roster. Recognize server messages by shape, not sender:

```js
// client
net.on("message", (data, fromId) => {
  if (data.kind === "start") beginRound(data);
});
```

## Limits

The script runs in a strict sandbox with hard budgets:

| Limit | Value |
| --- | --- |
| CPU per event or timer callback | ~50 ms (sustained heavy use is also capped) |
| Memory | 32 MB |
| Live timers | 16, intervals at least 50 ms |
| Broadcasts / state writes | ~60 and ~30 per second (3× a player's rate) |
| Message and state-value size | 8 KB JSON, same as clients |
| State keys | 64 per session, 128-character names |
| Script size | 512 KB |

An infinite loop, runaway allocation, or persistent errors end the script (the session survives, as above). `server.setTimeout(cb, ms)`, `server.setInterval(cb, ms)`, and `server.clearTimer(id)` replace the browser timer functions.

**`server.js` is a public file.** It ships in your published bundle like every other project file and anyone can read it. Never put secrets in it — its authority comes from *where it runs*, not from being hidden.

## A complete lobby

```js
// server.js — ready checks, countdown, kick support
const ready = new Set();
let countdownId = 0;

function publishLobby() {
  const players = server.players();
  for (const id of ready) {
    if (!players.some((p) => p.id === id)) ready.delete(id);
  }
  server.setState("server:lobby", { players: players.length, ready: ready.size });
  maybeStart(players);
}

function maybeStart(players) {
  if (countdownId || players.length < 2 || ready.size < players.length) return;
  let remaining = 3;
  server.setState("server:countdown", remaining);
  countdownId = server.setInterval(() => {
    remaining -= 1;
    server.setState("server:countdown", remaining);
    if (remaining <= 0) {
      server.clearTimer(countdownId);
      countdownId = 0;
      server.setState("server:phase", "playing");
      server.send({ kind: "start" });
    }
  }, 1000);
}

server.setState("server:phase", "lobby");
server.on("playerjoin", publishLobby);
server.on("playerleave", publishLobby);
server.on("message", (data, from) => {
  if (data && data.kind === "ready") { ready.add(from); publishLobby(); }
  if (data && data.kind === "unready") { ready.delete(from); publishLobby(); }
});
```

The matching client stays ordinary `Portals.net` code:

```js
// client
const net = Portals.net;
await net.join();

readyButton.onclick = () => net.send({ kind: "ready" });

net.on("state", (key, value) => {
  if (key === "server:lobby") renderLobby(value);        // {players, ready}
  if (key === "server:countdown") renderCountdown(value);
  if (key === "server:phase" && value === "playing") beginRound();
});

// Until "server:phase" appears the server hasn't connected yet —
// show "waiting for lobby…" and keep a solo mode available.
```

## Debugging

Test in the editor preview: it runs your **draft's** `server.js` against a real session, so you and a second tab can exercise the whole flow before publishing. After editing the script, reload the preview to swap the running server.

Sessions joined from a local dev server with a [dev token](https://portals.to/documentation/web-games/multiplayer-and-voice#test-multiplayer-locally) get a server too, but it runs your **published** `server.js` — Portals fetches the script from your live release, never from your machine. Iterate on the script itself in the editor preview; use local sessions to develop the client against the published server behavior.

`server.log(...)` output appears in the editor's **Server logs** tab: click the bug icon in the preview toolbar and switch tabs. It shows your script's lines (marked `[script]`) and the platform's own — session starts, crashes, budget violations — for the last hour up to the last day, and a script failure lights the bug icon by itself. For values you want to watch continuously, a state key still works well:

```js
server.setState("server:debug", { readyCount: ready.size, at: Date.now() });
```

## API reference

| Method | Result |
| --- | --- |
| `server.on(event, handler)` | Subscribe to `message`, `playerjoin`, `playerleave`, `state`. |
| `server.send(data)` | Broadcasts up to 8 KB of JSON to **all** players. |
| `server.setState(key, value)` | Writes shared state; only the server may write `server:*` keys. |
| `server.getState(key?)` | Reads one key, or the full state object, from the local mirror. |
| `server.players()` | Roster snapshot — the server itself is never in it. |
| `server.kick(sessionId)` | Disconnects that player from the session. |
| `server.setTimeout(cb, ms)` / `server.setInterval(cb, ms)` | Sandbox timers (max 16 live, ≥ 50 ms intervals). |
| `server.clearTimer(id)` | Cancels a timer from either function. |
| `server.log(...values)` | Diagnostic logging, shown in the editor's Server logs tab. |

For the client-side API, sessions, and channels, see [Multiplayer & Voice](https://portals.to/documentation/web-games/multiplayer-and-voice). To run the game simulation itself on the server — shared physics, snapshots, prediction, and graceful fallback — see [Server-Simulated Games](https://portals.to/documentation/web-games/server-sim).
