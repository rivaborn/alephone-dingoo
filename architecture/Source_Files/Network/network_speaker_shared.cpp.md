# Source_Files/Network/network_speaker_shared.cpp

## File Purpose
Handles receiving and decoding networked audio from remote players for local playback. Manages player muting and audio filtering based on team settings. Decodes compressed Speex audio when available.

## Core Responsibilities
- Receive network audio packets and decode compressed frames (Speex)
- Filter audio based on team-only chat settings and local player team
- Queue decoded audio data for playback via the speaker system
- Manage player microphone mute state (per-player ignore list)
- Clear all player mutes

## Key Types / Data Structures
| Name | Kind | Purpose |
| --- | --- | --- |
| network_audio_header | struct | In-memory audio packet metadata (mReserved=format flag, mFlags=settings like team-only) |
| network_audio_header_NET | struct | Wire-format audio header (platform-independent byte array) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
| --- | --- | --- | --- |
| sIgnoredPlayers | std::set\<short\> | static | Set of player indices whose microphone audio is ignored/muted |

## Key Functions / Methods

### received_network_audio_proc
- Signature: `void received_network_audio_proc(void *buffer, short buffer_size, short player_index)`
- Purpose: Callback invoked by network distribution system when audio packet arrives from a remote player
- Inputs: `buffer` (raw packet), `buffer_size` (bytes), `player_index` (sender's player ID)
- Outputs: None (side effects only)
- Side effects:
  - Checks sIgnoredPlayers; returns early if player is muted
  - Converts network header format via netcpy()
  - Checks team-only flag and local/remote player teams; may skip playback
  - If Speex enabled: decodes up to 12 variable-length frames (160 samples/frame) into static buffer
  - Queues decoded audio (160 * 2 * numFrames bytes) for playback
- Calls: netcpy(), get_player_data(), speex_bits_read_from(), speex_decode_int(), queue_network_speaker_data()
- Notes:
  - Buffer layout: network_audio_header_NET followed by audio payload
  - Speex: mReserved==1 signals compression; each frame prefixed with 1-byte length
  - Team filter: blocks unless (no team-only flag) OR (local_player->team == remote_player->team)

### mute_player_mic
- Signature: `void mute_player_mic(short player_index)`
- Purpose: Toggle mute status for a single player's microphone
- Inputs: `player_index`
- Outputs: None
- Side effects: Inserts/erases player from sIgnoredPlayers; prints status via screen_printf()
- Calls: std::set::find(), erase(), insert(), screen_printf()
- Notes: Acts as toggle; provides on-screen feedback

### clear_player_mic_mutes
- Signature: `void clear_player_mic_mutes()`
- Purpose: Unmute all players
- Inputs: None
- Outputs: None
- Side effects: Clears sIgnoredPlayers
- Calls: std::set::clear()

## Control Flow Notes
Callback-driven: Network receive event ΓåÆ received_network_audio_proc() ΓåÆ decode ΓåÆ queue for playback. Mute functions invoked from input/UI layers. No init/shutdown in this file (those are in companion files network_speaker_*.cpp per platform).

## External Dependencies
- **network_sound.h**: queue_network_speaker_data() (playback queuing), forward declarations
- **network_data_formats.h**: network_audio_header_NET struct, netcpy() (networkΓåöhost format conversion)
- **network_audio_shared.h**: network_audio_header struct, kNetworkAudioForTeammatesOnlyFlag, audio format constants
- **player.h**: player_data struct, get_player_data(), local_player global
- **shell.h**: screen_printf()
- **speex/speex.h** (optional, #ifdef SPEEX): speex_bits_read_from(), speex_decode_int()
- **network_speex.h**: gDecoderBits, gDecoderState globals
- **cseries.h**: Common engine types (byte, uint32, etc.)
- Standard: \<set\>
