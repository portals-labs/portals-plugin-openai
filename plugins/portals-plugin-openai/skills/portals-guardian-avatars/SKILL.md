---
name: portals-guardian-avatars
description: Put the player's Guardian avatar into a Three.js game on Portals — body types, skin/hair/eye colour, hair styles, wearables, retargeted animation clips, facial animation, and a first/third-person character controller. Use when working with PortalsGuardians, _portals/guardians-sdk.js, PortalsAvatars, createAvatar, avatar.wearables, avatar.animations, avatar.face, createController, SceneCollider, resolveAssetUrl, ledge grabbing, or pinning guardiansSdk in portals.json.
---

# Guardian avatars

The Guardian avatar SDK renders the same avatars players wear across Portals. It is a **Three.js library, not a renderer**: the game owns the scene, camera, and render loop, and the SDK adds avatars to it. Reach for it instead of hand-rolling a character whenever a 3D game wants a player avatar.

Read [references/guardian-avatars.md](references/guardian-avatars.md) for the full API, code examples, and local-development setup. It is the official documentation, copied verbatim. Types for autocomplete: `https://portals.to/portals-sdk/guardians.d.ts`.

## Wiring

Portals stamps the SDK into every processed preview and published bundle. Load it beside the Portals SDK; never edit, bundle, or ship a copy:

```html
<script src="./_portals/sdk.js"></script>
<script src="./_portals/guardians-sdk.js"></script>
<script src="./game.js"></script>
```

```js
const { THREE, PortalsAvatars } = await PortalsGuardians.ready();
```

Module-based games can skip the loader and `import { PortalsAvatars } from '@portals/avatars'` instead — Portals writes the import map that resolves `@portals/avatars`, `three`, and `three/addons/`. Both routes load the same module and can be mixed.

## Rules that are easy to get wrong

- **Call `avatars.update(dt)` once per frame** or nothing animates.
- **Take `THREE` from the SDK**, not from a CDN. There is exactly one shared instance; the game CSP keeps `script-src` at `'self'`, so a second copy will not load and would break instance checks anyway.
- **Assets resolve through Portals.** Published games run under `connect-src 'self'`; the SDK rewrites asset URLs onto managed same-origin paths automatically. Two exceptions need action: run a wearable's `thumbnail` through `resolveAssetUrl` before using it in an `<img>` (`img-src` is `'self'` too), and equip an externally hosted wearable as a catalog item **with its `id`** rather than a bare URL, since the item proxy addresses it by inventory item id.
- **The version is frozen.** A game keeps the avatar SDK version it was first stamped with; republishing and platform releases never change it. To move, pin `{ "guardiansSdk": "0.20.0" }` in `portals.json` at the project root. Pinning an unreleased version fails the publish.
- **Ground and collision are yours to supply.** The defaults are a flat floor at `y = 0` and free movement. `SceneCollider` is the quickest real level — register level geometry only, never avatars, and call `collider.refresh()` after adding or removing meshes. With existing physics, pass `getGroundHeight`, `resolveMovement`, `raycast`, and `checkClearance` directly.
- **Ledge grabbing needs `raycast`.** Without it nothing is climbable. Both shoulders must find the ledge, so a ledge narrower than the character cannot be grabbed, and the probe only runs while airborne — the player always jumps or falls into a climb.
- **Clips load on demand.** `createAvatar` resolves as soon as the body is on screen; the locomotion GLB downloads on first movement. Call `avatars.preloadAnimations()` to pay that cost behind a loading screen.
- **Clean up with `avatars.dispose()`** when leaving a scene, or `avatars.removeAvatar(avatar)` for one.

## What the API covers

- Appearance: `setSkinColor`, `setHairColor`, `setEyeColor`, `getHairStyles`, `setHairStyle`, `setFacialHair`, plus `configure()` / `getConfig()` for a whole `GuardianConfig` — pair `getConfig()` with the Portals SDK save API to persist a look.
- Wearables: `registerCatalog`, `equip`, `unequip`, `getEquipped`. Equipping replaces the slot, hides covered body meshes, and rebinds skinned items. Right-hand items infer a carry pose and use action from the item name, or set `handItemType`; listen via `controller.onItemUse`.
- Animation: `avatar.animations.load([...])` retargets clips onto the Guardian rig, including foreign Mixamo-rigged clips. The controller drives locomotion itself — return `true` from `controller.onBeforeAnimationUpdate` to take over for a frame. Faces are a separate layer: `avatar.face.setEmotion`, `setTalking`, `setAutoBlink`.
- Controller: WASD, sprint, jump, crouch, first/third-person camera. `configureMovement` values are clamped to `MOVEMENT_LIMITS`.

## Related

- Identity, saves, and leaderboards: the `portals-sdk` skill.
- Multiplayer and voice: the `portals-multiplayer-and-voice` skill.
