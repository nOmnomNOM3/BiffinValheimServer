# Changelog — SmoothServer

## 0.6.0 (2026-09-18)

Hit registration

* **CombatOwnership (client, ON by default): claim on hit for static objects.** On a dedicated server, damage is applied by
  whichever player's PC happens to own the object, so a swing at a vine, an ore vein or a tree that somebody else owns
  travels you → server → owner and its result comes back owner → server → you, each leg behind that peer's send queue.
  That is the half-second before a vine breaks. Now, when the local player is the attacker, the client takes ownership of
  the object first (`ZNetView.ClaimOwnership()`, the same local, instant call vanilla itself uses in 14 places) and
  vanilla applies the hit on your own machine in the same frame; only the health change travels, one leg instead of
  three. Statics only: `Destructible`, `MineRock5`, `MineRock`, `WearNTear`, `TreeBase`, `TreeLog`. Guards: never a player
  ZDO, never anything on a ship or cart, never inside a ward you cannot access, never a building piece another online
  player owns, never beyond `MaxDistance` (16 m), and any failure in the check falls back to the vanilla swing. **Creatures
  are deliberately not claimed** in this version (Stage 2 needs its own hysteresis so two players cannot both run a
  Seeker's death and duplicate its loot); they still benefit from the queue work below. `[CombatOwnership] Enabled` is
  server-synced and turns off live. Interop note: NoVikingLeftBehind's FastMining bonuses now run on the attacker after a
  claim (same code, different machine) - ore yields and regrowth are on the re-test list.
* **PriorityLane (server, OFF by default): a shaper in front of Steam's single reliable lane.** Holds the outgoing backlog
  in a managed priority queue (routed RPCs and freshly damaged objects first, bulk world updates after) and hands Steam
  only about one bandwidth-delay product at a time, so a hit RPC no longer waits behind tens of kilobytes inside Steam's
  own buffer. Send order only, every byte stays vanilla-legal, so unmodded clients benefit too. Hard ordering guards: a
  routed RPC is never promoted ahead of the ZDO it targets, `DestroyZDO` is never promoted, the compression handshake
  RPCs are barriers nothing may overtake, and no package is ever split or copied. 31 headless self-test cases cover the
  wire classifier, the queue and the guards, but the shaper has not yet moved a real player's bytes, so it ships **off**;
  switch `[PriorityLane] Enabled = true` for a session you will watch (needs a restart) and read `[PeerTelemetry]`
  socketQueue and `[LagProbe]` afterwards.

Diagnostics

* **LagProbe times mining and woodcutting too.** The hit-registration timer hooked `Character`, `Destructible` and
  `WearNTear` only, so a pickaxe on ore (`MineRock5`/`MineRock`) and an axe on a tree (`TreeBase`/`TreeLog`) - half of
  what a group does in a cave - were never measured. They are now.
* **LagProbe counts the swings you own.** A hit on an object this client owns is applied in the same frame; it used to be
  skipped outright, which meant a future claim-on-hit fix would look like the histogram "improving" by losing its samples.
  It is now counted as `local=N` next to `hits=N` in both the client summary and the server line. Report format v2; a
  0.5.3 client's v1 report is still read (`local=?`).

## 0.5.3 (2026-09-17)

Compatibility

* **SendBudget now works on a hex-patched `assembly_valheim.dll`** ([issue #2](https://github.com/MJensen01/SmoothServer/issues/2),
  part two). Some hosts and old "network fix" guides raise the game's 10240-byte per-peer send-queue literal in
  `ZDOMan.SendZDOs` by editing the DLL (the reporter's G-Portal file was byte-for-byte Steam's 1.0.14 except those two
  operands, 10240 → 30720). SendBudget's strict "exactly 2x 10240" check refused such a binary and the whole Harmony
  patch failed. The transpiler now recognises that shape - two identical raised literals plus the untouched 2048 and
  nothing else - takes it over exactly like vanilla, and names the value it found in the log and the module summary
  (`binary literal 30720`). `[SendBudget] HighWaterBytes` then applies as configured.
* **IL the module does not recognise is no longer a failure.** With no other mod on the method, an unexpected
  `ZDOMan.SendZDOs` used to throw out of the transpiler (a wall of stack trace, `FAILED(IL Compile Error)`). It now hands
  the method back untouched, reports `disabled(IL mismatch)`, and logs one block with every int literal it saw, the
  instruction count, the game version, and the game assembly's MVID, size, modified date and md5 - enough to answer a
  report from the log alone. Everything else keeps running. `disabled(conflict)` (another mod owns the method) is unchanged.
* ILSelfTest: 5 new cases (both take-over values, the two shapes that must not be taken over, the foreign-transpiler
  variant; the old "unrecognised IL throws" case is now "unrecognised IL is a clean mismatch") - 12 cases total, silent.

## 0.5.2 (2026-09-17)

* **SendBudget/CreateBudget/AdaptiveBudget/OwnershipRelease/ClientNet no longer FAIL when another mod
  has already transpiled the same method — they log the other mod and stay off ([issue #2]).** The
  report was `SendBudget transpiler: expected exactly 2x 10240 and 1x 2048 in ZDOMan.SendZDOs, found
  0 and 1 — game IL changed`, on a server where every other module applied. The game IL was fine:
  Harmony *chains* transpilers, so the second mod on a method is handed the first one's output, and
  several networking mods rewrite exactly those two 10240 literals. Our scan could not tell that from
  an Iron Gate change and blamed the wrong party. Each literal-swapping module now asks Harmony who
  else is on its target before patching; if somebody is, it leaves the method alone, logs one line —
  `[SendBudget] another mod already changed ZDOMan.SendZDOs (owners: <ids>) - SendBudget left off so
  the two do not fight; set [SendBudget] Enabled=false to silence this` — and reports the new status
  **`disabled(conflict)`** instead of `FAILED`. Nothing is relaxed: with no foreign transpiler on the
  method the exact match counts are still mandatory and a real game IL change still refuses to patch,
  loudly. AdaptiveBudget applies through SendBudget's call site, so the summary now says so when
  SendBudget is not applied. New `ILSelfTest` module (on by default, pure data, no patches) runs the
  transpiler bodies against synthetic IL at startup and logs one PASS/FAIL line, including the
  "another mod got here first" case.
* **LagProbe: session-edge filter — the multi-second "RTT" spikes were not latency.** The first real
  data from 0.5.1 put every 1–5 s round trip at exactly one of two moments: a player joining (the
  client's main thread is loading the world and answers nothing) or leaving (it is saving and
  quitting). Both are now dropped whole. Any probe sent within the new `[LagProbe] EdgeIgnoreSec`
  (30 s, machine-local) of a peer becoming ready, and every probe once the server knows the peer is
  going away (`ZNet.Disconnect`, or a socket already reporting disconnected), never touches
  RTT/EMA/jitter and is never charged as loss when it goes unanswered — the per-peer summary line
  counts them separately as `edgeDropped=N`. The line also carries the peer's session age
  (`age=Ns`) and, while a session is inside a window, `joining (Ns of edge window left)` or
  `leaving`. A peer that has never answered a single post-edge ping now prints
  `client-mod=none (N sent, 0 answered)` instead of `loss=100%`: an old or missing client half is
  not packet loss, and the `[PeerTelemetry]` line says the same.
* **LagProbe: ZDO churn by prefab — "what generates the traffic?"** 140–160 kB/s and ~800 ZDO
  updates/s to each player near the base was the symptom; this names the cause. A prefix/finalizer
  pair on `ZDOMan.SendZDOs(ZDOPeer, bool)` — the single funnel every outgoing ZDOData package goes
  through, vanilla's round robin, `SendAllZDOs` and SendCadence's own sweep alike, so nothing can be
  double counted — opens a window in which a postfix on `ZDO.Serialize(ZPackage)` counts one update
  against the ZDO's prefab hash and adds the bytes just written plus vanilla's fixed 42-byte per-ZDO
  header. Every `SummaryIntervalSec` the server prints `[LagProbe] zdo churn 60s: total=N updates
  (B kB) to P peers; top: <prefab>=n (x%), …`, StatsLog's record gains `zdoChurnTop` /
  `zdoChurnTotal` / `zdoChurnBytes`, and `ss.lag churn` / `ss.lag churn reset` print the top 25 and
  clear the table on demand. New `[LagProbe] ChurnEnabled` (true) and `ChurnTopN` (10), both
  machine-local. Off the send path the postfix is a single bool read, so the autosave's own
  serialisation costs nothing, and a background thread is ignored outright.
* **LagProbe: client hit reports reach the server (`SS_LagReport`).** The hit-registration histogram
  is measured on the client, where nobody reads a log. Every `SummaryIntervalSec` the client half
  now sends its own summary to the server as one small routed RPC — hits, p50/p95/max, late,
  timeouts, ownership churn per minute, the client's frame-time p50 and up to 8 owners — and the
  server logs one line per client per interval (`[LagProbe] client 'Name' 60s: hits=… p50=…ms
  p95=…ms max=…ms late=… timeouts=… churn=…/min cfps=… | owners: 'Name' n=… avg=…ms max=…ms, …`),
  writes `hitCount`/`hitP50Ms`/`hitP95Ms`/`hitMax`/`hitLate`/`hitTimeouts`/`hitChurnPerMin`/
  `hitReportAgeSec`/`hitOwners` into StatsLog's per-peer record, and repeats it in `ss.lag`. Reports
  from a peer that is not ready are ignored and an oversized package is dropped unread; a client
  without the mod (or on 0.5.1) simply never sends, and nothing changes for it.
* **LagProbe is a temporary diagnostic, not a permanent feature.** It exists to measure the
  hit-registration-latency work (`docs/HIT-LATENCY-PLAN.md`); once that fix ships and proves out,
  `[LagProbe] Enabled` will default to `false` in the following release (still available to turn back
  on), with removal of the ping/report RPCs entirely considered later. See
  `docs/LAGPROBE-LIFECYCLE.md`.

[issue #2]: https://github.com/MJensen01/SmoothServer/issues/2

## 0.5.1 (unreleased)

* **PeerTelemetry read zeros on dedicated servers, so AdaptiveBudget never adapted — fixed.** Every
  `[PeerTelemetry]` line showed `ping=0ms qual=0.00/0.00 … steamRate=0B/s` for every player, forever
  (15,761 samples on the live server without a single non-zero), and `[AdaptiveBudget] samples=0
  base=65536B (steady)` — the per-peer budget silently fell back to one fixed number for everybody.
  The Steam interface was never the problem: `peer.m_socket as ZSteamSocket` was returning null.
  ServerSync (vendored by SmoothServer, by NoVikingLeftBehind *and* by third-party mods) swaps a
  decorator socket into `ZNetPeer.m_socket` during the login RPC and restores it from a coroutine,
  but each copy's restore only recognises its own `BufferingSocket` type — with several copies
  loaded the unwind does not complete and a player keeps another mod's decorator for the rest of the
  session. Telemetry now unwraps the decorator chain to the real socket, a single refused connection
  no longer latches the Steam read off for the whole process, and every outcome is logged exactly
  once (`live stats OK for <name>: ping=Nms`, `GetConnectionRealTimeStatus failed (<result>) — stats
  unavailable`, `peer '<name>' socket is <Type> with no ZSteamSocket behind it`) — it can never read
  zeros silently again.
* **New `LagProbe` module** (`[LagProbe]`, on by default, diagnostics only — it changes no gameplay
  value): server→player RTT/jitter/loss pings (`SS_Ping`/`SS_Pong`, measured on the server's own
  clock and independent of Steam's statistics, surfaced in the PeerTelemetry line and StatsLog), a
  client-side hit-registration latency histogram (time from a hit on something you do not own until
  the owner's answer lands, with p50/p95/max and a per-owner breakdown), ZDO ownership churn within
  30 m, and an `ss.lag` console command that prints the current summary on demand.

**Compression: a client that also runs SmoothServer could be dropped into an empty world.** The server
started unframing one round trip before the client started framing ([issue #1]), so every plain packet in
that window was read as a frame and the peer's stream was lost in one direction. The switch now happens on
the `SS_Ready` message on both sides, a failure disables compression on both ends, and 0.5.0 peers stay
uncompressed.

* `[Compression]` handshake: `SS_Ready` is the in-band switch marker - "everything I send after this message
  is framed". A side starts framing right after sending its own Ready and starts unframing exactly at the
  peer's Ready, so both flips key off the same byte position in the socket's reliable FIFO.
* New `SS_Off` message: if a frame ever fails to unframe, that side tells the peer (framed, while it can
  still be read), then both ends drop to plain for the rest of the connection and never re-arm. One
  `[Compression] <peer> compression disabled (reason) - running plain` line per side.
* Wire proto bumped to 2: a 0.5.0 peer (proto 1) is never framed, in either direction - it logs one info
  line and stays uncompressed. Servers whose clients have no mod negotiate nothing, exactly as before.
* The client could send its capabilities before the server had assigned it a peer id, so the offer was
  dropped and never retried — compression silently never negotiated. The offer now waits for the peer to be
  ready, and both sides log the handshake.
* New machine-local `[Compression] SelfTest` (default off): runs the two-peer handshake through an
  in-process simulation at load and logs a PASS/FAIL line per case.
* **Both the compression handshake and PeerTelemetry now look through ServerSync's `BufferingSocket`
  wrapper; on servers running several ServerSync-based mods the peer socket stayed wrapped for the
  whole session, so neither could see the Steam socket.** `SocketOf` returned null, `OnCaps` bailed
  out, and compression could never be negotiated on a server like that no matter what both ends ran.
  Every peer-socket lookup now resolves the real `ZSteamSocket` behind the decorator chain - the same
  instance the `SendQueuedPackages`/`Recv` patches fire on, so the per-socket state still matches -
  and logs it once per peer (`peer socket wrapped by <Type>, using the ZSteamSocket behind it`, or
  `peer '<host>' socket is <Type> with no ZSteamSocket behind it - staying plain`). A fifth self-test
  case covers it.

[issue #1]: https://github.com/MJensen01/SmoothServer/issues/1

## 0.5.0 (2026-09-09) — Valheim 1.0

**Rebuilt for Valheim 1.0.7 (network version 39).** Requires BepInExPack_Valheim 5.4.2350. Not compatible
with 0.221.x — stay on 0.4.0 if your server is held on the `default_pre1_0` branch.

* `VPOServer`, `SyncListCache`: ported to 1.0's per-peer `SimulationDistance` and `Vector2s` zones; the
  WearNTear port mirrors 1.0's dirty-check so `s_support` is only broadcast when it actually changed; the
  ownership scan uses 1.0's point-distance active-area test.
* `AsyncSave`: the pre-size optimisation is retired — 1.0 saves per chunk and pre-sizes itself. The module
  now instruments only (StatsLog still gets save timings) and says so loudly at boot.
* `MapSelfTest`: worlds path via `SaveSystem.GetWorldsSaveRootPath`.
* All three transpiler assertions hold at their original counts on 1.0; every patch target re-verified.

## 0.4.0 (2026-09-07)

**`[Profiles] Profile` — one setting that tunes the whole mod.** A synced enum
(`Default` / `FastLink` / `Custom`) that writes a fixed table of values into ten existing
settings at load, whenever the profile changes, and after every live cfg reload. It is a preset,
not a new feature: nothing is shadowed or hidden, the values are written into the entries
themselves, so the cfg file, `cfg.py get` and each module's own startup log line all agree on
the single number that is actually running.

* `FastLink` — *"make it feel like LAN"*, for a small group on strong PCs and good links:
  `[SendCadence] SendHz=60`, `[AdaptiveBudget] CeilingBytes=262144 FloorBytes=32768`,
  `[SteamRates] SendRateMax=4194304`, `[Client] HighWaterBytes=131072
  SendRateMaxBytesPerSec=4194304`, `[Compression] Enabled=true`, `[LowLatency] NagleMicros=0`,
  `[FrameRate] TargetFrameRate=60`, `[CreateBudget] MaxCreatedPerFrame=20`. Nothing else is
  touched.
* `Default` — those same ten keys are forced back to their shipped defaults.
* `Custom` — the plugin overrides nothing and every value in the file stands.

Precedence is documented on each affected key's own description in the generated cfg, so it is
discoverable from the file rather than from this changelog. Because `Profile` is synced, a client
that joins a `FastLink` server runs `FastLink` too — including the machine-local keys
(`[Client] SendRateMaxBytesPerSec`, `[FrameRate] TargetFrameRate`) that ServerSync does not push.

> **Upgrading from 0.3.x:** the shipped default is `Profile=Default`, which means any of those
> ten keys you had hand-tuned is reset to its default on first boot (the plugin logs exactly what
> it changed). If you want your own numbers kept, set `[Profiles] Profile = Custom` — that is
> what it is for.

**New module `LowLatency`** (`[LowLatency]`, both ends, on by default). Steam coalesces small
reliable sends for `k_ESteamNetworkingConfig_NagleTime` microseconds before putting them on the
wire — 5000µs by default, which vanilla Valheim never changes: a straight 0-5 ms of
added latency on every ZDO update, in both directions, for a saving that mattered on a modem.
SmoothServer already batches (`SendCadence`) and compresses (`Compression`), so what reaches
Steam is already a big packet. `NagleMicros=0` sends it immediately. The shipped default is
vanilla's 5000 (the module logs a before/after readback but changes nothing until you lower it,
or until `FastLink` does); `SendBufferBytes` (0 = leave Steam's 512 KB alone) is there for a
server pushing large budgets to several peers. Picks the Steam interface its own build was
compiled for — `SteamGameServerNetworkingUtils` on the dedicated server,
`SteamNetworkingUtils` on the client (NOTES §20) — and goes quiet if that interface is
not initialised.

**New module `SmoothMotion`** (`[SmoothMotion]`, client only, **off by default**). Tunes how
*other* players and mobs are drawn between the updates you receive. Purely local and purely
cosmetic: nothing is sent, stored or simulated differently, and objects you own are never touched
(vanilla's own `if (zdo.IsOwner()) return;` runs first). Two knobs, both transpiled over the
hardcoded literals in `ZSyncTransform.SyncPosition` with an exact-match-count assertion, so a
game patch that changes the maths makes the module refuse to load rather than silently do
something else:

* `InterpolationFactor` (0.2 = vanilla) — how much of the gap to the target position a
  non-owned object closes each physics tick. Higher = snappier, lower = smoother.
* `ExtrapolateMs` (2000 = vanilla) — how far ahead an object may be dead-reckoned along its
  last known velocity. **Valheim already does this**, uncapped for a full two seconds; this knob
  only tightens it. 150-400 hides a lost packet without the two-second overshoot that produces
  the rubber-band snap when someone stops dead. 0 disables prediction entirely.
* `ApplyTo` (`Characters` / `All`) — players and mobs only, or every synced object.

**`tools/cfg.py preset <mod> fastlink|default|custom`** — sets `[Profiles] Profile` and
confirms the live reload, like any other `cfg.py set`.

**`ConfigWatcher` gained a post-reload hook** (`AfterReload`) and a `Reloading` flag. BepInEx
raises `SettingChanged` entry-by-entry *as* `Config.Reload()` walks the file, so a handler that
reacts to one entry by writing others is half-undone by the entries the reload has not reached
yet. The profile is therefore applied once, after the whole file is read, instead of flapping
mid-reload.

## 0.3.1 (2026-09-07)

**Fix: server-side Steam socket modules used the client Steam interface on dedicated servers
(connections failed).** The two builds of `assembly_valheim.dll` are compiled differently:
`ZSteamSocket` calls `SteamNetworkingSockets` on the client and `SteamGameServerNetworkingSockets`
on the dedicated server, and only the matching half of Steamworks is initialised in each process.
`SendQueueGuard` re-implemented vanilla's send drain with the client interface hard-coded, so on a
dedicated server every send threw `Steamworks is not initialized.`, no client could finish its
handshake, and joiners were dropped with *"Socket closed by peer"*. It no longer re-implements the
drain at all: it wraps the vanilla one with a prefix/finalizer pair, so the game picks the
interface its own build was compiled for. Back-off and once-per-episode logging are unchanged
(progress is now measured through `m_totalSent` instead of the `EResult`). `PeerTelemetry` probes
the build's own interface first and stops calling `GetConnectionQuality` where vanilla itself
calls the wrong one, taking ping/quality/throughput off the real-time status instead. `SteamRates`
no longer warns on the missing client interface — the first refusal per interface is noted once at
Info and that interface is never touched again. New `[General] SteamSelfTest` (off by default,
machine-local) logs which half of Steamworks is live and, straight out of `ZSteamSocket`'s IL,
which interfaces this build actually calls — turn it on once after a game update to re-prove it.

**StatsLog**: persistent JSONL stats + events for multi-day analysis. `[StatsLog]` (server-side,
on by default) writes `stats-YYYY-MM-DD.jsonl` every `IntervalSec` (default 10s) — fps/frame
time, ZDO/scene counts, per-peer RTT/pending/in-flight/queued bytes, AdaptiveBudget target +
congestion, Compression framing state, and compression byte deltas — plus `events-YYYY-MM-DD.jsonl`
for player join/leave, world-save stalls, GC sweeps, AdaptiveBudget backoff transitions,
SendQueueGuard drops and config reloads, as they happen. Files land in
`BepInEx/config/smoothserver/stats/` by default, flush on every write, rotate daily (UTC) and are
pruned past `RetentionDays` (default 30). `IntervalSec`/`RetentionDays` hot-reload. A companion
`tools/analyze.py` (stdlib-only) turns a few days of logs into a markdown report — sessions,
frame-time percentiles, per-player RTT/pending-bytes, AdaptiveBudget backoff time, compression
ratio, save stalls, ZDO growth, and the worst 10-second windows.

## 0.3.0 (2026-09-07)

**SmoothServer is now a both-ends mod.** The same `SmoothServer.dll` runs on a dedicated server
and in a player's BepInEx profile; `[General] Mode` (Auto/Server/Client) picks the half, and a
module that does not belong to the running half reports `disabled(side)` and patches nothing.

`[General] EnforceClientMod` **defaults to `false`**, so nothing changes for existing server
admins: SmoothServer still works as a pure server-side mod with zero client installs, and
vanilla clients can still join. Turn it on only if your whole group installs the mod.

### Read this before updating

- **BetterNetworking must be uninstalled** — on the server *and* on every client. SmoothServer's
  `Compression` module replaces it and the two cannot coexist (both wrap `ZSteamSocket`'s send
  queue). If BetterNetworking is present, `Compression` refuses to patch and reports
  `FAILED(...)`; the rest of the plugin still loads.
- **ServerSideMap must be uninstalled** — on the server *and* on every client. `SharedMap`
  replaces it. On first start SmoothServer imports your existing
  `<world>.mod.serversidemap.explored` file automatically (once), so no exploration is lost.
- **The zip now contains six DLLs, not one.** `SmoothServer.dll` plus `ZstdSharp.dll`,
  `System.Memory.dll`, `System.Buffers.dll`, `System.Numerics.Vectors.dll` and
  `System.Runtime.CompilerServices.Unsafe.dll`. A mod manager installs all of them; if you copy
  files by hand, copy all six into `BepInEx/plugins/SmoothServer/`.

### New server modules

- **PeerTelemetry** `[PeerTelemetry]` — per-peer ping, throughput and Steam send-queue sampling.
  No patches; it is the input to AdaptiveBudget and the source of the per-peer log line.
- **AdaptiveBudget** `[AdaptiveBudget]` — per-peer ZDO send budget derived from each peer's
  measured bandwidth-delay product, clamped and EMA-smoothed, backing off while Steam's *pending*
  byte count says the pipe is congested. SendBudget's configured value becomes the fallback.
- **SteamRates** `[SteamRates]` — raises Steam's `SendRateMax` on the game-server networking
  interface (default 1 MB/s). `SendRateMin` is deliberately left at vanilla.
- **SendQueueGuard** `[SendQueueGuard]` — defers (never drops) sends while a peer's Steam queue
  is backed up, and swallows the "Steamworks is not initialized" throws that can happen mid-handshake.
- **SyncListCache** `[SyncListCache]` — caches the per-peer sector scan in `ZDOMan.CreateSyncList`
  for a short window; the filter and sort still run on every send.
- **VPOServer** `[VPOServer]` — server-safe parts of ValheimPerformanceOptimizations (MIT):
  `WearNTear` support caching and a faster `ReleaseNearbyZDOS` scan.
- **AsyncSave** `[AsyncSave]` — pre-sizes `ZDOMan.GetSaveClone()`'s list from the live ZDO count.
  Measured on a 136 000-ZDO world: main-thread autosave stall **67 ms → 47 ms** and **78 ms → 45 ms**.
- **GcThrottle** `[GcThrottle]` — rate-limits `Resources.UnloadUnusedAssets` on an idle server.
- **OwnershipRelease** `[OwnershipRelease]` — shortens the ZDO ownership-release interval from
  vanilla's 2 s (default 0.5 s), so objects change hands faster as players move.

### New both-ends / client modules

- **Compression** `[Compression]`, both ends — zstd (ZstdSharp, pure managed) over Steam sockets
  using BetterNetworking's trained dictionaries. Explicit 1-byte frame tag and an
  `SS_Caps`/`SS_Ready` handshake, so the receiver never guesses and never uses an exception as a
  signal; a peer without the mod simply stays uncompressed. Self-tested at load.
- **SharedMap** `[Map]`, both ends — server-side shared map exploration and pins, replacing
  ServerSideMap. Bit-packed and compressed store (**1.7 KB** where ServerSideMap wrote **4.2 MB**
  for the same world), map size read from the client instead of hardcoded, deltas batched at
  1 Hz into sparse chunks instead of one RPC per pixel. Merges through vanilla `Minimap.Explore`,
  so shared pixels bake into your own map file. Optional death-pin sharing.
- **MapSelfTest** `[MapSelfTest]`, server — headless unit tests for the map codec, run at load.
- **ClientNet** `[Client]`, client only — the client's own ZDO send high-water mark (default
  48 KB) and Steam `SendRateMax`. Never active on a dedicated server.

### Other

- Modules are now discovered by reflection: one `[BepInPlugin]`, one config file, one module
  summary line, hot reload covering every module.
- Config sharing via ServerSync — the server's values win on clients that run the mod.
- `SendBudget`'s high-water hook now defers to AdaptiveBudget per peer;
  `[AdaptiveBudget] UseCallSiteSwap` is a legacy path and now defaults to `false`.

## 0.2.0 (2026-09-06)
- **Renamed** from `OrionNet`. New plugin GUID `Nosferatu.SmoothServer`, new config file
  `Nosferatu.SmoothServer.cfg`. Old `net.mjensen.orion.net.cfg` settings are **not** migrated —
  re-apply them once.
- Live config reload (hot reload): edits to the cfg file are picked up on a running server
  without a restart (`[General] HotReload`).

## 0.1.0 (2026-09-06)
- Initial release. Modules: `Telemetry`, `FrameRate`, `SendCadence`, `SendBudget`, `CreateBudget`.
- Server-only; no client install required.
