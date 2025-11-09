# Audio Replicator Plugin

Blueprint-accessible plugin for encoding PCM16 audio into Opus frames and replicating them across the network. Designed for short voice clips and burst-style audio transmissions in multiplayer Unreal Engine games.

## Requirements

- **Unreal Engine**: 5.6+
- **Platform**: Windows (tested)
- **Dependencies**: Included in ThirdParty directory

## Installation

1. Copy the plugin to your project's `Plugins` folder
2. Enable "Audio Replicator" in Edit → Plugins
3. Restart the editor

## Quick start
### 1. Setup Component
Add a replicated `UAudioReplicatorComponent` to a replicated client-owned actor (a `PlayerState` is ideal).
### 2. Broadcast Audio

**Option A: From WAV file**

```
StartBroadcastFromWav(WavAsset)
```

**Option B: From Opus packets**

```
StartBroadcastOpus(Packets, StreamHeader)
```
   ![[Pasted image 20251109212010.png]]
### 3. Subscribe

**Using Registry Subsystem** (recommended):

```
SubscribeToPlayer(PlayerState), SubscribeToChannel(SessionID)
```

**Using Component Directly**:

```
Bind to: OnTransferStarted, OnChunkReceived, OnTransferEnded
```
  ![[Pasted image 20251109213233.png]]
### 4. Receive and Decode Audio
 Once the transfer ends, `GetReceivedPackets` returns the assembled frame list and header so you can decode or save the data locally.
   ![[Pasted image 20251109213403.png]]

## Core Concepts

### Audio Flow

```
Client A              Server              Client B
   |                     |                    |
   |--[Header]---------->|--[Header]--------->|
   |--[Chunks 1-32]----->|--[Chunks 1-32]---->|
   |--[Chunks 33-64]---->|--[Chunks 33-64]--->|
   |--[End Marker]------>|--[End Marker]----->|
   |                     |                    |
                    Decode & Play
```

### Key Components

- **UAudioReplicatorBPLibrary** - Encode/decode utilities
- **UAudioReplicatorComponent** - Network replication handler
- **UAudioReplicatorRegistrySubsystem** - Multi-player discovery system

### Data Types

- **FOpusStreamHeader** - Stream metadata (sample rate, channels, bitrate)
- **FOpusPacket** - Single compressed audio frame
- **FOpusChunk** - Replicated packet wrapper with index

## API Reference

### Component Methods

|Method|Description|
|---|---|
|`StartBroadcastFromWav(WAV)`|Encode and stream a WAV file|
|`StartBroadcastOpus(Packets, Header)`|Stream pre-encoded Opus data|
|`CancelBroadcast()`|Stop current transmission|
|`GetReceivedPackets()`|Retrieve assembled frames after transfer|

### Registry Methods

|Method|Description|
|---|---|
|`SubscribeToChannel(GUID, Callback)`|Listen for specific session|
|`SubscribeToPlayer(PlayerState, Callback)`|Listen for specific player|
|`FindReplicatorForPlayer(PlayerState)`|Get component reference|

### Library Functions

|Function|Description|
|---|---|
|`TranscodeWavToOpus(WAV)`|Convert WAV to Opus packets|
|`DecodeOpusToWav(Packets, Header)`|Convert Opus to WAV|
|`DecodeOpusToPCM16(Packets, Header)`|Convert Opus to raw samples|

## Configuration

Default stream settings (adjustable via `FOpusStreamHeader`):

- **Sample Rate**: 48 kHz
- **Channels**: Mono
- **Frame Size**: 20 ms
- **Bitrate**: 32 kbps
- **Packets Per Tick**: 32

## Debugging

### Debug Functions

- `FormatOutgoingDebugReport()` - Outgoing transfer stats
- `FormatIncomingDebugReport()` - Incoming transfer stats
- `OpusStreamHeaderToString()` - Stream configuration details

### Debug Data Structures

- `FAudioReplicatorOutgoingDebug` - Chunk tracking, byte counts, progress
- `FAudioReplicatorIncomingDebug` - Missing indices, buffer state, bitrate

## Best practices & constraints

* Input WAV files must contain PCM16 little-endian samples; other encodings should be converted before use.
* Keep broadcasts client-authoritative: only the owning client should call `StartBroadcast*` so the server RPCs execute successfully.
* Attach the component to actors that exist on every client (e.g., controllers or pawns) and ensure the actor replicates.
* Default stream settings target 48 kHz audio, mono channel, 20 ms frames, and 32 kbps bitrate; adjust `FOpusStreamHeader` as needed for stereo or higher quality content.
* Each `FOpusChunk` carries a single Opus packet whose payload is typically much smaller than the 65 KB limit enforced by the chunking helpers, making it safe for Unreal RPC transport.


