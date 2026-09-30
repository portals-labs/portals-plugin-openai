# Ability casting & world indicators in a Portals game

Targeting reticles, cast telegraphs and "something is here" world markers,
built the way a Portals hosted game can actually ship them: procedural
geometry, inline GLSL, no external fetches, `three` left external.

The architecture below is distilled from
[`achrefelouafi/LinearAbiltyCastingThreeJS`](https://github.com/achrefelouafi/LinearAbiltyCastingThreeJS),
a Three.js skillshot sandbox with five elemental abilities (Frost Lance,
Storm Lance, Cinder Fall, Nova Beam, Voltaic Snare), a League-of-Legends-style
aim system and 938 live parameters.

> **It is a sandbox, not a dependency.** There is no npm package, its assets
> are FBX/HDR files under `public/`, its editor is `lil-gui`, and its licence
> reads "code is provided as-is for the purposes of this project". Do not
> `npm install` it, do not vendor its source into a game bundle. Take the
> patterns — they are the valuable part — and write them against the
> constraints in [Sandbox rules](#sandbox-rules).

---

## The one idea worth stealing: no dimensions on the CPU

The sandbox's headline property is that every one of its 938 parameters stays
live — you can drag a slider *while the game is paused* and an in-flight
lightning bolt re-kinks. That is not a UI trick. It falls out of a single
authoring rule:

**Nothing stores a world-space dimension. Effects store fractions, and resolve
them against a settings object every frame.**

```js
// config/settings.js — the single source of truth, a plain mutable object.
export const settings = {
  global: { timeScale: 1 },
  zone:   { radius: 3.2, rimWidth: 0.12, pillarHeight: 2.4, spin: 0.35 },
};
```

```js
// WRONG — the radius is baked at spawn. Changing settings.zone.radius later
// does nothing to anything already on screen, and every effect needs a
// rebuild-on-change path.
class ZoneMarker {
  constructor(pos) {
    this.mesh = new THREE.Mesh(new THREE.RingGeometry(
      settings.zone.radius * 0.9, settings.zone.radius, 64));
  }
}

// RIGHT — geometry is a unit primitive; the fraction lives on the instance;
// the dimension is resolved in update(). settings.zone.radius is read fresh
// every frame, so it is editable, tweenable and per-frame animatable for free.
class ZoneMarker {
  constructor(pos) {
    this.growth = 0;                                    // 0..1, a fraction
    this.mesh = new THREE.Mesh(UNIT_DISC, this.material); // radius 1
    this.mesh.position.copy(pos);
  }
  update(dt) {
    this.growth = Math.min(1, this.growth + dt * 4);
    const r = settings.zone.radius * ease(this.growth);
    this.mesh.scale.set(r, 1, r);
  }
}
```

Everything else in this document assumes that rule. Follow it and live tuning,
slow-motion (`settings.global.timeScale`), difficulty scaling and
accessibility toggles all come free; break it and each one becomes a feature.

**Corollary — drive every clock through the scale.** One `Time` object owns
`dt`, multiplies by `settings.global.timeScale`, and hands the result to every
`update()`. Nothing reads `performance.now()` directly, so pause is
`timeScale = 0` and a hit-stop is `timeScale = 0.05` for 80 ms.

---

## The three indicator shapes

Nearly every ability, quest marker or point of interest is one of these. The
sandbox ships the first two; the third is what a world marker needs.

| Shape | Sandbox name | Use for |
|---|---|---|
| **Line** — arrow swinging from the caster | `AimIndicator` | Skillshots, casts aimed by direction: "which way?" |
| **Zone** — circle placed under the cursor, range-clamped | `ZoneIndicator` | Ground-targeted casts, teleports: "where?" |
| **Beacon** — ring + pillar at a fixed world point | *(not in the sandbox)* | Quest markers, secret spots, loot: "there." |

### Line indicator — one SDF quad

The whole arrow is a **single quad** with the shape drawn in the fragment
shader as a signed-distance field. Not a mesh, not a texture — an SDF, because
it stays crisp at any length and the shaft/head proportions become uniforms.

```js
const aimMaterial = new THREE.ShaderMaterial({
  transparent: true, depthWrite: false, depthTest: false,
  blending: THREE.AdditiveBlending, side: THREE.DoubleSide,
  uniforms: {
    uColor:  { value: new THREE.Color('#7fd4ff') },
    uLen:    { value: 1 },      // quad length / quad width — keeps the head square
    uHead:   { value: 0.22 },   // head length, as a fraction of total
    uEdge:   { value: 0.035 },  // feather, in local units
    uTime:   { value: 0 },
    uCharge: { value: 0 },      // 0..1 — fills the shaft as the cast charges
  },
  vertexShader: /* glsl */`
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }`,
  fragmentShader: /* glsl */`
    uniform vec3 uColor; uniform float uLen, uHead, uEdge, uTime, uCharge;
    varying vec2 vUv;

    // Distance to an arrow lying along +y in a unit-width strip.
    float arrow(vec2 p) {
      float shaftEnd = 1.0 - uHead;
      // shaft: a capsule from y=0 to y=shaftEnd, half-width 0.12
      vec2 s = vec2(p.x, p.y - clamp(p.y, 0.0, shaftEnd));
      float d = length(s) - 0.12;
      // head: a triangle from shaftEnd to the tip
      float t = clamp((p.y - shaftEnd) / uHead, 0.0, 1.0);
      float halfW = mix(0.30, 0.0, t);
      float inHead = max(abs(p.x) - halfW, max(shaftEnd - p.y, p.y - 1.0));
      return min(d, inHead);
    }

    void main() {
      vec2 p = vec2((vUv.x - 0.5) * 1.0, vUv.y);   // x normalised to the strip
      float d = arrow(p);
      float a = 1.0 - smoothstep(-uEdge, uEdge, d);
      // charge wipe + a travelling highlight so a held cast reads as alive
      float fill = step(vUv.y, uCharge);
      float pulse = 0.6 + 0.4 * sin(uTime * 6.0 - vUv.y * 12.0);
      a *= mix(0.35, 1.0, fill) * pulse;
      if (a < 0.01) discard;
      gl_FragColor = vec4(uColor, a);
    }`,
});
```

Place it flat on the ground, one end at the caster:

```js
const quad = new THREE.Mesh(
  new THREE.PlaneGeometry(1, 1).translate(0, 0.5, 0).rotateX(-Math.PI / 2),
  aimMaterial,
);
quad.renderOrder = 3;

function aimAt(origin, dir, length) {
  quad.position.copy(origin).y += 0.02;          // above the ground plane
  quad.rotation.y = Math.atan2(dir.x, dir.z);
  quad.scale.set(length * 0.35, 1, length);      // width follows length
  aimMaterial.uniforms.uLen.value = length / (length * 0.35);
}
```

**Depth stance.** `depthTest: false` + `renderOrder` keeps the reticle visible
over uneven terrain, which is what players expect from a MOBA-style indicator.
If it must be occluded by walls instead, turn depth test back on and lift the
quad ~2 cm; z-fighting against a ground plane is the usual first bug.

### Zone indicator — the circle, and its snap-out

The zone circle is a disc + rim, clamped to the caster's max range. The
sandbox's touch worth copying is the **snap-out**: when the cursor pushes past
max range the circle does not stop dead at the boundary, it eases toward the
clamped point with a short spring. Hard clamping reads as a bug; the ease reads
as the ability resisting.

```js
const desired = raycastGround(pointer);                 // where the cursor is
const offset  = desired.clone().sub(casterPos);
const range   = settings.zone.maxRange;
if (offset.length() > range) offset.setLength(range);   // clamp
target.copy(casterPos).add(offset);
// spring toward the clamped target instead of teleporting to it
zone.position.lerp(target, 1 - Math.exp(-settings.zone.snap * dt));
```

`1 - Math.exp(-k * dt)` rather than a bare `lerp(a, b, 0.2)`: the exponential
form is frame-rate independent, so the feel is identical at 30 and 144 fps.
Use it for every smoothing term in the game.

### Beacon — a world marker for a point of interest

Ring on the water/ground, soft pillar above it, optional tendrils. Three draw
calls, no textures, visible from across a level.

```js
// Ring: a unit disc whose shader draws an animated annulus.
const ringMat = new THREE.ShaderMaterial({
  transparent: true, depthWrite: false, blending: THREE.AdditiveBlending,
  side: THREE.DoubleSide,
  uniforms: {
    uColor: { value: new THREE.Color('#ffd166') },
    uTime:  { value: 0 },
    uFade:  { value: 1 },   // 0..1 master alpha — the only "is it dying" state
  },
  vertexShader: /* glsl */`
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }`,
  fragmentShader: /* glsl */`
    uniform vec3 uColor; uniform float uTime, uFade;
    varying vec2 vUv;
    void main() {
      float r = length(vUv - 0.5) * 2.0;                  // 0 centre, 1 rim
      float rim  = smoothstep(0.82, 0.94, r) * (1.0 - smoothstep(0.97, 1.0, r));
      float wave = smoothstep(0.0, 0.5, fract(r * 2.0 - uTime * 0.5)) * 0.25;
      float a = (rim + wave * (1.0 - r)) * uFade;
      if (a < 0.01) discard;
      gl_FragColor = vec4(uColor, a);
    }`,
});
const ring = new THREE.Mesh(
  new THREE.CircleGeometry(1, 64).rotateX(-Math.PI / 2), ringMat);

// Pillar: an open-ended cylinder, brightest at the base, gone by the top.
const pillarMat = new THREE.ShaderMaterial({
  transparent: true, depthWrite: false, blending: THREE.AdditiveBlending,
  side: THREE.DoubleSide,
  uniforms: { uColor: ringMat.uniforms.uColor, uTime: ringMat.uniforms.uTime,
              uFade: ringMat.uniforms.uFade },
  vertexShader: /* glsl */`
    varying vec2 vUv;
    void main() {
      vUv = uv;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }`,
  fragmentShader: /* glsl */`
    uniform vec3 uColor; uniform float uTime, uFade;
    varying vec2 vUv;
    void main() {
      float up   = 1.0 - vUv.y;                             // fade with height
      float band = 0.5 + 0.5 * sin(vUv.y * 18.0 - uTime * 2.4);
      float a = up * up * (0.25 + 0.35 * band) * uFade;
      if (a < 0.01) discard;
      gl_FragColor = vec4(uColor, a);
    }`,
});
const pillar = new THREE.Mesh(
  new THREE.CylinderGeometry(1, 1, 1, 32, 1, true).translate(0, 0.5, 0),
  pillarMat);
```

Share the uniform *objects* (`uColor`, `uTime`, `uFade`) between the two
materials, as above: one write per frame drives the whole beacon, and fading
the marker out is a single assignment.

```js
function updateBeacon(dt) {
  ringMat.uniforms.uTime.value += dt * settings.global.timeScale;
  const r = settings.beacon.radius * ease(growth);
  ring.scale.set(r, 1, r);
  pillar.scale.set(r * 0.8, settings.beacon.pillarHeight * growth, r * 0.8);
}
```

---

## Pooling and draw-call budgets

The sandbox caps concurrent abilities at **4** and publishes a per-effect draw
budget. Doing the same is the difference between an effect that ships and one
that halves the frame rate on a mid-range phone.

| Effect | Draws | How |
|---|---|---|
| Ice field, up to 288 crystals | 3 | `InstancedMesh` per crystal LOD |
| Lightning, 24 filaments × 72 samples | 2 | one geometry, offsets in an attribute |
| Snare zone (cage, field, circle) | 4 | shared uniforms, no per-instance material |
| Beam (tube, coils, discs, charge) | 6 | |

Two rules get you there:

1. **One `InstancedMesh` per visual kind, allocated once at load, never
   resized.** Per-instance variation rides on `instanceMatrix` and
   `instanceColor`; a dead instance is a black one (with additive blending,
   fading a colour to black *is* the fade), not a removed one.
2. **Ring-buffer the pool.** Spawning takes the oldest slot rather than
   searching for a free one — bounded work, no allocation, and the visual cost
   of stomping a still-living effect at full saturation is imperceptible.

```js
const POOL = 24;
const mesh = new THREE.InstancedMesh(geo, mat, POOL);
mesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);
mesh.instanceColor = new THREE.InstancedBufferAttribute(new Float32Array(POOL * 3), 3);
let next = 0;
const spawn = (pos) => { const i = next; next = (next + 1) % POOL; write(i, pos); };
```

**Set bounding volumes by hand.** Three.js derives an `InstancedMesh`'s
bounding sphere exactly once and then caches it — with matrices rewritten every
frame that cache goes stale and the whole pool gets frustum-culled at the worst
moment. Assign `geometry.boundingSphere` and `mesh.boundingSphere` explicitly,
covering the whole area the pool can reach.

---

## Sandbox rules

What the reference project does freely and a Portals hosted game cannot.

**`three` and `three/addons/*` stay external.** They resolve through the
managed import map to the one vendored runtime. Never bundle them — two copies
of Three.js in a page break `instanceof` everywhere, silently.

```html
<script src="./_portals/sdk.js"></script>
<script type="importmap">{"imports":{"three":"…managed…"}}</script>
```

**No external network.** Published games run under a strict CSP; `connect-src`
is effectively `'self'`. Every asset the reference project loads —
`Idle.fbx`, `cast1.fbx`, `diffuse.png`, `spruit_sunrise.hdr` — is a fetch that
will not happen. This is why the indicators above are **procedural**:
`CircleGeometry`, `CylinderGeometry`, `PlaneGeometry` plus a fragment shader
cost nothing to download and cannot 404. When a texture is genuinely needed,
draw it into a `<canvas>` at load and wrap it in `THREE.CanvasTexture`.

**No CDN tooling.** `lil-gui` from a CDN is blocked. If a game wants live
tuning, expose the settings object on `window` behind a `?debug=1` query flag
and tune from the console — same benefit, zero bytes.

**No secrets, no client-authoritative rewards.** An ability's visual state is
client-side and trivially forged. Never let a VFX callback grant currency,
items or entitlements; route anything valuable through the session's server
script or a server-verified action.

**Colour space.** These materials are `ShaderMaterial`s, so their output is
linear and passes through tone mapping unchanged. Author the emissive colours
brighter than they look — additive blending over a bright scene needs headroom.
If a marker vanishes against water or snow, the fix is usually additive
blending plus `depthWrite: false`, not more opacity.

**Dispose on teardown.** Geometries, materials and canvas textures created per
effect must be disposed when the effect (or the room) goes away. Pool
everything you can precisely so there is nothing left to dispose per cast.

---

## Checklist for a new effect

- [ ] Geometry is a **unit** primitive; every dimension is a fraction resolved
      in `update()` against a settings object.
- [ ] `dt` arrives pre-multiplied by `settings.global.timeScale`; nothing reads
      the wall clock.
- [ ] Smoothing uses `1 - Math.exp(-k * dt)`, not a fixed lerp factor.
- [ ] Instances come from a fixed-size ring-buffer pool allocated at load.
- [ ] `geometry.boundingSphere` and `mesh.boundingSphere` are set explicitly.
- [ ] `transparent: true`, `depthWrite: false`, an explicit `renderOrder`, and
      a deliberate `depthTest` decision.
- [ ] Zero network fetches; textures, if any, are canvas-drawn at load.
- [ ] Shared uniform objects across the effect's materials, so one write per
      frame drives all of it.
- [ ] A `dispose()` that releases every geometry, material and texture it made.
