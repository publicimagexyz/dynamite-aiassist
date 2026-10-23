dynamite
===

dynamite is a 2 player co-op mod for Metal Gear Solid V: The Phantom Pain in early development stage.

> [!WARNING]
> You MUST roll back your game to the previous version before installing, see [related issue](https://github.com/unknown321/dynamite/issues/136).

[Download](https://github.com/unknown321/dynamite/releases/latest)

Things won't work like you expect them to; please make sure to read user documentation. Expect bugs.

https://github.com/user-attachments/assets/e82c43c7-eace-4084-b1fd-c4fb8ef3abe2

### Requirements

1. Unmodified Steam version of MGSV:TPP (regular version, JP is not supported)
2. Online connection
3. Some free space
4. A friend to play with

### Documentation

[Making of](./docs/MAKING_OF.md)

[User documentation](./webui/docs/docs.md) (shipped with executable)

[Build from source](./docs/BUILD.md)

### Fork status: audit and roadmap

This fork continues [unknown321/dynamite](https://github.com/unknown321/dynamite) with the goal of full co-op
across the game. The notes below come from a code audit (Oct 2026). Nothing has been tested in-game yet; line numbers
refer to the upstream code at commit `e559c3d`.

#### How it works today

- **Go launcher** (`main.go`, `installer/`, `patch/`, `webui/`, `masterserver/`, `savebackup/`): repacks game archives
  (datfpk), runs a local [fuse](https://github.com/unknown321/fuse) master server, backs up saves, serves the config UI.
- **C++ `dinput8.dll` hook** (`hook/src`): MinHook detours plus in-memory byte patches. Uses **333 hardcoded addresses
  for game version 1.0.15.3 only** (`mgsvtpp_addresses_1_0_15_3_en.h`).
- **Lua edits** (`moddata/`): 172 game scripts, formatted with stylua so they diff cleanly against vanilla.
- **Networking**: co-op piggybacks on the FOB (PvP) netcode. The host is the "attacker" and runs enemy AI; the client is
  a "defender" that gets movement commands. Extra state (markers, svars, Quiet boss, emblems) is sent as FlatBuffers
  messages (`DynamiteSyncSchema.fbs`) over the Steam UDP socket, with an 8-byte `DYNAMITE` header.

Mission status (`webui/templates/missionlist.md`): about 30 of 50 can be finished, 8 are unconfirmed
(16, 28, 29, 30, 31, 35, 41, 45), 13 are disabled. There's no free roam, no side ops, and no more than 2 players.
Not synced: vehicles, D-Horse position, Quiet/D-Dog, support heli model, loadout/supply drops, interrogations, gimmicks.

#### Bugs and risks found

| # | Where | Problem |
|---|---|---|
| 1 | `hook/src/DynamiteCore.cpp:49` `GetNearestPlayer` | One global target. It picks the player closest to *any* living soldier, and every enemy then targets that one player. This is the core blocker for real co-op AI. |
| 2 | `DynamiteCore.cpp:450`, `DynamiteSyncImpl.cpp:117` | Background `std::jthread`s call game functions (object lookup, `BlockHeapAlloc`, Steam sends) outside the game thread every 500 ms. `offensePlayerID` / `packetNumber` / `packetSeen` aren't atomic. Likely source of random crashes. |
| 3 | `hook/src/dynamite.cpp:102` | No game version check before installing hooks at hardcoded addresses. Byte patches check expected bytes (`patch.cpp`), but MinHook detours don't. The exe md5 check in `patch/patch.go` (`vendorValid`) is commented out. An updated game crashes or corrupts memory instead of showing an error. |
| 4 | `DynamiteSyncImpl.cpp:901, 936, 990, 1049` | Quiet boss handlers `memcpy(..., sizeof(struct))` from network vectors without checking length or null. Entity index from the peer is passed to game functions unchecked. |
| 5 | `DynamiteSyncImpl.cpp:655` (and the other `Sync*Var`) | `SyncVar` writes `array_start + i` into any svar/gvar the peer names, with no bounds check. Remote memory-corruption risk; by default anyone with your SteamID can join while you host. |
| 6 | `DynamiteSyncImpl.cpp` `RecvRaw` | Verifier runs over the whole 1024-byte buffer instead of the received length. No ack or resend, and reordered packets are dropped (`packet_num <= packetSeen`), so markers and var syncs can be lost silently. |
| 7 | `DynamiteCore.cpp:370` `CreateEmblem` | `ecode` is allocated, overwritten by the return value of `CreateEmblemParameters`, then `BlockHeapFree`d. The allocation leaks; if the return value is `&errcode` (typical for these engine functions, unverified), it frees a stack address. `params` also leaks on the "not ready" path. |
| 8 | minor | `GetSVar`/`SyncVar` leak the handle on error paths and check `handle == nullptr` after using it. `DynamiteCore.cpp:568` `spdlog::error` is missing its format argument. The blacklist accept path (`DynamiteHook.cpp` ~125) doesn't reset `packetSeen`. Several network handlers dereference optional FlatBuffers fields (`pos()`, `category()`, `name()`) without null checks. |

#### Roadmap

**Phase 1: stability (no game needed to write, needs in-game testing)**

1. **Version guard.** In `Dynamite::Dynamite()` (`dynamite.cpp`), check the running exe before `CreateHooks()`. Use
   the md5 `7cc5f282b068f741adda2bb1076fb721` from `patch/patch.go`, after confirming it against a real 1.0.15.3 exe, or
   a cheaper PE `TimeDateStamp`/`SizeOfImage` check. On mismatch: log the hash, show a message box, skip hooks.
   Consider a `[debug]` config key to override.
2. **Move tick work to the game thread.** Add a `Dynamite.Update` Lua function and register it in the fixed update
   list in `moddata/standalone/TppMain.lua` (the `a = { TppSequence.Update, ... }` table near line 570; `TppMain.OnUpdate`
   runs every frame on the Lua thread). Have it run `GetNearestPlayer` every 500 ms when targeting is enabled. Keep
   `StartNearestEnemyThread` / `StopNearestEnemyThread` as Lua names (they just toggle a flag) so the ~40 mission
   scripts don't change. Make `offensePlayerID`, `packetNumber`, `packetSeen` `std::atomic`. Delete the unused
   `WaitForSync` thread.
3. **Harden network handlers.** Null-check every optional field. Require `vector->size() == sizeof(T)` before `memcpy`.
   Bounds-check `array_start + size` against the var's array size (handle `+0x8`, as in `SyncVar`). Verify only the
   received length in `RecvRaw`.
4. **Fix the small bugs** in items 6-8 above.

**Phase 2: real co-op AI (needs Ghidra + debug build)**

5. **Per-soldier targeting.** Find where each soldier's AI picks its target (the callers of
   `GetFobOffensePlayerInstanceIndex`) and pick the nearest or most visible player per soldier instead of one global id.
6. Sync alert/caution state per command post. Fix "enemies switching targets" and "teleporting enemies".
7. **Reliable channel.** Add acks and resend for important messages (markers, svars, objectives).

**Phase 3: content**

8. Entity sync: vehicles (second instance per player?), D-Horse, buddies, support heli, supply drops, interrogation
   subtitles, gimmicks. The game has 11 sync instances; check which entity types they cover.
9. Work through the unconfirmed missions (`docs/issue_7.md` is a good model), then the disabled ones. Replace the
   respawn and fulton-fails-mission workarounds.
10. Free roam and side ops, then more than 2 players. Upstream reports the game fails to start with more than 2.
11. Port addresses to the current game version, or keep requiring the rollback.

#### Dev setup notes

- Game rolled back to 1.0.15.3, two Steam accounts (two PCs is easiest), Ghidra with the debug exe for symbol names.
- Build: `make prepare && make release` (Docker), or see [BUILD.md](./docs/BUILD.md). The hook needs a mingw-w64 GCC
  with C++23 and posix threads (upstream uses GCC 16.1 via `hook/builder/Dockerfile`).
- `go generate` writes gitignored build outputs (`*.fpkd`, `*.lng2`, `webui/templates/docs.tmpl`). `go build` needs
  `hook/cmake-build-release-docker-mingw15/dinput8.dll` to exist (it is embedded).

### Support the project

<a href="https://boosty.to/unknown321/donate" target="_blank">Boosty</a>

### Gameplay

<a href="https://www.youtube.com/watch?v=dja6IPYZ_p4">Mission 5 OVER THE FENCE (41:06)</a>

<a href="https://www.youtube.com/watch?v=4xIB7_qEVwI">Mission 15 FOOTPRINTS OF PHANTOMS (07:25)</a>

<a href="https://www.youtube.com/watch?v=JJQbVnNBghE">General gameplay, funny cuts (13:59)</a>

<a href="https://www.youtube.com/watch?v=JMAAO8YrxB0">Gameplay, v0.6.0 (Oct 31, 2025), Portugese (22:55)</a>

### Press

[PC Gamer](https://www.pcgamer.com/games/action/after-10-years-theres-a-functioning-metal-gear-solid-5-co-op-mod-so-you-and-a-friend-can-be-played-like-a-damn-fiddle-together/)
