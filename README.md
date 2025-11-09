# Audio Replicator Plugin

Audio Replicator provides Blueprint-accessible helpers to encode PCM16 audio into Opus frames, move those frames around the network, and rebuild playable audio on the receiving side. The plugin is designed for short voice clips or burst-style transmissions that can be encoded up front before being streamed to peers.

## Supported Platforms
This plug-in was last built against Unreal Engine 5.6 and Windows.

All required libraries are included in the ThirdParty directory and work out of the box.

## Feature summary

* **Local encode/decode utilities** – `UAudioReplicatorBPLibrary` loads PCM16 WAV files, converts the samples to Opus packets, and restores packets back to PCM16 or WAV output when needed.
* **Packet persistence helpers** – Blueprint nodes expose packing/unpacking for serialized Opus data so the frames can be written to disk or cached in save data.
* **Network-ready actor component** – `UAudioReplicatorComponent` handles reliable header delivery, chunked frame replication, and transfer bookkeeping so gameplay code only needs to trigger broadcasts and react to events.
* **World registry subsystem** – `UAudioReplicatorRegistrySubsystem` tracks every replicator in the current world, exposes Blueprint subscription hooks, and resolves the most recent sender for a given Opus session.
* **Blueprint-friendly data types** – `FOpusStreamHeader`, `FOpusPacket`, and `FOpusChunk` wrap stream metadata and per-frame payloads to comply with UFUNCTION restrictions while keeping packet ordering intact.
* **Runtime debugging** – Optional helpers expose formatted status text and per-session diagnostics for both outgoing and incoming transfers to help visualize replication health.

## Quick start

1. Add a replicated `UAudioReplicatorComponent` to a replicated client-owned actor (a `PlayerState` is ideal).
2. **Prepare source audio** with the Blueprint library: load a PCM16 WAV, encode it to Opus packets, or call the convenience node `TranscodeWavToOpusAndBack` to validate round-tripping.
3. **Start a broadcast** from the owning client:
   * Use `StartBroadcastFromWav` to encode and stream straight from a WAV file, or
   * Use `StartBroadcastOpus` if you already have packets plus a `FOpusStreamHeader` describing the stream.
4. **React to replication events** on every client: bind to `OnTransferStarted`, `OnChunkReceived`, and `OnTransferEnded` to drive UI or progress tracking. Once the transfer ends, `GetReceivedPackets` returns the assembled frame list and header so you can decode or save the data locally.
5. **Leverage the registry subsystem** when you need to discover replicators dynamically. The world subsystem fires `OnReplicatorAdded` / `OnReplicatorRemoved` and supports Blueprint subscriptions that map Opus session IDs or `APlayerState` owners to active components.
6. **Optionally cancel** in-flight transfers with `CancelBroadcast`, or query `GetOutgoingDebugInfo` / `GetIncomingDebugInfo` to surface detailed state in debug widgets.

The component automatically sequences frames, sends the Opus stream header reliably, throttles the number of packets sent each tick, and replicates completion markers once the queue drains. Incoming clients maintain an indexable buffer that is safe against missing headers and can tolerate unknown packet counts by expanding on demand.

## Registry subsystem overview

`UAudioReplicatorRegistrySubsystem` centralizes discovery for gameplay code that needs to coordinate multiple replicators. The subsystem automatically registers components as they begin play, re-scans the `AGameStateBase` for existing `APlayerState` owners, and tracks which component most recently broadcast for each Opus session. Blueprint callbacks can subscribe by session GUID or player owner, receive immediate notifications when a matching replicator is available, and listen for lifecycle events to drive UI affordances such as speaker lists.

## Data flow overview

1. **Header broadcast** – The owning client generates a `FOpusStreamHeader` (sample rate, channel count, bitrate, frame size, and optional packet count) either from encoding a WAV file or by supplying its own values. The server relays this header reliably to all clients before any frame data is transmitted.
2. **Chunked frame replication** – Each Opus packet is wrapped in a `FOpusChunk` with a monotonically increasing index. The component sends up to `MaxPacketsPerTick` chunks per frame (default 32) and tracks which chunks have been delivered.
3. **Transfer completion** – When all chunks are sent, a reliable end marker is multicast so listeners know the payload is ready. Clients can then decode the packets back into PCM16 or write them to disk via the Blueprint library helpers.

## Debugging helpers

* **Blueprint debug strings** – `FormatAudioTestReport`, `OpusStreamHeaderToString`, `FormatOutgoingDebugReport`, and `FormatIncomingDebugReport` convert runtime stats into log-friendly strings for UI widgets or on-screen messages.
* **Structured transfer snapshots** – `FAudioReplicatorOutgoingDebug` and `FAudioReplicatorIncomingDebug` expose chunk-level bookkeeping, missing indices, byte totals, and estimated duration/bitrate so you can quickly identify replication problems.

## Best practices & constraints

* Input WAV files must contain PCM16 little-endian samples; other encodings should be converted before use.
* Keep broadcasts client-authoritative: only the owning client should call `StartBroadcast*` so the server RPCs execute successfully.
* Attach the component to actors that exist on every client (e.g., controllers or pawns) and ensure the actor replicates.
* Default stream settings target 48 kHz audio, mono channel, 20 ms frames, and 32 kbps bitrate; adjust `FOpusStreamHeader` as needed for stereo or higher quality content.
* Each `FOpusChunk` carries a single Opus packet whose payload is typically much smaller than the 65 KB limit enforced by the chunking helpers, making it safe for Unreal RPC transport.

## Related files

* `AudioReplicator.uplugin` – module descriptor with category and startup configuration.
* `Source/AudioReplicator/Public` – Blueprint API headers (function library, component, registry subsystem, data types, debug helpers).
* `Source/AudioReplicator/Private` – Implementation of encoding, chunking, registry bookkeeping, and network replication behavior.
* `Source/ThirdParty/Opus` – Build script and static library wrapping libopus for encoding/decoding support.

