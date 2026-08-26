---
name: portals-pose-tuner
description: Tune procedural animation live inside a Portals web game with the Pose Tuner dev tool — carry tables, grip offsets, recoil shares, forced pose inputs, and a 3D gizmo for bone rotations, bone positions and held-weapon placement, exported as a paste-ready diff or a boneAdjust map. Use when tuning aimWeapon numbers, weapon carry or ADS poses, boneAdjust maps, ragdoll or IK constants, or any hand-tuned animation table; when a game needs createPoseTuner, _portals/dev/pose-tuner.js, attachBoneEditor, tuner.apply, tuner.dt, tuner.table, or avatars.onAfterUpdate; and when adding the free-cam and no-pause-screen support a tuning session requires.
---

# Pose Tuner

A procedural pose is a pure function of *numbers* (carry tables, grip offsets, recoil shares) and *inputs* (pitch, kick, scoped). No authored clip can be "the" gun-holding animation, because the hold has to survive things a clip cannot know — camera pitch, strafe direction, recoil, scope state. Those numbers get found by running the game and changing them, so the Pose Tuner puts a panel in the running game that binds the live tables the frame loop already reads: a slider mutates the table in place and the very next frame shows it.

Read [references/pose-tuner.md](references/pose-tuner.md) for the full API, the panel reference, and the caveats. It pairs with `$portals-guardian-avatars` — bone editing targets a Guardian rig, and its export is the `boneAdjust` shape `avatar.animations.load` takes.

## It is a dev tool, not runtime

Portals stamps it into **drafts** at `_portals/dev/pose-tuner.js` (same origin, so the strict game CSP is satisfied) and strips the whole `_portals/dev/` prefix out of **published** bundles. So the import is gated and its rejection caught, or the published game throws on load:

```js
if (new URLSearchParams(location.search).has('poseTuner')) {
  try {
    const { createPoseTuner } = await import('./_portals/dev/pose-tuner.js');
    tuner = createPoseTuner({ title: 'My game', storageKey: 'my-game-tuner' });
  } catch { /* published build — no dev tools */ }
}
```

For local development, download the same bytes from `https://portals.to/portals-sdk/pose-tuner.js` into the project's `_portals/dev/` directory. Files under `_portals/` are managed by Portals and never upload, so the local copy cannot drift into the bundle.

## Wiring

Every touch point no-ops when `tuner` is undefined, so these lines can stay in the shipped source:

```js
tuner?.table('HEAVY_CARRY', HEAVY_CARRY);            // bind the live table, mutated in place
aimWeapon(avatar, tuner?.apply(inputs) ?? inputs);   // forced inputs override pitch/kick/scoped
const dt = tuner?.dt(rawDt) ?? rawDt;                // pause / single-step / slow-mo
```

`attachBoneEditor` adds a 3D gizmo for bone rotations, bone positions and held-weapon placement. Its per-frame pass writes bones, so it obeys the same ordering rule as `Ragdoll` and `aimWeapon` — it must run after the mixer tick or be silently overwritten. On Guardian SDK 0.42.0+ register it and the ordering is the SDK's promise:

```js
avatars.onAfterUpdate(() => tuner?.boneEditor?.update());
```

## Two things the game MUST add while the tuner is on

Both follow from the panel being a DOM overlay: reaching for a slider means leaving pointer lock, and the default behaviour of a normal third-person game turns that into a dead session.

**1. Suppress every pause screen.** A game that pauses on pointer-lock exit, window blur, or Escape covers the viewport and stops the loop the moment you touch the panel — which is the exact frame you were trying to judge. While the tuner exists, keep rendering and skip the overlay entirely. Do not treat a click back onto the canvas as "resume" either; in edit mode that click is a bone pick, which is what `tuner.capturing` reports.

```js
function shouldPause() {
  if (tuner) return false;                 // tuning: never pause, never overlay
  return !document.pointerLockElement;
}
if (!tuner?.capturing) resumeFromPause();  // canvas click — a pick, not a resume
```

**2. Add a free camera.** A pose is judged from angles gameplay never gives you — down the barrel, from the side, behind the shoulder, under the chin. Bind a key that detaches the camera from the character controller and flies it (WASD + drag-look, character frozen in the stance you activated), call `controller.setEnabled(false)` while it is on so WASD flies instead of walking the character out of the pose, and hand the same rig to the bone editor's `orbit` callbacks so right-drag and wheel keep working once edit mode owns the canvas:

```js
tuner.attachBoneEditor({
  /* … */
  orbit: { look: (dx, dy) => freeCam.look(dx, dy), zoom: (dy) => freeCam.dolly(dy) },
});
```

## Rules

- The panel is a tuning surface, never a data format. What it produces is a paste-ready diff for the committed source; the committed source stays the source of truth.
- Export diffs are measured against what the code held at bind time, so a persisted edit from an earlier session still reads as changed — nothing tuned can silently fail to make it back.
- Bone rotations export as `boneAdjust` and paste straight into clip registration. Bone **positions** do not — `boneAdjust` cannot bake them, so apply them at runtime.
- The gizmo needs `three/addons/controls/TransformControls.js`, a hosted addon from Guardian SDK 0.42.0. On an older pin the import fails, the editor falls back to sliders, and only a `console.warn` says so.
- Bones a later procedural pass rewrites (an IK'd arm, an aim-bent spine) show that pass, not the edit. Tune those through their tables instead.
