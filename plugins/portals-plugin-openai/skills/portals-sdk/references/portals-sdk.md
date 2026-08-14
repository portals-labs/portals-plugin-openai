# Portals SDK

Source: [https://portals.to/documentation/advanced-tooling/portals-sdk](https://portals.to/documentation/advanced-tooling/portals-sdk) — verbatim copy of the official Portals documentation.

The Portals SDK connects a hosted web game to the Portals player and host. It supports identity, saved progress, casual scores, leaderboard reads, and closing the game from either standalone or in-room play. For real-time multiplayer, in-game text chat, and voice chat, see [Multiplayer, Chat, and Voice](https://portals.to/documentation/advanced-tooling/multiplayer-and-voice). To put the player's Guardian avatar in a Three.js game — wearables, animation and a character controller — see [Guardian Avatars](https://portals.to/documentation/advanced-tooling/guardian-avatars).

Portals injects the SDK into every processed preview and published bundle. Your project includes it from its own game origin:

```html
<script src="./_portals/sdk.js"></script>
<script src="./game.js"></script>
```

Do not download or edit `_portals/sdk.js`. Portals replaces the managed copy when it processes or publishes the project.

Guardian avatars ship as a second, optional managed script. Add it beside this one only if you use it:

```html
<script src="./_portals/guardians-sdk.js"></script>
```

It gives you the global `PortalsGuardians` — avatars, wearables, animation and a character controller, gated by `await PortalsGuardians.ready()` the same way `Portals` is gated by `await Portals.ready()`. See [Guardian Avatars](https://portals.to/documentation/advanced-tooling/guardian-avatars).

## Starter

Wait for the host before reading the player or using a hosted capability:

```js
async function start() {
  const session = await Portals.ready();
  console.log("Running in", session.context);

  if (session.player.playerId) {
    console.log("Signed in as", session.player.displayName);
  }
}

start().catch(console.error);
```

`session.context` is `standalone` on a game page and `room` when the same game is open inside a Portals room.

## Sign in from the game

Call sign-in from a direct player action such as a button click. Portals owns the sign-in interface. Never ask for a Portals password or other account credential inside your game.

```js
const signInButton = document.querySelector("#sign-in");

signInButton.addEventListener("click", async () => {
  try {
    const player = await Portals.identity.requestLogin();
    signInButton.textContent = `Playing as ${player.displayName || "Player"}`;
  } catch (error) {
    console.error("Sign-in was not completed", error);
  }
});
```

Subscribe when the game UI should react to later sign-in or sign-out changes:

```js
const unsubscribe = Portals.identity.onChange((player) => {
  document.body.dataset.signedIn = player.playerId ? "true" : "false";
});

// Call unsubscribe() if your UI is removed.
```

`playerId` is stable for that player within one game. It is deliberately different across games. It is not a Portals account ID and must not be used to join player activity across games.

## Save progress

Saved state belongs to the signed-in player and the current game:

```js
const previous = await Portals.loadState();
const progress = previous || { level: 1, coins: 0 };

progress.coins += 1;
await Portals.saveState(progress);
```

State must be JSON-serializable and no larger than 64 KB after JSON encoding. Signed-out players cannot save. `loadState()` returns `null` when no saved state exists.

Keep saves compact and version them when their shape may change:

```js
await Portals.saveState({
  schemaVersion: 1,
  level: 3,
  unlocked: ["dash", "double-jump"],
});
```

## Submit and read casual scores

Scores require sign-in. Higher values rank first and only the best score for each player and mode is kept.

```js
await Portals.submitScore(1250);
await Portals.submitScore(48, "daily");
```

A mode may contain lowercase letters, numbers, and hyphens and may be at most 32 characters. Omit it to use `default`.

Read the top scores after the host has enforced access to the game:

```js
const leaderboard = await Portals.getLeaderboard({
  mode: "daily",
  limit: 10,
});

for (const entry of leaderboard.entries) {
  console.log(entry.rank, entry.displayName, entry.score);
}
```

The limit defaults to 10 and may be from 1 to 100. Each row contains:

| Field | Meaning |
|---|---|
| `rank` | One-based position in the returned board. |
| `playerId` | Stable identifier scoped to this game. |
| `displayName` | Current public display name, or `null`. |
| `avatarUrl` | Current public avatar URL, or `null`. |
| `score` | Best submitted score for the selected mode. |

Game scores are client-reported and intended for social and casual competition. Never use them to award currency, paid prizes, access, or another valuable entitlement.

## Close the game

```js
document.querySelector("#quit").addEventListener("click", () => {
  Portals.quit();
});
```

The current host decides how to close the game and restore player controls.

## API reference

| Method | Result |
|---|---|
| `Portals.ready()` | Resolves to the current player and host context. |
| `Portals.getPlayer()` | Resolves to the latest player. |
| `Portals.identity.requestLogin()` | Opens Portals sign-in when needed and resolves to the signed-in player. |
| `Portals.identity.onChange(listener)` | Subscribes to player changes and returns an unsubscribe function. |
| `Portals.saveState(data)` | Saves JSON state for the signed-in player. |
| `Portals.loadState()` | Loads JSON state or returns `null`. |
| `Portals.submitScore(score, mode?)` | Keeps the player's highest casual score for a mode. |
| `Portals.getLeaderboard(options?)` | Reads up to 100 top casual scores. |
| `Portals.quit()` | Requests that the host close the game. |

TypeScript declarations are available at [portals.d.ts](https://portals.to/portals-sdk/portals.d.ts). They declare the global `Portals` object and every public SDK type.

## Access and error handling

Free games may read leaderboards while signed out. Saving and score submission require sign-in. Paid games receive SDK capabilities only after Portals verifies purchase access.

Every asynchronous method can reject when the host is unavailable, the request is invalid, access is missing, or a network operation fails. Catch errors at the player action that caused them and keep the game playable when an optional Portals feature is unavailable.

Do not place Firebase tokens, API keys, payment details, or signed asset URLs in game code, saved state, score modes, logs, or leaderboard UI.
