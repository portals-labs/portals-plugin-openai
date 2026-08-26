# Pose Tuner

Source: [https://portals.to/documentation/web-games/pose-tuner](https://portals.to/documentation/web-games/pose-tuner) — the official Portals documentation. The bone editor's `boneAdjust` export and its post-mixer ordering rule belong to the Guardian avatar SDK; see [Guardian Avatars](https://portals.to/documentation/web-games/guardian-avatars) for that side.

The Pose Tuner is an in-game panel for tuning procedural animation: the number tables your pose code reads every frame, the live inputs that drive them, and — through a 3D gizmo — bone rotations, bone positions and the placement of a held weapon. You tune in the real renderer, on the real rig, and export exactly what changed as a paste-ready snippet. Your committed source stays the source of truth; the panel is never a data format.

It is a **dev tool, not part of the game runtime**:

- **Draft previews** get it stamped at `_portals/dev/pose-tuner.js`, on the game's own origin, so the strict game CSP is satisfied.
- **Published bundles never include it.** Portals strips the whole `_portals/dev/` prefix at publish, so a game that lazy-imports it behind a dev flag works in preview and quietly no-ops in production.
- **Local development:** download the same bytes from [portals.to/portals-sdk/pose-tuner.js](https://portals.to/portals-sdk/pose-tuner.js) into your project's `_portals/dev/` directory. Files under `_portals/` are managed by Portals and never upload, so the local copy cannot drift into your bundle.

```js
// Behind a flag only you use — absent in production, so the import rejects
// and the game plays on untouched.
if (new URLSearchParams(location.search).has('poseTuner')) {
  try {
    const { createPoseTuner } = await import('./_portals/dev/pose-tuner.js');
    tuner = createPoseTuner({ title: 'My game', storageKey: 'my-game-tuner' });
  } catch {
    /* published build — no dev tools */
  }
}
```

## Why a panel instead of an animation editor

A procedural pose is a pure function of *numbers* (carry tables, grip offsets, recoil shares) plus *inputs* (pitch, kick, flags). No authored clip can be "the" gun-holding animation, because the hold has to survive things clips cannot know: camera pitch, strafe direction, recoil, scope state. So the tool binds the live objects your frame loop already reads — a slider mutates the table in place and the very next frame shows the result — and forces the inputs so the failure states are one click instead of a play session.

## Wiring

Three touch points, each a no-op when the tuner is off:

```js
// 1. Bind the live tables your pose code reads (nested objects and numeric
//    arrays are walked; per-key ranges optional).
tuner.table('HEAVY_CARRY', HEAVY_CARRY, { tuck: { min: -1, max: 1 } });

// 2. Name forced inputs after your pose function's argument keys, then merge
//    them over the real inputs each frame.
tuner.control('pitch', { min: -1.35, max: 1.35, step: 0.01 });
tuner.control('scoped', { type: 'toggle' });
aimWeapon(avatar, tuner.apply({ pose, lookYaw, pitch, kick, scoped }));

// 3. Route the frame delta through pause / single-step / slow-mo.
const dt = tuner.dt(rawDt);
```

Written with optional chaining they can stay in the shipped source unchanged:

```js
tuner?.table('HEAVY_CARRY', HEAVY_CARRY);
aimWeapon(avatar, tuner?.apply(inputs) ?? inputs);
const dt = tuner?.dt(rawDt) ?? rawDt;
```

## What the game must add while the tuner is on

The panel is a DOM overlay, so touching it means leaving pointer lock. Two defaults of a normal third-person game turn that into a dead tuning session.

### Suppress every pause screen

A game that pauses on pointer-lock exit, window blur, or Escape will cover the viewport and stop the loop the moment you reach for a slider — which is the exact frame you were trying to look at. While the tuner exists, keep rendering and skip the overlay entirely:

```js
function shouldPause() {
  if (tuner) return false;                 // tuning: never pause, never overlay
  return !document.pointerLockElement;
}
```

Do not treat a click back onto the canvas as "resume" either. While bone-edit mode owns the canvas a left-click is a bone pick, a right-drag is an orbit, and the wheel zooms — `tuner.capturing` reports exactly that window:

```js
if (!tuner?.capturing) resumeFromPause();
```

### Add a free camera

A pose is judged from angles gameplay never gives you: down the barrel, from the side, behind the shoulder, under the chin. Bind a key that detaches the camera from the character controller and flies it — WASD plus drag-look, with the character frozen in whichever stance you activated. Call `controller.setEnabled(false)` while free cam is on, so WASD flies the camera instead of walking the character out of the pose you are tuning.

Hand the same rig to the bone editor's `orbit` callbacks so right-drag and wheel keep working once edit mode owns the canvas:

```js
orbit: { look: (dx, dy) => freeCam.look(dx, dy), zoom: (dy) => freeCam.dolly(dy) },
```

## Editing bones and weapons in 3D

`attachBoneEditor` adds direct manipulation: pick a stance (activation goes through your game's real code paths), then click a joint and drag a local-space gizmo. Edit mode grows a **clickable sphere on every bone** (green on registered targets, orange on the selection) so the pick targets are visible, and the skeleton overlay shows by default. Pass `defaultOn: true` to start with edit mode already enabled — note the canvas then belongs to the editor from the first frame, so give the game another way into pointer lock (an action button calling `requestPointerLock()`). A button switches the gizmo between **rotation** and **position**; holding **Shift** hides it so you can click straight through to the next joint; right-drag orbits and the wheel zooms via callbacks you supply.

```js
tuner.attachBoneEditor({
  THREE, scene, camera, domElement: renderer.domElement,
  getBones: () => avatar.bones,
  poses: [
    { id: 'hipPose', label: 'hip', activate: () => holdHipStance() },
    { id: 'adsPose', label: 'ads', activate: () => setAds(true) },
  ],
  loadTransformControls: () =>
    import('three/addons/controls/TransformControls.js').then((m) => m.TransformControls),
  orbit: { look: (dx, dy) => freeCam.look(dx, dy), zoom: (dy) => freeCam.dolly(dy) },
});
```

`three/addons/controls/TransformControls.js` is part of the managed Three.js runtime from Guardian SDK 0.42.0, so the import resolves in hosted previews and in local projects that use the managed import map. On an older pin the import fails, the editor falls back to sliders, and only a `console.warn` says so.

- **Bone rotations** are stored per pose as Euler XYZ radians and exported as a `boneAdjust` map — exactly the shape [`avatar.animations.load`](https://portals.to/documentation/web-games/guardian-avatars) takes at clip registration, and applied with the same math, so what you see while dragging is what the baked clip will look like.
- **Bone positions** are local offsets over the clip, exported separately (`boneAdjust` cannot bake positions — apply them at runtime).
- **Non-bone targets** — a held weapon's mount, a prop — register as `targets`: the gizmo attaches in translate or rotate mode and every drag calls your `fold(object, poseId, mode)` to convert the dragged transform back into whatever live table your game poses it from. See below.

### Register every gun and pickable object

Bones are only half of a hold. A weapon's look is its **mount transform** — where it sits in the hand and how it is angled — as much as the arm pose behind it, and the same goes for anything the character picks up: a torch, a tool, a thrown object, a carried crate. Those transforms live in your own tables rather than on the skeleton, so the gizmo reaches them through `targets`. Register every weapon and pickable prop the game can hold; one left out is one whose numbers still have to be guessed by hand.

```js
targets: [
  { id: 'rifleMount', label: 'rifle', mode: 'translate',
    getObject: () => heldWeapon?.model,
    fold: (obj, poseId) => { MOUNTS[poseId].pos.copy(obj.position); } },
  { id: 'rifleAngle', label: 'rifle angle', mode: 'rotate',
    getObject: () => heldWeapon?.model,
    fold: (obj, poseId) => { MOUNTS[poseId].rot.copy(obj.rotation); } },
  { id: 'torch', label: 'torch', mode: 'translate',
    getObject: () => props.torch,
    fold: (obj, poseId) => { PROP_MOUNTS.torch[poseId].copy(obj.position); } },
]
```

Each entry is `{ id, label, mode: 'translate' | 'rotate', getObject(), fold(object, poseId, mode) }`, selected in the picker as `@id`. Four rules make them behave:

- **`getObject()` is called fresh on every pick and every frame — return the current object, never a captured reference.** Held items get swapped, re-parented and destroyed; a stale handle edits a weapon that is no longer in the hand. Returning `undefined` when nothing is held is fine — the target simply is not pickable then.
- **`fold` writes your table; it does not keep the drag.** The tuner deliberately lets your code re-apply that table on the next frame — the same trick the bone edits use, and what keeps a drag stable instead of fighting your pose code. A `fold` that mutates the object without writing anything back snaps to nothing as soon as the game re-poses.
- **A target declares a single `mode`.** To move *and* rotate the same object, register two entries against the same `getObject`, as above. `mode` is also passed to `fold` as its third argument.
- **Bind the table it folds into with `tuner.table()` as well.** Sliders bound that way are refreshed after every fold, so the gizmo and the numbers stay one view of one value, and the drag shows up in `Copy changes` with everything else.

`poseId` is whichever stance is active, so a single target covers the hip mount, the ADS mount and every other pose — as long as it folds into a per-pose table.

Run the editor's per-frame pass in the post-mixer slot. On Guardian SDK 0.42.0 and later, register it once and the ordering is the SDK's promise:

```js
avatars.onAfterUpdate(() => tuner.boneEditor?.update());
```

On older pinned versions, call it in your frame loop directly after `avatars.update(dt)` and before your own bone writers (procedural aim, IK, ragdolls) — anything that writes bones before the mixer pass is silently overwritten.

## The panel

| Section | What it does |
| --- | --- |
| Force state | Sliders/toggles that override pose inputs; one-click corner-state snapshots (pitch extremes, strafe diagonals, scoped raises) and *release all*. |
| Time | Pause, single-step, slow-motion — via `tuner.dt()`. |
| Actions | Buttons wired to your game's real paths (fire once, ADS on/off, cycle weapon). |
| Tables | One collapsible section per bound table, per-row reset dots, persisted across reloads under `storageKey`. |
| Bones | Pose buttons, bone/target picker, gizmo mode button, rot/pos rows, `boneAdjust` export. |
| Inspect | Skeleton overlay drawn through the mesh. |
| Undo / redo | ↶/↷ in the header, plus Ctrl/⌘ Z and Ctrl/⌘ ⇧Z. Covers everything that edits data — table sliders, bone edits, gizmo drags (one entry per drag), resets. Forced inputs are session state and stay out. |
| Copy changes / Reset all | Export a `TABLE.key = value;` diff of everything that differs from the code's values; restore the code's values and clear persistence. |

Export diffs are always measured against what the *code* held at bind time — persisted slider edits from a previous session still show as changed, so nothing tuned can silently fail to make it back into your source.

## Caveats

- Keystrokes inside the panel are kept from the game where possible, but a capture-phase listener registered before the tuner mounts (the Guardian controller's input is one) still sees them first. Pause time while typing in a number field, or use the sliders.
- `tuner.dt()` pausing stops what you step with that delta; eases timed off wall-clock keep moving. Force their inputs instead (that is what a `kick` control is for).
- Bones a later procedural pass rewrites (an IK'd arm, an aim-bent spine) show that pass, not your edit — tune those through their tables, or edit them under a stance the pass leaves alone.
