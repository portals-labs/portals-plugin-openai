# Portals Wearable Standard

Source: [https://portals.to/documentation/web-games/wearable-standard](https://portals.to/documentation/web-games/wearable-standard) — the official Portals documentation, copied verbatim with its site-relative links pointed at portals.to.

**Version 1.** This page is the contract between a wearable file and every place a Portals avatar appears:
- `/avatar`;
- the Shop;
- the Play lobby;
- hosted games;
- games built with the Portals engine.

A wearable that follows it behaves the same way on all of them, with no code in any game. For the rig itself — bone names, the template, and skinned versus rigid attachment — see [Wearable standards](https://portals.to/wearable-standards).

| Part | Status |
| --- | --- |
| File rules, attachment forms, budgets and upload validation | Supported now |
| Separate static and dynamic files | In development. Until then, the validated upload is the one file every runtime loads |
| Extra bones on skinned wearables (capes, tails) | In development. Uploads are accepted; until the static file ships, runtimes before the release that enables them fold these bones onto the hips |
| Named animation clips with triggers, material animation | In development |
| Secondary motion (spring bones: capes, hair, tails) | In development |
| Particle emitters and trails | In development |

Everything marked in development is specified here so files can be authored against it. Each part is enabled by the Guardian SDK release named in the [changelog](https://portals.to/documentation/web-games/guardian-avatars-changelog).

## Principles

1. **A wearable is data, never code.** The GLB describes what should happen; the Guardian SDK does it. Wearables may not contain:
   - scripts;
   - custom shaders;
   - external file references;
   - lights or cameras.
2. **The file describes itself.** Everything a wearable does is declared inside its GLB. Any runtime handed the file gets the full behaviour, with no side-channel flags.
3. **Standards first.** Motion uses core glTF animation. Secondary motion uses [`VRMC_springBone`](https://github.com/vrm-c/vrm-specification/tree/master/specification/VRMC_springBone-1.0). Material animation uses a subset of [`KHR_animation_pointer`](https://github.com/KhronosGroup/glTF/tree/main/extensions/2.0/Khronos/KHR_animation_pointer). One Portals extension, `PORTALS_wearable`, covers only what no standard does: clip triggers, particle emitters and trails.
4. **It must look right standing still.** With every dynamic feature off, a wearable must look correct in its rest pose. That is how it appears:
   - in thumbnails;
   - to players who ask for reduced motion;
   - to distant or over-budget avatars;
   - in games pinned to an older SDK.
5. **The platform owns the budget.** Limits are checked at upload *and* enforced by the SDK at runtime. A wearable renders on other players' devices, so no file is trusted to stay within budget.

## File rules

- **Format.** One self-contained binary glTF 2.0 file (`.glb`). Every buffer and image is embedded, and there are no external URIs. The file must pass the [Khronos glTF Validator](https://github.khronos.org/glTF-Validator/) without errors.
- **Textures.** PNG or JPEG.
- **Rig.** The full Portals rig, attached as a skinned mesh or as rigid parts under rig bones ([Wearable standards](https://portals.to/wearable-standards)).
- **Extensions.** `extensionsRequired` may list only `KHR_mesh_quantization`, a geometry encoding every Portals runtime decodes. A file that requires any other extension is rejected, because an older runtime could not show it at all.
- **Extensions a wearable may use:**

  | Extension | Use |
  | --- | --- |
  | `KHR_texture_transform` | UV offset, scale and rotation |
  | `KHR_mesh_quantization` | Compact vertex data |
  | `KHR_materials_emissive_strength` | Glow brighter than 1.0 |
  | `KHR_materials_unlit` | Flat, unlit materials |
  | `KHR_materials_specular`, `KHR_materials_ior`, `KHR_materials_clearcoat`, `KHR_materials_transmission` | Richer surfaces |
  | `KHR_animation_pointer` | Material animation (allowlisted properties, below) |
  | `VRMC_springBone` | Secondary motion |
  | `PORTALS_wearable` | Clip triggers, emitters, trails |

  Any other extension is removed at upload and reported to the creator. Lights (`KHR_lights_punctual`) and cameras are removed too.
- **Animation moves only the wearable.** A channel that animates a rig bone, or a node above one, would pose the avatar, so it is removed at upload and reported. Such channels are usually character motion exported with the garment. A clip left with nothing to move is rejected.

## What Portals makes from an upload

Every upload is validated on the server. Portals then derives the files avatars actually load:

| File | Contents | Used by |
| --- | --- | --- |
| Static | The wearable in rest pose on the canonical rig, with no clips, springs or effects | thumbnails, reduced motion, distant avatars, games on older SDKs |
| Dynamic | The validated file with its declared behaviour | every current Guardian surface and game |

A wearable with nothing dynamic has only a static file. Both files are immutable, and each is named by its content. The creator's original upload is kept privately so it can be processed again when the standard adds features.

## `PORTALS_wearable`

A root-level extension object:

```json
{
  "extensionsUsed": ["PORTALS_wearable"],
  "extensions": {
    "PORTALS_wearable": {
      "standard": 1,
      "clips": [
        { "animation": 0, "trigger": "always" },
        { "animation": 1, "trigger": "jump", "loop": "once", "fade": 0.1, "priority": 1 }
      ],
      "emitters": [],
      "trails": []
    }
  }
}
```

- `standard` (required) is the version of this document the file was written for. A runtime that knows an older version plays the parts it understands and ignores the rest.
- A file without `PORTALS_wearable` is a static wearable: its animations never play. Items from before this standard that Portals marks as animated keep looping their first animation.

### Clips

Each entry binds a glTF animation to a trigger:

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `animation` | integer | required | Index into the file's `animations` |
| `trigger` | trigger | required | When it plays (see [Triggers](#triggers)) |
| `loop` | `"repeat"` \| `"once"` | `"repeat"` | Loop, or play once and hold the last frame |
| `fade` | number, 0–2 s | `0.2` | Cross-fade in and out |
| `priority` | integer, 0–10 | `0` | When several clips match, the highest priority plays |

A clip may animate only nodes inside the wearable, including its extra bones, plus the allowlisted material properties below. It can never pose the avatar.

### Triggers

A closed list. Portals derives the avatar's state, so the same wearable behaves the same in every game:

- `always`: while worn;
- `equip`: once, when the item goes on;
- the locomotion states: `idle`, `walk`, `run`, `sprint`, `jump`, `fall`, `land`, `crouch`, `swim`, `climb` and `sit`;
- `emote`: while any emote plays.

Games cannot define their own triggers. That keeps a wearable's behaviour independent of any one game's code.

### Emitters

Particles attached to a node of the wearable:

| Field | Type | Meaning |
| --- | --- | --- |
| `node` | integer | Node the emitter follows |
| `shape` | object | `{ "type": "point" }`, `{ "type": "sphere", "radius": r }`, `{ "type": "box", "size": [x, y, z] }` or `{ "type": "cone", "radius": r, "angle": degrees }` |
| `rate` | number | Particles per second |
| `bursts` | array | Optional `{ "time": seconds, "count": n }` bursts on each trigger start |
| `lifetime` | `[min, max]` seconds | Particle life |
| `speed` | `[min, max]` m/s | Initial speed along the shape's emission direction |
| `size` | `{ "start", "end" }` metres | Size over life |
| `color` | `{ "start": [r,g,b,a], "end": [r,g,b,a] }` | Linear colour and alpha over life |
| `gravity` | number | Multiplier on world gravity (0 floats, negative rises) |
| `texture` | integer | Optional index into the file's `textures` |
| `flipbook` | `{ "columns", "rows", "fps" }` | Optional sprite-sheet animation of `texture` |
| `blending` | `"alpha"` \| `"additive"` | How particles combine |
| `space` | `"local"` \| `"world"` | Particles move with the wearable, or stay where they were emitted |
| `trigger` | trigger | When it emits; default `always` |
| `maxParticles` | integer | Cap on live particles for this emitter |

### Trails

Ribbons that follow a node, such as a sword swing or a wingtip:

| Field | Type | Meaning |
| --- | --- | --- |
| `node` | integer | Node the trail follows |
| `width` | number, metres | Ribbon width |
| `lifetime` | number, seconds | How long a point of the trail lasts |
| `color` | `{ "start", "end" }` | Colour along the trail's age |
| `blending` | `"alpha"` \| `"additive"` | How it combines |
| `trigger` | trigger | When it draws; default `always` |

## Secondary motion

Capes, skirts, hair, tails, ears and chains swing using [`VRMC_springBone` 1.0](https://github.com/vrm-c/vrm-specification/tree/master/specification/VRMC_springBone-1.0), authored with the VRM add-on for Blender or any tool that writes it.

- **Spring joints** are the wearable's own extra bones (below). They can't be rig bones.
- **Body colliders belong to Portals.** Collider groups named `portals:head`, `portals:neck`, `portals:torso`, `portals:hips`, `portals:upperArms`, `portals:lowerArms`, `portals:hands`, `portals:upperLegs` and `portals:lowerLegs` must be placed on rig bones. At runtime the SDK replaces their shapes with the canonical collider set for the wearer's body type, so a cape collides with every body the same way. The rig template ships with these groups.
- **Wearable colliders** (for example a shield the cape should drape over) may be placed on the wearable's own nodes.
- Springs do not collide with the game world. Games can set a wind vector that every spring feels.

## Material animation

`KHR_animation_pointer` channels may target only these properties of the wearable's own materials:

- `/materials/{i}/pbrMetallicRoughness/baseColorFactor`
- `/materials/{i}/emissiveFactor`
- `/materials/{i}/extensions/KHR_materials_emissive_strength/emissiveStrength`
- `/materials/{i}/{texture}/extensions/KHR_texture_transform/offset`, where `{texture}` is one of the following:
  - `pbrMetallicRoughness/baseColorTexture`;
  - `emissiveTexture`;
  - `normalTexture`.

Any other pointer is removed at upload and reported.

## Extra bones

A skinned wearable may carry bones beyond the rig: cape chains, tail segments, ear bones.

- Every extra bone descends from a rig bone. The one exception is `neutral_bone`, which Blender's exporter adds for vertices with no bone weights. Runtimes bind it to the rig root.
- An extra bone may not reuse a rig bone's name. A node with a rig bone's name is bound to that bone of the avatar, so it must sit under the same parent as in the rig. A file may carry a second full copy of the rig.
- Extra bones belong to the one wearable instance, so two wearables can each have a bone named `Cape_01`.

In the static file, the weights on extra bones are folded into their nearest rig ancestor, so the static file still looks right in rest pose.

## Budgets

Every wearable must fit the limits below, and the SDK enforces the same limits on every avatar at runtime. Limits for the in-development parts are published with the SDK release that enables them.

| Limit | Cosmetic | Full avatar |
| --- | --- | --- |
| File size | 50 MB | 50 MB |
| Triangles | 120,000 | 150,000 |
| Largest texture side | 4096 px | 4096 px |
| Decoded texture memory | 128 MB | 128 MB |
| Materials | 16 | 48 |
| Mesh primitives (draw calls) | 64 | 96 |
| Animation clips | 8 | 8 |
| Emitters / live particles per emitter | 4 / 256 | 4 / 256 |
| Trails | 2 | 2 |

## How games run it

A game needs nothing beyond the `avatars.update(dt)` call it already makes. The SDK:

- plays clips, springs and effects from that call, using the game's `dt`, so pausing the game pauses them;
- runs the springs and effects after anything that poses bones (`onAfterUpdate`), so they follow ragdolls and aim poses;
- runs the local player at full quality, and lowers quality for distant and crowded avatars, one feature at a time;
- shows the static file to a player who asks for reduced motion;
- resets springs after a teleport, so a cape never stretches across the map;
- never lets one wearable exceed its budget.

## Validation errors

The upload rejects a file for any of these reasons:

- it is not a self-contained GLB, or the glTF Validator reports an error;
- it requires an extension;
- a texture is not PNG or JPEG;
- it exceeds a budget;
- a `PORTALS_wearable` field is out of range or points at something that does not exist;
- a clip plays an animation with nothing a wearable may move;
- a bone breaks the rules above.

Each error names the field or limit it failed, and the upload lists every error at once. The rest-pose check joins this list when Portals starts deriving separate static files.
