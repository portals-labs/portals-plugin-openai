---
name: portals-sdk
description: Use the Portals SDK in a hosted web game for player identity, Portals sign-in, saved progress, casual scores, leaderboards, and the host-owned game UI safe area. Use when working with the global Portals object, _portals/sdk.js, Portals.ready, Portals.identity.requestLogin, saveState/loadState, submitScore, getLeaderboard, Portals.quit, playerId, standalone vs room host context, or avoiding the persistent top-left Portals controls.
---

# Portals SDK

The `Portals` global connects a hosted web game to the Portals player and host: identity, saved progress, casual scores, leaderboard reads, and closing the game.

Read [references/portals-sdk.md](references/portals-sdk.md) for the full API, code examples, and the method reference table. It is the official documentation, copied verbatim.

## Wiring

Portals stamps the SDK into every processed preview and published bundle. Include it from the game's own origin, before game code:

```html
<script src="./_portals/sdk.js"></script>
<script src="./game.js"></script>
```

Never download, edit, or bundle `_portals/sdk.js` — Portals replaces the managed copy on every process and publish. TypeScript declarations live at `https://portals.to/portals-sdk/portals.d.ts`; they are not part of the game's files.

## Host UI reserve

Every launched game keeps a trusted Portals controls trigger above the game iframe. The current closed trigger is 44×44 CSS pixels at `left: max(12px, env(safe-area-inset-left))` and `top: max(12px, env(safe-area-inset-top))`; a party badge may extend 4px beyond it. Keep essential and interactive HUD UI outside that top-left footprint. For a top HUD, use `padding-left: calc(max(12px, env(safe-area-inset-left)) + 56px)`; for a left HUD, use the equivalent top inset. A full-bleed non-interactive playfield is fine.

The host control is outside and above untrusted game code. Never try to hide, restyle, intercept, or out-z-index it. Its expanded panel may temporarily cover more of the game and owns focus while open, so pause or otherwise keep gameplay safe until focus returns.

## Rules that are easy to get wrong

- `await Portals.ready()` before reading the player or using any hosted capability. `session.context` is `standalone` on a game page, `room` inside a Portals room.
- Call `Portals.identity.requestLogin()` only from a direct player action such as a button click. Portals owns the sign-in UI — never ask for a Portals password or account credential in-game.
- `playerId` is stable per player *within one game* and deliberately different across games. It is not a Portals account ID and must not be used to correlate a player across games. It is `null` for signed-out players, so always render a fallback name.
- Saved state must be JSON-serializable and at most 64 KB encoded. Signed-out players cannot save; `loadState()` returns `null` when nothing is saved. Version the shape (`schemaVersion`) when it may change.
- Scores and saves require sign-in. A score mode is lowercase letters, numbers, and hyphens, max 32 characters, defaulting to `default`. `getLeaderboard` takes a limit of 1–100, default 10.
- Scores are client-reported. Never use them to award currency, paid prizes, access, or any other valuable entitlement.
- Every async method can reject — no host, invalid request, missing access, network failure. Catch at the player action that caused it and keep the game playable when an optional Portals feature is unavailable.
- Never put API keys, Firebase tokens, payment details, or signed asset URLs in game code, saved state, score modes, logs, or leaderboard UI.

## Related

- Selling in-game products for Coins — the one sanctioned way to grant a paid entitlement: the `portals-game-economy` skill.
- Real-time multiplayer, text chat, and voice: the `portals-multiplayer-and-voice` skill.
- Authoritative server-side game logic: the `portals-server-scripts` skill.
- Player avatars in a Three.js game: the `portals-guardian-avatars` skill.
