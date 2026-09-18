# Portals SDK

Source: [https://portals.to/documentation/web-games/portals-sdk](https://portals.to/documentation/web-games/portals-sdk) — verbatim copy of the official Portals documentation.

The Portals SDK connects a hosted web game to the Portals player and host. It supports identity, saved progress, casual scores, leaderboard reads, game-specific Coin products, and closing the game from either standalone or in-room play. For setup and the complete purchase contract, see [Sell game products for Coins](https://portals.to/documentation/web-games/in-game-coins). For real-time multiplayer, in-game text chat, and voice chat, see [Multiplayer, Chat, and Voice](https://portals.to/documentation/web-games/multiplayer-and-voice). To put the player's Guardian avatar in a Three.js game — wearables, animation and a character controller — see [Guardian Avatars](https://portals.to/documentation/web-games/guardian-avatars).

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

It gives you the global `PortalsGuardians` — avatars, wearables, animation and a character controller, gated by `await PortalsGuardians.ready()` the same way `Portals` is gated by `await Portals.ready()`. See [Guardian Avatars](https://portals.to/documentation/web-games/guardian-avatars).

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

`session.context` is `standalone` on a game page and `room` when the same game is open inside a Portals room. `session.matchmaking` is non-null when the player arrived through a Portals-managed lobby — see [managed lobbies](https://portals.to/documentation/web-games/multiplayer-and-voice#managed-lobbies-and-matchmaking).

## Leave room for Portals controls

Every launched game keeps a trusted Portals controls button above the game in the top-left corner.
It is host UI outside the game iframe, so game code cannot hide, restyle, intercept, or out-z-index
it.

The current closed trigger is **44 × 44 CSS pixels**. Its top and left positions are each the
larger of `12px` and the matching device safe-area inset. A party-count badge may extend 4px beyond
the button. Keep essential labels and interactive game UI outside that footprint; the canvas or
other non-interactive playfield can still remain full-bleed.

For a top HUD, reserve the left edge. For a left HUD, reserve the top edge:

```css
.game-hud--top {
  padding-left: calc(max(12px, env(safe-area-inset-left)) + 56px);
}

.game-hud--left {
  padding-top: calc(max(12px, env(safe-area-inset-top)) + 56px);
}
```

The additional 56px covers the 44px button, the possible badge extension, and a small visual gap.
When the player opens Portals controls, the larger host-owned panel may temporarily cover more of
the game and takes pointer and keyboard focus. Keep gameplay safe while it is open and restore the
game UI cleanly when focus returns.

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

## Read the current username and active avatar

Use `Portals.player.get()` when a game needs the player's current public profile snapshot rather than only the lightweight identity from `Portals.ready()`:

```js
const player = await Portals.player.get();

const label = player.username
  ? `@${player.username}`
  : player.displayName || "Guest";

console.log(label, player.avatar);
```

`username` is the active public profile handle without the `@`. `avatar` is the playable look saved on the Portals `/avatar` screen: Guardian body configuration plus selected wearables, or a selected full-avatar replacement. It is `null` for a guest. The older `avatarUrl` field is only the player's 2D profile image; do not load it as a 3D character.

The first signed-in read is cached for that hosted-game load. Call it again after `Portals.identity.requestLogin()`; if the player changes their saved look in another tab, reopen or reload the game. To render it in a Three.js game, pass the result to `avatars.createAvatarFromPlayer(player)` from the [Guardian avatar SDK](https://portals.to/documentation/web-games/guardian-avatars#load-the-current-portals-player).

### Open the trusted avatar picker

Let a signed-in player update their global Portals look without leaving the game:

```js
document
  .querySelector("#customize-avatar")
  .addEventListener("click", async () => {
    try {
      const refreshedPlayer = await Portals.avatar.openPicker();
      await replaceRenderedAvatar(refreshedPlayer);
    } catch (error) {
      console.error("Avatar customization did not finish", error);
    }
  });
```

Call `openPicker()` from a direct click or tap. Portals owns the interface and signs the player in when needed. Changes remain a local draft until the player chooses **Done**; **Cancel** discards the draft and rejects the promise. A completed picker resolves only after the global `/avatar` look is saved and returns a refreshed `PortalsPlayerProfile`, so the game can replace its rendered avatar without reloading.

The picker is unavailable in the editor preview because that host does not open account-wide interfaces. Catch the rejection and keep the preview playable; test this interaction in a published game host.

Games cannot read the player's inventory or directly equip global items. They receive only the sanitized active presentation in `player.avatar`; its `wearables` are render data, not ownership proof. The picker only presents eligible items in the player's existing Shop inventory, and the server validates ownership and compatibility on save. The SDK does not expose Marketplace listings, purchases, sales, owned-item records, or a method that accepts arbitrary wearable IDs.

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

Scores require sign-in. Higher values rank first and, by default, only the best score for each player and mode is kept. Pass `{ replace: true }` as the third argument to store a score even when it is worse than the player's stored one — for a game whose score can legitimately go down.

A mode can rank the other way round. In **My Games → your game → Settings → Leaderboard ranking**, set a mode to **Lowest score** for golf strokes, move counts, or mistakes: the lowest value ranks first and a new submission replaces the player's stored score only when it is lower. A mode can also rank as a time; see [Time leaderboards](#time-leaderboards) below.

Draft play — the editor preview and a shared `?draft=` link — reads and writes a separate draft leaderboard, so you can play a board through before publishing without preview scores ever reaching the published game's ranking.

```js
await Portals.submitScore(1250);
await Portals.submitScore(48, "daily");
await Portals.submitScore(12, "daily", { replace: true });
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

| Field         | Meaning                                     |
| ------------- | ------------------------------------------- |
| `rank`        | One-based position in the returned board.   |
| `playerId`    | Stable identifier scoped to this game.      |
| `displayName` | Current public display name, or `null`.     |
| `avatarUrl`   | Current public avatar URL, or `null`.       |
| `score`       | Stored score for the selected mode. Elapsed milliseconds on a time mode. |

Game scores are client-reported and intended for social and casual competition. Never use them to award currency, paid prizes, access, or another valuable entitlement.

### Time leaderboards

A score is only a number; nothing in `submitScore` says whether it is points or a time. The mode's **ranking**, a setting you own in the creator dashboard, tells Portals how to treat that number. Every mode ranks as **Highest score** until you change it to **Lowest score**, **Shortest time**, or **Longest time**.

**1. Submit a time from your game.** Measure elapsed **milliseconds** and post them to a mode of your choice. Use a dedicated mode for each time board so it never mixes with point scores.

```js
const startedAt = performance.now();

// ...the player finishes the run...

const elapsedMs = Math.round(performance.now() - startedAt);
await Portals.submitScore(elapsedMs, "speedrun");
```

**2. Set the mode's ranking.** Open **My Games → your game → Settings → Leaderboard ranking**. Every mode your game has submitted to is listed, including modes played only in the editor preview or a draft link. Set the mode to **Shortest time** or **Longest time**. The setting autosaves and applies to the live and draft boards immediately.

Do this before you publish. A mode is stored under highest-score rules until you change its ranking, so play the game once in the editor preview to create the mode, set the ranking, then publish.

**3. Portals keeps the right time.** On a shortest-time mode, a new submission replaces the player's stored time only when it is lower; on a longest-time mode, only when it is higher. `{ replace: true }` still stores the posted time regardless, exactly as it does for scores.

**4. The board sorts by the ranking.** `getLeaderboard` and the featured leaderboard on the game page rank a shortest-time mode lowest first and a longest-time mode highest first. Rank 1 is always the winner.

**5. The game page formats the value.** When the featured leaderboard is a time mode, Portals sends the board's ranking together with its rows, and the game page shows each stored value as a time: `83456` becomes `1:23.456`, and times of an hour or more read `h:mm:ss.mmm`. Point boards keep the plain number format.

`getLeaderboard` returns the raw millisecond values and no ranking, so an in-game leaderboard formats them itself:

```js
const board = await Portals.getLeaderboard({ mode: "speedrun", limit: 10 });

function formatTime(ms) {
  const total = Math.round(ms);
  const minutes = Math.floor(total / 60000);
  const seconds = Math.floor((total % 60000) / 1000);
  const millis = total % 1000;
  return `${minutes}:${String(seconds).padStart(2, "0")}.${String(millis).padStart(3, "0")}`;
}

for (const entry of board.entries) {
  console.log(entry.rank, entry.displayName, formatTime(entry.score));
}
```

Always submit milliseconds. A value in seconds still ranks in the right order but displays as a fraction of a second on the game page.

## Close the game

```js
document.querySelector("#quit").addEventListener("click", () => {
  Portals.quit();
});
```

The current host decides how to close the game and restore player controls.

## API reference

| Method                                                | Result                                                                  |
| ----------------------------------------------------- | ----------------------------------------------------------------------- |
| `Portals.ready()`                                     | Resolves to the current player and host context.                        |
| `Portals.getPlayer()`                                 | Resolves to the latest player.                                          |
| `Portals.player.get()`                                | Reads the current public username and playable `/avatar` look.          |
| `Portals.avatar.openPicker()`                         | Opens trusted global avatar UI and resolves to the refreshed profile.   |
| `Portals.identity.requestLogin()`                     | Opens Portals sign-in when needed and resolves to the signed-in player. |
| `Portals.identity.onChange(listener)`                 | Subscribes to player changes and returns an unsubscribe function.       |
| `Portals.matchmaking.current()`                       | Resolves to the managed-match context, or `null` in a casual session.   |
| `Portals.matchmaking.onChange(listener)`              | Subscribes to managed-match phase changes; returns an unsubscribe.      |
| `Portals.saveState(data)`                             | Saves JSON state for the signed-in player.                              |
| `Portals.loadState()`                                 | Loads JSON state or returns `null`.                                     |
| `Portals.submitScore(score, mode?, options?)`         | Records a casual score; keeps the best for the mode's ranking unless `{ replace: true }`. |
| `Portals.getLeaderboard(options?)`                    | Reads up to 100 top casual scores.                                      |
| `Portals.economy.getCatalog()`                        | Reads products frozen into the current release.                         |
| `Portals.economy.getInventory()`                      | Reads this player's game-specific entitlements.                         |
| `Portals.economy.purchase(sku)`                       | Opens Portals-owned Coin confirmation from a player action.             |
| `Portals.economy.consume(sku, quantity, operationId)` | Idempotently uses a consumable grant.                                   |
| `Portals.quit()`                                      | Requests that the host close the game.                                  |

TypeScript declarations are available at [portals.d.ts](https://portals.to/portals-sdk/portals.d.ts). They declare the global `Portals` object and every public SDK type.

## Access and error handling

Free games may read leaderboards while signed out. Saving and score submission require sign-in. Paid games receive SDK capabilities only after Portals verifies purchase access.

Every asynchronous method can reject when the host is unavailable, the request is invalid, access is missing, or a network operation fails. Catch errors at the player action that caused them and keep the game playable when an optional Portals feature is unavailable.

Do not place Firebase tokens, API keys, payment details, or signed asset URLs in game code, saved state, score modes, logs, or leaderboard UI.
