# Horizon Server: design

Horizon Server is a Minecraft: Java Edition 26.3 server for Horizon players first, with vanilla Java clients
welcome. It aims to be the fastest server we can build while keeping vanilla's rules exactly. Two bars:

1. **Better than Pumpkin on every number we can measure**: players at 20 TPS, MSPT p99, memory, bandwidth and
   join time, with Pumpkin's released binary run as a black box against the same bots.
2. **Vanilla-exact where Pumpkin isn't.** Gameplay is Horizon's engine, checked by its 324 vanilla parity
   harnesses (each one runs the real 26.3 jars and Horizon on the same inputs and compares the results).

This page holds the measured baseline (section 1), the targets (2), what this repo takes from
[horizon-rust/horizon](https://github.com/horizon-rust/horizon) and what it writes fresh (3), the design of each
part with the measured cost it removes (4 to 8), and the plan (9). The supervisor, `server.toml`, the panel,
backups and the control API are already designed and mostly built in horizon's `docs/server.md` and
`docs/hosting.md`. This page doesn't repeat them; it says where they go (section 3.3).

**Originality.** Nothing here comes from Pumpkin's, Paper's, Folia's or any other server's code; none of it
was read. About other servers we use only what a user can see: their documentation, their generated config
file, and black-box runs of their released binaries. Vanilla's behaviour comes from the 26.3 jars, as for the
rest of Horizon.

## 1. Baseline (measured 2026-10-29, Mac mini M4 Pro, 14 cores, 64 GB)

**Servers.**
- Horizon: `horizon-client --dedicated` at horizon `668419b92`, `fast` profile. This is main *before* the three
  tick fixes still in review (#3815, #3834, #3847).
- Vanilla: 26.3 `server.jar` on Temurin 25, `-Xms4G -Xmx4G`.
- Pumpkin: release `0.2.1+26.3-26.51`, `pumpkin-ARM64-macOS`, SHA-256 checked against the release's
  `checksums.sha256`.

**Same settings for all three.** Seed 12345, a fresh world, view and simulation distance 10, offline mode, creative
with forced game mode, max players 200, difficulty easy, bound to 127.0.0.1. Pumpkin also ran with its
Bedrock listeners, LAN broadcast, plugins and telemetry turned off, through its own `pumpkin.toml`.

**Load.** `horizon-bots --behaviour mixed` (60 % walk within 48 blocks of spawn, 20 % explore outwards at
22 blocks/s, 20 % idle). One bot joins every 0.5 s, then a 30 s warm-up, then a 60 s window. The bots don't fight:
`horizon-bots` has no combat yet (issue S-1, section 9.3).

**How each number is taken.** The server measures itself, the same way on all three:
- MSPT: the console's `tick query` every 5 s, which reports the average, P50, P95 and P99 over the last
  100 ticks.
- TPS: `time query gametime` at the start and end of the window, as game ticks per wall-clock second.
- Memory: RSS, sampled every second.
- CPU: the server process's CPU seconds divided by the window.
- Bandwidth: the bots' loopback bytes, from macOS `nettop`, per player.
- Join time: from the bot's connect to its spawn position.

The driver, `sn_run.py`, comes into this repo as `bench/` with issue S-2.

**Caveat.** The Mac mini is shared. Its load average was 57 to 132 on 14 cores during the runs (other agents' builds), and
the bots ran on the same machine. So a tick can include waiting for a core, and the numbers carry more noise
than a dedicated machine's would. All three servers ran interleaved under the same load (10, then 50, then
100 bots, each in the order Horizon, vanilla, Pumpkin). Treat differences under about 20 % as noise.

| Bots | Server | TPS | MSPT avg | p50 | p99 (median / worst 5 s) | CPU cores | RSS MB | KB/s per player | KB per chunk | Chunks sent | Join ms p50 / max | Pulled back |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| 10 | Horizon | **20.0** | **11.3** | 10.9 | **17.7 / 23.6** | 1.01 | 1165 | 87.9 | **3.55** | **15 272** | 74 / **103** | 1 469 |
| 10 | vanilla | 16.7 | 41.1 | 7.8 | 243 / 352 | 0.79 | 3334 | 61.4 | 6.75 | 5 646 | 3 730 / 5 550 | 1 366 |
| 10 | Pumpkin | 20.0 | 23.0 | 16.6 | 87 / 143 | 1.20 | **538** | 46.3 | 4.50 | 6 397 | **55** / 1 980 | 0 |
| 50 | Horizon | **20.0** | **28.7** | 27.4 | **38.5 / 51.7** | 2.11 | 1535 | 76.5 | **3.64** | **65 431** | 93 / **249** | 12 141 |
| 50 | vanilla | 5.3 | 58.8 | 7.8 | 416 / 537 | 0.70 | 3466 | 38.4 | 7.43 | 16 115 | 4 791 / 15 082 | 2 577 |
| 50 | Pumpkin | 20.0 | 45.8 | 40.2 | 151 / 283 | 1.08 | **869** | 34.5 | 3.77 | 28 427 | **54** / 4 075 | 0 |
| 100 | Horizon | **14.3** | 79.2 | 83.8 | **108 / 138** | 4.42 | 2054 | 95.4 | 3.70 | **160 042** | 83 / **244** | 14 638 |
| 100 | vanilla | 2.2 | 108.3 | 63.5 | 539 / 556 | 1.37 | 3699 | 45.0 | 5.18 | 53 786 | 933 / 10 524 | 14 790 |
| 100 | Pumpkin | 13.6 | **78.6** | 73.3 | 231 / 348 | 2.54 | **1421** | 51.6 | **3.61** | 88 625 | **43** / 494 | 0 |

Idle RSS after start (fresh world, nobody on): Horizon 760–780 MB, vanilla 1 550–1 630 MB, **Pumpkin 32–35 MB**.
Start to ready, fresh world: Horizon 9.2–14.1 s, vanilla 25–34 s, **Pumpkin 0.1–0.2 s** (it generates no spawn
area before it accepts players). The driver and the raw JSON come into this repo with issue S-2.

**What it says.**
- **Up to 50 players**, Horizon and Pumpkin both hold 20 TPS. Horizon's tick is about 40 % shorter (28.7 against
  45.8 ms at 50) and its p99 is 4 to 5 times lower (51.7 against 283 ms at worst). Over the same window it also
  sent 2.3 times the chunks. Vanilla falls to 5.3 TPS at 50.
- **At 100 players nobody holds 20 TPS.** Horizon runs at 14.3, Pumpkin at 13.6, vanilla at 2.2. Horizon still has the
  lowest p99 (138 against 348 and 556 ms) and sends 1.8 times Pumpkin's chunks. Two of the Horizon fixes in review
  (#3815, #3834) took a 100-bot run from 14.5 to 17.8 TPS (different mob counts, so read it as a direction).
- **Memory is Pumpkin's clear lead.** It idles at 32 MB against Horizon's 760 MB, and uses 1.4 GB against 2.05 GB
  at 100. Vanilla's heap alone is 4 GB.
- **Bandwidth per chunk is the same** for Horizon and Pumpkin (3.6–3.7 KB), and 5–7 KB for vanilla. Horizon's
  higher KB/s per player is the extra chunks. So bandwidth can't drop much further on vanilla's format. The Horizon
  fast path (section 8) is where it can.
- **Join.** Pumpkin's median is lowest (43–55 ms), but its worst reached 4.1 s at 50 bots. Horizon's stays at
  74–93 ms with a worst of 244 ms. Vanilla takes 0.9 to 4.8 s.
- **The work isn't fully equal.** Pumpkin never pulled a bot back (0 against ~14 600 for Horizon and vanilla at
  100): it doesn't run vanilla's "moved too quickly" check, so its explore bots never wait for it. The mob counts
  of vanilla and Pumpkin can't be seen from outside; Horizon had 1 834 mobs at the end of the 100-bot window.
  Vanilla's self-reported MSPT leaves out the chunk work it runs between ticks, which is why its TPS (from game
  time) is the number to read, not its MSPT.

**Where Horizon's tick goes at 100 bots** (macOS `sample` of the tick thread, from the profiling behind #3815, #3834 and #3847, same machine):

| Cost | Share of the tick thread | Status |
|---|---|---|
| Mobs: AI and movement | 12–19 % | flat profile (collision scan, AABBs, chunk reads) |
| Natural spawning (`spawn_for_chunk`) | 10–12 % | already tight per position |
| Hub (`tick_lan`): tracking, chunk send, play state | 15–22 % | section encoding is already on helper threads |
| Light views of proto chunks on the tick thread | 18.7 % → 0.3 % | **fixed** in #3815 |
| Scheduled ticks: one `GameLevel` per due tick | tick_blocks 15 → 10 ms at 50 bots | **fixed** in #3834 |
| Spawners: a level built with nobody in range | 1.6 % → 0.5 % | **fixed** in #3847 |
| Random ticks | 7–10 % | vanilla's own block reads |
| Fluids (`slope_distance`, `Vec` allocations) | ~4.5 % | open |

Everything above runs on **one thread**. The worker pool (generation, light, section encoding) has cores to
spare: the whole server used 1.0 to 4.4 of the 14 cores. Pumpkin's real leads over Horizon are memory and start
time (section 7). On scale the lesson is simple: the next 10× can't come from shaving one
thread. It comes from running the tick on more cores without changing what vanilla would do.

## 2. Targets

Each one is measured with the bench from section 1, on a dedicated 8-core machine (the bots on another
machine), with `mixed` bots, view distance 10, and a pre-generated world plus a fresh-world pass.

| Number | Pumpkin today | Target | Why it is reachable |
|---|---|---|---|
| Players at 20 TPS (walking, one level) | 50 (13.6 TPS at 100) | **200** on 8 cores | parallel phases (4.2) + the hub off the tick thread (5) |
| Players at 20 TPS spread over 3 dimensions | — | **300** | levels in parallel (4.1) |
| MSPT p99 at 100 players | 231 ms (worst 5 s: 348) | **< 35 ms** (p50 < 15) | the same, plus saving off the tick thread (5.5) |
| RSS idle, fresh world | 32 MB | **< 100 MB**. One of the two numbers where we may stay behind: vanilla's registries, worldgen and loot tables are loaded up front | headless engine (7) |
| RSS at 100 players | 1.42 GB | **< 1.0 GB** | per-player state measured and capped (7) |
| Bandwidth, walking vanilla client | 3.6 KB per chunk | **≤ vanilla's bytes per chunk**, compression level picked by measurement | 5.3 |
| Bandwidth, walking Horizon client | — | **−50 %** against vanilla clients on the same server | the fast path (8) |
| Join to spawn, p99 | 43 ms p50, 494 ms max (4.1 s at 50) | **< 300 ms** on loopback, **< 2 s** for a full view | 5.2 |
| Start to ready | 0.1–0.2 s (no spawn area generated first) | **< 1 s**, the other one | headless engine (7), spawn area on the workers |
| Vanilla parity | not checked | **every harness that covers server code passes** in CI at the pinned horizon commit | 3.1 |

The 200-player target fits within one tick: at 100 bots, the tick thread's CPU time per tick was 31 ms
(with #3815 and #3834). That leaves 8 cores × 50 ms = 400 core-ms per tick for 200 players,
if most of the work moves to parallel phases.

## 3. What this repo is, and what it takes from horizon

### 3.1 Gameplay stays in horizon, as a library

Gameplay is about 300 000 lines of vanilla-exact Rust in `horizon-client`: `game/` (the game binary's
`impl Game`) is 133 k, the lib's `mobs/` 62 k, `lan/` 15 k and the flat lib modules 90 k. On top of that comes
`horizon-world` (124 k). The 324 harnesses guard it. Writing gameplay fresh for a server would throw away the
one thing Pumpkin can't match. Forking it would make two copies drift. So:

- **Decision: gameplay has one copy, in horizon.** Single player (the server thread), LAN, `--dedicated` and
  Horizon Server all run the same code. A gameplay fix lands once in horizon and serves all of them.
- **What has to move:** the server half of `Game` is in the game *binary* (`crates/horizon-client/src/game/`),
  so no other crate can call it. The crate split (horizon roadmap priority 1; #3186/#3187 moved rendering to
  `horizon-client-render`) continues with **`horizon-sim`**: a library crate holding `Game` minus the window.
  - `game.rs` and `game/` move into it. Window, input, audio and drawing stay behind a `client` feature, or move
    back to the bin.
  - `horizon-client` becomes a thin bin on top of it.
  - `horizon-sim` exposes a narrow server API: open a universe, `tick_server()`, accept a connection, console
    lines, status. That is what `game/dedicated.rs` uses today.
  - It's done like the earlier moves: files move without being edited, `pub(super)` becomes `pub(crate)`, and
    every PR passes the full `pr_ready` with parity. That way the game can't break silently.
  - The order is in 9.1 (W2). It's mechanical but large: a series of move PRs like B8 and B9.
- **How this repo depends on it:** a git dependency on `horizon-rust/horizon`, pinned to a commit (`rev = …`),
  for `horizon-core`, `horizon-protocol`, `horizon-world`, `horizon-items`, `horizon-command`,
  `horizon-server-files` and `horizon-sim`.
  - A daily bot PR bumps the pin.
  - This repo's CI then runs, against the pinned commit, the harnesses that cover server code
    (`dedicated_server`, `online_mode`, `secure_chat`, `server_files`, `server_entities`, `server_player_*`,
    `server_dimension_players`, `user_cache`, and `parity.py --changed` over the bump's diff), plus this repo's
    own tests and bench smoke run.
  - A bump that fails isn't merged. Nobody chases horizon's main by hand.

### 3.2 What this repo writes fresh

Only server-scale machinery that has no vanilla rule inside it, and so can't drift from vanilla:

| Crate | Owns |
|---|---|
| `hzs` (binary) | the server program: `server.toml`, the console, starting `horizon-sim` in-process |
| `hzs-net` | the network front: accept, login and configuration handed to horizon's login code, per-player send queues and budgets, I/O threads, compression and encryption off the tick thread (section 5) |
| `hzs-sched` | the parallel tick driver: levels in parallel and the phase pipeline inside one level (section 4). It drives `horizon-sim`'s level API and doesn't own rules |
| `hzs-fast` | the server half of the Horizon fast path (section 8). The client half lives in horizon, next to the client's own decoder |
| `hzs-ops` | the supervisor, panel, control API and MCP, backups, metrics, the Kira bridge (section 9.2) |
| `bench/` | the driver from section 1, Pumpkin/vanilla/Paper runners, report and comparison table |

Engine changes that every mode benefits from land in **horizon**, with this repo's bench as their target:
levels that can tick on separate threads, the entity change log for the tracker, chunk snapshots for
off-thread saving. Single player's server thread and LAN hosts get them too. This repo never patches
gameplay. It can only ask for, and review, a horizon PR.

### 3.3 What becomes of horizon's `horizon-server` crate and `--dedicated`

- **`crates/horizon-server` (the supervisor) moves here** as `hzs-ops`, history included (`git filter-repo` on
  the crate's path). It links no game code already, so the move is clean.
  - The launcher's hosting page and `install.sh` switch to this repo's releases.
  - Horizon then deletes its copy in one PR, once this repo has shipped a release that the launcher uses.
  - Until then, fixes land in horizon and are cherry-picked here. There is no second line of development.
- **`--dedicated` stays in horizon-client**, as the reference server of the harnesses. `test_dedicated_server_parity.py`
  and the server harnesses run against it, and LAN shares its hub.
  - Once `horizon-sim` exists, it is ~100 lines over `horizon-sim`'s server API, the same API `hzs` calls, so
    it costs nothing to keep.
  - `hzs` is the product: the release players install for hosting.
  - `--dedicated` keeps vanilla's single-threaded order. It's the baseline every parallel phase is compared
    against (4.3).

## 4. Ticking on more cores

Vanilla ticks one level at a time (`MinecraftServer.tickChildren`: each `ServerLevel.tick` in order, then the
connections). Inside a level it ticks in a fixed order:
1. time and weather;
2. scheduled block and fluid ticks, in their queue's order;
3. raids;
4. chunks: random ticks and natural spawning, in a shuffled chunk order that draws on the level's random;
5. block events;
6. entities, in list order;
7. block entities, in list order.

Horizon follows the same order (`game/levels.rs` `tick_server`, which swaps each level into the one `Game`).

### 4.1 Levels in parallel (exact)

Each level ticks on its own thread. Three things stay serial, in vanilla's level order, at a barrier after the
parallel phase:
- anything that crosses levels: entities and players changing dimension, `execute in`, ender pearls, portal
  search into the other level's chunks;
- shared server state: scoreboard, advancements and statistics, game rules, boss bars, the player list,
  command functions and `/schedule`, the world border's sync across levels;
- the connections.

A level's tick writes those effects to an ordered log, which the barrier applies in vanilla's order.

The one place where vanilla's order is visible across levels is an entity that leaves a level that ticks
earlier for one that ticks later: in vanilla it can tick in both levels during one server tick. The barrier
replays that case serially (the arriving entity's tick runs in the destination's serial slot). A harness that
throws entities through portals every tick checks it against the serial path, tick by tick.
- **Removes:** the levels' sum becomes their maximum. It helps only when players are spread over dimensions;
  with everyone in the overworld it gains nothing. That's why 4.2 exists.
- **Engine work (horizon):** levels already have separate state (`levels.rs` parks the inactive ones). The
  work is making `Game`'s per-level part a value a thread can own, with the shared part behind the barrier log.

### 4.2 One level on more cores (exact by default)

Vanilla's order inside a level can't be split into regions freely. Random ticks and natural spawning draw from
one level random in chunk order. Scheduled ticks run in one global order. Entity IDs come from one counter.
Mobs see the moves of the mobs that ticked before them. Running regions in parallel changes these draw
sequences. No player can see the difference, but the parity harnesses do, and so would anyone comparing seeds
with vanilla. Two modes:

- **`tick = "vanilla"` (default): exact.** Vanilla's order of effects is kept. Only work that doesn't depend
  on that order runs in parallel:
  1. *Per-entity pure work, run ahead in parallel, committed in order.* Pathfinding (`PathFinder` over a
     read-only snapshot of the level), sensor scans (nearest player, nearest item, line of sight), collision
     shape gathering for `Entity.move`, and brain sensors run on all cores at the start of the entity phase.
     None of this draws a random number, and an entity's own draws use its own random (`Entity.random`),
     so no level draw moves. The serial
     entity loop then uses each result if nothing it read changed since. It checks the blocks and entities in
     the result's read set against a per-section change counter, and recomputes in place when they did. Mobs
     that don't move or path (most of them: 1 400+ mobs against 100 players in section 1) commit
     without any recompute. **Removes:** the AI and movement share (12–19 %), minus the serial commit.
  2. *Hub work after the tick.* Entity tracking, chunk sending and packet encoding read the level once it has
     ticked, so they run in parallel per player while the next tick's serial phases start (section 5).
     **Removes:** the hub's 15–22 %.
  3. *Random ticks: block reads batched per chunk.* The draws stay serial and in order, but each chunk's
     sections are pinned once instead of a map lookup per read. **Removes:** part of the 7–10 %.
  4. *Natural spawning: a per-tick grid of "players near this chunk" answers*, built in parallel. The draws
     stay serial. **Removes:** ~2 % (the per-chunk map entry and `Vec` measured while profiling #3847).
- **`tick = "regions"` (opt-in, off by default): parallel regions, vanilla rules, not vanilla's random
  sequences.**
  - Groups of loaded chunks more than 2 chunks apart, with no entity, ticket or redstone link between them,
    tick on separate threads. Each group gets its own random, split from the level's, plus its own entity ID
    range and scheduled-tick queue. Groups merge and split at the barrier as players move.
  - Every rule stays vanilla's (the harnesses that compare outcomes still pass). What changes is only which
    random number a given chunk draws. The config says it in those words.
  - Off by default, like every Horizon change to vanilla behaviour (horizon's `docs/design-system.md`).
- **What stays serial in both modes:** cross-level effects (4.1), commands and functions, the player list, chat
  order, advancements, the scoreboard, raids (one per village, but their spawning draws the level random),
  block events (vanilla runs them to a fixed point in one queue), and the order of packets to one player.

### 4.3 How exact is checked

Every parallel phase has a switch that runs it serially (`HZS_SERIAL=entities,hub,...`). A harness runs
the same fixture both ways and compares a digest of the level every tick: block states, entity positions and
NBT, the random's state. On top of that, the horizon harnesses that cover server code run on `hzs` with the
default mode. A phase that can't pass that comparison doesn't ship in `tick = "vanilla"`.

## 5. Network and I/O off the tick thread

**What already exists in horizon's hub:**
- section encoding once per chunk, on helper threads (#623, `lan/chunks.rs`);
- chunk frames compressed once for every player (`SharedFrames`, `lan/conn.rs`);
- vanilla's chunk batch pacing (`ChunkBatchReceived`) and its move checks (#586);
- a reader and a writer thread per connection (#649 made the writer wake once per tick).

#3036 (hectoreccles) moves batch bodies to helper threads. W3 builds on it rather than redoing it.

### 5.1 I/O threads instead of two threads per player

200 players means 400 threads today. `hzs-net` uses a fixed pool of I/O threads (one per two cores) on
`kqueue`/`epoll` through `mio`, with one bounded queue per player. The tick thread only pushes packet bodies.
Framing, compression, encryption and the socket writes happen on the pool. A player whose queue passes its
budget gets its chunk sending throttled first (as vanilla's batch pacing does). Vanilla's timeouts and
disconnects stay unchanged. **Removes:** thread switches and lock traffic per packet. Bench: CPU per player at
200 idle bots.

### 5.2 Join

Login and configuration (registries, tags, known packs) are the same for every player. They are encoded once
per server start and sent as prepared frames. The first chunks around the spawn come from the chunk cache
(5.3), not the generator. **Removes:** Horizon already joins in 74–93 ms p50 on loopback, worst 244 ms. Pumpkin's median is lower (43–55 ms) but its worst reached 4.1 s at 50 bots. Prepared frames bring the median under Pumpkin's and keep the worst flat.

### 5.3 Chunk cache and compression level

- A chunk's encoded and compressed packet is kept until a block in it changes. Block changes go to the players
  holding it as vanilla's section updates, and drop the cached frame. Walking players who re-enter chunks then
  cost a memcpy, not an encode.
- Compression uses vanilla's threshold (256) and the level that gives the best bytes per core-ms on the bench.
  zlib 6 is what Horizon uses today; Pumpkin's config defaults to 4. Both are vanilla's wire format.
- **Removes:** the hub's encode cost for re-sent chunks. In section 1, Horizon's walking bots received
  each chunk several times, and those re-sends are where most of the bandwidth went.

### 5.4 Entity tracking as deltas

- Today the tracker rebuilds each tracked entity's state per tick (`lan/tracking.rs`, `Entities::tick`,
  `mob_meta` ~3 % of the tick thread).
- Instead, the engine writes a change log as it changes an entity (position, rotation, metadata index, equipment,
  passengers). The tracker turns each changed entity into packets once, and sends them to every player who
  sees it.
- A grid of players per chunk section finds who sees it.
- Vanilla's update cadence is kept exactly, because clients interpolate on it: `ServerEntity.sendChanges`,
  with its update interval, its relative moves, its 60-tick position check, and its rotation and head
  thresholds.
- **Removes:** the per-viewer encode (×players) and the full state rebuild for entities that didn't change.

### 5.5 Saving and light

- Light already runs on the worker pool. Since #3815 the tick thread no longer builds proto-chunk views
  (18.7 % → 0.3 %).
- Saving: at autosave (every 6 000 ticks) and on unload, a chunk is snapshotted on the tick thread (a copy of
  its sections and block entities' NBT, which is cheap). The workers serialize, compress and write the region
  files, through vanilla's `RegionFile` format (`horizon-world`).
- The bench gets an autosave window so that the spike is measured, not guessed.

## 6. Who can join

`server.toml`:

```toml
[clients]
allow = "any"        # "any" (vanilla Java and Horizon) or "horizon" (Horizon clients only)
```

- With `"horizon"`, the server refuses a client that didn't send brand `horizon` and the fast-path hello
  (section 8) in configuration. The refusal message names the Horizon launcher.
- **Be honest about what this is:** a convenience filter, not security. Horizon's client is open source, so a
  modified client can claim to be Horizon. It keeps vanilla players from joining a server they can't play
  fully, nothing more.
- **The real gate stays online mode.** Each player must own a Minecraft account, checked by the session
  service (Mojang's rules: no play without an owned account). The whitelist and bans apply on top of it.
  - `hzs` runs offline mode only on loopback (benches and tests). On any other address it refuses to start
    with `online-mode=false`. This follows horizon's own rule: no play without an owned account.
  - No other auth providers, ever.
  - The official server runs offline mode today, so moving it to `hzs` means moving it to online mode. That is
    Ludovic's decision (9.2).

## 7. Memory and start

- Pumpkin idles at 32 MB and Horizon at 760 MB. The difference is mostly what the dedicated path
  loads only for a window: the HUD atlas, the item renderer, block-entity and entity models, the panorama,
  the fog and music tables, and the mesh tables. horizon's `docs/server.md` §2 already decided the headless
  engine. In `horizon-sim` the dedicated path skips all of them.
- After that, every per-player and per-chunk structure gets a measured size in the bench report (bytes
  per loaded chunk, per tracked entity, per player). The target is under 1 GB at 100 players.
- A loaded chunk's lighting and heightmaps are kept packed (#3253's storage primitives, by hectoreccles, are
  the base).
- Start: resources load in 1.6 s today. Without the client's tables, and with the registries mapped from one
  prepared file, the target is under 1 s.

## 8. The Horizon fast path

Vanilla's protocol stays the default and is never changed for a vanilla client. Horizon clients ask for more,
in configuration:

1. The client sends brand `horizon` (it does today, `net.rs`), then a `horizon:hello` custom payload:
   protocol revision plus a capability bit set.
2. The server answers `horizon:hello` with the subset it accepts. Horizon's server already answers a Horizon
   brand with `horizon:quiet` (`lan/conn.rs`), and this is the same mechanism.
3. A client that sends no hello, or a capability the server doesn't know, gets vanilla. The fast path is
   always negotiated, never assumed.

| Capability | What it changes | Measured cost it removes |
|---|---|---|
| `chunk-zstd` | chunk packets in Horizon's own section encoding with a shared zstd dictionary, as one `horizon:chunk` payload | bytes per chunk (section 1: 3.6–3.7 KB for Horizon and Pumpkin alike with vanilla's format) |
| `far-terrain` | the server sends LOD tiles beyond view distance from the saved world, at low priority under the send budget. The client's `lod.rs` draws them today from its own generator | the client generating far terrain itself. Players see the server's real world, builds included |
| `entity-batch` | one packet per tick with every tracked entity's change, quantized as vanilla's deltas are | per-packet overhead: dozens of small moves per tick per player |
| `server-stats` | TPS, MSPT and the player's own ping, for the client's debug overlay | none (a feature) |

Each capability has its own harness. A Horizon client with the capability, and one without, must end up with
identical client state (`horizon-ctl state` digests). The client half lands in horizon behind the same
negotiation, so a Horizon client still plays on every vanilla server.

## 9. Plan

### 9.1 Workstreams (in order; each is a few PRs under ~500 lines, with its target and bench)

| # | Workstream | Repo | Target | Proof |
|---|---|---|---|---|
| W0 | **Bench**: driver, Pumpkin/vanilla/Paper runners, pre-generated world, bots on another machine, report | here | ±5 % between two runs of the same binary | the report, published |
| W1 | **Skeleton**: workspace, `hzs` running `--dedicated`'s loop through horizon's crates pinned by `rev`, CI with the server harnesses, daily pin bump | here | `hzs` passes `dedicated_server` and the server harnesses | CI |
| W2 | **`horizon-sim`**: `game/` into a lib, `client` feature, server API | horizon | the game unchanged: full `pr_ready` + parity per PR | horizon CI |
| W3 | **Network front**: I/O pool, prepared login frames, chunk cache, compression level by measurement (with #3036) | here + horizon | join p99 < 300 ms, CPU per idle player halved | bench |
| W4 | **Tracker deltas**: the engine's change log, encode once per entity | horizon | `tick_lan` share ÷ 3 at 100 bots | bench + `server_entities` harness |
| W5 | **Parallel levels**, then the exact entity phase (4.2.1), then `tick = "regions"` | horizon (engine) + here (driver) | 200 players at 20 TPS on 8 cores | bench + the serial-vs-parallel digest harness |
| W6 | **Headless memory and start** | horizon | idle < 150 MB, start < 1 s | bench |
| W7 | **Fast path**: hello, `chunk-zstd`, `entity-batch`, `far-terrain` | here + horizon (client) | −50 % bytes for Horizon clients | per-capability harness |
| W8 | **Operations**: move the supervisor (`hzs-ops`), metrics endpoint (Prometheus text, loopback/tailnet only), scheduled restarts, Kira bridge | here | the official server's checklist (9.2) | an hour-long soak with 100 bots |

W0 and W1 come first: without a bench that repeats within 5 %, no tick PR can show its gain. W2 then gates
W4 to W6, and W3 and W8 can run beside it.

### 9.2 Before the official server can leave Pumpkin

horizon.bunchy7s.dev runs Pumpkin today, hosted by bunchy7s. Before it switches, we need from him (asked in
#mod-chat through the orchestrator):
1. **The plugins and features he relies on.**
   - The kira-pumpkin plugin's `/game/ask`. In `hzs`, a built-in `[kira]` bridge registers the command and
     calls Kira's API on a worker thread. Kira can also read status and players through the control API
     (`hzs-ops`, token on loopback).
   - Anything else he uses: permissions, proxies (Velocity?), the query port, management.
2. **Bedrock players?** Pumpkin also serves Bedrock. `hzs` is Java only. If Bedrock players are on the server,
   that is a decision for Ludovic.
3. **The world.** Pumpkin's config names its own chunk format (`[world.chunk] type = "pump"`). Either the world
   is exported to Anvil once (a converter, if the format is documented; we don't read Pumpkin's code), or the
   official server starts a fresh world.
4. **Operations:** backups he keeps, restart schedule, the log he watches, how he deploys (systemd, Docker).
5. **Online mode.** The server runs offline mode today, and `hzs` needs online mode on a public address
   (section 6). Every player then needs an account that owns Java Edition. That is Ludovic's decision.

The switch-over itself:
- `hzs` runs beside Pumpkin on another port for a week, with the same whitelist and ops;
- Kira's bridge is checked end to end;
- the 100-bot soak passes on his machine;
- then DNS moves.

### 9.3 Issues anyone can take (good first ones for a new contributor)

- **S-1 bots: a `fight` behaviour.** Bots in survival that attack the nearest mob and each other. Today's
  bench has no combat. In horizon's `horizon-bots`, the base of every bench.
- **S-2 bench: bring `sn_run.py` here as `bench/`**, with a `--pre-generated` world copy and a Markdown table
  out of the JSON.
- **S-3 bench: bots on another machine.** `--bots-via ssh host`, as horizon's `tools/server_bench.py` has.
- **S-4 skeleton: the workspace** with the pinned git dependencies and a CI job that builds them on Linux and
  macOS.
- **S-5 docs: an owner's page** comparing `server.properties` to `server.toml`, from horizon's
  `docs/hosting.md`.
- **S-6 bots: report bytes and join time** (`bytes_in`/`bytes_out` per bot from the connection's counters,
  join percentiles), so the bench doesn't need `nettop`.
