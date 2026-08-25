# Guardian Avatars

Source: [https://portals.to/documentation/web-games/guardian-avatars](https://portals.to/documentation/web-games/guardian-avatars) — the official Portals documentation, extended with the SDK reference for every released capability through 0.37.0. Version markers like (0.36.0) name the release a feature first shipped in; a game pinned below that version does not have it. Types: [portals.to/portals-sdk/guardians.d.ts](https://portals.to/portals-sdk/guardians.d.ts) — the source of truth for the API. Read it before writing code against a method not shown here.

The Guardian avatar SDK renders the same avatars players wear across Portals — body types, skin/hair/eye colour, hair styles, the wearables system, retargeted locomotion clips, facial animation, and a first/third-person character controller with the Portals feel.

It is a Three.js library, not a standalone renderer: you own the scene, camera and render loop, and the SDK adds avatars to it. For sign-in, saved progress, leaderboards and host control, see [Portals SDK](https://portals.to/documentation/web-games/portals-sdk).

Portals stamps the SDK into every processed preview and published bundle. Include it exactly like the [Portals SDK](https://portals.to/documentation/web-games/portals-sdk):

```html
<script src="./_portals/sdk.js"></script>
<script src="./_portals/guardians-sdk.js"></script>
<script src="./game.js"></script>
```

That gives you the global `PortalsGuardians`. Wait for it the same way you wait for `Portals`:

```js
const { PortalsAvatars } = await PortalsGuardians.ready();
```

`ready()` is required — the global exists immediately but its exports are only populated once the module lands. Do not edit `_portals/guardians-sdk.js`; Portals manages that copy. Your game keeps the SDK version it was first stamped with — see [Versions](#versions) for how to pin one or develop against it locally.

### Importing it instead

The SDK is an ES module underneath — that is how it shares your game's Three.js instance instead of loading a second copy. If your game is already module-based, skip the loader and import it by name:

```html
<script type="module" src="./game.js"></script>
```

```js
import * as THREE from 'three';
import { PortalsAvatars } from '@portals/avatars';
```

Portals writes the import map that resolves `@portals/avatars`, `three` and `three/addons/` when it processes the project; keep it if you hand-author `index.html`:

```json
{
  "imports": {
    "three": "./_portals/vendor/three/three.module.js",
    "three/addons/": "./_portals/vendor/three/addons/",
    "@portals/avatars": "./_portals/guardians-sdk.module.js"
  }
}
```

Note the two files: `guardians-sdk.js` is the classic-script loader, `guardians-sdk.module.js` is the SDK. Mapping the bare specifier at the loader hands the game the wrong file. Both routes load the same module, so mixing them in one project is fine.

Only these Three.js addons are hosted: `controls/OrbitControls.js`, `loaders/GLTFLoader.js`, `utils/BufferGeometryUtils.js`. Importing any other `three/addons/*` path fails at runtime.

## Versions

The avatar SDK is released in versions. Your game is **frozen on the version it was first stamped with**: publishing again, or a newer platform release, never changes the avatar SDK under a working game. To move to a different released version, pin it in a `portals.json` at your project root:

```json
{ "guardiansSdk": "0.36.0" }
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
    "@portals/avatars": "https://portals.to/portals-sdk/guardians/0.36.0/guardians-sdk.module.js"
  }
}
</script>
```

Then record the same version in `portals.json`. When you push the game to Portals, the known-CDN URLs are rewritten to the managed same-origin runtime and the pin keeps the SDK at the release you tested.

## Load the current Portals player

`Portals.player.get()` reads the player's active public username and the playable look saved on `/avatar`. `createAvatarFromPlayer()` turns that result into the exact Guardian configuration and wearables — or the selected full-avatar asset — in one call:

```js
await Portals.ready();
const { THREE, PortalsAvatars } = await PortalsGuardians.ready();

const avatars = new PortalsAvatars({ scene });
const player = await Portals.player.get();

nameLabel.textContent = player.username
  ? `@${player.username}`
  : player.displayName || "Guest";

let avatar = await avatars.createAvatarFromPlayer(player);
if (!avatar) {
  // Guests and hosts without a saved look get the game's ordinary fallback.
  avatar = await avatars.createAvatar({ bodyType: "female" });
}
```

The result is `null` for a guest, so keep a playable default. Both the profile read and asset load can reject; catch them at startup and use that same fallback rather than blocking the game.

The first signed-in profile is cached for the hosted-game load. After a player signs in, read the profile again — the lightweight object returned by `requestLogin()` does not include the playable avatar:

```js
await Portals.identity.requestLogin();
const signedInPlayer = await Portals.player.get();
const avatar = await avatars.createAvatarFromPlayer(signedInPlayer);
```

`player.avatarUrl` is the 2D public profile image — never pass it to a GLTF loader. The 3D `/avatar` look is `player.avatar`; letting `createAvatarFromPlayer()` load it also routes creator-hosted wearables through Portals' managed same-origin item proxy.

Multiplayer avatars: send the public `player.avatar` shape over `Portals.net`, not a private inventory response. Remote players can be rebuilt with `createAvatarFromPlayer({ ...identity, username, avatar })`.

### Let the player change their global avatar

Call `Portals.avatar.openPicker()` from a click or tap. Portals opens its trusted avatar interface over the game, signs the player in when needed, and resolves only after the global `/avatar` look is saved. The result is the refreshed player profile, ready to render:

```js
customizeButton.addEventListener("click", async () => {
  const player = await Portals.avatar.openPicker();

  if (avatar) avatars.removeAvatar(avatar);
  avatar = await avatars.createAvatarFromPlayer(player);
  controller = avatars.createController(avatar, {
    camera,
    domElement: renderer.domElement,
  });
});
```

Choosing **Done** changes the player's avatar across Portals, not just inside the current game; **Cancel** discards the draft. Catch rejection so cancelation, an unavailable host, or a failed save does not interrupt play. The editor preview intentionally rejects `openPicker()`; keep the preview playable on rejection and verify the interaction in a published game host.

The game never gets raw inventory, owned-instance IDs, marketplace operations, credentials, or direct global item mutation. `player.avatar.wearables` is a sanitized, render-ready snapshot of the equipped look — not inventory or ownership proof. The Portals-owned picker only offers eligible items from the signed-in player's existing Shop inventory, and the server validates ownership and compatibility again. The Guardian SDK's `wearables.equip()` changes an avatar object inside the game; it does not change the player's global Portals look.

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

`avatars.update(dt)` drives every avatar and controller — call it once per frame or nothing animates. It is also the mixer tick that rewrites every bone local from the clips, so **anything that writes bones — `Ragdoll.update()`, `aimWeapon()`, `applyPose()` — must run AFTER it** in the frame. Stepped before it, they are silently overwritten: the corpse plays its death clip and the gun never points anywhere. Nothing errors; it just does not work.

## Three.js is shared, not bundled

The SDK does not ship its own copy of Three.js — it uses the same managed runtime, so the page has exactly one instance and objects the SDK creates work with your own scene graph, raycasters and materials.

Take `THREE` from the SDK rather than loading it yourself. From the classic script it comes out of `ready()`, as above. From a module you can import it directly, since the import map points at the same file:

```js
import * as THREE from 'three';
```

Loading a second copy from a CDN will not work — the game CSP keeps `script-src` at `'self'` — and would break instance checks even if it did.

## Assets load through Portals

Guardian models, wearables and animation clips live on the Portals CDN. Published games run under `connect-src 'self'` and cannot fetch from another host, so the SDK rewrites asset URLs onto managed same-origin paths that Portals serves for you (`/_portals/cdn/…`, `/_portals/media/…` and, for wearables stored elsewhere, `/_portals/item/<id>`). This is automatic — pass ordinary Portals URLs and they resolve. Hand-written `fetch()` to a CDN works in no environment.

Two consequences worth knowing:

- **Thumbnails need the same treatment.** `img-src` is `'self'` too, so run a wearable's `thumbnail` through `resolveAssetUrl` before putting it in an `<img>`:

  ```js
  import { resolveAssetUrl } from '@portals/avatars';
  img.src = resolveAssetUrl(def.thumbnail);
  ```
- **Arbitrary hosts need an item id.** A wearable whose GLB is hosted outside Portals can only be fetched through the item proxy, which addresses it by inventory item id. Equip it as a catalog item (with its real inventory `id`) rather than as a bare URL. A raw URL to an unmanaged host logs a warning and is returned unchanged, so the browser blocks it — that is the failure mode to look for when a wearable silently does not appear.

Relative, `blob:` and `data:` URLs pass through untouched — assets shipped inside your own bundle need nothing.

## Appearance

```js
avatar.setSkinColor('tan');           // preset id or any CSS colour
avatar.setHairColor('#584435');
avatar.setEyeColor('green');
avatar.getHairStyles();               // [{ id, name }]
avatar.setHairStyle('Spiky_Bun');     // null / 'none' for bald
avatar.setFacialHair(null);           // remove
```

Colour setters return `this` and chain. `getHairStyles()` differs per body type — never hardcode the list. `avatar.configure({ … })` applies a whole `GuardianConfig` at once, and `avatar.getConfig()` reads it back — useful for persisting a look with the Portals SDK's save API.

## Wearables

```js
avatar.wearables.registerCatalog(myItems);      // WearableDef[]
await avatar.wearables.equip('basic-tshirt');   // catalog id
await avatar.wearables.equip({ id: 'hat-1', name: 'Cap', slot: 'hat', url: '…' });
await avatar.wearables.unequip('hat');          // by slot or id
avatar.wearables.getEquipped();
```

Equipping replaces whatever occupies the item's slots, hides the body meshes the slot covers, and re-binds skinned wearables to the avatar skeleton. Multi-slot items (dresses, full-body suits) declare every slot they occupy via `slots`. `urlFemale` is used automatically on a female Guardian.

Right-hand items get a carry pose and a use action inferred from the item name — an axe swings, a pistol shoots — or set `handItemType` explicitly. Listen for the action with `controller.onItemUse`.

## Animation

The Guardian models ship without clips. Clips are **declared at avatar creation and fetched on first use**, one GLB per behaviour — a game that only walks and jumps fetches one locomotion file and never downloads the swim, sit or spell clips at all. `createAvatar` resolves as soon as the body is on screen; call `avatars.preloadAnimations()` to pay the cost behind a loading screen instead.

```js
avatar.animations.getNames();             // available — loaded OR declared
avatar.animations.isLoaded('walk');       // in memory right now
avatar.animations.play('wave');           // false if not loaded yet; starts the fetch
await avatar.animations.ensure('wave');   // explicit load
await avatars.preloadAnimations();        // everything
```

`play()` returns `false` until the bytes arrive and the current pose simply holds — **do not treat a `false` return as an error**. Poll `isLoaded()` or `await ensure()` if you need certainty.

Custom clips work the same way, and any Mixamo- or UE5-mannequin-rigged GLB is retargeted onto the Guardian rig automatically:

```js
await avatar.animations.load([
  { name: 'wave', url: './anims/wave.glb', loop: false },
]);
```

The character controller drives locomotion states itself. Return `true` from `controller.onBeforeAnimationUpdate` to take over for a frame.

### Clip names

**Always available** (declared on every avatar, played by the controller itself):

`idle` `walk` `run` `sprint` `jump` `fall` `land` `roll` `climbTop`

Two 8-way directional sets, used when strafing. The controller picks by speed, so a slow sidestep is a walked one rather than a jog at half rate.

Walk pace — `walk` is forward:
`walkBack` `walkLeft` `walkRight` `walkFwdLeft` `walkFwdRight` `walkBackLeft` `walkBackRight`

Run pace — `run` is forward:
`runBack` `runLeft` `runRight` `runFwdLeft` `runFwdRight` `runBackLeft` `runBackRight`

The run band comes in two public styles (`RUN_STYLES`): `'run4'` (default — a natural forward run, ≈3.6 m/s world) and `'jog2'` (a relaxed ≈2.0 m/s forward jog over the same strafe band). Pick with `new PortalsAvatars({ runStyle: 'jog2' })` or switch live with `await avatars.setRunStyle('run4' | 'jog2')`; every other clip is unaffected. Sprinting always breaks strafing: while the sprint key is held and the character moves, the body turns into the direction of travel and plays the forward sprint — walk and run strafe normally.

Crouch, same idea — `crouchWalk` is forward:
`crouchIdle` `crouchWalk` `crouchBack` `crouchLeft` `crouchRight` `crouchFwdLeft` `crouchFwdRight` `crouchBackLeft` `crouchBackRight`

Right-hand item overlays, fetched when the avatar first holds something:
`carry` `meleeA` `meleeB` `shoot` `shoot2h` `throw`

**Opt-in groups** — `await avatar.animations.load(ANIMATION_SETS.<set>)`, one GLB each:

| Set | Names |
| --- | --- |
| `emote` | `dance` `celebrate` `cry` `talkGesture` `paper` `rock` `scissors` `holdTorch` |
| `sit` | `sitEnter` `sitIdle` `sitIdle2` `sitIdle3` `sitTalking` `sitNodding` `sitExit` `groundSitEnter` `groundSitIdle` `groundSitExit` `driving` |
| `interact` | `interact` `pickUpHigh` `pickUpLow` `pushEnter` `push` `pushExit` `repair` `drink` |
| `counter` | `counterEnter` `counterIdle` `counterShow` `counterGive` `counterAngry` `counterExit` |
| `unarmed` | `punchJab` `punchCross` `kick` `fightEnter` `fightExit` |
| `hit` | `hitHead` `hitChest` `hitStomach` `hitShoulderLeft` `hitShoulderRight` `death` `death2` |
| `sword` | `swordEnter` `swordIdle` `swordAttack` `swordAttackStanding` `swordExit` |
| `pistol` | `pistolIdle` `pistolAimUp` `pistolAimNeutral` `pistolAimDown` `pistolShoot` `pistolReload` |
| `spell` | `spellEnter` `spellIdle` `spellShoot` `spellExit` `spellDoubleEnter` `spellDoubleIdle` `spellDoubleShoot` `spellDoubleExit` |
| `swim` | `swimIdle` `swimFwd` |
| `crawl` | `crawlEnter` `crawlIdle` `crawlFwd` `crawlBack` `crawlLeft` `crawlRight` `crawlExit` |
| `climb` | `climbEnter` `climbIdle` `climbUp` `climbDown` `climbLeft` `climbRight` `climbExit` |
| `acrobatics` | `backflip` `dodgeLeft` `dodgeRight` |
| `locomotionExtra` | `sprintEnter` `sprintExit` `walkFormal` `turnLeft` `turnRight` `idleLookAround` `idleTired` |
| `lean` | `runLeanLeft` `runLeanRight` |
| `crouchTransitions` | `crouchEnter` `crouchExit` |
| `walkCarry` | `walkCarry` |
| `mantle` | `climbUp1m` `getOffWall` `stepUp` |
| `parkour` | `wallRunLeft` `wallRunRight` `wallRunJumpLeft` `wallRunJumpRight` `slideStart` `slide` `slideExit` `vault` `kipUp` `doubleJump` `runFlip` |
| `ninjaJump` | `ninjaJumpStart` `ninjaJumpDouble` `ninjaJumpAir` `ninjaJumpLand` |
| `turn180` | `turn180Left` `turn180Right` |
| `overhandThrow` | `overhandThrow` |
| `swordLight` | `swordLightA`…`swordLightD` + `…Rec` + `swordLightCombo` |
| `swordHeavy` | `swordHeavyA`…`swordHeavyD` + `…Rec` + `swordHeavyCombo` |
| `swordRegular` | `swordRegularA`…`swordRegularC` + `…Rec` + `swordRegularCombo` |
| `swordSpecial` | `swordAerialA` `swordAerialARec` `swordAerialB` `swordAerialCombo` `swordAerialIdle` `swordBlock` `swordDash` `swordGroundPound` `swordUppercut` |
| `meleeCombo` | `meleeChain` `meleeHook` `meleeHookRec` `meleeKnee` `meleeKneeRec` `meleeUppercut` |
| `bow` | `bowNotch` `bowAimUp` `bowAimNeutral` `bowAimDown` `bowShoot` `bowRapidShoot` |
| `shield` | `shieldIdle` `shieldBreak` `shieldDash` `shieldBash` `shieldSprint` |
| `launched` | `liftAir` `liftAirIdle` `liftAirHitLeft` `liftAirHitRight` `liftAirFall` `liftAirFalling` `liftAirImpact` `knockback` |
| `farm` | `plantSeed` `scatterSeeds` `water` `harvest` `pickFromTree` |
| `fish` | `fishCast` `fishWait` `fishHold` `fishReel` `fishReelFailed` |
| `gather` | `mine` `chopTree` `openChest` `bandage` `consume` |
| `idleExtra` | `idleFoldArms` `idleLantern` `idleNo` `idlePhone` `idleRail` `idleRailCall` `surprise` `yes` |
| `lay` | `idleToLay` `layToIdle` |
| `zombie` | `zombieSpawn` `zombieIdle` `zombieBite` `zombieScratch` + 8-way `zombieWalk*` and `zombieRun*` |
| `monster` | `monsterTransform` |

Two traps: `talkGesture` is the body gesture, `talking` is the **face** (`avatar.face.setTalking`); and `climbTop` (the ledge grab, always loaded) is not `climbUp` (ladder climbing, in the `climb` set).

Faces are a separate layer:

```js
avatar.face.setEmotion('smile');   // neutral | smile | serious | frown
avatar.face.setTalking(true);      // drive from mic or dialogue state
avatar.face.setAutoBlink(true);    // on by default
```

`setEmotion` and `setTalking` also return early while their clip loads, then apply themselves when it lands.

## Character controller

`createController` returns a character controller with WASD movement, sprint, jump, crouch and a first/third-person camera rig. Movement values clamp to the ranges in `MOVEMENT_LIMITS`.

```js
controller.configureMovement({ walkSpeed: 2, runSpeed: 4, jumpHeight: 1.2 });
controller.toggleCameraMode();
controller.setEnabled(false);      // e.g. while a menu is open
```

Keys are read from the window, and only real text entry (`<input type=text>`, `<textarea>`, `contenteditable`, `<select>`) takes them away from the game — sliders, checkboxes and buttons do not, so a click on a UI control never leaves WASD dead. A focused widget still keeps the keys it natively owns (arrows for a slider, Space for a button); if your own UI keeps focus after a click, blur it on `pointerup`. Held keys are released on `blur` and whenever the page is hidden, so alt-tabbing away never leaves the character running.

**Gravity, stairs and step-downs** (0.25.0). Falling runs at `gravity` 12 m/s² capped at `maxFallSpeed` 12 m/s (both settable in `configureMovement`), and the default 1.6 m jump apex is derived from the gravity — raising it makes the character heavier without also shortening its jump. Drops up to `stepDown` (0.4 m) are walked down while staying grounded, which keeps a run down a staircase from playing the falling clip on every tread; anything deeper falls normally, once the ground loss outlasts `fallAnimDelay` (0.12 s of coyote time).

**Swimming** (0.26.0). A movement mode the GAME toggles at the water's edge — the SDK has no idea where a level's water is:

```js
controller.setSwimming(true, { surfaceY: waterY - 0.2 }); // entered water
controller.setSwimming(false);                            // back on land
```

While on, gravity is off and movement runs at `swimSpeed` (default 1.9 m/s, a `ControllerOptions` field and `configureMovement` key). Space swims up and C dives (programmatic: `setJumpHeld` / `setSwimVertical(-1..1)`); jumping, crouching, the ledge grab and `pickUpItem()` are suspended, while hand items keep working over the swim pose. The `swimFwd`/`swimIdle` clips drive themselves, cadence tracking real speed. `surfaceY` is an optional ceiling the ROOT may not rise above; the root is not at the feet in water — the clips hang the body off it flat and chest-high, so a surface swimmer wants about `waterY − 0.2`, and a full body-depth below that simply submerges it. Omit it for open water. The bottom (`getGroundHeight`) and walls (`resolveMovement`) keep working. Turning it off hands the body straight back to normal physics — sinking to the bottom underwater, falling above it — so toggle exactly at the boundary.

Since 0.36.0 the swimmer collides as that PRONE body rather than as the upright capsule: a horizontal capsule along the facing, `swimRadius` (0.35 m) with `swimReach` (0.55 m) either side of the root. It stops 0.9 m short of a wall head-on, 0.35 m from one it drifts into sideways, and settles a radius clear of the bed instead of half inside it. `SceneCollider` drives this through a `depenetrateSegment` hook it supplies automatically; a hand-rolled collider still gets the widened swept probe, and can implement the hook for the overlap recovery too.

**Touch (mobile).** On coarse-pointer devices the controller automatically adds on-screen touch controls: a virtual joystick (deflection sets the pace), drag-to-look with pinch-to-zoom (pinching fully in enters first person), and semi-transparent buttons — jump, sprint, crouch, camera, plus use/throw while a hand item is held. Control it with the `touch` option (`'auto'` default, `true`/`false` to force, or a `TouchControlsOptions` object — e.g. `{ joystick: 'dynamic' }` spawns the stick under the thumb). Restyle via the `.pa-touch*` CSS classes; `controller.touch.onTap` reports taps that weren't drags, for your own tap-to-interact raycasts. A game with its own mobile UI passes `touch: false` and drives the programmatic input API instead: `setMoveVector(x, y)` (analog, camera-relative), `setJumpHeld`, `setSprint`, `beginPrimaryAction`/`endPrimaryAction` (tap = strike, hold = melee combo), `setCrouch`, `throwItem`.

**Strafing.** By default the body turns to face where it is going, so A/D walk the character left and right rather than sidestepping. Pass `strafe` to hold the camera's heading and slide instead, playing the directional clips by direction of travel:

```js
avatars.createController(avatar, { domElement, camera, strafe: 'always' });
// 'off' (default) | 'always' | 'aim'
controller.setStrafe('aim');   // free until aimed
controller.setAiming(true);    // …which this turns on
```

`'aim'` engages by itself while a `gun` / `gun2h` right-hand item is equipped, so a shooter needs no extra wiring. First person always strafes whatever the mode. Without the directional clips loaded the facing still changes and the forward run clip plays, so this never breaks — it just looks less good.

**Shooter-grade strafing** (0.36.0). Four refinements on the above, all **off by default** so an existing game moving to a newer SDK animates unchanged:

```js
controller.setStrafeTuning({
  collapse: 'auto',            // 'off' | 'hip' | 'aim' | 'auto'
  diagonalTurn: Math.PI / 4,   // turn the body onto a forward diagonal
  sprintCap: true,             // keep the strafe lock while sprinting
  hysteresis: true,            // dead band on the walk/run boundary
});
```

`collapse: 'aim'` folds every diagonal onto the pure sidestep (a gunfight reads better as a clean shuffle, and the diagonal takes swing a held weapon across the body); `'hip'` sends the forward diagonals to the plain forward clip and the back ones to the sidestep; `'auto'` switches between them on `aiming`. Pair `'hip'`/`'auto'` with `diagonalTurn` or the body crabs — that turns the whole body to face its travel, and the live offset is on `controller.diagonalYaw`, which you hand to `aimWeapon` as `diagYaw` and replicate (the body yaw is no longer simply the camera's). `sprintCap` stops Shift dropping the strafe lock, capping off-axis directions at run pace instead. `hysteresis` replaces the single walk/run threshold with a 2.0 / 2.6 m/s dead band.

**Weapon posing** (0.36.0). Hand items give a gun its carry pose and `shoot` overlay; this is the stance on top — aimed grip, one-handed hip point, two-handed waist carry — plus the pitch bend, procedural recoil and barrel aim that no clip can do.

```js
await loadGunPoses(avatar);   // adsPose / hipPose / heavyPose / hipShoot
await avatar.wearables.equip({ ...gunDef, offset: GUARDIAN_PISTOL_GRIP });

// each frame, AFTER avatars.update(dt)
pose = applyPose(avatar, pose, wantedPose(weaponUp, aiming, twoHanded));
aimWeapon(avatar, { pose, lookYaw, pitch, diagYaw: controller.diagonalYaw, kick });
```

`GUARDIAN_PISTOL_GRIP` / `GUARDIAN_TUBE_GRIP` are solved offsets for a weapon authored barrel-down-`+Z`, sights-up, grip-toward-`−Y`; mark the barrel tip with an empty named `muzzle` and resolve it by name after equipping (equip attaches a clone, so a reference to the mesh you handed in points at nothing). `carryFor(slim)` picks `RAIL_CARRY` over `HEAVY_CARRY` for a thin weapon a wrapped arm would look like it was clutching air around. In multiplayer, run every remote body through the same functions from its replicated yaw/pitch/flags — the body is the only aim indicator a peer gets.

**Shouldering a scoped weapon** (0.37.0). `aimWeapon(avatar, { ..., scoped: aiming })` brings the waist carry up to the cheek over ~200 ms: `SCOPE_DEFAULTS` is a destination (hand target, elbow pole, shoulder lift, head follow, its own kick shares) that the existing two-bone solve walks to, and the barrel then tracks the LOOK rather than the torso's share of it. Override with `scope: { ...RAIL_CARRY, ...yours }` — spread a carry in, or the fields you omit fade toward zero. Same flag at both ends of the wire. If two callers pose ONE body in a frame (a killcam rewind plus the live loop), give each its own `driver` string or one advances the ease and the other does not; pass `dt` from a fixed-step loop.

**Where the shot comes from** (0.37.0). `muzzleWorld(avatar)` is the barrel tip for tracers and flashes. For a hitscan use `shotOrigin(avatar, { raycast: collider.raycast, crouching })` and trace TWICE: the camera trace says where the shot is going, a second trace from this origin says whether anything stops it (floor the aim point a stride past the origin, or a crosshair landing behind the barrel fires backwards). A third-person lens sees over cover the barrel is still behind — trace only from the camera and every wall is cover for the other guy. A barrel buried in geometry falls back to the chest, because a ray starting inside a wall meets none of its faces.

**Ragdoll** (0.36.0). A position-based solver on the skeleton, not a physics engine.

```js
const ragdoll = new Ragdoll(avatar, { ...collider.controllerOptions(), bounds: 40 });
controller.setBodyLocked(true);   // or the controller keeps walking the corpse
ragdoll.start({ ...shotKick({ from, point, head }), velocity: controller.velocity });
ragdoll.update(dt);               // in the loop, AFTER avatars.update(dt)
ragdoll.stop();                   // on respawn
```

Spread `controllerOptions()` whole — the extra keys are ignored, and `getGroundHeight`/`raycast` are what land a body on a rooftop and keep one thrown at a wall out of it. Leave `bounds` off for a map with a void. Keep a death clip playing underneath: it holds the bones the solver does not simulate (fingers, jaw, toes). `start()` returns `false` on a rig it cannot drive; tune via `RAGDOLL_DEFAULTS` keys passed as `tuning`.

**Restricting camera rotation** (0.24.0). `cameraLimits` bounds how far the camera can be turned — angles in radians:

```js
avatars.createController(avatar, {
  domElement, camera,
  cameraLimits: { minYaw: -0.6, maxYaw: 0.6, lockPitch: true },  // side-on stage
});
controller.setCameraLimits({ lockYaw: true, lockPitch: true });  // cutscene
controller.setCameraLimits({ minYaw: null, maxYaw: null });      // window off
```

Ranges (`minYaw`/`maxYaw`/`minPitch`/`maxPitch`) bound the camera however it moves, input and SDK camera writes alike; locks (`lockYaw`/`lockPitch`) stop only *input*, so the game can still place the camera for a rail or a scripted pan. `setCameraLimits` changes only the keys you pass, moves a camera already outside a new range to its nearest edge, and survives an avatar swap. Yaw uses `cameraRig.yaw`'s convention — 0 looks along −Z, increasing turns left; pass both ends (one alone means nothing on a circle) and `null` to clear. Movement is camera-relative and the body faces the camera while strafing or in first person, so a narrow yaw window also narrows where the character can walk and face. Custom control schemes should call `controller.cameraRig.applyLookDelta(yawRadians, pitchRadians)` instead of writing `yaw`/`pitch` — that is what gets them inversion, the locks and the limits.

### Ground and collision

Ground and collision default to a flat floor at `y = 0` and free movement. A real level must supply them. `SceneCollider` derives every hook from the meshes you register, triangle-exact, so ramps, cylinders and imported geometry all work:

```js
const collider = new SceneCollider();
collider.add(ground).add(levelGeometry);   // level meshes only, never avatars

avatars.createController(avatar, {
  camera,
  domElement: renderer.domElement,
  ...collider.controllerOptions(),
});
```

Call `collider.refresh()` after adding or removing meshes under a registered root, and keep the set to level geometry — never avatars or props. Since 0.28.0 the collider builds a BVH per registered geometry (a one-time cost at `add()`, ~1 ms per ~1k triangles) and supplies a `depenetrate` hook: a capsule overlap pass, run after the vertical step, that pushes the body back out of walls. Every raycast goes through the BVH too. `capsule: false` in `SceneColliderOptions` turns both off if a huge scene makes the build hitch matter more than wall penetration. Since 0.32.0 the ground probe is capsule-aware: the controller passes its radius to `getGroundHeight`, and the collider answers with a centre ray plus four rim rays, so a landing that is only half on a ledge, a plank gap or a pillar's rim stands on it instead of falling through.

Avatars are solid to each other too (0.32.0, capsule-vs-capsule): every controller and NPC created through `PortalsAvatars` is pushed back out of any other avatar's capsule each frame — players cannot run through NPCs or each other. The push is horizontal only (landing on someone's head slides off) and hidden avatars are skipped. Opt a character out with `collideWithAvatars: false` (a spectator, a cutscene ghost); setups built outside the factory wire the obstacle set with `controller.setAvatarObstacles(() => avatarsIterable)`.

Wiring `raycast` also switches on **camera avoidance** (0.23.0, on by default): the third-person boom shortens when a wall comes between the character and the camera — snapping in, easing back out — and drops to first person when the character's back is against something and there is nothing left to frame, returning to third person once clear. Other avatars never block the view, because `SceneCollider` skips skinned meshes. Turn it off with `cameraCollision: false`, or tune it:

```js
avatars.createController(avatar, {
  camera, domElement: renderer.domElement,
  ...collider.controllerOptions(),
  cameraCollision: { minDistance: 0.7, firstPerson: true, probe: myCameraOnlyRaycast },
});
```

`controller.cameraRig.allowedOccluders` (0.34.0) is a set of objects allowed to hide the player — **empty by default, filled only by the game**: `allowedOccluders.add(mesh)` (or a group, covering its descendants) and that object may come between the character and the camera without the boom pulling in. The camera still refuses to sit *inside* a listed object, so keep it closed and consistently wound; everything not listed pulls the boom in as always.

Games with their own physics pass the hooks directly instead:

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

`raycast` is what enables ledge climbing: two rows of rays, one per shoulder, sweep down across the character's reach, and a hit within 30° of horizontal is a ledge it pulls itself onto. Both rows must hit (so brushing a wall never triggers it, and a ledge narrower than the shoulders is not grabbable), and it only probes while airborne — a climb always follows a jump or a fall, from the first airborne frame onward. Since 0.36.0 the surface grabbed must also be the first thing in front of the character: anything more than 0.25 m taller standing between the two rejects the grab, so a wall is never climbed *through* to reach the roof behind it — including a wall taller than the probe band, or one modelled as a single quad with no top face for the downward rays to find. Lower obstacles (a kerb at the foot of the wall) still let the climb happen. Without the hook the character just bumps into walls. `checkClearance` keeps it from climbing into a ceiling.

## NPCs

`createNPC(avatarOptions, npcOptions)` / `createNPCFromUrl(url, extra, npcOptions)` load the same Guardian as `createAvatar` (wearables, face, animations — all of it) wrapped in a code-driven `NPCController` instead of player input. The shared `avatars.update(dt)` drives them; nothing extra to call.

```js
const npc = await avatars.createNPCFromUrl(playerLookUrl, { position: { x: 4, y: 0, z: 2 } });
await npc.moveTo({ x: 0, z: 0 }, { run: true });  // promise resolves on arrival (false if superseded/stopped)
npc.follow(player.root, { distance: 2 });          // indefinite; walks when the gap opens, idles when close
npc.lookAt(player.root);                           // smooth idle turn; { instant: true } snaps
await npc.playAnimation('sitIdle');                // fetches the clip if needed
npc.stopAnimation();                               // hand a looping clip back to idle/locomotion
npc.hide(); npc.show();
avatars.removeNPC(npc);
```

Locomotion (idle/walk/run selection, stride matching) is automatic while moving. Gotchas: NPCs keep their spawn height unless you pass `getGroundHeight` (per NPC or SDK-wide via `PortalsAvatarsOptions.npcDefaults`); `playAnimation` of a looping clip owns the body until `stopAnimation()` — for gestures during movement use `npc.avatar.animations.playOverlay()`; NPCs collide with the player and each other by default (0.32.0) — `collideWithAvatars: false` for a walk-through ghost; everything else an avatar can do is on `npc.avatar`.

## Cleanup

`avatars.removeAvatar(avatar)` disposes the avatar, its GPU resources and its controllers. `avatars.removeNPC(npc)` does the same for an NPC. `avatars.dispose()` does the lot — call it when leaving a scene.

## Anti-patterns

| Do not | Instead |
| --- | --- |
| `import * as THREE from 'https://unpkg.com/three'` | bare `'three'` via the import map, or `THREE` from `ready()` |
| use the SDK synchronously after the script tag | `await PortalsGuardians.ready()` first |
| map `@portals/avatars` at `guardians-sdk.js` | map it at `guardians-sdk.module.js` |
| `new GLTFLoader().load('https://addressables-cdn…')` | let the SDK load it, or `resolveAssetUrl` first |
| `equip({ id: 'x', url: 'https://mysite.com/hat.glb' })` for an inventory item | `equip(realInventoryItemId)` |
| assume `play()` succeeding means the clip ran | check `isLoaded()` or `await ensure()` |
| forget `avatars.update(dt)` | call it once per frame |
| run `ragdoll.update()` / `aimWeapon()` before `avatars.update(dt)` | bone writers run AFTER the mixer tick |
| `avatar.root.parent = scene` | `PortalsAvatars` adds it for you |
| hardcode hair style ids | `avatar.getHairStyles()` |
| load `player.avatarUrl` as the game character | `Portals.player.get()` then `avatars.createAvatarFromPlayer(player)` |
| copy `/avatar` API calls or account tokens into game code | use the host-mediated `Portals.player.get()` contract |
| build a game-owned global inventory picker or send wearable ids to the host | call `Portals.avatar.openPicker()` from a player action |
