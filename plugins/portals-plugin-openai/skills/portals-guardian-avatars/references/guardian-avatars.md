# Guardian Avatars

Source: [https://portals.to/documentation/advanced-tooling/guardian-avatars](https://portals.to/documentation/advanced-tooling/guardian-avatars) — verbatim copy of the official Portals documentation.

The Guardian avatar SDK renders the same avatars players wear across Portals — body types, skin/hair/eye colour, hair styles, the wearables system, retargeted locomotion clips, facial animation, and a first/third-person character controller with the Portals feel.

It is a Three.js library, not a standalone renderer: you own the scene, camera and render loop, and the SDK adds avatars to it. For sign-in, saved progress, leaderboards and host control, see [Portals SDK](https://portals.to/documentation/advanced-tooling/portals-sdk).

Portals stamps the SDK into every processed preview and published bundle. Include it exactly like the [Portals SDK](https://portals.to/documentation/advanced-tooling/portals-sdk):

```html
<script src="./_portals/sdk.js"></script>
<script src="./_portals/guardians-sdk.js"></script>
<script src="./game.js"></script>
```

That gives you the global `PortalsGuardians`. Wait for it the same way you wait for `Portals`:

```js
const { PortalsAvatars } = await PortalsGuardians.ready();
```

Do not edit `_portals/guardians-sdk.js`; Portals manages that copy. Your game keeps the SDK version it was first stamped with — see [Versions](https://portals.to/documentation/advanced-tooling/guardian-avatars#versions) for how to pin one or develop against it locally.

### Importing it instead

The SDK is an ES module underneath — that is how it shares your game's Three.js instance instead of loading a second copy. If your game is already module-based, skip the loader and import it by name:

```html
<script type="module" src="./game.js"></script>
```

```js
import { PortalsAvatars } from '@portals/avatars';
```

Portals writes the import map that resolves `@portals/avatars`, `three` and `three/addons/` when it processes the project; you only need to keep it if you are hand-authoring `index.html`. Both routes load the same module, so mixing them in one project is fine.

Types for editor autocomplete: [portals.to/portals-sdk/guardians.d.ts](https://portals.to/portals-sdk/guardians.d.ts).

## Versions

The avatar SDK is released in versions. Your game is **frozen on the version it was first stamped with**: publishing again, or a newer platform release, never changes the avatar SDK under a working game. To move to a different released version, pin it in a `portals.json` at your project root:

```json
{ "guardiansSdk": "0.20.0" }
```

The pin travels with the project — through the editor, the AI builder, zip imports and GitHub builds — so the version you developed against is the version that publishes. Pinning a version that was never released fails the publish with an error naming the current release.

### Local development

Every released version is downloadable, so you can develop outside Portals against the exact build your game will run:

```
https://portals.to/portals-sdk/guardians/<version>/guardians-sdk.js
https://portals.to/portals-sdk/guardians/<version>/guardians-sdk.module.js
```

Locally you own the import map (Portals rewrites it to same-origin managed paths when it processes the project). Map `three` to a CDN build matching the SDK's Three.js release, and `@portals/avatars` to the versioned module URL:

```html
<script type="importmap">
{
  "imports": {
    "three": "https://cdn.jsdelivr.net/npm/three@0.178.0/build/three.module.js",
    "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.178.0/examples/jsm/",
    "@portals/avatars": "https://portals.to/portals-sdk/guardians/0.20.0/guardians-sdk.module.js"
  }
}
</script>
```

Then record the same version in `portals.json`. When you push the game to Portals, the known-CDN URLs are rewritten to the managed same-origin runtime and the pin keeps the SDK at the release you tested.

## Starter

`game.js`, loaded with a plain `<script src="./game.js">`:

```js
const { THREE, PortalsAvatars } = await PortalsGuardians.ready();

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, innerWidth / innerHeight, 0.1, 200);
const renderer = new THREE.WebGLRenderer({ antialias: true });
document.body.appendChild(renderer.domElement);

const avatars = new PortalsAvatars({ scene });

const avatar = await avatars.createAvatar({
  bodyType: 'female',
  skinColor: '#EAAB7F',
  hairColor: '#131110',
});

const controller = avatars.createController(avatar, {
  camera,
  domElement: renderer.domElement,
});

const clock = new THREE.Clock();
renderer.setAnimationLoop(() => {
  avatars.update(clock.getDelta());
  renderer.render(scene, camera);
});
```

`avatars.update(dt)` drives every avatar and controller — call it once per frame or nothing animates.

Animation clips are loaded on demand: `createAvatar` resolves as soon as the body is on screen, and the locomotion GLB downloads the first time the character actually moves. Call `avatars.preloadAnimations()` if you would rather pay that cost behind a loading screen.

## Three.js is shared, not bundled

The SDK does not ship its own copy of Three.js — it uses the same managed runtime, so the page has exactly one instance and objects the SDK creates work with your own scene graph, raycasters and materials.

Take `THREE` from the SDK rather than loading it yourself. From the classic script it comes out of `ready()`, as above. From a module you can import it directly, since the import map points at the same file:

```js
import * as THREE from 'three';
```

Loading a second copy from a CDN will not work — the game CSP keeps `script-src` at `'self'` — and would break instance checks even if it did.

## Assets load through Portals

Guardian models, wearables and animation clips live on the Portals CDN. Published games run under `connect-src 'self'` and cannot fetch from another host, so the SDK rewrites asset URLs onto managed same-origin paths that Portals serves for you (`/_portals/cdn/…` and, for wearables stored elsewhere, `/_portals/item/…`). This is automatic — pass ordinary Portals URLs and they resolve.

Two consequences worth knowing:

- **Thumbnails need the same treatment.** `img-src` is `'self'` too, so run a wearable's `thumbnail` through `resolveAssetUrl` before putting it in an `<img>`:

  ```js
  import { resolveAssetUrl } from '@portals/avatars';
  img.src = resolveAssetUrl(def.thumbnail);
  ```
- **Arbitrary hosts need an item id.** A wearable whose GLB is hosted outside Portals can only be fetched through the item proxy, which addresses it by inventory item id. Equip it as a catalog item (with its `id`) rather than as a bare URL. A raw URL to an unmanaged host logs a warning and will be blocked by the browser.

Assets you ship inside your own bundle are unaffected — relative paths are already same-origin.

## Appearance

```js
avatar.setSkinColor('tan');           // preset id or any CSS colour
avatar.setHairColor('#584435');
avatar.setEyeColor('green');
avatar.getHairStyles();               // [{ id, name }]
avatar.setHairStyle('Spiky_Bun');
avatar.setFacialHair(null);           // remove
```

`avatar.configure({ … })` applies a whole `GuardianConfig` at once, and `avatar.getConfig()` reads it back — useful for persisting a look with the Portals SDK's save API.

## Wearables

```js
avatar.wearables.registerCatalog(myItems);
await avatar.wearables.equip('basic-tshirt');   // catalog id
await avatar.wearables.equip({ id: 'hat-1', name: 'Cap', slot: 'hat', url: '…' });
await avatar.wearables.unequip('hat');          // by slot or id
avatar.wearables.getEquipped();
```

Equipping replaces whatever occupies the item's slots, hides the body meshes the slot covers, and re-binds skinned wearables to the avatar skeleton. Multi-slot items (dresses, full-body suits) declare every slot they occupy via `slots`.

Right-hand items get a carry pose and a use action inferred from the item name — an axe swings, a pistol shoots — or set `handItemType` explicitly. Listen for the action with `controller.onItemUse`.

## Animation

The Guardian models ship without clips; clips load from separate GLBs and are retargeted onto the Guardian rig automatically, including foreign Mixamo-rigged clips.

```js
await avatar.animations.load([
  { name: 'idle', url: './anims/locomotion.glb', clip: 'idle', loop: true },
  { name: 'wave', url: './anims/wave.glb', loop: false },
]);
avatar.animations.play('wave', { fade: 0.2 });
```

The character controller drives locomotion states itself. Return `true` from `controller.onBeforeAnimationUpdate` to take over for a frame.

Faces are a separate layer:

```js
avatar.face.setEmotion('smile');
avatar.face.setTalking(true);
avatar.face.setAutoBlink(true);
```

## Controller

`createController` returns a character controller with WASD movement, sprint, jump, crouch and a first/third-person camera rig. Movement values are clamped to the ranges in `MOVEMENT_LIMITS`.

```js
controller.configureMovement({ walkSpeed: 2, runSpeed: 4, jumpHeight: 1.2 });
controller.toggleCameraMode();
```

Ground and collision are yours to supply — the defaults are a flat floor at `y = 0` and free movement. The quickest way to a level that behaves is `SceneCollider`, which reads collision off the meshes you register:

```js
const collider = new SceneCollider();
collider.add(ground).add(levelGeometry);   // level meshes only, never avatars

avatars.createController(avatar, {
  camera,
  domElement: renderer.domElement,
  ...collider.controllerOptions(),
});
```

It casts against real triangles, so ramps, cylinders and imported geometry work as they look. Call `collider.refresh()` after adding or removing meshes under a registered root, and keep the registered set to level geometry — every ray is tested against all of it.

If you already have physics, pass the four hooks yourself instead:

```js
avatars.createController(avatar, {
  camera,
  domElement: renderer.domElement,
  getGroundHeight: (x, z, y) => terrain.heightAt(x, z),
  resolveMovement: (current, proposed, radius) => world.slide(current, proposed, radius),
  raycast: (origin, direction, maxDistance) => world.raycast(origin, direction, maxDistance),
  checkClearance: (point, radius, height) => world.capsuleFree(point, radius, height),
});
```

`raycast` is what enables ledge grabbing: two rows of rays, one at each shoulder, sweep down across the character's reach, and a surface within 30° of horizontal is a ledge it pulls itself onto. The line between the two rows' hits is the edge it squares up to, so curved and chamfered edges work too.

Two rules keep it deliberate. Both shoulders have to find the ledge, so brushing past a wall never triggers a grab — the cost is that a ledge narrower than the character's shoulders cannot be grabbed at all. And the probe only runs while airborne: the player always jumps (or falls) into a climb, never walks into one. It goes live on the first airborne frame, so pressing jump right at a ledge catches it immediately rather than at the apex. Walls with no reachable top are never climbable, and without a `raycast` nothing is.

## Cleanup

`avatars.dispose()` tears down every avatar and controller and releases their GPU resources. Call it when leaving a scene; individual avatars can go with `avatars.removeAvatar(avatar)`.
