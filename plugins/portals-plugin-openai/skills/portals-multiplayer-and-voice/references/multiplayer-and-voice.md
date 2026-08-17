# Multiplayer, Chat, and Voice

Source: [https://portals.to/documentation/advanced-tooling/multiplayer-and-voice](https://portals.to/documentation/advanced-tooling/multiplayer-and-voice) — verbatim copy of the official Portals documentation.

Hosted web games run in a sandbox with no outside network access: `fetch`, WebSocket, and WebRTC connections to other servers fail, and networking libraries cannot work. `Portals.net` and `Portals.voice` are the only multiplayer and voice transports — the Portals host page owns the real connections on the game's behalf.

Both come with the SDK your project already includes:

```html
<script src="./_portals/sdk.js"></script>
<script src="./game.js"></script>
```

## Sessions and channels

A session is the set of players of your game in the same bucket. On the game page, that is everyone who opened the game; inside a Portals room, it is the people in that room. Voice partitions the same way, so the people you play with are the people you talk to.

`join({ channel })` creates a private sub-lobby within that bucket — useful for parties or match codes. A channel may use letters, numbers, colons, underscores, and hyphens, must start with a letter or number, and may be at most 64 characters.

### Global channels

Sessions are regional: fast-paced games place each region's players in their own copy of a channel, so nearby players meet by default and stay close to the server. Prefix a channel with `global:` to make it worldwide instead — every region joins the same room:

```js
// One lobby for the whole world
await net.join({ channel: "global:lobby" });

// A match arranged there must be global too, or the players
// split back into their own regions' rooms and never meet.
await net.join({ channel: "global:match-x7q2" });
```

A global room runs in one place (the US home region), so distant players see higher latency. Use global channels for lobbies, matchmaking, and chat; keep fast-paced matches on regional channels unless a match deliberately spans regions. Slow-paced games (turn-based, chat-like traffic) are consolidated into one worldwide room automatically — the prefix guarantees it for a channel regardless of pace.

### Forcing a region

`join({ region })` puts the session on Portals' US or EU servers for every player, wherever they are. The only values are `"us"` and `"eu"`:

```js
// Everyone in this session connects to the EU servers
await net.join({ region: "eu" });

// A pin combines with a sub-lobby
await net.join({ channel: "match-x7q2", region: "us" });
```

A pin outranks everything else that picks a region: the player's own location, the automatic consolidation of slow-paced games, and a `global:` prefix. Three things follow from that:

- **Distant players pay the latency.** Pin when the game needs one known location for everyone — a scheduled event, a persistent world, a region your players share — and leave it off otherwise, since the default already keeps each player near their server.
- **Every player of a room must pass the same pin.** The pin chooses which room you join, so a game that pins `"eu"` for some players and nothing (or `"us"`) for others splits them into separate rooms. That is also how you run deliberate US and EU rooms of one channel: let the player pick, and treat the two as separate worlds.
- **A pin never falls back.** If the pinned region cannot take the session, `join()` rejects rather than quietly connecting somewhere else — one more reason for the catch that keeps the game playable solo.

## Join a multiplayer session

```js
const net = Portals.net;

async function startMultiplayer() {
  try {
    const session = await net.join();
    console.log("I am", session.self.id);
    console.log("Players", session.players);
    console.log("Shared state", session.state);
  } catch (error) {
    console.warn("Multiplayer is unavailable", error);
    // Keep the game fully playable solo.
  }
}
```

The resolved session is a plain data snapshot, not a handle: every method stays on `Portals.net` itself, before and after joining. `net.players()`, `net.self()`, and `net.getState()` read the live local mirror synchronously once joined.

Keep the roster current with events:

```js
net.on("playerjoin", (player, players) => renderRoster(players));
net.on("playerleave", (player, players) => renderRoster(players));
```

Each player is `{ id, playerId, displayName, avatarUrl }`. `id` identifies the connection (two tabs are two players); `playerId` is the game-scoped identity from the Portals SDK and is `null` for signed-out players, so always render a fallback name.

## Broadcast messages

`net.send(data)` broadcasts up to 8 KB of JSON to the **other** players — your own client does not receive its own broadcast:

```js
net.send({ kind: "shoot", x: 120, y: 48 });

net.on("message", (data, fromId) => {
  if (data.kind === "shoot") spawnShot(data.x, data.y, fromId);
});
```

Use `send` for transient events. Anything a late joiner must see belongs in shared state.

## Shared state

`net.setState(key, value)` writes last-write-wins state that late joiners receive in `join().state`. Values may be at most 8 KB of JSON, keys at most 128 characters, and a session may hold at most 64 keys.

```js
net.setState("phase", "playing");

net.on("state", (key, value) => {
  if (key === "phase") applyPhase(value);
});
```

The `state` event fires for everyone **including the writer**, so apply changes from the event instead of duplicating them at the call site.

## Rate limits and update cadence

Each player may send about 20 broadcasts and 10 state writes per second. Never send per-frame updates — sample input or positions on a fixed timer (100–150 ms) and interpolate between updates locally:

```js
setInterval(() => {
  if (joined) net.send({ kind: "pos", x: me.x, y: me.y });
}, 120);
```

## Give the session an authority

Without a server script there is no server-side game logic: every client is authoritative over what it sends, so the session is exactly as trustworthy as its least trustworthy player. Never let a peer's messages control currency, prizes, access, or another valuable entitlement — and for a multiplayer game, don't stop at trusting peers less: give the session an authority no player controls.

**The default design for a multiplayer game is a server-simulated one** — a root `server.js` that runs your game's simulation on Portals servers, so outcomes are witnessed rather than claimed and no player's tab holds an advantage. [Server-Simulated Games](https://portals.to/documentation/advanced-tooling/server-sim) is the architecture; [Server Scripts](https://portals.to/documentation/advanced-tooling/server-scripts) is the underlying contract, and its lighter referee form (lobbies, ready checks, authoritative countdowns, kicking) is the minimum a multiplayer game should ship with.

The exceptions are narrow: purely cosmetic shared spaces (a co-op drawing canvas, an emote wall) and slow turn-based games whose pace hides latency can ship trust-light, with clients simply rendering what peers report.

## Handle disconnects

There is no automatic reconnect. On connection loss, tell the player and offer a rejoin that calls `join()` again:

```js
net.on("status", (status) => {
  if (status === "disconnected") showRejoinButton();
});
```

## Text chat

In-game text chat rides the same multiplayer relay — never a third-party chat service. Use a `kind` field so chat coexists with gameplay messages:

```js
const chatInput = document.querySelector("#chat-input");
const chatLog = document.querySelector("#chat-log");

chatInput.addEventListener("keydown", (event) => {
  if (event.key !== "Enter" || !chatInput.value.trim()) return;
  const text = chatInput.value.trim().slice(0, 300);
  net.send({ kind: "chat", text });
  appendChat(net.self(), text); // own sends are not echoed back
  chatInput.value = "";
});

net.on("message", (data, fromId) => {
  if (data.kind !== "chat" || typeof data.text !== "string") return;
  const sender = net.players().find((p) => p.id === fromId);
  appendChat(sender, data.text.slice(0, 300));
});

function appendChat(sender, text) {
  const line = document.createElement("div");
  const name = document.createElement("strong");
  name.textContent = sender?.displayName || "guest";
  const body = document.createElement("span");
  body.textContent = " " + text;
  line.append(name, body);
  chatLog.append(line);
  chatLog.scrollTop = chatLog.scrollHeight;
}
```

Chat text is untrusted player input: cap its length and render it with `textContent`, never `innerHTML`.

Messages are transient with no history. When late joiners should see recent chat, keep a small rolling array in shared state and seed the UI from `join().state`:

```js
const log = (net.getState("chatlog") || []).slice(-20);
net.setState("chatlog", [...log, { name: myName, text }]);
```

Typing speed stays far below the rate limit, so no batching is needed.

## Voice chat

`Portals.voice` connects the players of the same session by voice. The Portals host owns the microphone and the connection: when your game asks to join, Portals shows the player its own microphone consent card and handles the browser permission. Your game never accesses the mic — never call `getUserMedia` and never add WebRTC or voice libraries.

Because consent is Portals' job, call `voice.join()` automatically during startup when your game has voice chat — no "join voice" button is needed:

```js
const voice = Portals.voice;

async function startVoice() {
  try {
    const session = await voice.join();
    renderVoiceRoster(session.participants);
    showMuteButton(session.muted);
  } catch (error) {
    // No host, editor preview, in-room play (the room's own voice already
    // runs), the player declined the microphone, or connecting failed.
    showVoiceOffControl(); // a small "voice off — turn on" that calls startVoice() again
  }
}
```

The join promise stays pending while the player decides on the consent card, so do not race it against a short timer. A rejection must leave the game fully playable — voice is always optional, and the retry control lets a player who declined turn voice on later.

### Mute, roster, and speaking indicators

Start players with an easy-to-find mute button:

```js
muteButton.addEventListener("click", () => {
  voice.setMuted(!voice.muted());
  muteButton.textContent = voice.muted() ? "unmute" : "mute";
});
```

Participants share the multiplayer player shape. Keep the roster current and highlight who is talking:

```js
voice.on("participantjoin", (participant, participants) => renderVoiceRoster(participants));
voice.on("participantleave", (participant, participants) => renderVoiceRoster(participants));

voice.on("speaking", (ids) => {
  // ids currently talking, self included — match against participant.id
  highlightSpeakers(ids);
});

voice.on("status", (status) => {
  if (status === "disconnected") showVoiceOffControl();
});
```

### Microphone selection

Players with a headset and a webcam need to pick which mic Portals captures from. `voice.devices()` lists the microphones and `voice.setDevice(deviceId)` switches to one — Portals still owns the device, so your game never touches `enumerateDevices` or `getUserMedia`:

```js
async function renderMicPicker(select) {
  // Rejects wherever voice itself is unavailable — hide the picker there.
  const devices = await voice.devices().catch(() => []);
  select.replaceChildren(
    ...devices.map((device, index) => {
      const option = document.createElement("option");
      option.value = device.deviceId;
      // Labels are empty until the player has granted the mic.
      option.textContent = device.label || "microphone " + (index + 1);
      option.selected = device.active;
      return option;
    }),
  );
}

select.addEventListener("change", async () => {
  try {
    await voice.setDevice(select.value);
  } catch (error) {
    renderMicPicker(select); // switch failed — show what is actually live
  }
});
```

`setDevice()` applies immediately during a session and is remembered for the next `join()`, so a player can also pick a mic before voice starts (unlabeled at that point, since the browser hides device names until permission is granted). The list is a snapshot — call `devices()` again when the player opens the picker so a headset plugged in mid-game shows up.

Voice UI belongs in your game's HTML overlay — mute toggle, mic picker, roster with speaking highlights, the "turn on" retry — not inside canvas rendering, so it stays clickable and accessible.

## Where each transport works

| Context | `Portals.net` | `Portals.voice` |
| --- | --- | --- |
| Game page (`portals.to/g/...`) | Yes | Yes |
| Inside a Portals room | Yes — the session is the room | No — the room's own voice chat already runs |
| Editor preview | Yes | No |
| Bundle opened outside Portals | Only with a [dev token](#test-multiplayer-locally) | No |

`join()` rejects wherever a transport is unavailable, which is why both joins need a catch that keeps the game playable.

## Test multiplayer locally

Normally multiplayer needs the Portals host page, so a game served from your own machine has none of it. A **dev token** removes that limit: with one declared, `net.join()` talks to the real Portals multiplayer service directly, so you can develop against live infrastructure from a local dev server (vite, `python -m http.server`, anything that serves your files).

Mint a token for a game you own — with your Portals access key, or a signed-in Firebase bearer token:

```sh
curl -X POST https://portals.to/api/v2/arcade/dev-token \
  -H "x-access-key: YOUR_ACCESS_KEY" \
  -H "Content-Type: application/json" \
  -d '{"gameId":"YOUR_GAME_ID"}'
```

The SDK itself is served at [portals.to/portals-sdk/sdk.js](/portals-sdk/sdk.js). Download it into your project at `_portals/sdk.js` — the same path publishing stamps it to, so your HTML works unchanged locally and on Portals (downloaded projects already have it there):

```sh
mkdir -p _portals && curl -o _portals/sdk.js https://portals.to/portals-sdk/sdk.js
```

The dev-token response is `{ token, expiresAt }`. Declare it on `window` **before** the SDK script tag:

```html
<script>
  window.__PORTALS_DEV__ = { token: "pdev_..." };
</script>
<script src="./_portals/sdk.js"></script>
<script src="./game.js"></script>
```

That is the whole setup — `net.join()`, messages, shared state, and events now work exactly as on Portals. Open the page in two tabs and each sees the other join.

What to know:

- **Local sessions are fenced off.** They join a `dev` channel namespace (`join({ channel: "lobby" })` becomes `dev:lobby`), so a half-built local build never meets the live players of your published game.
- **The token is a credential.** It grants multiplayer sessions on your game for 8 hours. Treat it like a password: never commit it, never ship it in a bundle. When it expires, `join()` rejects with a message saying so — mint a new one.
- **Server scripts run too.** A published `server.js` joins your local sessions like any other; Portals runs it, not your machine (see [server scripts](https://portals.to/documentation/advanced-tooling/server-scripts)).
- **Voice stays Portals-only.** `voice.join()` still rejects outside Portals.
- Everything else that needs the host — saves, leaderboards, identity — keeps its documented no-host behavior (no-ops and `null`s).

## API reference

| Method | Result |
| --- | --- |
| `net.join(options?)` | Joins the session, optionally with `{ channel, region }`; resolves to `{ self, players, state }`. |
| `net.leave()` | Leaves the session. |
| `net.send(data)` | Broadcasts up to 8 KB of JSON to the other players. |
| `net.setState(key, value)` | Writes shared last-write-wins state for the session. |
| `net.getState(key?)` | Reads one key, or the full state object, from the local mirror. |
| `net.players()` / `net.self()` | Read the live roster / own player synchronously. |
| `net.on(event, handler)` / `net.off(event, handler)` | Subscribe to `message`, `playerjoin`, `playerleave`, `state`, `status`. |
| `voice.join(options?)` | Joins voice; Portals asks the player for the microphone. Resolves to `{ self, participants, muted }`. |
| `voice.leave()` | Leaves voice chat. |
| `voice.setMuted(muted)` / `voice.muted()` | Toggle / read the local microphone mute. |
| `voice.devices()` | Lists the microphones as `{ deviceId, label, active }` for a mic picker. |
| `voice.setDevice(deviceId)` | Captures from that microphone, now and on the next join. |
| `voice.participants()` / `voice.self()` | Read the live voice roster / own participant synchronously. |
| `voice.on(event, handler)` / `voice.off(event, handler)` | Subscribe to `participantjoin`, `participantleave`, `speaking`, `status`. |

TypeScript declarations for every type on this page are available at [portals.d.ts](/portals-sdk/portals.d.ts). For identity, saved progress, and leaderboards, see [Portals SDK](https://portals.to/documentation/advanced-tooling/portals-sdk).
