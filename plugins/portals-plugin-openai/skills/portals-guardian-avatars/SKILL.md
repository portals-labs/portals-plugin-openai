---
name: portals-guardian-avatars
description: Put the player's Guardian avatar into a Three.js game on Portals — body types, skin/hair/eye colour, hair styles, wearables, retargeted animation clips, facial animation, NPCs, and a first/third-person character controller with swimming, touch controls, strafing, weapon posing, ragdoll and camera limits. Use when working with PortalsGuardians, _portals/guardians-sdk.js, PortalsAvatars, createAvatar, createAvatarFromPlayer, avatar.wearables, avatar.animations, ANIMATION_SETS, RUN_STYLES, avatar.face, createController, createNPC, SceneCollider, resolveAssetUrl, ledge grabbing, setSwimming, setStrafe, setStrafeTuning, aimWeapon, loadGunPoses, muzzleWorld, shotOrigin, Ragdoll, cameraLimits, ability VFX / targeting indicators, or pinning guardiansSdk in portals.json.
---

# Guardian avatars

The Guardian avatar SDK renders the same avatars players wear across Portals. It is a **Three.js library, not a renderer**: the game owns the scene, camera, and render loop, and the SDK adds avatars to it. Reach for it instead of hand-rolling a character whenever a 3D game wants a player avatar.

Read [references/guardian-avatars.md](references/guardian-avatars.md) for the full API, code examples, versioning, and local-development setup. For targeting reticles, cast telegraphs and world markers built within the sandbox rules, read [references/ability-vfx.md](references/ability-vfx.md). Types for autocomplete: `https://portals.to/portals-sdk/guardians.d.ts` — the source of truth for the API.

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

Module-based games can skip the loader and `import { PortalsAvatars } from '@portals/avatars'` instead — Portals writes the import map that resolves `@portals/avatars`, `three`, and `three/addons/`. Map the bare specifier at `guardians-sdk.module.js`, never at the `guardians-sdk.js` loader. Both routes load the same module and can be mixed. Only three addons are hosted: `controls/OrbitControls.js`, `loaders/GLTFLoader.js`, `utils/BufferGeometryUtils.js`.

## Rules that are easy to get wrong

- **Call `avatars.update(dt)` once per frame** or nothing animates.
- **Anything that writes bones runs AFTER `avatars.update(dt)`.** That call is the mixer tick, and it rewrites every bone local from the clips. `Ragdoll.update()`, `aimWeapon()` and `applyPose()` stepped before it are silently overwritten — nothing errors; it just does not work.
- **Take `THREE` from the SDK**, not from a CDN. There is exactly one shared instance; the game CSP keeps `script-src` at `'self'`, so a second copy will not load and would break instance checks anyway.
- **Assets resolve through Portals.** Published games run under `connect-src 'self'`; the SDK rewrites asset URLs onto managed same-origin paths automatically. Two exceptions need action: run a wearable's `thumbnail` through `resolveAssetUrl` before using it in an `<img>` (`img-src` is `'self'` too), and equip an externally hosted wearable as a catalog item **with its `id`** rather than a bare URL, since the item proxy addresses it by inventory item id.
- **The version is frozen.** A game keeps the avatar SDK version it was first stamped with; republishing and platform releases never change it. To move, pin `{ "guardiansSdk": "0.36.0" }` in `portals.json` at the project root. Pinning an unreleased version fails the publish. Features below are marked with the release they shipped in — a game pinned lower does not have them.
- **Ground and collision are yours to supply.** The defaults are a flat floor at `y = 0` and free movement. `SceneCollider` is the quickest real level — register level geometry only, never avatars, and call `collider.refresh()` after adding or removing meshes. With existing physics, pass `getGroundHeight`, `resolveMovement`, `raycast`, and `checkClearance` directly.
- **Ledge grabbing needs `raycast`.** Without it nothing is climbable. Both shoulders must find the ledge, so a ledge narrower than the character cannot be grabbed, and the probe only runs while airborne — the player always jumps or falls into a climb. Since 0.36.0 a wall standing in front of the ledge rejects the grab, so a wall is never climbed through.
- **Swimming is the game's toggle.** The SDK has no idea where a level's water is: call `controller.setSwimming(true, { surfaceY })` at the water's edge and `setSwimming(false)` exactly at the boundary on the way out.
- **Clips load on demand.** `createAvatar` resolves as soon as the body is on screen; the locomotion GLB downloads on first movement. `animations.play()` returns `false` until the clip's bytes arrive — that is not an error; `await ensure()` for certainty. Call `avatars.preloadAnimations()` to pay the cost behind a loading screen.
- **Clean up with `avatars.dispose()`** when leaving a scene, or `avatars.removeAvatar(avatar)` / `avatars.removeNPC(npc)` for one.

## What the API covers

- Appearance: `setSkinColor`, `setHairColor`, `setEyeColor`, `getHairStyles`, `setHairStyle`, `setFacialHair`, plus `configure()` / `getConfig()` for a whole `GuardianConfig` — pair `getConfig()` with the Portals SDK save API to persist a look.
- Player identity: `Portals.player.get()` then `avatars.createAvatarFromPlayer(player)` renders the signed-in player's saved look (`null` for a guest — keep a fallback); `Portals.avatar.openPicker()` opens the trusted global avatar picker from a click.
- Wearables: `registerCatalog`, `equip`, `unequip`, `getEquipped`. Equipping replaces the slot, hides covered body meshes, and rebinds skinned items. Right-hand items infer a carry pose and use action from the item name, or set `handItemType`; listen via `controller.onItemUse`.
- Animation: lazily loaded clips — a fixed always-available locomotion set (8-way walk/run/crouch strafe clips included), opt-in `ANIMATION_SETS` groups (sit, swim, sword, pistol, parkour, zombie, …), two `RUN_STYLES`, and `avatar.animations.load([...])` retargeting custom or Mixamo-rigged GLBs. Faces are a separate layer: `avatar.face.setEmotion`, `setTalking`, `setAutoBlink`.
- Controller: WASD, sprint, jump, crouch, first/third-person camera; `configureMovement` values clamp to `MOVEMENT_LIMITS`, including `gravity` / `maxFallSpeed` / `stepDown` (0.25.0). Automatic touch controls on mobile (`touch` option) and a programmatic input API (`setMoveVector`, `setJumpHeld`, `beginPrimaryAction`, …). Swimming with prone collision (0.26.0/0.36.0). Camera limits (`cameraLimits`, 0.24.0), camera wall avoidance (0.23.0) with `allowedOccluders` (0.34.0). Avatar-vs-avatar collision (0.32.0, `collideWithAvatars` to opt out).
- Shooter kit (0.36.0–0.37.0): `strafe` modes plus `setStrafeTuning` (diagonal collapse, `diagonalTurn`, `sprintCap`, `hysteresis`); weapon posing via `loadGunPoses` / `applyPose` / `aimWeapon` with solved grips (`GUARDIAN_PISTOL_GRIP`, `GUARDIAN_TUBE_GRIP`), procedural recoil, and the shouldered scope (`scoped`, `SCOPE_DEFAULTS`); `muzzleWorld` / `shotOrigin` for tracers and hitscans; `Ragdoll` — a position-based skeleton solver fed `collider.controllerOptions()`, stepped after `avatars.update(dt)` over a playing death clip.
- NPCs: `createNPC` / `createNPCFromUrl` wrap the same Guardian in a code-driven `NPCController` — `moveTo`, `follow`, `lookAt`, `playAnimation`; locomotion is automatic, and the shared `avatars.update(dt)` drives them.

## Reproduce the player's own items

A game that renders the signed-in player's Guardian inherits whatever that account owns and wears, so an item that breaks an animation or hides the wrong mesh is only debuggable with the real GLB in hand. Two `portals-web-games` MCP tools read the authenticated account's wardrobe — there is no route to another player's:

- `list_avatar_wearables` names the owned items, marks the ones currently worn, and reports the saved body type, skin, and hair. It also reports a full-body custom avatar when one is worn: that replaces the Guardian body, so a game rendering it will not show the wearables at all.
- `save_avatar_wearable` writes one item's GLB into the game's directory, fetched through the same per-item proxy a hosted game loads it from and resolved for the avatar's body type. Equip the saved file with `registerCatalog` + `equip` to see what the player sees.

## Related

- Identity, saves, and leaderboards: the `portals-sdk` skill.
- Multiplayer and voice: the `portals-multiplayer-and-voice` skill.
