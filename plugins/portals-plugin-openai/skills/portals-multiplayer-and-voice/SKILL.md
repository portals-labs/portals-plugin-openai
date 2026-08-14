---
name: portals-multiplayer-and-voice
description: Add real-time multiplayer, in-game text chat, and voice chat to a hosted Portals web game with Portals.net and Portals.voice. Use when working with net.join, sessions and channels, global channels, net.send broadcasts, shared state, playerjoin/playerleave events, rate limits, disconnect handling, voice.join, mute, mic pickers, speaking indicators, or testing multiplayer locally with a dev token.
---

# Portals multiplayer, chat, and voice

`Portals.net` and `Portals.voice` are the *only* multiplayer and voice transports available to a hosted game. The sandbox has no outside network access: `fetch`, WebSocket, and WebRTC to other servers all fail, and networking or voice libraries cannot work. The Portals host page owns the real connections.

Read [references/multiplayer-and-voice.md](references/multiplayer-and-voice.md) for the full API, code examples, availability matrix, and dev-token setup. It is the official documentation, copied verbatim.

Both transports ship with the SDK the project already includes:

```html
<script src="./_portals/sdk.js"></script>
<script src="./game.js"></script>
```

## Multiplayer rules that are easy to get wrong

- A session is everyone in the same bucket — all players of the game page, or the people in one Portals room. `join({ channel })` makes a private sub-lobby: letters, numbers, colons, underscores, hyphens; must start alphanumeric; max 64 characters.
- Sessions are **regional**. A `global:` channel prefix shares one worldwide room. A match arranged in a global lobby needs a global channel too (`global:match-x7`), or players split back into their own regions and never meet. Global rooms run in the US home region, so keep fast-paced matches regional.
- `join()` resolves to a plain snapshot, not a handle — every method stays on `Portals.net`. Use `net.players()`, `net.self()`, `net.getState()` for live synchronous reads after joining.
- `send()` does **not** echo to the sender; the `state` event **does** fire for the writer. Apply state changes from the event, not at the call site.
- Limits: 8 KB JSON per message and per state value, 128-character keys, 64 keys per session. About 20 broadcasts and 10 state writes per player per second — never send per-frame; sample on a 100–150 ms timer and interpolate locally.
- Anything a late joiner must see belongs in shared state, not in a broadcast.
- There is no automatic reconnect. On status `disconnected`, show a rejoin control that calls `join()` again.
- Without a server script, every client is authoritative over what it sends — the session is exactly as trustworthy as its least trustworthy player. **The default design for a multiplayer game is a server-simulated one** (the `portals-server-sim` skill); at minimum ship a server-script referee (the `portals-server-scripts` skill). Validate every incoming message's kind and shape, and never let a peer's messages control currency, prizes, access, or another entitlement. Only purely cosmetic shared spaces and slow turn-based games should ship trust-light.
- Text chat rides the same relay with a `kind` field — never a third-party chat service. Cap length (e.g. 300 chars) on send and receive, and render with `textContent`, never `innerHTML`. For history late joiners see, keep a rolling array (last ~20) in shared state.

## Voice rules that are easy to get wrong

- Portals owns the microphone and consent. Never call `getUserMedia` or `enumerateDevices`, and never add WebRTC or voice libraries.
- Because consent is Portals' job, call `voice.join()` automatically at startup — no "join voice" button. The promise stays pending while the player answers the consent card, so never race it against a short timer.
- A rejection must leave the game fully playable. Show a small "voice off — turn on" control that retries.
- Always render an easy-to-reach mute button. Device labels are empty until mic permission is granted, and `devices()` returns a snapshot — re-call it when the picker opens so a mid-game headset appears.
- Put voice UI in HTML overlays, not canvas rendering, so it stays clickable and accessible.

## Availability

`Portals.net` works on the game page, inside Portals rooms, and in editor preview. `Portals.voice` works only on the game page — rooms run their own voice, and preview has none. Outside Portals, `net` works only with a dev token and `voice` never does. Both `join()` calls need a `catch` that keeps the game playable.

## Local testing

A dev token lets a locally served build talk to the real multiplayer service. Mint it against a game you own, declare `window.__PORTALS_DEV__` *before* the SDK script tag, and serve the SDK from `_portals/sdk.js` so the HTML is identical locally and on Portals. Local sessions are fenced into a `dev:` channel namespace. The token is a credential valid for 8 hours — never commit it, never ship it in a bundle, and strip it before pushing.

## Related

- Identity, saves, and leaderboards: the `portals-sdk` skill.
- Lobbies, ready checks, authoritative countdowns, kicking: the `portals-server-scripts` skill.
- Server-authoritative real-time simulation — shared physics, snapshots, prediction: the `portals-server-sim` skill.
