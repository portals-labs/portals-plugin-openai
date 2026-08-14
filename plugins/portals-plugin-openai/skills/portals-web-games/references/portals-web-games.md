# Portals Web Games Reference

This is the MCP server's bundled instruction set, copied from `src/instructions.ts` so the plugin skill and server use the same platform guidance.

## Contents

- [Portals SDK — identity, saves, leaderboards](#portals-sdk--identity-saves-leaderboards)
- [Multiplayer — Portals.net](#multiplayer--portalsnet)
- [Voice — Portals.voice](#voice--portalsvoice)
- [Guardian avatars — the 3D character SDK](#guardian-avatars--the-3d-character-sdk)
- [Guardian appearance, wearables, faces](#guardian-appearance-wearables-faces)
- [Guardian animation](#guardian-animation)
- [Guardian character controller](#guardian-character-controller)
- [Guardians together with the Portals SDK](#guardians-together-with-the-portals-sdk)
- [Guardian SDK versions and local development](#guardian-sdk-versions-and-local-development)
- [Testing multiplayer on a local dev server](#testing-multiplayer-on-a-local-dev-server)
- [Generated assets — art, audio, and 3D models](#generated-assets--art-audio-and-3d-models)

Games pushed with push_web_game_source run on Portals with an injected SDK. Load it before your game code — the file is managed by Portals, never modify or bundle it:

  <script src="./_portals/sdk.js"></script>
  <script src="./game.js"></script>

TypeScript declarations are not part of the game's files — for a TS project download them: curl -o portals.d.ts https://portals.to/portals-sdk/portals.d.ts. The game runs in a sandbox; external network access is restricted — use the SDK below instead of your own backend.

## Portals SDK — identity, saves, leaderboards

- await Portals.ready() before touching player data; resolves to the current player and host context.
- Portals.getPlayer() → Promise of the latest player. playerId is deliberately different across games; it may be null for unsigned users.
- Portals.identity.requestLogin() → Promise of the signed-in player. Only call from a direct user action (button click). Never ask for passwords in-game.
- Portals.identity.onChange(listener) → unsubscribe fn; fires on sign-in/out.
- Portals.saveState(data) / Portals.loadState() — per-player persistence; requires sign-in. State must be JSON-serializable and no larger than 64 KB after JSON encoding. loadState resolves to the saved state or null.
- Portals.submitScore(score, mode?) — mode defaults to "default"; lowercase letters, numbers, hyphens, max 32 chars.
- Portals.getLeaderboard({ mode, limit }?) → { entries } with rank, playerId, displayName, avatarUrl, score. limit 1–100, default 10. Free games allow unsigned leaderboard reads; paid games require verification first.
- Portals.quit() — asks the host to close the game and restore player controls.

All async SDK methods can reject (network, access restrictions) — handle rejections. Casual scores are client-reported: never use scores or peer messages to award currency, paid prizes, access, or any valuable entitlement. Never put API keys, tokens, or other secrets in game code or saved state.

## Multiplayer — Portals.net

Multiplayer is what sets Portals games apart: when building or updating a game, default to including at least a lightweight Portals.net feature — live player presence, a shared score race, co-op or turn-taking, in-game chat — and propose one to the developer if they haven't asked. The minimum version is a few lines (join() plus a send() or setState()); only skip it when it truly doesn't fit the game.

A session groups all players in the same bucket (game page or room). join({ channel }) creates private sub-lobbies; channel names allow letters, numbers, colons, underscores, hyphens, must start alphanumeric, max 64 chars.

Sessions are regional: fast-paced games place each region's players in their own copy of a channel, so nearby players meet by default. A "global:" channel prefix shares one worldwide room instead — join({ channel: "global:lobby" }). Use global channels for lobbies, matchmaking, and chat; a match arranged in a global lobby needs a global channel too ("global:match-x7"), or the players split back into their own regions' rooms and never meet. A global room runs in one place (the US home region), so distant players see higher latency — keep fast-paced matches on regional channels unless a match deliberately spans regions.

  const net = Portals.net;
  const session = await net.join();        // { self, players, state } — a snapshot
  net.players(); net.self(); net.getState(); // live synchronous reads after joining

- send(data) — broadcast to the other players (sender does not receive its own message), max 8 KB JSON.
- setState(key, value) / getState(key?) — shared state late joiners see. Up to 64 keys, key max 128 chars, value max 8 KB. The "state" event fires for everyone, including the writer.
- Events via on/off: "message" (data, fromId), "playerjoin" / "playerleave" (player, players), "state" (key, value), "status". Player objects: { id, playerId, displayName, avatarUrl } — id is the connection, playerId the game-scoped identity (null when unsigned).
- Rate limits: ~20 broadcasts and ~10 state writes per player per second. Sample input on a 100–150 ms interval; never send per-frame.
- No automatic reconnection: on status "disconnected", show a rejoin button that calls join() again.
- Default multiplayer games to a server-simulated design — a root server.js runs the game's simulation on Portals servers (see the portals-server-sim skill); at minimum ship a server-script referee. Validate every incoming message's kind and shape either way; only purely cosmetic shared spaces and slow turn-based games should ship trust-light.

Text chat rides the same relay with a kind field, e.g. net.send({ kind: "chat", text }). Render chat with textContent, never innerHTML; cap text length (e.g. 300 chars) on both send and receive. For history late joiners see, keep a rolling array in shared state (slice to the last ~20 entries).

## Voice — Portals.voice

Portals owns the microphone and permissions — never call getUserMedia or use WebRTC libraries.

- join() resolves to { self, participants, muted }. The promise stays pending while the player answers the consent card — don't race it against short timers, and a rejection must leave the game fully playable.
- setMuted(muted) / muted() — always render an easy-to-reach mute button.
- devices() → [{ deviceId, label, active }] and setDevice(deviceId) for mic selection. Labels are empty until mic permission is granted; re-call devices() when the picker opens.
- Events: "participantjoin" / "participantleave" (participant, participants), "speaking" (ids), "status" ("disconnected" → show a voice-off control).
- Put voice UI in HTML overlays, not the canvas, for clickability and accessibility.

Availability: Portals.net works on the game page, inside Portals rooms (room session), and in editor preview; Portals.voice only on the game page (rooms run their own voice). Neither exists outside Portals — wrap both join() calls in try/catch and keep the game playable without them.

## Guardian avatars — the 3D character SDK

Every pushed game also gets the Guardian avatar SDK: the same avatars players wear across Portals — body types, skin/hair/eye colour, hair styles, the wearables system, retargeted locomotion and facial animation, and a first/third-person character controller with the Portals feel. It is a Three.js library, not a renderer: the game owns the scene, camera and render loop. Reach for it instead of hand-rolling a character whenever a 3D game wants a player avatar.

Load it beside the SDK. Both files are managed by Portals and stamped into the bundle on every push — never modify, bundle, or ship your own copy. The stamped SDK is versioned per game (see "Guardian SDK versions and local development" below):

  <script src="./_portals/sdk.js"></script>
  <script src="./_portals/guardians-sdk.js"></script>
  <script src="./game.js"></script>

  const { THREE, PortalsAvatars } = await PortalsGuardians.ready();

ready() is required: the global exists as soon as the tag runs but its exports only arrive once the module lands. Take THREE from it — a classic script has no other way to reach the managed Three.js, and a second copy of Three breaks instanceof checks between your objects and the SDK's.

From an ES module (<script type="module" src="./game.js">) import by name instead; both routes reach the same SDK:

  import * as THREE from 'three';
  import { PortalsAvatars, SceneCollider, resolveAssetUrl, ANIMATION_SETS } from '@portals/avatars';

Portals injects the import map that resolves those specifiers when it processes a push, and re-merges it on later pushes — so keep the <script type="importmap"> block that comes back in a pulled index.html. Hand-authoring it means mapping "three" to ./_portals/vendor/three/three.module.js, "three/addons/" to ./_portals/vendor/three/addons/, and "@portals/avatars" to ./_portals/guardians-sdk.module.js. Map the specifier at the module, not at guardians-sdk.js — that one is the classic-script loader.

Never load Three.js from a CDN or bundle it in a pushed game: script-src is 'self', and the managed runtime must be the only copy on the page. (A local dev server is the one exception — see "Guardian SDK versions and local development".) Only three addons are hosted — three/addons/controls/OrbitControls.js, three/addons/loaders/GLTFLoader.js, three/addons/utils/BufferGeometryUtils.js; any other three/addons/* path fails at runtime.

Minimum viable game:

  const { THREE, PortalsAvatars } = await PortalsGuardians.ready();
  const renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setSize(innerWidth, innerHeight);
  document.body.appendChild(renderer.domElement);

  const scene = new THREE.Scene();
  scene.add(new THREE.HemisphereLight('#cfe6ff', '#2a2f3a', 2));
  const camera = new THREE.PerspectiveCamera(55, innerWidth / innerHeight, 0.1, 200);

  const avatars = new PortalsAvatars({ scene });
  const avatar = await avatars.createAvatar({ bodyType: 'female', skinColor: '#EAAB7F' });
  const controller = avatars.createController(avatar, { camera, domElement: renderer.domElement });

  const clock = new THREE.Clock();
  renderer.setAnimationLoop(() => {
    avatars.update(clock.getDelta());   // forget this and nothing animates
    renderer.render(scene, camera);
  });

Assets: Guardian models, wearables and clips live on the Portals CDN, and a published game runs under connect-src 'self' — the SDK rewrites its own asset URLs onto same-origin managed paths, so pass ordinary Portals URLs and never fetch a CDN yourself. For URLs you hand to the browser (a wearable thumbnail in an <img>, since img-src is 'self' too) call resolveAssetUrl(url) first. Relative, blob: and data: URLs pass through untouched, so assets inside your own bundle need nothing.

## Guardian appearance, wearables, faces

- avatar.configure({ bodyType, skinColor, hairStyle, hairColor, eyeColor, facialHair }) applies a whole look; setSkinColor / setHairColor / setEyeColor / setHairStyle / setFacialHair set one at a time and chain. Colours take a preset id or any CSS colour.
- avatar.getHairStyles() and getFacialHairStyles() are read off the loaded model and differ per body type — never hardcode style ids.
- avatar.getConfig() returns a plain object that round-trips through Portals.saveState, so a player's look persists across sessions.
- avatars.createSetupPanel(avatar) mounts a ready-made customisation panel (body type, colours, hair, wearables) with a toggle button; AvatarSetupController is the same model headless if you want your own UI. A body-type switch reloads the avatar — rebuild anything bound to it in onAvatarReplaced, and call newController.adoptFrom(oldController) or the camera snaps and WASD changes direction under the player.
- avatar.wearables.registerCatalog(defs), await equip(idOrDef), await unequip(slotOrId), getEquipped(). Equipping fills the item's slots, hides the body meshes they cover, and re-binds skinned items to the skeleton; multi-slot items declare slots: ['top', 'bottom']. A wearable hosted outside Portals only reaches a published game through the item proxy, so equip it as a catalog item carrying its real inventory id (portalsItemId) — a bare URL to an unmanaged host is blocked by the browser and shows up as a wearable that silently never appears.
- Right-hand items get a carry pose and a use action inferred from the name (axe → melee, pistol → gun) unless handItemType says otherwise; handle the action with controller.onItemUse.
- avatar.face.setEmotion('neutral' | 'smile' | 'serious' | 'frown'), setTalking(bool) (drive it from dialogue or voice), setAutoBlink(bool).

## Guardian animation

Clips are declared at avatar creation and fetched on first use, one GLB per behaviour, so a game that only walks and jumps downloads one file and never pulls the swim or sword clips at all.

- play(name) returns false while its GLB is still downloading and the current pose simply holds — that is not an error. Use avatar.animations.isLoaded(name), await avatar.animations.ensure(name), or await avatars.preloadAnimations() behind a loading screen when you need certainty.
- Always available, played by the character controller itself: idle walk run sprint jump fall land roll climbTop pickup, the 8-way walk set (walkBack walkLeft walkRight walkFwdLeft walkFwdRight walkBackLeft walkBackRight), the matching run set (runBack runLeft runRight runFwdLeft runFwdRight runBackLeft runBackRight), the crouch set (crouchIdle crouchWalk crouchBack crouchLeft crouchRight crouchFwdLeft crouchFwdRight crouchBackLeft crouchBackRight), and the right-hand overlays (fetched once the avatar first holds something) carry meleeA meleeB comboA comboB comboFull shoot shoot2h throw.
- Everything else is opt-in: await avatar.animations.load(ANIMATION_SETS.emote), then play a name from that set. One GLB per set:
  emote: dance celebrate cry talkGesture paper rock scissors holdTorch
  sit: sitEnter sitIdle sitIdle2 sitIdle3 sitTalking sitNodding sitExit groundSitEnter groundSitIdle groundSitExit driving
  interact: interact pickUpHigh pickUpLow pushEnter push pushExit repair drink
  counter: counterEnter counterIdle counterShow counterGive counterAngry counterExit
  unarmed: punchJab punchCross kick fightEnter fightExit
  hit: hitHead hitChest hitStomach hitShoulderLeft hitShoulderRight death death2
  sword: swordEnter swordIdle swordAttack swordAttackStanding swordExit
  pistol: pistolIdle pistolAimUp pistolAimNeutral pistolAimDown pistolShoot pistolReload
  bow: bowNotch bowAimUp bowAimNeutral bowAimDown bowShoot bowRapidShoot
  shield: shieldIdle shieldBreak shieldDash shieldBash shieldSprint
  spell: spellEnter spellIdle spellShoot spellExit spellDoubleEnter spellDoubleIdle spellDoubleShoot spellDoubleExit
  meleeCombo: meleeChain meleeHook meleeHookRec meleeKnee meleeKneeRec meleeUppercut
  swordLight: swordLightA swordLightARec swordLightB swordLightBRec swordLightC swordLightCRec swordLightD swordLightCombo (swordHeavy is the same eight names with Heavy)
  swordRegular: swordRegularA swordRegularARec swordRegularB swordRegularBRec swordRegularC swordRegularCombo
  swordSpecial: swordAerialA swordAerialARec swordAerialB swordAerialCombo swordAerialIdle swordBlock swordDash swordGroundPound swordUppercut
  launched: liftAir liftAirIdle liftAirHitLeft liftAirHitRight liftAirFall liftAirFalling liftAirImpact knockback
  swim: swimIdle swimFwd
  crawl: crawlEnter crawlIdle crawlFwd crawlBack crawlLeft crawlRight crawlExit
  climb: climbEnter climbIdle climbUp climbDown climbLeft climbRight climbExit
  acrobatics: backflip dodgeLeft dodgeRight
  parkour: wallRunLeft wallRunRight wallRunJumpLeft wallRunJumpRight slideStart slide slideExit vault kipUp doubleJump runFlip
  ninjaJump: ninjaJumpStart ninjaJumpDouble ninjaJumpAir ninjaJumpLand
  mantle: climbUp1m getOffWall stepUp
  turn180: turn180Left turn180Right
  locomotionExtra: sprintEnter sprintExit walkFormal turnLeft turnRight idleLookAround idleTired
  lean: runLeanLeft runLeanRight
  crouchTransitions: crouchEnter crouchExit
  walkCarry: walkCarry
  overhandThrow: overhandThrow
  idleExtra: idleFoldArms idleLantern idleNo idlePhone idleRail idleRailCall surprise yes
  lay: idleToLay layToIdle
  farm: plantSeed scatterSeeds water harvest pickFromTree
  fish: fishCast fishWait fishHold fishReel fishReelFailed
  gather: mine chopTree openChest bandage consume
  zombie: zombieSpawn zombieIdle zombieBite zombieScratch plus the 8-way zombieWalk zombieWalkBack zombieWalkLeft zombieWalkRight zombieWalkFwdLeft zombieWalkFwdRight zombieWalkBackLeft zombieWalkBackRight and the same eight as zombieRun*
  monster: monsterTransform
- Two name traps: talkGesture is the body gesture while talking is the face (avatar.face.setTalking), and climbTop is the ledge grab (always loaded) while climbUp is ladder climbing (in the climb set).
- Custom clips work the same way — await avatar.animations.load([{ name: 'wave', url: './anims/wave.glb', loop: false }]) — and Mixamo- or UE5-mannequin-rigged GLBs are retargeted onto the Guardian rig automatically.
- The run band has two styles: 'run4' (default, a natural run) and 'jog2' (a relaxed jog). Pick with new PortalsAvatars({ runStyle: 'jog2' }) or switch live with await avatars.setRunStyle('jog2').

## Guardian character controller

createController gives WASD movement, sprint, jump, crouch and a first/third-person camera rig; speeds clamp to MOVEMENT_LIMITS.

- controller.configureMovement({ walkSpeed, runSpeed, sprintSpeed, jumpHeight, gravity }), toggleCameraMode(), setEnabled(false) while a menu owns the screen.
- Strafe: 'off' (default — the body turns toward the direction of travel, so A/D walk left and right), 'always' (hold the camera heading and sidestep with the directional clips), or 'aim' (free until setAiming(true), which a gun / gun2h item does by itself). First person always strafes.
- Never assign avatar.root.rotation.y — the controller owns it every frame. Use controller.setFacing(yaw) to turn the character (and the camera), and controller.respawn({ position, facing }) to put it back on the ground with every in-flight action cancelled. setBodyLocked(true) hands the body to your own full-body clips (cutscenes, fishing, mini-games) while gravity and the camera keep working.
- Ground and collision default to a flat floor at y = 0 and free movement — a real level must supply them. Easiest is SceneCollider, which reads triangle-exact collision off the meshes you register, so ramps, cylinders and imported geometry behave as they look:

  const collider = new SceneCollider();
  collider.add(ground).add(levelGeometry);   // level meshes only, never avatars
  avatars.createController(avatar, { camera, domElement: renderer.domElement, ...collider.controllerOptions() });

  Call collider.refresh() after adding or removing meshes under a registered root, and keep the registered set to level geometry — every ray is tested against all of it. A game with its own physics passes the four hooks directly instead: getGroundHeight, resolveMovement, raycast, checkClearance.
- raycast is what enables ledge climbing: both shoulders must find the ledge and the probe only runs while airborne, so a climb always follows a jump or a fall. Without the hook the character cannot climb at all.
- avatars.removeAvatar(avatar) disposes one avatar with its controller and GPU resources; avatars.dispose() tears down everything when leaving a scene.

## Guardians together with the Portals SDK

They are complementary: Portals handles identity, saves, leaderboards, multiplayer and voice; the Guardian SDK handles the character.

  await Portals.ready();
  const saved = await Portals.loadState();
  const avatar = await avatars.createAvatar(saved?.look ?? { bodyType: 'female' });
  await Portals.saveState({ look: avatar.getConfig() });

For multiplayer avatars, send avatar.getConfig() over Portals.net and build remote players with createAvatar(config) — its wearables array is equipped as part of the load. Construct the manager as new PortalsAvatars({ scene, defaultOutfit: false }) when your configs already carry a full outfit, since the basic set is otherwise equipped on every avatar. Drive a remote avatar's clips from the state you already sync (position, speed, grounded) rather than sending animation names per frame. avatar.face.setTalking(true) on the players Portals.voice reports in its "speaking" event makes a voice chat read as characters talking.

TypeScript declarations for editor autocomplete: curl -o guardians.d.ts https://portals.to/portals-sdk/guardians.d.ts — it is the authoritative API surface, worth reading before using a method not shown above.

## Guardian SDK versions and local development

The avatar SDK is released in versions, and a game is frozen on the version it was first pushed with — later pushes and newer platform releases never change the avatar SDK under a working game. To move to a different released version, pin it in a portals.json at the project root and push:

  { "guardiansSdk": "0.20.0" }

portals.json is an ordinary project file that travels with every push, so the version a game was developed against locally is the version that runs on Portals. Never invent a version number: pinning an unreleased version fails the push with an error that names the current release. The current release is 0.20.0, built against three 0.178.0.

Every released version is downloadable, so a Guardian game CAN run on a local dev server:

  https://portals.to/portals-sdk/guardians/<version>/guardians-sdk.js
  https://portals.to/portals-sdk/guardians/<version>/guardians-sdk.module.js

Three pieces make the local setup, and all of it is push-safe (no cleanup needed):

1. An import map in index.html — locally the page owns it; on push Portals overrides the managed specifiers and rewrites known three CDN URLs onto the managed runtime:

  <script type="importmap">
  {
    "imports": {
      "three": "https://cdn.jsdelivr.net/npm/three@0.178.0/build/three.module.js",
      "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.178.0/examples/jsm/",
      "@portals/avatars": "https://portals.to/portals-sdk/guardians/0.20.0/guardians-sdk.module.js"
    }
  }
  </script>

  Match the three version to the SDK release's build (0.178.0 for 0.20.0) — a mismatched three breaks retargeting in ways that look like broken assets. A module game needs nothing else; a classic-script game (PortalsGuardians.ready()) additionally downloads the pair into the local _portals/ directory beside sdk.js (the loader resolves the module relative to its own URL), and _portals/ is dropped from every push:

  mkdir -p _portals
  curl -o _portals/guardians-sdk.js https://portals.to/portals-sdk/guardians/0.20.0/guardians-sdk.js
  curl -o _portals/guardians-sdk.module.js https://portals.to/portals-sdk/guardians/0.20.0/guardians-sdk.module.js

2. Direct CDN assets while local. The downloaded SDK is the sandbox build: it rewrites Guardian model/wearable/clip URLs onto same-origin /_portals/cdn/... paths that only exist on Portals. Switch that off when the page is not on Portals — the guard is what makes it push-safe:

  import { setAssetRewriting } from '@portals/avatars';
  if (['localhost', '127.0.0.1'].includes(location.hostname)) setAssetRewriting('never');

  The Portals CDN allows cross-origin reads, so models and clips then load directly. (Classic-script games do the same via the PortalsGuardians global after ready().)

3. The same version in portals.json, so the push runs exactly the release that was tested.

## Testing multiplayer on a local dev server

A game served from your own machine has no host page, so Portals.net is unavailable by default. To develop multiplayer locally, call create_dev_multiplayer_token for the game and hardcode the token it returns into the game's code — declared on window before the SDK loads:

  <script>
    window.__PORTALS_DEV__ = { token: "pdev_..." };
  </script>
  <script src="./_portals/sdk.js"></script>

Pasting the token in as a literal is the intended setup; it needs no env var or config plumbing. net.join(), messages, shared state, and events then work exactly as on Portals — open two tabs and each sees the other join. Serving locally also means fetching the managed SDK yourself (a pull does not include it): mkdir -p _portals && curl -o _portals/sdk.js https://portals.to/portals-sdk/sdk.js

Local sessions join a "dev" channel namespace (join({ channel: "lobby" }) becomes dev:lobby), so a half-built local build never meets the published game's live players. The token lasts about 8 hours; when it expires join() rejects saying so, and a new one can be minted. Remove the window.__PORTALS_DEV__ block before committing or pushing — a published game's source is readable, and the token authorizes multiplayer sessions on the game until it expires. Voice, saves, leaderboards, and identity still need the Portals host page and keep their documented no-host behavior locally.

## Generated assets — art, audio, and 3D models

The AI Lab tools make assets for the game: generate_ai_image (sprites, backgrounds, UI, icons — transparent=true for a PNG with alpha), generate_ai_texture (seamless tiling material, the cheapest at 3 credits), text_to_3d_model and image_to_3d_model (GLB, 1–5 minutes; keep calling check_3d_model_task until it answers with a file, since polling is what finalizes the model), text_to_speech, generate_sound_effect and generate_music (MP3, synchronous). Each takes an outputPath and writes the finished file into the game's own directory; list_generated_assets and save_generated_asset reuse or collect an earlier generation.

Every generated asset must ship inside the bundle and be referenced by a path relative to index.html:

  <img src="./assets/ship.png">        // yes
  <img src="https://cdn.theportal.to/ai-images/...">  // blocked on Portals

A published game is served under img-src 'self' / connect-src 'self' / media-src 'self', so the CDN URL a generator returns loads on a local dev server and then fails on Portals — the exact bug that looks like "it worked yesterday". The tools save the file for this reason; keep it in the pushed directory and never fetch an asset URL at runtime. Generated files count against the bundle limits (500 files, 50 MB per file, 100 MB total), so mind the size of music tracks and hero models.

Generation costs credits and cannot be undone, so prefer reusing what is already there: check list_generated_assets before generating another variant, and re-prompt rather than regenerate in a loop. A texture is 3 credits, an image 20, a sound effect 20, speech 45 per 1,000 characters, music 70 per minute, a 3D model 60–75.

Full docs: https://portals.to/documentation/advanced-tooling/portals-sdk, https://portals.to/documentation/advanced-tooling/multiplayer-and-voice and https://portals.to/documentation/advanced-tooling/guardian-avatars
