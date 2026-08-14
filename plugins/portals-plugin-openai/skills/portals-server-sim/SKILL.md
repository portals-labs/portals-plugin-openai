---
name: portals-server-sim
description: Build a server-authoritative real-time game on Portals where server.js simulates the game itself — one shared sim module, snapshots and inputs over the wire, client prediction and dead reckoning, ping measured from echoed sequence numbers, and a fallback ladder to host authority when the server is away. Use for continuous contested multiplayer (shared ball, racing, ramming), server-side physics, snapshot protocols, prediction and reconciliation, or testing a built server.js on a virtual clock.
---

# Server-simulated games

The `portals-server-scripts` skill covers the contract — a root `server.js` that Portals runs as an invisible authoritative participant. This is the next level: the server script **simulates the game itself**, stepping the same physics the clients ship and broadcasting the result, so nobody's tab owns the world.

Read [references/server-sim.md](references/server-sim.md) for the full pattern with the worked FUTBOL example — protocol, tick, prediction, fallback ladder, measured budgets, and the virtual-clock test harness. It is the official documentation, copied verbatim.

## The default for multiplayer games

**If the game is multiplayer, build it server-simulated.** Host-authoritative multiplayer (one tab simulates, the rest follow) is not the starting point — it is the fallback rung a server sim keeps underneath for the moments the server is away. The server sim buys fairness (games are **continuous and contested** more often than they first look — a shared ball, ramming, racing, even a grabbed pickup — and a host's zero self-latency is a real competitive advantage), **witnessed outcomes** rather than claims, and **symmetry** — the server sits at the relay, ~25 ms from everyone on a regional channel.

The exceptions are narrow: purely cosmetic shared spaces (a co-op drawing canvas, an emote wall) and slow turn-based games whose pace hides latency, with a plain server script as referee where anything needs deciding. When in doubt, take the server sim — retrofitting authority into a shipped host-authoritative game is much harder than building on it, while the host-authority fallback comes with the server sim for free.

## The rules that make or break it

- **One simulation, shared byte-identically.** The client and server must run the same game rules from one module. A ball simulated two slightly different ways is a desync nobody can debug.
- **The sim module stays plain.** Pure data in, pure data out (`{ x, z }` fields, no THREE.js, no DOM, no clock of its own). Side effects go through hooks so each authority decides what a goal or a sound *means*.
- **Integration must be step-rate independent.** The server ticks at 20 Hz against the sandbox's 50 ms timer floor; clients render at 60–120 Hz. Solve movement exactly over `dt` (`vel = settle + (vel - settle) * Math.exp(-dt * k)`) instead of accumulating per-frame adds, or the two ends settle at different speeds and prediction argues with its authority for as long as a key is held.
- **The sandbox takes one file** — no `import`, no `require` — so share at *build* time. Bundle the referee entry plus the sim module into `dist/server.js` with esbuild (`format: "iife"`, unminified so it reads next to its logs); that bundle is what a push ships.
- **The server marks its snapshots.** One field (`sv: 1`) is the entire handshake: the first marked snapshot flips every tab, host included, into a follower; when the marked stream goes quiet the host resumes simulating. One protocol, two places it can run.
- **Inputs go up on change, not on a metronome** — no faster than every 60 ms, and every 100 ms regardless so a held key stays alive. Carry a sequence number. On the server an input drives whatever seat the team sheet assigned that player, **never whatever the packet asks for**, and a stale input stops driving its actor after ~700 ms.
- **The clock ticks on the server**, never as a published deadline — a deadline is read against each player's own clock, and a wrong device clock would show a wrong match.
- **Predict yourself, dead-reckon everyone else, replay contested moments from events.** Ease your own actor onto the authority's version rather than snapping. Dead-reckon others on the *sim's own drag constants* — extrapolating at snapshot velocity in a straight line walks a released actor well past where the physics stopped it. Never predict a contested outcome; play it when `ev` says so.
- **Subtract the ocean.** A snapshot is already a round trip old. Echo the last acted-on sequence number in each actor's row; the gap between sending `q` and reading it back — both ends off the guest's own clock — is the whole loop. Project the ball and opponents forward by one downlink, your own actor by the full trip.
- **The fallback ladder must stay walkable.** A server sim that takes the game down with it has failed. Three rungs, each a working game: server simulates → server referees only while the host simulates → no server at all (peer-run, which is the ordinary state of every unpublished game). Silence is the signal: guests fall back after a beat and a half, the host starts simulating half a beat later, both measured from the same last-marked snapshot so guests are listening before the host talks.
- **Boot resumes shared state.** Shared state outlives the script. A hot-swapped server reads its own `server:*` keys back, prunes them against the live roster, and carries on — being swapped mid-match must not hand anybody a fresh 0-0.
- **Coalesce state writes.** Scoreboard once a second, directory-style state on a ~120 ms timer; the smooth clock rides the snapshots. Keep every value inside 8 KB: packed arrays over objects, two-decimal rounding, capped list sizes.

## Budgets that shape the design

The sandbox limits are load-bearing here, not commentary. The 50 ms timer floor *is* the 20 Hz tick and 20 snapshots/s. The ~60 broadcasts/s budget means one snapshot per tick with events piggybacked on it. CPU is not the constraint — a 3v3 physics tick measured ≈ 0.7 µs against a ~50 ms ceiling — architecture is the cost. Server RTT is ~25 ms regional versus ~230 ms from the wrong side of a `global:` channel, so keep matches on regional channels.

## Testing and debugging

- **Test the shipped file on a virtual clock.** The built `server.js` is a plain script talking to one `server` global — mock that global, run the file in `node:vm`, and ninety seconds of gameplay land in milliseconds, deterministically. The mock must also fake `Date.now()`, since the tick measures real elapsed time against it. Drive the lobby, a match, the fallback, and a hot swap through it; exit non-zero on the first failed expectation. Publish-and-preview is too slow a loop for physics.
- **Editor preview** runs the draft's `server.js` against a real session; two preview panes (the 2p device) are a two-player match on one screen.
- **Server logs** — the script's own `server.log(...)` lines (marked `[script]`) and the platform's (session starts, crashes, `script fatal`) — appear under the preview's bug icon in the **Server logs** tab. Fleet failures light the icon on their own.
- **Local dev sessions** get the **published** server script, never the one on disk — so a draft has no server there, which exercises exactly the fallback rung the design must keep working. `net.getState("server:...")` in the console says which world you are in.

## Related

- The `server.js` contract, sandbox limits, and protected `server:` state: the `portals-server-scripts` skill.
- Client-side sessions, channels, and the `Portals.net` API: the `portals-multiplayer-and-voice` skill.
