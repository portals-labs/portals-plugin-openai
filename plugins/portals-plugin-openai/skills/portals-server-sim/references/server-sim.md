# Server-Simulated Games

Source: `content/docs/code-and-custom-ui/server-sim.md` in portal-backend-2 — verbatim copy of the official Portals documentation. Not yet published on portals.to.

[Server Scripts](https://portals.to/documentation/advanced-tooling/server-scripts) covers the contract: put a `server.js` at your project root and Portals runs it as an invisible, authoritative participant in every multiplayer session. That page shows a lobby — turn-taking, ready checks, verified scores. This page is the next level: the server script **simulates the game itself**, stepping the same physics your clients ship and broadcasting the result, so that nobody's tab owns the pitch.

The worked example throughout is FUTBOL, a 3D stickman football game: 90-second matches at up to 3v3, a worldwide lobby, and a ball that must read identically on every screen. Its server script — the *referee* — runs the football on Portals servers, every tab follows its snapshots, and the match survives the server crashing mid-game.

## The default for multiplayer games

**If your game is multiplayer, build it server-simulated.** Host-authoritative multiplayer (one player's tab simulates, the rest follow) is not the starting point here — it is the fallback rung a server sim keeps underneath for the moments the server is away. Defaulting to the server sim buys you, from day one:

- **Fairness.** The game is **continuous and contested** more often than it first looks — a shared ball, ramming, racing, even a grabbed pickup. Under host authority one player pays zero latency on their own actions, which in any competitive moment is a real advantage.
- **Witnessed outcomes.** A goal the server saw in its own simulation cannot be fabricated by any client; under host authority, every result is ultimately someone's claim.
- **Symmetry.** On a regional channel the server sits at the relay, ~25 ms from everyone, instead of 0 ms from one player and a full round trip from the other.

The exceptions are narrow: a purely cosmetic shared space (a co-op drawing canvas, an emote wall) or a slow turn-based game whose pace hides latency can ship [trust-light](https://portals.to/documentation/advanced-tooling/multiplayer-and-voice), with a plain [server script](https://portals.to/documentation/advanced-tooling/server-scripts) as referee where anything needs deciding. When in doubt, take the server sim — retrofitting authority into a shipped host-authoritative game is much harder than building on it, while the host-authority fallback comes with the server sim for free.

## One simulation, shared

The whole pattern rests on one rule: **the client and the server must run byte-identical game rules.** A ball simulated two slightly different ways is a desync nobody can debug. FUTBOL keeps everything that *is* the football in one module, `src/sim.js`, and both programs import it.

That module stays deliberately plain:

- **Pure data in, pure data out.** It works on plain `{ x, z }` fields — the client passes its render-backed actors straight in, the server passes bare objects. No THREE.js, no DOM, no clock of its own.
- **Side effects go through hooks.** Where a rule needs a consequence the two ends handle differently — a sound, an event for the wire, a goal — the sim calls a hook and keeps its hands off the outcome:

```js
// sim.js — the rules call hooks; each authority decides what they mean
if (goalLine(ball)) world.hooks.onGoal(ball.pos.z < 0 ? 0 : 1);

// client (host fallback): claim the goal on the wire
// server: score it — seeing is scoring
```

- **Step-rate independent integration.** The server ticks at 20 Hz against the sandbox's 50 ms timer floor; clients render at 60–120 Hz. Solve your movement exactly over the step rather than accumulating per-frame adds, or the two ends settle at different speeds and the prediction argues with its authority for as long as a key is held:

```js
// dv/dt = drive − k·v, solved exactly over dt: lands in the same place
// whether it is asked 20 or 120 times a second.
const fade = Math.exp(-dt * k);
const settle = drive / k;
vel = settle + (vel - settle) * fade;
```

The server sandbox takes **one file** — no `import`, no `require` — so the sharing happens at build time. FUTBOL's `npm run build` runs esbuild over `src/referee/referee.js` and flattens it plus `sim.js` into `dist/server.js`, which is what a push ships:

```js
// scripts/build-server.mjs
await build({
  entryPoints: ["src/referee/referee.js"],
  outfile: "dist/server.js",
  bundle: true,
  format: "iife",
  target: "es2017",
  minify: false, // readable next to its logs
});
```

## The wire protocol

Three kinds of message carry the whole match. Keep each inside the 8 KB value budget — pack arrays, round numbers:

**Snapshots** — authority → everyone, at the tick rate (20/s from the server). Everything needed to draw the pitch: every actor's position, velocity, facing, timers, which seat it is and the last input sequence it heard, plus the ball, score, and phase. One extra field matters more than all of them: the server marks its snapshots.

```js
server.send({
  k: "s",
  sv: 1,              // the mark — "this stream is the server's"
  p: "playing", t: 61.4, sc: [1, 0],
  s: [ /* one packed array per actor */ ],
  b: [x, y, z, vx, vy, vz],
  ev: ["goal:0"],     // one-shot moments for followers to replay
});
```

The mark is the entire handshake. The first `sv` snapshot flips every tab — the host's included — into a follower; when the marked stream goes quiet, the host resumes simulating. The wire cannot tell the two authorities apart except by the mark, **which is the point: one protocol, two places it can run.**

**Inputs** — followers → authority, sent when the input *changes* rather than on a metronome (no faster than every 60 ms, and every 100 ms regardless, so a held key still says it is alive). Include a sequence number; the ping section below is built on it:

```js
net.send({ k: "i", x: stick.x, z: stick.z, q: seq++ });
```

On the server, an input drives whatever seat the team sheet assigned that player — never whatever the packet asks for — and a stale input stops driving its actor after ~700 ms, because a shirt sprinting on a dead man's key is worse than one standing still.

**Actions** — discrete moves (FUTBOL has exactly one: the tackle) announced by the client as they happen and **judged by the authority** with the same rule the client predicts them with. Whether a lunge connected arrives a round trip later; the animation plays locally and immediately.

## The server's tick

```js
// referee.js — 20 Hz against the sandbox's 50 ms timer floor
sim.timer = server.setInterval(stepSim, 50);

function stepSim() {
  const now = Date.now();
  // A stalled callback integrates as a capped step, not one giant leap.
  const dt = Math.min(0.1, (now - sim.lastStepAt) / 1000);
  sim.lastStepAt = now;

  for (const actor of world.live) applyRemoteInput(actor, dt, now);
  for (const actor of world.live) {
    advanceTimers(actor, dt);
    integrateActor(actor, dt);
    interactActor(world, actor);   // the rules: contacts, knockdowns
  }
  resolveActors(world.live);
  updateBall(world, dt);

  sendSnapshot();
}
```

Details that earn their keep:

- **The clock ticks on the server** rather than publishing a deadline. A deadline is read against each player's own clock, and a device with a wrong clock would show a wrong match.
- **A leaver is an actor off the pitch** — drop their shirt from the sim on `playerleave` and the snapshots say so by omission. No extra bookkeeping if each packed actor names its own seat.
- **State writes are coalesced.** The scoreboard publishes once a second and directory-style state on a ~120 ms timer, well inside the ~30 writes/s budget; the smooth clock rides the snapshots instead.

## Prediction on the client

Followers do three different things with a snapshot, and the distinction is what makes the game feel right:

1. **Your own actor is predicted.** Keys apply locally through the same shared sim, then the actor eases onto the authority's version. The keys stay crisp; corrections are a blend, not a snap.
2. **Everyone else is dead-reckoned** between snapshots — carried forward on the same drag constants the sim exports. That agreement matters: extrapolating a released actor at snapshot velocity in a straight line walks it well past where the physics stopped it.
3. **Moments are replayed from events.** Contested outcomes (a tackle connecting, a goal) are never predicted — they arrive in `ev` and play out when the authority says so.

### Subtracting the ocean

A snapshot describes a moment that had already happened when it was written. Read it as the present and the whole pitch sits a round trip in the past — a couple of hundred milliseconds on a worldwide channel. The fix is to measure the loop and carry the world forward by it:

- Every input carries a sequence number `q`; the authority echoes the last one it acted on in each actor's snapshot row.
- The gap between sending `q` and reading it back — both ends off the guest's *own* clock, so nothing needs agreeing about what time it is — is the whole round trip: uplink, the authority's wait for its next tick, downlink.
- Everything the authority owns is projected forward by its share: one downlink for the ball and the opponents, the full trip for your own actor, which the authority has not even heard about until an uplink ago.

FUTBOL relays each player's reading back down in the snapshots so every screen can print a ping card for the whole pitch — the only place a guest can learn about another guest's connection.

## The fallback ladder

A server can crash, run over budget, or be hot-swapped by a publish — and Portals keeps the session going **without** it. A server sim that takes the game down with it has failed at its one job, so the design is a ladder, each rung a working game:

1. **Server simulates** (it heard the team sheet): everyone follows `sv` snapshots, goals are witnessed.
2. **Server referees only** (booted mid-match by a hot swap, so the actors died with the old instance's memory): the host tab simulates exactly as before the server existed, while the server still owns the clock and the score — and a goal claim must be consistent with the ball the host was publicly showing everyone a moment earlier. That does not make a cheating host impossible; it makes one *visible*, and unable to invent a scoreline outright.
3. **No server at all** (a draft that has never been published, a crash): clients fall back to fully peer-run — host-owned clock, peer-kept room list. This is the ordinary state of every unpublished game, which is why the rung must stay working.

Two mechanics make the ladder walkable:

- **Silence is the signal.** When the marked stream stops, guests go back to believing unmarked snapshots after a beat and a half, and the host starts simulating half a beat later — in that order on every tab, both measured from the same last-marked snapshot, so the guests are already listening when the host starts talking.
- **Boot resumes shared state.** Shared state outlives the script, so a swapped-in server reads its own keys back (`server:match`, `server:rooms`), prunes them against the live roster, and carries on — being hot-swapped mid-match must not hand anybody a fresh 0-0.

```js
// referee.js boot — resume, prune, republish
const was = server.getState("server:match");
if (was && (was.phase === "play" || was.phase === "count")) {
  match.scores = [...was.scores];
  match.time = was.time;
  startTick();   // referee the host's sim until the next kick-off deals a sheet
}
```

## Budgets that shape the design

The [sandbox limits](https://portals.to/documentation/advanced-tooling/server-scripts#limits) are not commentary — each one is load-bearing here, and FUTBOL's probe run confirmed them in practice:

| Budget | Measured | What it forces |
| --- | --- | --- |
| Timer floor 50 ms | exactly — shorter intervals clamp | the 20 Hz tick, and 20 snapshots/s |
| Broadcasts ~60/s | 20/s sustained clean; 40-message bursts intact | one snapshot per tick, events piggybacked on it |
| State writes ~30/s | — | coalesce: scoreboard 1/s, directories on a 120 ms timer |
| 8 KB per value | six packed actors fit with room to spare | arrays over objects, two-decimal rounding, caps on list sizes |
| CPU ~50 ms/callback | one 3v3 physics tick ≈ 0.7 µs | plain arithmetic is nowhere near the ceiling — architecture is the cost, not compute |
| Server RTT | ~25 ms regional, ~230 ms from the wrong side of a global channel | regional channels for matches; `global:` only where meeting matters more than ping |

## Test it without publishing

Server code otherwise iterates through publish and editor preview, which is too slow a loop for physics. The built `server.js` is a plain script that talks to one `server` global — so mock the global, run the file in `node:vm` on a **virtual clock**, and ninety seconds of football land in milliseconds, deterministically:

```js
// test-referee.mjs (abridged) — the file under test is byte-for-byte what ships
const server = {
  on: (ev, fn) => handlers[ev] = fn,
  send: (m) => sent.push(m),
  setState: (k, v) => state[k] = v,
  getState: (k) => state[k],
  players: () => roster,
  setInterval: (fn, ms) => queue(fn, ms, true),
  setTimeout: (fn, ms) => queue(fn, ms, false),
  clearTimer: (id) => timers.delete(id),
  kick: () => {}, log: () => {},
};
vm.runInContext(readFileSync("dist/server.js", "utf8"), context);

ref.message({ k: "seats", sheet }, host);
ref.message({ k: "mkick" }, host);
ref.advance(3200);                       // countdown → play, virtually
expect("snapshots marked sv", ref.lastSnapshot().sv === 1);
```

Drive the lobby, a simulated match, the fallback and a hot swap through it; exit non-zero on the first failed expectation. The one thing the mock must also fake is `Date.now()`, since the tick measures real elapsed time against it.

## Debugging live

- The **editor preview** runs your draft's `server.js` against a real session — two preview panes (the 2p device) are a two-player match on one screen.
- **Server logs** from both your script (`server.log(...)`, shown as `[script]`) and the platform around it (session starts, crashes, `script fatal` lines) appear in the editor: open the preview's bug icon and switch to the **Server logs** tab. Fleet failures light the icon on their own.
- **Local dev sessions** (via a [dev token](https://portals.to/documentation/advanced-tooling/multiplayer-and-voice#test-multiplayer-locally)) get the **published** server script, never the one on your disk — so a draft has no server there, which exercises exactly the fallback rung your design must keep working. `net.getState("server:...")` in the console tells you which world you are in.

## The shape of it

A server sim is not more simulation — it is the same simulation, run in a fairer place, with everything around it built to lose that place gracefully:

1. One shared, pure, step-rate-independent sim module; the server bundle is built, not copied.
2. Snapshots down, inputs and actions up; the server's stream differs by one mark.
3. Predict yourself, dead-reckon the rest, replay contested moments from events.
4. Measure the loop with echoed sequence numbers; project the world forward by it.
5. Fall back rung by rung — referee without simulating, then host authority — and resume from shared state on every boot.
6. Test the shipped file on a virtual clock; read `server.log` in the editor's Server logs tab.
