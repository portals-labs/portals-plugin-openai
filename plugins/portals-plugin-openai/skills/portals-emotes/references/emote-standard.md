# Portals Emote Standard

Source: [https://portals.to/documentation/web-games/emote-standard](https://portals.to/documentation/web-games/emote-standard) — the official Portals documentation, copied verbatim with its site-relative links pointed at portals.to.

**Version 1.** This page is the contract between an emote file and every place a Portals avatar plays one:
- the Play lobby;
- `/avatar` and the Shop;
- hosted games;
- games built with the Portals engine.

An emote that follows it plays the same way on all of them. Games need no code for it: the [Portals avatar SDK](https://portals.to/documentation/web-games/guardian-avatars) gives every player an emote wheel.

## What an emote is

An emote is motion on the Portals avatar rig, played over whatever the avatar is doing and faded back to it. It can:
- play once (a wave), or loop until the player moves (a dance);
- open with an entry and close with an exit;
- hold a facial expression while it plays;
- carry a prop — a flag, a microphone, a guitar — shown only while it plays;
- play a sound or music.

Every player has a starter set. Players buy and trade more in the Shop, and arrange up to eight on their wheel on `/avatar`.

## Principles

1. **An emote is data, never code.** The GLB describes the motion; the avatar SDK plays it. Emotes may not contain scripts, shaders, external file references, lights or cameras.
2. **The file describes itself.** Its mode, expression and what its feet do are declared inside the GLB, in `PORTALS_emote`.
3. **One rig.** Emotes are authored on the Portals avatar rig, the same rig wearables use. A file on any other rig is refused at upload and skipped by every runtime: runtimes never retarget someone else's upload.
4. **Cosmetic only.** An emote changes how the avatar looks, never where it is or what it can do. Games decide when emotes are allowed.

## The file

### Rig

The file carries the complete Portals avatar rig — every bone of the [template](https://portals.to/wearable-standards) with its canonical name and parent, in the canonical orientation — as an armature on its own. Export the armature without the body; the only mesh an emote carries is its [prop](#prop). Blender exports an armature with no mesh as plain joint nodes, which is the expected form.

Author on either body's template. The SDK fits the file to the wearer's body as it loads: each key turns that body's joints the same way, the hips' rise and fall is scaled to it, and hands that meet on your template meet on it. One file serves both.

### Clips

| Clip name | Required | Plays |
| --- | --- | --- |
| `Emote` | yes | The emote. A loop repeats it; a one-shot plays it once. |
| `Emote_Enter` | no | Once, before `Emote`. |
| `Emote_Exit` | no | Once, after a loop is stopped or a one-shot ends. |

Other clips are removed at upload. Within the three clips:
- any rig bone may rotate;
- only `mixamorig:Hips` may move, and only up and down — sideways travel is removed, so the avatar stays where it stands;
- the prop's own nodes may move, rotate, scale and change morph weights;
- scale channels on the rig, and channels on any other node, are removed.

### Prop

An emote may carry one prop. Parent every prop mesh under one node, and parent that node to the rig bone it rides on: `mixamorig:Hand.R` for a microphone, `mixamorig:Spine2` for a guitar on the back. The prop may have bones of its own, a flag's cloth or a guitar's strap, animated by the emote's clips; it may not be skinned to the avatar's bones.

The prop follows the [wearable standard's](https://portals.to/documentation/web-games/wearable-standard) file rules — self-contained, no external files — with the budgets under [Limits](#limits). It is shown only while the emote plays, grows in and shrinks out with the emote's fade, and keeps the size and placement it has on the file's rig on either body.

### Sound

An emote may carry one sound: an MP3, embedded in the file. Upload it beside the GLB in the Shop form, and the form puts it in the file; a file that already embeds one declares it in `PORTALS_emote`. ID3 tags are stripped at upload.

The sound starts with `Emote`, not `Emote_Enter`, and runs on the emote's clock. In a looping emote it loops, so make it the loop's length, and it stops when the loop does. In a one-shot it plays through, into `Emote_Exit` if it is longer. Encode it with LAME's gapless header (any LAME-based encoder writes one) so a loop loops on the music and not on the encoder's silence.

### `PORTALS_emote`

A root-level extension, listed in `extensionsUsed` and never in `extensionsRequired`:

```json
{
  "extensionsUsed": ["PORTALS_emote"],
  "extensions": {
    "PORTALS_emote": { "standard": 1, "mode": "loop", "face": "smile", "feet": "planted" }
  }
}
```

| Field | Values | Meaning |
| --- | --- | --- |
| `standard` | `1` | The version of this standard the file follows. |
| `mode` | `loop`, `once` | Loop until the player moves or stops it, or play through once. |
| `face` | `neutral`, `smile`, `serious`, `frown` | Optional. The expression the avatar holds while the emote plays. |
| `feet` | `free`, `planted` | Optional. `planted` holds each foot where it stood when the emote began, and the legs bend to reach it. Because the hips never travel sideways, the feet of an emote that stands in place would otherwise slide. It works over any idle, on either body. `free` (the default) plays the file's legs as they are, for an emote whose feet step. |
| `prop` | `{ "node": 12, "bone": "mixamorig:Hand.R" }` | Present with a prop: the node rooting it, and the rig bone it rides on. |
| `sound` | `{ "bufferView": 7 }` | Present with a sound: the buffer view holding its MP3. |

The upload form writes this for you from the mode, expression and feet you choose, the prop it finds and the sound you add.

### Limits

| Limit | Value |
| --- | --- |
| File size | 5 MB |
| `Emote`, one-shot | 60 s |
| `Emote`, loop | 30 s |
| `Emote_Enter`, `Emote_Exit` | 5 s each |
| Hips travel up or down from rest | 1.5 m |
| Keyframes across all clips | 200,000 |
| Prop triangles | 10,000 |
| Prop materials | 2 |
| Prop mesh primitives (draw calls) | 4 |
| Prop textures | 2, at most 1024 px a side |
| Sound | 1 MB, 30 s |

## Reference files

Three reference emotes sit on the canonical rig and pass every upload rule. Use them as templates:
- [`nod.glb`](https://portals.to/portals-sdk/emotes/templates/nod.glb) — a one-shot with a held smile: one `Emote` clip and the declaration;
- [`sway-loop.glb`](https://portals.to/portals-sdk/emotes/templates/sway-loop.glb) — a loop with `Emote_Enter`, `Emote_Exit` and a hips bob;
- [`rally-dance.glb`](https://portals.to/portals-sdk/emotes/templates/rally-dance.glb) — a looping dance with a prop and music: a flag on the right hand, waved by its own bone, and a one-bar loop at 120 BPM.

## How runtimes play an emote

- **Over the body.** The emote is a full-body layer applied after the avatar's own animation. It fades in over 0.2 s and back out over 0.3 s, and bones the emote does not key keep moving underneath it.
- **Cancelled by action.** With the SDK's character controller, moving, jumping, crouching, using an item or leaving the ground ends the emote at once. A looping emote the player stops otherwise plays out its exit.
- **Seen from outside.** In first person the camera pulls back to third person for the emote and returns afterwards.
- **Wearables join in.** A wearable clip or effect bound to the `emote` trigger of the [wearable standard](https://portals.to/documentation/web-games/wearable-standard) plays while any emote plays.
- **Props within a budget.** Props share a budget across every avatar in view: the player's own first, then by distance. Past it, an emote plays without its prop.
- **Every sound at one level.** Each runtime measures the sound on the player's device (ITU-R BS.1770-4) and plays it at -16 LUFS, never above -1 dBTP, and raises a quiet one at most 10 dB. A file can't be louder than another.
- **The player decides.** The player's Portals setting — everyone's emote sounds, only their own, or none — and the game's own volume always apply. Other players' sounds come from where they stand, at half the volume of the player's own, and only the nearest three are heard at once. Emote sounds drop 12 dB while someone else talks in voice chat.
- **Skipped, never broken.** A file that breaks this standard, or that fails to load, plays nothing.

## Selling an emote

Emotes are sold in the Shop like wearables: listed, bought, resold, auctioned, offered on and traded. A player's copy goes on their wheel on `/avatar`; one they sell or trade away leaves their wheel.

To list one, choose **emote** on `/sell-in-shop`:
1. Upload the file, and its sound if it has one. The form checks both against this standard with the mode, expression and feet you choose, and shows the ingest report: how it plays, how much of each limit it and its prop use, and anything it removed. The preview plays what buyers will get, with sound.
2. Add a Shop image. It is also the emote's icon on the wheel.
3. Say where the motion comes from: made by you, your own motion capture, or licensed — with the licensor and the licence. Then agree to the rights statement.
4. Set the price, supply and release as for any item, and submit it for review. Reviewers play it on both bodies in the review room and read the provenance beside it.

**Provenance.** Motion copied from a game, a film, another person's performance, or an asset pack whose terms forbid selling animations on their own can't be sold. Some packs that call themselves free forbid exactly that, so check the licence, not the label. The same goes for the sound: music, samples and recordings need the right to be sold as part of an emote.

**Approved emotes.** You can change an approved emote's name, description and images. Its motion, prop and sound can't change: buyers bought them, and review checked their provenance. To sell a new version, list it as a new emote.

**Creator tools.** Agents can check emote files, create and edit emote drafts and submit them through `/api/v2/arcade/emotes/{validate,upload-url,drafts,submit}` with an access key that has the **Shop emote drafts** permission. Grant it at `/mcp`. The price, the supply and the agreement to the rights statement are set in the Shop form, not through a key.

## Upload errors

| Error | Fix |
| --- | --- |
| `rig` | Author on the Portals template rig; every bone must be present under its canonical parent. |
| `orientation` | Apply the armature's rotation so the rig stands upright facing +Z. |
| `descriptor` | Add `PORTALS_emote` with a `standard`, a `mode` and, optionally, a valid `face` and `feet`. |
| `no-clip` | Name the emote's clip `Emote`, and make sure it moves the rig. |
| `geometry` | Export the armature without the body. A prop's meshes go under one node parented to a rig bone. |
| `prop` | Carry one prop, under one node on a rig bone, holding only its own nodes and skinned only to its own bones. |
| `budget` (prop) | Bring the prop within its triangles, materials, primitives, textures and texture size. |
| `texture` | Export the prop's textures as PNG or JPEG, embedded in the file. |
| `sound` | Upload an MP3 of at most 1 MB and 30 s. |
| `not-self-contained` / `required-extension` | Export one self-contained `.glb` that requires no extension but `KHR_mesh_quantization`. |
| `duration` | Shorten the clip to the limit for its kind. |
| `hips-travel` | Keep the hips within 1.5 m of their rest height. |
| `too-large` / `budget` | Reduce the export's sample rate. |
| `EMOTE_PROVENANCE_REQUIRED` | Say where the motion comes from before submitting. |
| `EMOTE_LICENSOR_REQUIRED` / `EMOTE_LICENSE_REF_REQUIRED` | For licensed motion, name the licensor and link or reference the licence. |
| `EMOTE_ATTESTATION_REQUIRED` | Agree to the rights statement in the Shop form. |
