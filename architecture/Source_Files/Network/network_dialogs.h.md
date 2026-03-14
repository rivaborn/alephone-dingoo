# Source_Files/Network/network_dialogs.h

## File Purpose
Header file defining dialog classes and structures for network game setup, player gathering, joining, and postgame reporting in the Aleph One multiplayer system. Provides abstract base classes with platform-specific implementations (Mac/SDL) for pre-game and postgame UI.

## Core Responsibilities
- Define dialog classes for gathering players (server), joining games (client), and configuring netgame parameters
- Declare structures for player rankings, game outcome data, and network-specific UI state
- Define callback interfaces for network events (player joins/drops, chat messages)
- Declare postgame carnage report visualization functions and data structures
- Provide string IDs and dialog control IDs for resource management across platforms
- Declare helper functions for player color assignment, level selection mapping, and graph rendering

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `net_rank` | struct | Stores kill/death counts, rankings, friendly fire stats, and player identity for postgame display |
| `NetgameOutcomeData` | struct | Mac/NIB-specific: holds control references for postgame UI display (player buttons, kills/deaths text) |
| `GatherDialog` | class | Abstract base for server-side dialog managing incoming player joins, chat, and game start |
| `JoinDialog` | class | Abstract base for client-side dialog displaying available gatherers, handling join attempts, and chat |
| `SetupNetgameDialog` | class | Abstract base for configuring game parameters (limits, difficulty, map, teams, metaserver) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rankings` | `struct net_rank[]` | global | Stores computed player rankings for postgame carnage report |

## Key Functions / Methods

### GatherDialog::Create
- Signature: `static std::auto_ptr<GatherDialog> Create()`
- Purpose: Abstract factory creating platform-specific (Mac/SDL) gather dialog instance
- Inputs: none
- Outputs/Return: `auto_ptr<GatherDialog>` owning new instance
- Side effects: Allocates platform-specific dialog subclass
- Calls: (implementation-dependent, at link-time)
- Notes: Concrete type determined at link-time; called by network setup code

### GatherDialog::GatherNetworkGameByRunning
- Signature: `bool GatherNetworkGameByRunning()`
- Purpose: Runs dialog event loop, waits for players to join, returns on start or cancel
- Inputs: none
- Outputs/Return: bool success/cancel status
- Side effects: Pumps message loop, populates m_ungathered_players and m_pigWidget
- Calls: Run() (pure virtual), Stop() (pure virtual), idle()
- Notes: Server-side entrypoint; manages join acceptance callbacks

### GatherDialog::JoinSucceeded / JoiningPlayerDropped / JoinedPlayerDropped / JoinedPlayerChanged
- Signature: `virtual void JoinSucceeded(const prospective_joiner_info*)`, etc.
- Purpose: Network callbacks invoked when players join/drop during gathering phase
- Inputs: Pointer to prospective_joiner_info (player being added/removed)
- Outputs/Return: void
- Side effects: Updates m_ungathered_players map, updates UI widgets
- Calls: update_ungathered_widget()
- Notes: Final virtual methods (not overridden by subclasses); implements GatherCallbacks interface

### JoinDialog::JoinNetworkGameByRunning
- Signature: `const int JoinNetworkGameByRunning()`
- Purpose: Runs dialog event loop waiting for acceptance or join result
- Inputs: none
- Outputs/Return: int join_result code
- Side effects: Polls for join acceptance, updates chat and status widgets
- Calls: Run() (pure virtual), Stop() (pure virtual), gathererSearch(), attemptJoin()
- Notes: Client-side entrypoint; returns before game starts

### SetupNetgameDialog::SetupNetworkGameByRunning
- Signature: `bool SetupNetworkGameByRunning(player_info*, game_info*, bool, bool&)`
- Purpose: Runs dialog loop for game parameter configuration; populates game_info/player_info on success
- Inputs: player_information (to populate), game_information (to populate), ResumingGame (context), outAdvertiseGameOnMetaserver (to populate)
- Outputs/Return: bool success; fills output structs
- Side effects: Modifies game_info and player_info structures, updates m_allow_all_levels state
- Calls: Run() (pure virtual), Stop() (pure virtual), limitTypeHit(), gameTypeHit(), chooseMapHit(), informationIsAcceptable()
- Notes: Server-side; called via network_game_setup() function

### draw_new_graph / draw_player_graph / draw_totals_graph (Postgame Report)
- Signature: `extern void draw_new_graph(NetgameOutcomeData &outcome)`, etc.
- Purpose: Render postgame carnage report graphs (player kills, totals, scores, team tallies)
- Inputs: outcome (result display data), optional index (for single-player graph)
- Outputs/Return: void (renders directly)
- Side effects: Direct drawing to active display context
- Calls: (implementation in network_dialogs.cpp)
- Notes: Five graph types: _player_graph, _total_carnage_graph, _total_scores_graph, _total_team_carnage_graph, _total_team_scores_graph

**Notes on trivial helpers**: calculate_rankings(), rank_compare(), team_rank_compare(), score_rank_compare() compute sorted rankings; reassign_player_colors() assigns team colors to players; menu_index_to_level_entry()/level_index_to_menu_index() map between UI indices and level data.

## Control Flow Notes
- **Init phase**: SetupNetgameDialog runs to populate game_info before NetStart()
- **Pre-game gather**: GatherDialog runs on server, accepts JoinSucceeded/Dropped callbacks from network layer
- **Pre-game join**: JoinDialog runs on client, polls for acceptance via NetCheckForNewJoiner()
- **In-game**: Chat callbacks (ReceivedMessageFromPlayer) fire during active netgame
- **Post-game**: Carnage report functions render outcome data after NetDown state

## External Dependencies
- **Headers included**: player.h (MAXIMUM_NUMBER_OF_PLAYERS=8), network.h, network_private.h (JoinerSeekingGathererAnnouncer), FileHandler.h (file dialogs), network_metaserver.h, metaserver_dialogs.h (GlobalMetaserverChatNotificationAdapter), shared_widgets.h (ButtonWidget, EditTextWidget, etc.)
- **Defined elsewhere**: GatherCallbacks, ChatCallbacks (network.h), player_info, game_info (network.h), entry_point (map.h), prospective_joiner_info (network.h), ColorfulChatWidget, FileChooserWidget, EditNumberWidget, SelectorWidget (shared_widgets.h)
- **Conditional compilation**: USES_NIBS (Mac resource forks), DISABLE_NETWORKING (disable all networking)
