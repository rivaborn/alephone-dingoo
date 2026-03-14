# Source_Files/Network/network_dummy.cpp

## File Purpose
Provides stub implementations of network functions when networking is disabled or unavailable. Allows single-player or LAN-disabled builds of the Aleph One game engine to link successfully by providing no-op network interface functions that return safe defaults.

## Core Responsibilities
- Implement all declared network functions with safe dummy behavior
- Enable single-player-only game builds without conditional compilation in game logic
- Return sensible defaults (false, NULL, 0, 1) indicating no network is active
- Disable network-only cheats (crosshair, tunnel vision, behind-view)
- Support map initialization and game queries in non-networked mode

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### NetExit
- Signature: `void NetExit(void)`
- Purpose: Clean up network subsystem on shutdown.
- Inputs: None.
- Outputs/Return: None.
- Side effects: None.
- Calls: None.
- Notes: No-op stub; actual network teardown happens in real implementation.

### NetSync
- Signature: `bool NetSync(void)`
- Purpose: Synchronize game state across network players before advancing frame.
- Inputs: None.
- Outputs/Return: `true` (always synchronized in dummy mode).
- Side effects: None.
- Calls: None.
- Notes: Single-player mode is trivially synchronized; returns immediately.

### NetUnSync
- Signature: `bool NetUnSync(void)`
- Purpose: Acknowledge frame processing and desynchronize for next tick.
- Inputs: None.
- Outputs/Return: `true` (always succeeds).
- Side effects: None.
- Calls: None.
- Notes: No-op in single-player; no actual unsync needed.

### NetGetLocalPlayerIndex
- Signature: `short NetGetLocalPlayerIndex(void)`
- Purpose: Query index of the local human player.
- Inputs: None.
- Outputs/Return: `0` (always player 0 in single-player).
- Side effects: None.
- Calls: None.
- Notes: Hardcoded to first player slot.

### NetGetNumberOfPlayers
- Signature: `short NetGetNumberOfPlayers(void)`
- Purpose: Query total player count in current game.
- Inputs: None.
- Outputs/Return: `1` (single player only).
- Side effects: None.
- Calls: None.
- Notes: Always returns 1; supports single-player games only.

### NetGetPlayerData
- Signature: `void *NetGetPlayerData(short player_index)`
- Purpose: Retrieve opaque player state blob for given player index.
- Inputs: `player_index` (ignored in dummy).
- Outputs/Return: `NULL`.
- Side effects: None.
- Calls: None.
- Notes: No player data available in stub mode.

### NetGetGameData
- Signature: `void *NetGetGameData(void)`
- Purpose: Retrieve opaque game state blob (rules, parameters, scores).
- Inputs: None.
- Outputs/Return: `NULL`.
- Side effects: None.
- Calls: None.
- Notes: No shared game data needed in single-player.

### NetChangeMap
- Signature: `bool NetChangeMap(struct entry_point *entry)`
- Purpose: Broadcast map change request to all players and wait for acknowledgment.
- Inputs: `entry` ΓÇö destination level/entry point.
- Outputs/Return: `false` (operation not supported).
- Side effects: None.
- Calls: None.
- Notes: Map changes are not coordinated across network in dummy mode.

### NetGetNetTime
- Signature: `int32 NetGetNetTime(void)`
- Purpose: Get synchronized network tick count (for animation, event scheduling).
- Inputs: None.
- Outputs/Return: `0` (no network time in dummy mode).
- Side effects: None.
- Calls: None.
- Notes: Single-player uses local frame counter; network time is irrelevant.

### NetAllowCrosshair / NetAllowTunnelVision / NetAllowBehindview
- Signature: `bool NetAllow{Crosshair,TunnelVision,Behindview}(void)`
- Purpose: Check if network game rules allow specific cheat features.
- Inputs: None.
- Outputs/Return: `false` (all cheats disabled).
- Side effects: None.
- Calls: None.
- Notes: Even in single-player dummy mode, these cheats are locked out by convention.

### Void stubs (display_net_game_stats, network_speaker_idle_proc, network_microphone_idle_proc, current_game_has_balls)
- Summarized: All are no-op functions that perform no action.
- `network_gather()` / `network_join()` both return `false` ΓÇö player gathering/joining not supported.

## Control Flow Notes
- Not integrated into normal game frame (init/update/render/shutdown).
- These functions are called by game logic whenever network queries are needed; in dummy mode, they short-circuit to safe defaults without blocking or state transitions.
- Single-player game flow treats `NetGetNumberOfPlayers() == 1` as authoritative; no synchronization overhead.

## External Dependencies
- **Includes:**  
  - `cseries.h` ΓÇö core game framework types/macros (includes `cstypes.h` for fixed-width integer types like `int32`, `short`).
  - `map.h` ΓÇö world geometry structures; provides `struct entry_point` for level/spawn data.
  - `network.h`, `network_games.h` ΓÇö function declarations that are implemented here.
- **Defined elsewhere:**  
  - All function signatures are declared in `network.h` and `network_games.h`; this file provides their definitions for DISABLE_NETWORKING builds.
  - `struct entry_point` defined in `map.h`.
