---
name: portals-game-economy
description: Sell in-game products for Coins inside a hosted Portals web game with Portals.economy — microtransactions, IAP, durable and consumable SKUs, purchase confirmation, player inventory, and spending consumables. Use when working with Portals.economy.getCatalog, getInventory, purchase, consume, operationId idempotency, the product catalog behind get_game_economy_catalog and update_game_economy_catalog, the purchase sandbox, or why a live purchase is blocked.
---

# In-game purchases

`Portals.economy` is how a hosted game sells to its own players. The currency is Coins, which players already hold on Portals; the game never sees a card, a dollar price, or a wallet balance, and it never renders a payment form. Portals owns the confirmation UI and the money.

Read [references/portals-game-economy.md](references/portals-game-economy.md) for the full API, the catalog fields, every error code, and worked examples.

## The catalog is not in the game's code

A game can only sell a **SKU that exists in its catalog**, authored per game and pinned to a release. `purchase("anything_else")` cannot invent a product. So the order of work is catalog first, code second:

1. `get_game_economy_catalog` — read what the game already sells, and its `draft_revision`.
2. `update_game_economy_catalog` — draft a SKU (`upsert`), or withdraw one (`retire`).
3. Write game code against those exact SKU strings.
4. `test_game_economy_purchase` — run it against the owner's simulated wallet.

Drafted products reach players only after Portals reviews the catalog **and** the owner publishes the game. Editing the draft never changes what a live game charges today.

A product is one of two kinds, and the choice is permanent for that SKU:

- **`durable`** — bought once, owned forever. `grantQuantity` is always 1. Remove-ads, a character, a level pack.
- **`consumable`** — grants a stackable `grantQuantity` the game spends with `consume()`. Lives, hints, currency packs.

## Wiring

`Portals.economy` ships in the managed SDK; there is nothing extra to include.

```html
<script src="./_portals/sdk.js"></script>
```

```js
await Portals.ready();
const products = await Portals.economy.getCatalog();
const owned = await Portals.economy.getInventory();
```

Buying must start from a real player gesture:

```js
buyButton.addEventListener("click", async () => {
  const result = await Portals.economy.purchase("extra_lives_5");
  if (result.status === "cancelled") return;   // normal, not an error
  lives += result.quantity;
});
```

Spending a consumable is idempotent on an `operationId` the game chooses. Reuse the **same** id when retrying the **same** gameplay event, and a fresh one for a new event:

```js
const result = await Portals.economy.consume("extra_lives_5", 1, `revive-${runId}-${deathCount}`);
lives = result.quantity;   // remaining, after the operation
```

## Rules that are easy to get wrong

- **A cancelled purchase is a normal outcome, not a rejection.** `purchase()` resolves with `status: "cancelled"` when the player dismisses the Portals confirmation. Code that only handles the promise rejecting will silently grant nothing and say nothing.
- **`purchase()` must be called directly from a click or tap handler.** Calling it after an `await`, from a timer, or on load rejects with `PLAYER_ACTION_REQUIRED` — the player gesture has expired by then. Do the awaiting *after* the purchase, not before it.
- **Never grant an entitlement the server did not.** Award only what `purchase()` and `consume()` return. Client-reported scores and peer messages must never move Coins, inventory, or access — see the `portals-sdk` and `portals-multiplayer-and-voice` skills.
- **`consume()` needs a stable `operationId`,** 8–128 URL-safe characters. A random id per retry double-spends the player's items on a flaky network; that is the whole reason the argument exists. Derive it from the gameplay event (`run-42-revive-1`), never from `Math.random()` at the call site.
- **Re-read inventory rather than tracking it locally.** The player may own things from an earlier session, another device, or a purchase made mid-session. `getInventory()` is the truth; a local counter is a guess.
- **Outside a Portals host there is no economy.** On a local dev server `getCatalog()` and `getInventory()` resolve to `[]`, and `purchase()`/`consume()` reject. Keep the game playable — gate the shop UI on a non-empty catalog instead of assuming it is there.
- **Retiring a SKU is permanent.** The id is burned for the game's lifetime and cannot be re-created, because players who bought it keep an entitlement that must keep resolving. Code that reads a retired SKU has to keep working.
- **A price is 10–5,400 Coins,** and a game may hold 50 active and 100 lifetime SKUs. Players also have daily spend caps across a game and across Portals, so a purchase can fail with `DAILY_GAME_LIMIT` even though the player has the Coins.
- **Icons are game files.** `iconPath` points inside the game's own pushed source (`assets/lives.png`); a CDN URL fails the published game's CSP exactly like every other external asset.

## When purchases are blocked

`ECONOMY_UNAVAILABLE` or `NOT_READY` from a live game is usually not a code bug. Live purchases additionally require the owner's monetization readiness — verified email, Stripe identity verification, account in good standing, a published game with a reviewed live catalog — and access is still rolling out to creators in stages. None of that is visible to the MCP by design; the owner reads it at **portals.to/my-games → Economy**. Say so plainly rather than rewriting working purchase code.

## Related

- Identity, saves, and leaderboards: the `portals-sdk` skill. Sign-in is a precondition for any purchase.
- Pushing, testing, and publishing the game itself: the `portals-web-games` skill.
- Keeping a multiplayer session from awarding entitlements: the `portals-multiplayer-and-voice` and `portals-server-scripts` skills.
