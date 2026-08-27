# In-game purchases on Portals

`Portals.economy` lets a hosted web game sell products to its own players for **Coins** — the currency players already hold on Portals. Portals owns the wallet, the price display, and the confirmation UI. The game names a product and reacts to the outcome; it never handles money, never renders a payment form, and never learns the player's balance.

The API ships in the managed SDK at `_portals/sdk.js`, alongside identity, saves, and leaderboards. TypeScript declarations for every type below are in [portals.d.ts](https://portals.to/portals-sdk/portals.d.ts).

## Two halves

| Half | Where it lives | Who changes it |
|---|---|---|
| The **catalog** — which SKUs exist, what they cost | Portals, per game, pinned to a release | The owner, via the MCP tools or portals.to/my-games |
| The **game code** — when to offer and what to grant | The game's own source | Whoever writes the game |

The catalog is authoritative. `purchase(sku)` can only sell a SKU the released catalog contains, so a product has to be drafted before any code can reference it.

## The catalog

### Product fields

| Field | Rule |
|---|---|
| `sku` | 3–64 characters, `^[a-z][a-z0-9_-]{2,63}$`. The string game code passes to `purchase()`. Permanent. |
| `title` | Player-facing name in the confirmation UI. Up to 80 characters. |
| `description` | Player-facing detail. Up to 500 characters. |
| `kind` | `durable` or `consumable`. |
| `coinPrice` | 10–5,400 Coins, integer. |
| `grantQuantity` | Units granted per purchase. Always 1 for a durable; 1–10,000 for a consumable. |
| `purchaseLimitPerPlayer` | 1–10,000, or null for no cap. |
| `iconPath` | PNG/JPEG/WebP path **inside the game's own pushed source**, e.g. `assets/lives.png`. Never absolute, never `..`, never a CDN URL. |

A game may hold **50 active** and **100 lifetime** SKUs.

### durable vs consumable

- **`durable`** is an entitlement the player owns once and keeps. Buying twice is refused with `ALREADY_OWNED`. Read it from `getInventory()` at startup — remove-ads, an unlocked character, a level pack.
- **`consumable`** is a stackable quantity. Each purchase adds `grantQuantity` to the player's balance for that SKU, and the game draws it down with `consume()`. Extra lives, hint tokens, a soft-currency pack.

The kind is fixed once the SKU is drafted, because it decides what ownership means for everyone who already bought it.

### Lifecycle

```
draft  ──review──▶  approved (sellable in draft launches)  ──publish game──▶  live for players
```

Edits land in the **draft** catalog. A draft carries a `draftRevision`, which is an optimistic-concurrency token: write with the revision you last read, and a change made in between fails with `DRAFT_CONFLICT` instead of silently overwriting.

Approval releases the exact reviewed draft as an immutable catalog of its own: real-Coin purchases then work in **draft launches** — the owner's editor preview and the staging share link — before the game is ever published. Publishing captures the approved catalog into the immutable live release. Any later draft edit or a rejection closes draft selling until the next approval, and never changes what a live game charges today.

**Retirement is permanent.** A retired SKU is withdrawn from sale and its id is burned for the game's lifetime; it cannot be re-created under the same name. Players who already bought it keep their entitlement, and it keeps appearing in their inventory, so code that reads it must go on working.

### The MCP tools

- **`get_game_economy_catalog`** — every drafted product, the review status, the live catalog revision, and the `draft_revision` to write against. `configured: false` means the game has no catalog yet.
- **`update_game_economy_catalog`** — `action: "upsert"` creates a SKU or **replaces every field** of an existing one (read it first and resend what should stay), `action: "retire"` withdraws it permanently.
- **`test_game_economy_purchase`** — drives the owner's simulated wallet: `view`, `purchase`, `consume`, `refund`, `reset`. See below.

## Game API

```js
await Portals.ready();
```

Everything below requires the host. Sign-in is a precondition for purchases and inventory.

### getCatalog()

```js
const products = await Portals.economy.getCatalog();
// [{ sku, title, description, kind, coinPrice, grantQuantity,
//    purchaseLimitPerPlayer, iconPath }]
```

The exact immutable catalog bundled with the running release. Build the shop UI from this rather than from hardcoded product data — the price shown then always matches the price charged. Resolves to `[]` with no host.

### getInventory()

```js
const owned = await Portals.economy.getInventory();
// [{ sku, kind, quantity, status: "active" | "revoked", purchaseCount }]
```

What the signed-in player owns for this game. A durable appears with `quantity: 1`; a consumable's `quantity` is what is left to spend. `status: "revoked"` means the entitlement was reversed (a refund or a moderation action) and must **not** be honoured. Resolves to `[]` with no host.

### purchase(sku)

```js
const result = await Portals.economy.purchase("extra_lives_5");
// { status: "purchased" | "cancelled", sku, receiptId, quantity }
```

Opens Portals-owned confirmation UI showing the title and Coin price, and resolves when the player decides. **Must be called directly from a click or tap handler** — the host checks that the gesture is still live and rejects with `PLAYER_ACTION_REQUIRED` otherwise. Anything awaited before the call spends the gesture, so purchase first and await afterwards.

`status: "cancelled"` is a normal outcome with `receiptId: null` — the player said no. It is not an error and must not be reported as one.

The call waits on a human, so it has a **10-minute** timeout rather than a network one.

### consume(sku, quantity, operationId)

```js
const result = await Portals.economy.consume("extra_lives_5", 1, `revive-${runId}-${deathCount}`);
// { sku, quantity, idempotent }
```

Spends a consumable and resolves with the **remaining** aggregate quantity. `quantity` is a positive integer; `operationId` is 8–128 URL-safe characters (`^[A-Za-z0-9_-]{8,128}$`).

`operationId` is the idempotency key, and it is the argument most worth getting right. Replaying the same id returns the same result with `idempotent: true` and spends nothing further — which is exactly what a retry after a dropped response should do. Derive it from the gameplay event that caused the spend, so a retry naturally reuses it and a genuinely new spend naturally gets a new one:

```js
// yes — one id per real event, stable across retries
`revive-${runId}-${deathCount}`

// no — a new id every attempt, so one bad connection charges the player twice
`revive-${Math.random()}`
```

## Worked example

```js
const catalog = await Portals.economy.getCatalog();
if (catalog.length === 0) return;            // no host, or nothing on sale

let inventory = await Portals.economy.getInventory();
const owns = (sku) =>
  inventory.some((i) => i.sku === sku && i.status === "active" && i.quantity > 0);

if (owns("remove_ads")) disableAds();

for (const product of catalog) {
  const button = renderShopRow(product);     // title, description, coinPrice
  button.addEventListener("click", async () => {
    button.disabled = true;
    try {
      const result = await Portals.economy.purchase(product.sku);
      if (result.status === "purchased") {
        inventory = await Portals.economy.getInventory();
        applyEntitlements(inventory);
      }
    } catch (error) {
      showMessage(
        error.code === "INSUFFICIENT_COINS"
          ? "Not enough Coins for that."
          : "That purchase could not be completed."
      );
    } finally {
      button.disabled = false;
    }
  });
}
```

Note the shape: `purchase()` is the first thing in the handler, the outcome decides whether anything is granted, and the grant comes from re-reading inventory rather than from a local guess.

## Errors

Rejections carry a `code`. The ones worth branching on:

| Code | Meaning |
|---|---|
| `PLAYER_ACTION_REQUIRED` | `purchase()` was not called from a live click or tap. |
| `AUTHENTICATION_REQUIRED` | The player is signed out. Prompt with `Portals.identity.requestLogin()` from a gesture. |
| `INSUFFICIENT_COINS` | The player cannot afford it. Worth its own message; everything else is not the player's fault. |
| `ALREADY_OWNED` | A durable the player already has, or a SKU whose `purchaseLimitPerPlayer` is reached. |
| `INSUFFICIENT_ENTITLEMENT` | `consume()` asked for more than the player holds. |
| `ENTITLEMENT_KIND_MISMATCH` | `consume()` was called on a durable. Durables are owned, not spent. |
| `DAILY_GAME_LIMIT` / `DAILY_GLOBAL_LIMIT` | Player spend caps — 13,500 Coins in one game, 27,000 across Portals, per day. Reachable by a player who *does* have the Coins. |
| `PRODUCT_NOT_FOUND` / `CATALOG_NOT_FOUND` | The SKU is not in the released catalog — or, in a draft launch, the current draft has no operator approval yet. A code/catalog mismatch, not a player problem. |
| `ACCESS_REQUIRED` | The player lacks access to the game, or the launch (or the approved draft behind a draft launch) is no longer current. |
| `ECONOMY_UNAVAILABLE` / `NOT_READY` | Purchases are not enabled for this game or host. See *When purchases are blocked*. |
| `CONTROL_BLOCKED` | Portals has frozen purchasing for this game or account. |
| `RATE_LIMITED` | Carries `retryAfter` in seconds. |

Rate limits are per player: **30 purchases per 10 minutes** and **240 consumes per minute**. A game that calls `consume()` per frame rather than per event will hit the second one.

Treat every other code as a generic failure, keep the game playable, and never grant the item anyway.

## No host

On a local dev server, or anywhere outside a Portals game host, `getCatalog()` and `getInventory()` resolve to `[]` and `purchase()` / `consume()` reject. Gate shop UI on a non-empty catalog so the game stays playable while it is being developed — the same discipline the rest of the SDK needs.

## Testing without real Coins

`test_game_economy_purchase` runs against a simulated wallet that belongs to the game's **owner**: a 10,000-Coin starting balance and its own inventory, touching no real Coins and no player's entitlements.

Its value is the failure paths. `scenario` forces the outcome:

| Scenario | What it reproduces |
|---|---|
| `success` | The player confirms. |
| `cancelled` | The player dismisses the confirmation — the case most often mishandled. |
| `insufficient` | The player cannot afford it. |
| `retryable` | A transient host failure. |

`refund` reverses a receipt so revocation handling can be exercised, and `reset` restores the starting balance and clears everything the wallet holds.

Test at minimum: a successful purchase grants exactly once; a cancelled purchase grants nothing and shows no error; a repeated `consume()` with the same `operationId` spends once.

## Testing with real Coins

Once the draft catalog is approved, the editor preview and the staging share link run the real economy against the approved draft — the full confirmation UI, real Coin debits, real inventory. An owner buying from their own game is a supported **test purchase**: the owner pays real Coins, Portals retains the full price, and the creator earns nothing from it. Purchases made during draft launches are ordinary entitlements — players keep them when the game is published.

## When purchases are blocked

A real purchase needs more than correct code. All of the following must hold, and none of it is visible to the MCP:

- The catalog is operator-approved for the launch being played: the reviewed catalog captured on the current release for a live play, or the approved current draft for an editor-preview or staging-link play. A draft edit after approval closes draft purchases until the next approval.
- The owner's email is verified and their Stripe identity verification is complete.
- The owner's account is in good standing, with no purchase freeze on the account or the game.
- The owner's account is included in the current rollout — in-game purchases are still being opened to creators in stages.

The owner sees all of it, with the specific blocker and next action, at **portals.to/my-games → Economy**. `ECONOMY_UNAVAILABLE` or `NOT_READY` from an otherwise correct game means one of these is unmet; report it rather than rewriting working purchase code.

## Trust

The economy is server-authoritative and must stay that way:

- Grant only what `purchase()` and `consume()` return. A client-reported score, a peer's multiplayer message, and a local flag are all forgeable — never let one award Coins, items, or access. See the `portals-sdk` and `portals-multiplayer-and-voice` skills.
- Treat `getInventory()` as the source of truth for what a player owns, re-read after every purchase, and honour `status: "revoked"` by taking the entitlement away.
- Never place tokens, keys, receipts, or signed URLs in game code, saved state, or logs.
