---
name: portals-wearables
description: Make wearables for the Portals Shop — hats, glasses, tops, back items and full avatars for the Portals avatar — to the Portals wearable standard, then check, draft and submit them with the portals-web-games MCP tools. Use when authoring or exporting a wearable GLB for Portals, the Portals rig template new-rig.glb, skinned or rigid attachment, wearable budgets, PORTALS_wearable clips, VRMC_springBone springs, emitters and trails, validate_wearable, create_wearable_draft, update_wearable_draft, submit_wearable_draft, the Shop wearable drafts access-key permission, or a wearable upload the Shop rejects.
---

# Portals wearables

A Portals wearable is one GLB that follows the **Portals wearable standard**, the contract between the file and every place a Portals avatar appears: `/avatar`, the Shop, the Play lobby and games. Read [references/wearable-standard.md](references/wearable-standard.md) before authoring or fixing a file. It is the official spec, verbatim, and the public copy is at https://portals.to/documentation/web-games/wearable-standard.

The `portals-web-games` MCP tools take a wearable from a local file to Portals review on the creator's own account. **Price and release happen on Portals, not here.**

## What works today

- **Enforced at upload now:** the file rules, the two attachment forms, the rig, the budgets and glTF validity. Every Shop upload is checked on the server, and so is every `validate_wearable` call.
- **Specified, played as SDK releases enable them:** named clips with triggers (`PORTALS_wearable`), material animation, secondary motion (`VRMC_springBone`), particle emitters and trails. Upload validates them against the standard today. Each only plays in games and on surfaces running a Portals avatar SDK release that enables it (the [changelog](https://portals.to/documentation/web-games/guardian-avatars-changelog) names each one). Everywhere else the wearable shows in its rest pose.

So author the static wearable first, and make it look right standing still (principle 4 of the standard). That is how it appears in thumbnails, to reduced-motion players, on distant avatars and in games pinned to older SDKs. Add motion only when you mean it, and never make it carry the look.

## The rig

Every wearable ships the full **Portals rig**, the 90-bone Mixamo skeleton with `mixamorig:` bone names. Download the template, bind to it and export:

- Rig template: https://portals.to/rigs/new-rig.glb
- Bone reference and export tips: https://portals.to/wearable-standards

Attach the mesh one of two ways:

- **Skinned:** weight-painted to rig bones so it deforms with the body. Use it for clothing.
- **Rigid:** no skin weights. Every rendered mesh is parented under a rig bone and follows it as a solid object, like a hat under `mixamorig:Head` or wings under `mixamorig:Spine2`. Use it for props. A mesh with skin weights can't be attached rigidly.

Export one binary glTF (`.glb`) with +Y up, in the rig's own facing: a sideways or mirrored export is refused. An old 65-bone Portals rig missing only the 25 face bones is upgraded automatically. Any other rig is refused with the list of missing bones. Skinned capes and tails may add extra bones below a rig bone (see "Extra bones" in the reference).

## Reference templates

Six reference wearables on the real rig show each part of the standard authored correctly. They are also the fixtures the SDK and the upload validator are tested against. Download them from `https://portals.to/portals-sdk/wearables/templates/<name>.glb`:

| Template | Shows |
| --- | --- |
| `static-hat` | A plain rigid hat under the head. This is the static baseline |
| `animated-wings` | Rigid wings with clips bound to triggers: an `always` flutter, a `jump` flap and an `equip` unfold |
| `spring-cape` | A skinned cape on an extra-bone chain driven by springs that collide with the body |
| `spring-tail` | A skinned tail on an extra-bone chain from the hips |
| `ember-hat` | A glowing hat whose tip emits sparks |
| `trail-sword` | A right-hand sword whose tip leaves a trail |

Start from the closest template rather than a blank scene.

## Budgets

The upload checks these limits and the SDK enforces the same ones at runtime. `validate_wearable` reports the file's use next to each limit.

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

`type: "cosmetic"` is anything worn on the Portals avatar. `type: "avatar"` is a full avatar that replaces it. The type picks the budgets. The Shop render image is a PNG, JPEG or WebP of at most 5 MB.

## Rules that are easy to get wrong

- **Self-contained, or refused.** Embed every buffer and texture, and use PNG or JPEG textures only. `extensionsRequired` may list only `KHR_mesh_quantization`, so Draco and meshopt compression are refused.
- **Ingest strips some things silently, and says so.** Unlisted extensions, lights, cameras, and animation channels that move a rig bone (character motion exported with the garment) are removed at upload and reported in `removals`. The Shop stores the cleaned copy, so export clean and nothing changes.
- **A clip moves only the wearable.** It can never pose the avatar. A clip left with nothing to move is refused.
- **Validation lists every problem at once.** Fix them all, then validate again. Every message is the server's own and safe to show the creator verbatim.
- **Only the creator's own uploads count.** Model and render files go through the tools' upload. An outside URL, or another creator's file, is refused.
- **No legacy `animated` flag.** A standard file declares its own motion with `PORTALS_wearable`. The tools offer no switch for it.

## Check, draft, submit

**0. The permission.** The tools need the access key's **Shop wearable drafts** permission. The creator turns it on at https://portals.to/mcp while signed in, and generating a new key turns it off again. Without it every call answers `ACCESS_KEY_PERMISSION_REQUIRED` with the page to visit. Relay that. An agent can never grant it.

1. **`validate_wearable`** with `glbPath`, `type` and `category`. It uploads the file and runs the server's dry run of Shop ingest: orientation, rig, the standard and the glTF Validator, with nothing drafted. Iterate until `ok` is true, then keep its `glb_url`.
2. **`create_wearable_draft`** with `name`, `type`, `category`, `genderSupport`, `glb` (a local path, or the `glb_url` to reuse that upload) and `render` (a local image). Optional fields are `description`, `blockedSlots` (other slots it takes, like a full-body suit taking `top` and `bottom`) and `removesHair` (a hat covering the whole head). A `unisex` item needs `femaleGlb` and `femaleRender` too. The save runs the same ingest as the Portals Shop form, so a failing file comes back with its issues.
3. **The creator finishes it on Portals.** Hand over the returned `edit_url`, where they set the price, supply and any sale dates. Opening the draft there lets the viewer capture the Shop thumbnail, and saving keeps it. None of this can be done through the tools.
4. **`submit_wearable_draft`** sends it to Portals review. If it isn't ready, it lists every blocker with who fixes it and where: a payout account to connect (`/monetization`), sale settings or the thumbnail in the Shop form, or draft fields to fix.
5. **`update_wearable_draft`** changes only the fields you pass. Saving a rejected item returns it to draft. Replacing a model clears the thumbnail captured for the old one, so the Shop form captures a new one when it is next opened.

## Price and release happen in Portals

The tools stop at the draft. Price, supply, sale dates, game gating, purchase limits and anything touching Coins are set only in the signed-in Shop form, behind Portals' Stripe payout check. Submitting puts the item in review; a Portals moderator approves or returns it, and the item goes on sale by the settings the creator chose. Never promise a price, a date or approval.

## Related

- Wearing items in a game, including trying a local wearable GLB on a Portals avatar: the `portals-avatars` skill (`registerCatalog` + `equip`).
- The MCP workflow and authentication: the `portals-web-games` skill.
