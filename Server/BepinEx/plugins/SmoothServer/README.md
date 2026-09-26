# SmoothServer

> **Actively maintained for Valheim 1.0.** Rebuilt for 1.0.7 on launch day and kept current as patches land;
> version 1.x of the mod is in the works. Bugs and ideas: [GitHub issues](https://github.com/MJensen01/SmoothServer/issues).
>
> Designed and thought up by **Nosferatu** from real pain points hit running a co-op server.
> Coded using Claude, tested by humans.

Networking and performance tuning for Valheim dedicated servers — **and, since 0.3.0, an optional
client half in the same DLL**.

The server half is the point of the mod and it still needs **zero client installs**:
`[General] EnforceClientMod` defaults to `false`, so vanilla players can always join. Install the
same package on players' machines as well and you additionally get packet compression, a client
send budget and a server-wide shared map.

## Installing

**Server admins** — install into the *server's* profile
(`BepInEx/plugins/SmoothServer/`). Everything under "Server modules" below starts working
immediately; nobody has to install anything. Config file: `Nosferatu.SmoothServer.cfg` in the
server's `BepInEx/config/`.

**Players** — install into your own profile only if the server also runs SmoothServer 0.3.0+.
`[General] Mode` is `Auto`: a dedicated server runs the server half, your game runs the client
half. You get compression, the client send budget and the shared map; server-owned settings are
pushed to you by the server.

**The package contains six DLLs**, not one: `SmoothServer.dll` plus `ZstdSharp.dll`,
`System.Memory.dll`, `System.Buffers.dll`, `System.Numerics.Vectors.dll` and
`System.Runtime.CompilerServices.Unsafe.dll` (the zstd codec and its dependencies — none ship
with Valheim or BepInEx). A mod manager handles this; hand-installers must copy all six.

## Mods this replaces — uninstall them first

| Uninstall | Replaced by | Where |
| --- | --- | --- |
| **CW_Jesse / BetterNetworking** | `Compression` + `SteamRates` + `SendQueueGuard` | server **and** clients |
| **Mydayyy / ServerSideMap** | `SharedMap` | server **and** clients |

Both must go from *every* machine. If BetterNetworking is still installed, `Compression` refuses
to patch (`FAILED(...)` in the module summary) and the rest of SmoothServer loads normally — it
will not break your server, it just will not compress. SharedMap imports an existing
`<world>.mod.serversidemap.explored` file once on first start, so no exploration is lost.

## Server modules

Each module has its own `Enabled` toggle in its config section.

- **Telemetry** `[Telemetry]` — periodic fps / frame-time / ZDO-rate log line. No gameplay effect.
- **FrameRate** `[FrameRate]` — raises the dedicated server's Unity frame cap. 60 is a good
  default on modern hardware: roughly +4pp of one CPU core for half the tick latency.
- **SendCadence** `[SendCadence]` — sends ZDO updates to every peer on a fixed interval
  (`SendHz`, default 20) instead of vanilla's one-peer-per-frame round robin. Scales with player count.
- **SendBudget** `[SendBudget]` — the ZDO send-queue high-water mark and minimum chunk size,
  exposed as config: `HighWaterBytes` defaults to 65536 (vanilla 10240), `MinChunkBytes` to 2048
  (same as vanilla). Acts as the fallback when AdaptiveBudget is off.
- **CreateBudget** `[CreateBudget]` — objects created per frame, exposed as config (default
  matches vanilla, i.e. a no-op until raised).
- **PeerTelemetry** `[PeerTelemetry]` — per-peer ping, throughput and Steam send-queue sampling.
  No patches; feeds AdaptiveBudget.
- **AdaptiveBudget** `[AdaptiveBudget]` — a *per-peer* send budget from each peer's measured
  bandwidth-delay product, clamped, EMA-smoothed, and backed off while Steam's pending byte count
  says the pipe is congested. A player on fibre next door and one on satellite no longer share
  one number.
- **SteamRates** `[SteamRates]` — raises Steam's `SendRateMax` (default 1 MB/s).
  `SendRateMin` stays at vanilla on purpose: it is a floor on Steam's own bandwidth *estimate*,
  and raising it stops the congestion controller backing off for weak connections.
- **SendQueueGuard** `[SendQueueGuard]` — defers, never drops, while a peer's Steam send queue is
  backed up, and survives the Steamworks throws that can occur mid-handshake.
- **SyncListCache** `[SyncListCache]` — caches the per-peer sector scan in `ZDOMan.CreateSyncList`
  briefly; the filter and sort still run on every send.
- **VPOServer** `[VPOServer]` — the server-safe parts of ValheimPerformanceOptimizations (MIT):
  `WearNTear` support caching and a faster `ReleaseNearbyZDOS` scan.
- **AsyncSave** `[AsyncSave]` — pre-sizes the save clone list. Measured on a 136 000-ZDO world,
  main-thread autosave stall **67 ms → 47 ms** and **78 ms → 45 ms**.
- **GcThrottle** `[GcThrottle]` — rate-limits `Resources.UnloadUnusedAssets` on an idle server.
- **OwnershipRelease** `[OwnershipRelease]` — shortens the ZDO ownership-release interval from
  vanilla's 2 s (default 0.5 s) so objects change hands faster as players move.
- **MapSelfTest** `[MapSelfTest]` — headless unit tests for the shared-map codec at load; shows up
  as `applied  PASS` in the module summary.
- **PriorityLane** `[PriorityLane]` — *both ends.* Holds this machine's outgoing backlog in a
  priority queue instead of letting it pile up inside Steam's own buffer, and hands Steam only
  about one bandwidth-delay product at a time, latency-critical traffic first. A melee hit no
  longer waits behind tens of kilobytes of bulk world data: the residual delay drops from the
  80–430 ms band to roughly 25 ms. **Unmodded clients benefit too** — only the send *order*
  changes, every byte stays vanilla. Coexists with SendQueueGuard and Compression; the ordering
  rules that keep it safe (never let a destroy overtake the object it destroys, never promote a
  message whose target the receiver has not been sent, never cross Compression's handshake) are
  unit-tested at load.
- **StatsLog** `[StatsLog]` — writes `stats-YYYY-MM-DD.jsonl` (fps/frame time, ZDO counts,
  per-peer network/budget/compression state) every `IntervalSec` (default 10s) and
  `events-YYYY-MM-DD.jsonl` (joins/leaves, save stalls, GC sweeps, AdaptiveBudget backoff
  transitions, SendQueueGuard drops, config reloads) as they happen, to
  `BepInEx/config/smoothserver/stats/` by default. Rotates daily, prunes past `RetentionDays`
  (default 30). Run `tools/analyze.py` against a few days of these files for a markdown report —
  see the repo's `tools/README.md`.

## What the client half adds

- **Compression** `[Compression]` — *both ends*. zstd over Steam sockets using
  BetterNetworking's trained dictionaries, with an explicit per-message frame tag and an
  `SS_Caps`/`SS_Ready` capability handshake, so the receiver always knows whether a message is
  compressed. Peers without the mod stay uncompressed with no penalty. Typical self-test ratio on
  real ZDO traffic: **~8 %** of the original size.
- **SharedMap** `[Map]` — *both ends*. The whole server shares map exploration and pins. The store
  is bit-packed and compressed (**1.7 KB** where ServerSideMap wrote **4.2 MB** for the same
  world), the map size is read from your client rather than hardcoded, and updates are batched at
  1 Hz into sparse chunks instead of one RPC per explored pixel. Merging goes through vanilla
  `Minimap.Explore`, so shared terrain bakes into your own map file and nothing is lost if the
  module is turned off. Pin sharing is configurable per pin type; death pins are off by default.
- **ClientNet** `[Client]` — *client only*. Your client's own ZDO send high-water mark (default
  48 KB, vanilla 10 KB) and Steam `SendRateMax`. Inactive on a dedicated server.
- **CombatOwnership** `[CombatOwnership]` — *client only.* On a dedicated server the ore, tree or
  vine you are swinging at is simulated on another player's PC, so every hit is a round trip to
  them and back — about half a second in a busy cave. This takes ownership of the object the
  moment you hit it, after which your damage lands on your own machine in the same frame and the
  hit message never goes on the wire at all. Ore, trees, vines, destructibles and building pieces;
  bows and AoE included. Creatures are deliberately left alone. Wards, other players' builds,
  ships and carts are all respected.
- **LagProbe** `[LagProbe]` — *both ends, diagnostics only; it changes nothing about the game.*
  The server pings each player every 5 s over a routed RPC and reports that player's real RTT,
  jitter, packet loss and frame rate (all measured on the server's own clock, so it works whatever
  Steam reports). On your client it times **hit registration** — from the moment you hit something
  someone else owns until the owner's answer comes back — and keeps a p50/p95/max histogram plus a
  per-owner breakdown, and it counts how often objects near you change owner. Type `ss.lag` in the
  console for the current summary on either side.

## Config

One file, `Nosferatu.SmoothServer.cfg`. Edits are picked up live on a running server
(`[General] HotReload`) — no restart. Settings the server owns are pushed to clients that run the
mod (ServerSync); `[General]` entries stay machine-local.

## Dependencies

- [BepInExPack_Valheim](https://valheim.thunderstore.io/package/denikson/BepInExPack_Valheim/) 5.4.2350

## Credits

- zstd dictionaries and the compression idea: CW_Jesse's **BetterNetworking** (MIT).
- `VPOServer`: **ValheimPerformanceOptimizations** (ontrigger, MIT).
- Config sync: blaxxun-boop's **ServerSync** (MIT-0).
- `SharedMap` is a clean-room reimplementation inspired by Mydayyy's **ServerSideMap**
  (MIT/Unlicense).
- Server-authoritative ownership idea: ddormer's **Serverside Simulations** (no published
  license — idea only, no code taken).

## Source & issues

https://github.com/MJensen01/SmoothServer — MIT licensed.
