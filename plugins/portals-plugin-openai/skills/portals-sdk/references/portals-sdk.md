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

## Threaded WebAssembly and Unreal web exports

A browser export that uses WASM threads needs shared memory. Opt in by adding
`crossOriginIsolation` to the existing **root `portals.json`** alongside any SDK pins:

```json
{
  "crossOriginIsolation": true
}
```

Keep this file in the **built output beside `index.html`**, then push/import that output.
Preview the new draft before publishing. The setting belongs to the source snapshot:
changing it affects the draft; a published release changes only after a new publish.
Set it to `false` or remove the field to opt out. The value must be a JSON boolean.
No new MCP setting or SDK method is required.

Portals serves the opted-in entry with
`Document-Isolation-Policy: isolate-and-require-corp`. This enables isolation inside
the existing game iframe while preserving the Portals host and its credential-free
`postMessage` SDK bridge. Creators cannot set server headers using HTML meta tags.
Do not install a service-worker header workaround or disable the iframe sandbox.

**Supported scope: desktop Chromium browsers with Document Isolation Policy**
(Chrome 137+; verify the actual capabilities in other Chromium browsers).
This is not a promise of Firefox, Safari, iPhone, Android, or embedded-WebView support.
Portals adds an early check for `crossOriginIsolated` and `SharedArrayBuffer`;
when unavailable it navigates to a browser-support notice. No user-agent guess
can replace checking these values in the actual game frame:

```js
const supportsThreads = globalThis.crossOriginIsolated === true &&
  typeof SharedArrayBuffer !== "undefined";
```

A standalone localhost check is not enough: verify a shared `WebAssembly.Memory`,
a real worker, the engine startup, gameplay, and `Portals.ready()` in the hosted
preview. The editor's mobile layout preview does not establish mobile browser support.

For Unreal, upload an already-built **browser/WebAssembly export**, not a native
executable, source project, or `.uproject`. Record the UE version, export toolchain,
required browser/WebGPU features, and memory budget. Enabling threads does not add
WebGPU support to a device or certify an Unreal toolchain. The complete game still
needs its own qualification.

Keep workers, WASM, JS, and game data in the bundle with relative URLs. Cross-origin
resources must satisfy both Portals CSP and CORS/CORP; isolation does not expand the
network allowlist. Prefer uncompressed `.js`/`.wasm` files (the CDN handles transport
compression). Precompressed `.br`/`.gz` exports require the correct serving MIME and
Content-Encoding and are not made compatible by this isolation flag. Current plugin
upload limits still apply; use the website import path for bundles beyond plugin limits.

For local testing, configure your development server to send the same isolation
header. Deployment requires both the backend processing changes and the CDN
viewer-response function; a source upload alone cannot enable an unconfigured CDN.

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

Anyone can post a score. A signed-in player is ranked under their Portals profile; a signed-out player is ranked under a name your game collects from them — see [Scores from signed-out players](https://portals.to/documentation/web-games/portals-sdk#scores-from-signed-out-players) below. Higher values rank first and, by default, only the best score for each player and mode is kept. Pass `{ replace: true }` as the third argument to store a score even when it is worse than the player's stored one — for a game whose score can legitimately go down.

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
| `displayName` | Current public display name, or the name a signed-out player gave, or `null`. |
| `avatarUrl`   | Current public avatar URL, or `null`. Always `null` for a signed-out player. |
| `score`       | Stored score for the selected mode. Elapsed milliseconds on a time mode. |

Game scores are client-reported and intended for social and casual competition. Never use them to award currency, paid prizes, access, or another valuable entitlement.

### Scores from signed-out players

A signed-out player has no Portals profile, so there is no name for the board to rank them under. Your game collects one and passes it as `options.name`:

```js
const player = await Portals.getPlayer();

if (player.playerId === null) {
  // Your own UI — a text input, a three-letter arcade entry, whatever fits
  // the game. Portals does not prompt for this.
  await Portals.submitScore(1250, "daily", { name: enteredName });
} else {
  await Portals.submitScore(1250, "daily");
}
```

A name is 1 to 24 characters. The submission is **rejected** for a signed-out player without a usable one, so collect it before you post the score rather than discovering the rejection afterwards. For a signed-in player the option is ignored — their Portals profile name always wins, and a game cannot rename them.

The name is stored on that score alone and is the only thing the board shows for that row: `avatarUrl` is `null` and the row does not link to a Portals profile. The player keeps the same board identity across sessions in the same browser, so beating their own score updates their row instead of adding a second one. Clearing browser storage starts a new one.

A signed-out player's name reaches a public leaderboard exactly as typed. If your game's audience makes that a concern, constrain the input — a fixed character set, a short length, or a pick-from-list — rather than accepting free text.

Paid games are unaffected: a signed-out player cannot purchase access, so they never reach the point of posting a score.

### Private leaderboards

A private leaderboard is a separate ranking for one group of players — a tournament, a classroom, a stream night — while your public board carries on untouched.

**1. Create it.** In **My Games → your game → Settings → Private leaderboards**, create one and give it a name, for example `friday-cup`. A name may use lowercase letters, numbers, and hyphens, must start with a letter or number, and may be at most 64 characters.

**2. Share its link.** The settings page gives you the link to copy:

```
https://portals.to/g/your-game?board=friday-cup
```

Everyone who follows that link competes on that board. You do not have to change your game for this — every `submitScore` and `getLeaderboard` call made in that session already goes there.

To label it, read the board from the session:

```js
const session = await Portals.ready();

if (session.board !== null) {
  showBanner(`playing the ${session.board} board`);
}
```

Your game can also name a board itself, on either call:

```js
await Portals.submitScore(1250, "daily", { board: "friday-cup" });
const board = await Portals.getLeaderboard({ mode: "daily", board: "friday-cup" });
```

A private leaderboard is **unguessable, not secret**. The name is the only thing that reaches it, so treat the link as the invitation: anyone you give it to can read and post there, and a name you publish — in your game's own code, say — is public. Pick something hard to guess for a board that should stay closed.

Some things to know:

- **Only a leaderboard you created exists.** A link naming anything else — a typo, or one you have deleted — plays the **public** board instead. Players are never blocked, but their scores go to the public ranking, so check the name when you share a link and expect old links to feed the public board once you delete a leaderboard.
- A private leaderboard is never listed on your game's page and can never be the featured board. Its scores are not counted in your leaderboard settings, and `session.board` is the only place a player can see which board they are on.
- Mode rankings are shared. A mode set to **Shortest time** ranks that way on every board.
- A session on a private leaderboard stays there. There is no way to read the public board from inside it.
- Draft play keeps the split: `?draft=` and `?board=` together give you the draft side of that private leaderboard.
- Deleting one deletes its scores with it.
- A game may hold up to 50 private leaderboards.
- A Portals admin resetting your whole leaderboard empties private leaderboards too, without archiving them as a version. The leaderboards themselves survive, so their links keep working.

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
| `Portals.ready()`                                     | Resolves to the current player, host context, and private board.        |
| `Portals.getPlayer()`                                 | Resolves to the latest player.                                          |
| `Portals.player.get()`                                | Reads the current public username and playable `/avatar` look.          |
| `Portals.avatar.openPicker()`                         | Opens trusted global avatar UI and resolves to the refreshed profile.   |
| `Portals.identity.requestLogin()`                     | Opens Portals sign-in when needed and resolves to the signed-in player. |
| `Portals.identity.onChange(listener)`                 | Subscribes to player changes and returns an unsubscribe function.       |
| `Portals.matchmaking.current()`                       | Resolves to the managed-match context, or `null` in a casual session.   |
| `Portals.matchmaking.onChange(listener)`              | Subscribes to managed-match phase changes; returns an unsubscribe.      |
| `Portals.saveState(data)`                             | Saves JSON state for the signed-in player.                              |
| `Portals.loadState()`                                 | Loads JSON state or returns `null`.                                     |
| `Portals.submitScore(score, mode?, options?)`         | Records a casual score; keeps the best for the mode's ranking unless `{ replace: true }`. A signed-out player needs `{ name }`. |
| `Portals.getLeaderboard(options?)`                    | Reads up to 100 top casual scores. `{ board }` reads a private board.   |
| `Portals.economy.getCatalog()`                        | Reads products frozen into the current release.                         |
| `Portals.economy.getInventory()`                      | Reads this player's game-specific entitlements.                         |
| `Portals.economy.purchase(sku)`                       | Opens Portals-owned Coin confirmation from a player action.             |
| `Portals.economy.consume(sku, quantity, operationId)` | Idempotently uses a consumable grant.                                   |
| `Portals.quit()`                                      | Requests that the host close the game.                                  |

TypeScript declarations are available at [portals.d.ts](https://portals.to/portals-sdk/portals.d.ts). They declare the global `Portals` object and every public SDK type.

## Access and error handling

Free games may read leaderboards and post scores while signed out; a signed-out score needs `{ name }`. Saving requires sign-in. Paid games receive SDK capabilities only after Portals verifies purchase access.

Every asynchronous method can reject when the host is unavailable, the request is invalid, access is missing, or a network operation fails. Catch errors at the player action that caused them and keep the game playable when an optional Portals feature is unavailable.

Do not place Firebase tokens, API keys, payment details, or signed asset URLs in game code, saved state, score modes, logs, or leaderboard UI.
