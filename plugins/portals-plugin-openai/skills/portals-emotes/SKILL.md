---
name: portals-emotes
description: Make emotes for the Portals Shop — dances, waves and flourishes a Portals avatar plays in the Play lobby, on /avatar and in every game, with an optional held prop and sound or music — to the Portals emote standard, then check, draft and submit them with the portals-web-games MCP tools. Also covers playing emotes in a Three.js game with the avatar SDK (avatar.emotes, the controller's emote wheel, syncEmotes, avatars.audio). Use when authoring or exporting an emote GLB for Portals, the Emote / Emote_Enter / Emote_Exit clips, PORTALS_emote, an emote's prop or MP3, emote limits and budgets, where the motion comes from and the rights statement, validate_emote, create_emote_draft, update_emote_draft, submit_emote_draft, the Shop emote drafts access-key permission, emote sound volume or the player's emote-sound setting, or an emote upload the Shop rejects.
---

# Portals emotes

A Portals emote is one GLB that follows the **Portals emote standard**: motion on the Portals avatar rig, played over whatever the avatar is doing and faded back to it. Read [references/emote-standard.md](references/emote-standard.md) before authoring or fixing a file. It is the official spec, verbatim, and the public copy is at https://portals.to/documentation/web-games/emote-standard.

The `portals-web-games` MCP tools take an emote from a local file to Portals review on the creator's own account. **Price, release and the rights statement happen on Portals, not here.**

## The file

- **The armature on its own.** Export the complete Portals rig — every bone of the template at https://portals.to/rigs/new-rig.glb with its canonical `mixamorig:` name and parent — without the body. Blender exports an armature with no mesh as plain joint nodes, which is expected. Any other rig is refused and never retargeted.
- **One optional prop.** Parent every prop mesh under one node, and that node to the rig bone it rides on (`mixamorig:Hand.R` for a microphone, `mixamorig:Spine2` for a guitar on the back). The prop may have bones of its own that the clips animate — a flag's cloth — but may not be skinned to the avatar's bones. It shows only while the emote plays, and upload declares it in `PORTALS_emote` for you.
- **Three clip names.** `Emote` (required), and optionally `Emote_Enter` and `Emote_Exit`. Other clips are removed at upload and reported.
- **Only the hips move, and only up and down.** Any rig bone may rotate. Sideways hips travel, scale channels and channels on other nodes are removed, so the avatar stays where it stands.
- **One file for both bodies.** Author on either body's template; the SDK fits the file to the wearer's body as it loads: each key turns that body's own joints the same way, the hips' rise and fall is scaled, and hands that meet on your template meet on it.
- **The mode, expression and feet.** A loop repeats until the player moves or stops it; a one-shot plays through once. An optional expression (`neutral`, `smile`, `serious`, `frown`) is held while it plays. `feet: "planted"` holds the Portals avatar's feet where they stood when the emote began, so an emote that stands in place (a wave, a clap, a cheer) never slides; `free`, the default, plays the file's legs as they are, for one whose feet step. The tools write these into the file's `PORTALS_emote`, as the Shop form does. Pass `mode`, `face` and `feet` together: any left out goes back to its default.
- **One optional sound.** An MP3, passed beside the GLB as `sound`; upload embeds it in the file and strips its tags. It starts with `Emote` (not `Emote_Enter`) and runs on the emote's clock: in a loop it loops, so make it the loop's length; in a one-shot it plays through. Encode with LAME (any LAME-based encoder) so its gapless header lets a loop loop on the music. Every runtime measures it and plays it at -16 LUFS, so its level in the file doesn't matter — never boost it to stand out.

## Limits

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
| Prop textures | 2, at most 1024 px a side, PNG or JPEG |
| Sound | 1 MB of MP3 audio, 30 s of music |

`validate_emote` reports the file's use next to each limit. The Shop image is a PNG, JPEG or WebP of at most 5 MB; it is also the emote's icon on the player's wheel.

## Reference emotes

Three reference emotes on the canonical rig pass every upload rule. Start from them rather than a blank scene:

- `https://portals.to/portals-sdk/emotes/templates/nod.glb` — a one-shot with a held smile.
- `https://portals.to/portals-sdk/emotes/templates/sway-loop.glb` — a loop with an entry, an exit and a hips bob.
- `https://portals.to/portals-sdk/emotes/templates/rally-dance.glb` — a looping dance with a prop and music: a flag on the right hand, waved by its own bone, and a one-bar 120 BPM loop.

## Where the motion comes from

Every emote declares its source before review: **original** (keyframed by the creator), **own-mocap** (the creator's own capture) or **licensed** — with the licensor and the licence. Then the creator agrees to the rights statement, on Portals.

- **Never sell motion you don't have the rights to sell.** Motion ripped from a game, a film, another person's performance, or an asset pack whose terms forbid reselling animations on their own can't be listed. Some packs that call themselves free forbid exactly that — check the licence, not the label.
- **The same goes for the prop and the sound.** The rights statement covers everything the emote carries: a song, a sample or a recording, and a prop model, need the right to be sold as part of an emote. Music generated for the creator is theirs only if the generator's terms say so.
- **The agreement is the creator's.** The tools can declare the source, licensor and licence, but never agree to the statement. A new file, or any change to the declaration, clears an earlier agreement, so the creator agrees to exactly what was declared.

## Check, draft, submit

**0. The permission.** The tools need the access key's **Shop emote drafts** permission — separate from Shop wearable drafts. The creator turns it on at https://portals.to/mcp while signed in, and generating a new key turns it off again. Without it every call answers `ACCESS_KEY_PERMISSION_REQUIRED` with the page to visit. Relay that. An agent can never grant it.

1. **`validate_emote`** with `glbPath`, `mode`, optionally `face` and `feet`, and `soundPath` for an MP3. It uploads the files and runs the server's dry run of emote ingest with nothing drafted, reporting the prop's and sound's budgets too. Iterate until `ok` is true, then keep its `glb_url` and `sound_url`.
2. **`create_emote_draft`** with `name`, `glb` (a local path, or the `glb_url` to reuse that upload), `sound` (a local `.mp3`, or the `sound_url`), `render` (a local image), `mode`, `face`, `feet`, and `source` (with `licensor` and `licenseRef` for licensed motion). The save runs the same ingest as the Portals Shop form, so a failing file comes back with its issues.
3. **The creator finishes it on Portals.** Hand over the returned `edit_url`: there they watch it on a Portals avatar, open it in the test room, agree to the rights statement, and set the price, supply and any sale dates. None of this can be done through the tools.
4. **`submit_emote_draft`** sends it to Portals review. While creator emote sales are closed it answers `CREATOR_EMOTE_SALES_CLOSED` and the draft stays saved. If the draft isn't ready it lists every blocker with who fixes it and where: a payout account, sale settings or the rights statement in the Shop form, or draft fields to fix.
5. **`update_emote_draft`** changes only the fields you pass. Changing `mode`, `face` or `feet` re-checks the file with the new declaration; `sound` embeds a new MP3, and `sound: "none"` removes it. Saving a rejected emote returns it to draft.

An approved emote's name, description and image can change; its motion, prop and sound can't. To sell a new version, list it as a new emote.

## Playing emotes in a game

Games need no code for the standard wheel: the avatar SDK's character controller gives every player one (avatar SDK 0.57.0).

- **The wheel.** `avatars.createController(avatar, …)` opens it with **B**, a touch button on mobile, and plays the slot a player picks; moving, jumping, crouching or using an item cancels the emote. Pass `emotes: false` for a game with no emotes, gate it with `controller.setEmotesEnabled(false)`, or keep the rules with your own UI via `emotes: { key: null }` and `controller.playEmote(slotOrKey)`.
- **The player's own wheel.** `createAvatarFromPlayer(player)` fills `avatar.emotes` from the wheel the player arranged on `/avatar`; guests get the starter set.
- **Code.** `avatar.emotes` plays on any avatar, including a remote player's or an NPC: `play(def, { elapsed })`, `stop()`, `current()`, `list()`, `on('start' | 'end', …)`. An emote plays as a full-body layer over whatever the avatar is doing.
- **Props and sound (0.59.0).** Both play with no game code. Props share one budget across avatars (`emotePropBudget`, the player's own first); sounds play through `avatars.audio` at one measured loudness, others' positional at half volume and the nearest three heard. It follows the player's Portals emote-sound setting (`Portals.preferences`, Portals SDK 1.22.0) and ducks under Portals voice chat on its own. Set your game's level with `avatars.audio.setVolume()`, route it into your mixer with `new PortalsAvatars({ audio: { context, destination } })`, and pause it with `avatars.audio.setMuted(true)`. Never play emote sounds any other way.
- **Multiplayer (0.58.0).** One call after `Portals.net.join()` keeps every player's emotes in step: `syncEmotes({ net: Portals.net, local: avatar, avatarFor: (id) => remoteAvatars.get(id) })`, exported beside `PortalsAvatars`. `dispose()` it on leave and `setLocal(newAvatar)` after rebuilding the player's avatar. A receiver only plays what the room vouched for on the player's roster entry (`player.emotes`), so a message can never make someone appear to own an emote.
- **Pin the version.** Pin `guardiansSdk` in `portals.json` to a release that has what you use — see the `portals-avatars` skill.

## Related

- The full avatar SDK API, including `avatar.emotes`: the `portals-avatars` skill.
- `syncEmotes` and `Portals.net`: the `portals-multiplayer-and-voice` skill.
- Wearables that react to emotes (the `emote` trigger): the `portals-wearables` skill.
- The MCP workflow and authentication: the `portals-web-games` skill.
