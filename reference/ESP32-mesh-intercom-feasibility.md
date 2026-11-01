# ESP32 feasibility study for the hands-free mesh intercom

Research date: October 31, 2026. This is a documentation-based engineering assessment, not a hardware test report. Proposed thresholds and numerical examples below are design assumptions, explicitly distinguished from vendor measurements.

## Decision

**ESP32-S3 is a credible candidate for a compact four-person intercom, including overlapping conversation. Its main unresolved risk is mobile mesh behavior, rather than whether it can send compressed speech.** Recommend a bounded prototype before committing the integrated hardware to an ESP32-only network.

Use existing audio drivers, codecs, and speech processing. Compare routerless Mesh-Lite against an ESP-NOW transport behind the same application interface. Mesh-Lite offers a quicker networking experiment but has root-dependence and support-policy concerns. ESP-NOW avoids Wi-Fi association but leaves us responsible for a substantial mesh protocol. Raw packet injection is not an attractive production foundation.

If interoperable 802.11s remains mandatory, ESP32's supported networking stacks do not satisfy that requirement. Retain the Linux/OpenWrt radio path, or use ESP32 as an audio endpoint connected to that infrastructure.

## Requirements and interpretation

The user confirmed at least four nodes and free conversation: one person's speech must not prevent anyone else from speaking. Previous project discussions established mesh as a baseline, operation without a separate router, automatic reconnection, and software that can migrate from prototype boards into integrated hardware.

For evaluation, assume four concurrent captured speech streams, one encoder and up to three remote decoders per endpoint, and local mixing. This is a conservative acceptance case, not an assumption that four people normally talk continuously. Test both direct connectivity and paths with two or three wireless hops. Range, environment, battery runtime, headset versus open speaker, and acceptable interruption during recovery remain unspecified.

“Full duplex” here describes the user experience. A single half-duplex radio can alternate short transmit and receive intervals while audio capture and playback continue. TDMA can provide this experience, but TDMA does not itself compress, route, mix, or conceal lost speech. The alternative is ordinary Wi-Fi contention, which can also support overlapping conversation if airtime and delay permit it. There is no need to restrict the product to push-to-talk simply because the radio cannot transmit and receive simultaneously.

“No external AP” is also different from “no node ever acts as an AP.” The former permits an ESP32 tree made from internal SoftAP/station links. The latter excludes those stacks and makes ESP-NOW more relevant.

## What lower-level Wi-Fi access actually gives us

| Capability | Available mechanism | Value for our intercom |
|---|---|---|
| Direct messages without association | ESP-NOW | Useful candidate transport for our own mesh |
| Signal and receive metadata | RSSI and receive-control information | Link assessment and troubleshooting; combine with measured loss and delay |
| Packet observation | Promiscuous/sniffer receive callbacks | Useful during testing; avoid continuous broad capture in the normal audio path |
| Selected raw frame transmission | `esp_wifi_80211_tx()` | Research tool; limited substitute for a supported network transport |
| Channel State Information | CSI callbacks | Optional propagation experiments; low initial priority |

The current S3 documentation permits selected management/action frames and non-QoS data through the raw transmit API, and excludes encrypted and QoS frame transmission through that API. CSI exposes estimates of subcarrier channel responses, rather than raw radio samples or decoded microphone sound. Metadata includes information such as RSSI and reception time. These features do not establish a supported arbitrary MAC implementation or hard real-time transmit slots. [Wi-Fi vendor features](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/wifi-driver/wifi-vendor-features.html)

Promiscuous capture can run alongside AP/station operation, but Espressif warns that it can significantly affect throughput. Its callback runs in the Wi-Fi driver task; move work to another task. Our own received packet metadata plus sequence-number statistics should be sufficient initially. [Wi-Fi modes and sniffer](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/wifi-driver/wifi-modes.html)

Parts of the Wi-Fi stack ship as precompiled libraries. Application-level access should not be confused with access to all firmware internals. Consequently, implementing interoperable 802.11s or precise TDMA would be a separate research effort with uncertain support and portability. [Official Wi-Fi library repository](https://github.com/espressif/esp32-wifi-lib)

CSI is unlikely to be the deciding feature. Start with packet arrival rate, burst loss, latency, queue depth, and bidirectional delivery quality. A strong RSSI can coexist with a busy channel or an asymmetric link.

## Existing network choices

| Choice | What already exists | What we would own | Assessment |
|---|---|---|---|
| Ordinary AP/station Wi-Fi + UDP | Association, IP, sockets, ordinary link security | Voice application and reconnect handling | Best baseline; a star alone does not meet mesh requirement |
| ESP-WIFI-MESH | Parent/child tree, forwarding, topology management | Voice, group membership, recovery integration | Valid reference implementation; target/version support must be checked |
| ESP-MESH-LITE | SoftAP/station tree and IP access | Voice, local sessions, root policy, mobility validation | Quick multi-hop experiment with material caveats |
| ESP-NOW + our mesh | Direct vendor action-frame communication | Discovery, routing, forwarding, recovery, secure groups | Most relevant custom embedded path; much more work |
| painlessMesh | Arduino-oriented discovery and message routing | Audio integration and substantial transport adaptation | Useful control demonstration; poor default voice transport |
| Raw injection + sniffing | Some frame-level APIs | Almost the entire usable communication system | Do not choose as the primary implementation |

ESP-WIFI-MESH organizes links as a tree and supplies self-organization and repair. Its network is not a set of interchangeable 802.11s peers. Mesh data APIs also differ from ordinary IP sockets at every node. [Mesh guide](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-guides/esp-wifi-mesh.html), [mesh API](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/network/esp-wifi-mesh.html)

Mesh-Lite uses SoftAP plus station mode and lets nodes use IP networking. That is useful for retaining a UDP media application. However, independent upstream IP connectivity does not prove unrestricted incoming connectivity between every branch. Explicitly verify child-to-child media, addressing, any NAT behavior, and socket recovery after reparenting. [Mesh-Lite repository](https://github.com/espressif/esp-mesh-lite)

### Mesh-Lite caveats that affect the decision

The official no-router example explicitly configures one board as root and the others as non-root. It proves a routerless configuration exists; it does not prove autonomous root replacement, continued operation in both halves of a split group, or automatic group merge. Those are separate acceptance tests. [No-router example](https://github.com/espressif/esp-mesh-lite/blob/release/v1.0/examples/no_router/README.md)

The public user guide lists 8–12 ms per-hop delay and repair times below 50 seconds for root loss and below 45 seconds for child disconnection, under its stated 50-device configuration. These are vendor example results, not voice guarantees or lower bounds. They show why “self-healing” cannot be interpreted as conversational continuity. [Mesh-Lite user guide](https://github.com/espressif/esp-mesh-lite/blob/release/v1.0/components/mesh_lite/User_Guide.md)

More significantly, the public repository's support policy says it is in maintenance mode, with essential security and critical bug fixes, while future development and commercial support move to a private repository. We should evaluate the public version as it stands and determine commercial access terms before making it a product dependency. [Support policy](https://github.com/espressif/esp-mesh-lite/blob/release/v1.0/components/mesh_lite/SUPPORT_POLICY.md)

The inspected public component manifest declares Mesh-Lite 1.0.2, ESP-IDF >=5.0 and IoT-Bridge 1.0.1. This is a snapshot of the inspected branch, not a claim that it is the latest commercial version. A minimum IDF constraint does not prove compatibility with every newer IDF/audio combination. Pin a tested dependency set. [Component manifest](https://github.com/espressif/esp-mesh-lite/blob/release/v1.0/components/mesh_lite/idf_component.yml)

The ESP-MDF master README itself advises against using that branch for product development. Do not adopt an old MDF tutorial as the product baseline without checking its dependency and target requirements. [ESP-MDF README](https://github.com/espressif/esp-mdf)

### ESP-NOW: useful transport, incomplete mesh

ESP-NOW sends connectionless vendor action frames. Current documentation lists 250-byte v1 payloads and 1,470-byte v2 payloads, with version interoperability limits. The default bit rate is 1 Mbit/s. The inspected ESP32 documentation lists 20 paired peers and up to 17 encrypted peers, with seven encrypted peers by default; verify the selected target and SDK. It supports encrypted unicast but not encrypted multicast. A successful send callback confirms MAC-level success, not end-to-end application receipt. Peers must meet the radio's channel requirements. [ESP-NOW API](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/network/esp_now.html)

Four endpoints fit comfortably within those documented peer counts. Peer count is not the hard part. We still need:

- Authenticated discovery, group identity, and enrollment.
- Bidirectional neighbor quality, route selection, route expiry, and hysteresis.
- Forwarding with source identity, sequence numbers, hop limits, and duplicate suppression.
- A controlled distribution strategy for each speaker's stream.
- Bounded queues, voice deadlines, and a small retry budget.
- Partition operation, merge handling, and channel rediscovery.
- Secure group media, replay rejection, and key changes when membership changes.

For a four-node experiment, bounded duplicate-suppressed flooding can establish feasibility quickly. It should be treated as an experiment, not presumed scalable. Compare it against explicit next-hop forwarding. Naive rebroadcasting increases channel competition and duplicate traffic precisely when reception is already poor.

painlessMesh describes JSON messaging over its node connections, warns about high message rates and dropped messages, and uses AsyncTCP dependencies. That is a mismatch with frequent deadline-sensitive binary audio packets. Its convenient mesh name does not establish voice readiness. [painlessMesh README](https://github.com/gmag11/painlessMesh)

## Audio technology we can reuse

Use ESP-IDF and FreeRTOS as the base. Build a small composition of audio components instead of automatically combining every Espressif framework.

| Layer | Reuse candidate | Project-specific work |
|---|---|---|
| Microphone and playback | I2S drivers, `esp_codec_dev`, board support | Microphone placement, wiring, levels, headphone/speaker hardware |
| Speech front end | ESP-SR AFE | Tune for actual acoustics, wind and double talk |
| Speech compression | `esp_audio_codec`, initially Opus | Select settings and measure combined workload |
| Standard IP calls, if needed | ESP-WebRTC or ESP media protocols SIP/RTP | Offline discovery/signaling and group integration |
| Four-person conference | Codec and DSP primitives | Per-stream buffering, decode, mixing, limiting and clock correction |
| Maintenance | ESP-IDF update and diagnostic facilities | Product update flow, logs, recovery and user controls |

Espressif's Opus component supports mono voice configurations, adjustable complexity, FEC and DTX; its documented encoder range starts at 20 kbit/s. Start at 24 kbit/s, 16 kHz mono, 20 ms packets and low complexity. The published S3R8 benchmark reports 24.9% encoding CPU and 5.86% decoding CPU for a different 48 kHz stereo configuration at 90 kbit/s and encoder complexity zero. These numbers establish useful prior evidence, but do not predict our four-party system. [Codec features and benchmarks](https://github.com/espressif/esp-adf-libs/blob/master/esp_audio_codec/README.md)

ESP-SR AFE provides echo cancellation, noise processing, VAD and gain-related processing. AEC needs the actual playback reference, which in our design is the final local mix. Validate double talk, clipping, acoustic coupling and wind with the intended microphone and enclosure. Voice detection alone does not make a usable hands-free intercom. [AFE documentation](https://docs.espressif.com/projects/esp-sr/en/latest/esp32s3/audio_front_end/README.html), [audio FAQ](https://docs.espressif.com/projects/esp-faq/en/latest/application-solution/audio-development-framework.html)

The AFE benchmark's single-microphone low-cost voice-communication configuration uses about 48.7 KB internal RAM and 819.7 KB PSRAM, with feed/fetch CPU figures of 30.6% and 4.7% of one core. Settings matter; do not add percentages across unrelated benchmarks as though they measured the integrated application. This is a strong reason to start with PSRAM. [AFE benchmark](https://docs.espressif.com/projects/esp-sr/en/latest/esp32s3/benchmark/README.html)

ESP-WebRTC supplies connection/media components and examples; SIP/RTP support is also available. These are alternatives for the IP media layer, not extra mesh routing layers. They cannot simply run unchanged on ESP-NOW without an IP adaptation or a different media transport. We do not need browser interoperability or SIP registration to prove a closed four-node intercom. [WebRTC solution](https://github.com/espressif/esp-webrtc-solution), [SIP stack](https://github.com/espressif/esp-adf-libs/blob/master/esp_media_protocols/docs/en/SIP_README.md)

An example named `peer_demo` is not necessarily a real microphone demonstration: the inspected solution README describes fake audio and chat data. Use a real capture/playback application to prove speech quality. The local doorbell example demonstrates device-hosted signaling and two-way audio, but is still not a four-node mobile mesh. [Peer demo](https://github.com/espressif/esp-webrtc-solution/blob/main/solutions/peer_demo/README.md), [local doorbell](https://github.com/espressif/esp-webrtc-solution/blob/main/solutions/doorbell_local/README.md)

## Capacity, latency and TDMA

The following calculations are ours, not measurements.

At 16 kHz and 16 bits, mono PCM is 256 kbit/s. At 24 kbit/s compressed speech, a 20 ms frame averages 60 bytes and produces 50 packets/s. Four active sources produce 96 kbit/s of unique compressed audio. A receiver consumes 72 kbit/s of remote codec payload before overhead.

With RTP/UDP/IPv4, 12 + 8 + 20 bytes of basic headers bring a 60-byte frame to 100 bytes, or 40 kbit/s per stream on one path before Wi-Fi framing, security, acknowledgements, retries, or forwarding. ESP-NOW omits those IP headers when used directly, but still has its own over-the-air framing and contention overhead.

Topology changes the result dramatically. In a four-node line A–B–C–D, independently unicasting every source to each of the other three receivers gives 20 directed hop transmissions per 20 ms round: 1,000 packet transmissions/s. At the illustrative 100 bytes and a hypothetical fixed 1 Mbit/s PHY, payload plus those basic headers alone occupy 80% of airtime. Real overhead makes that design untenable at that rate in one collision domain. This is an illustrative lower-bound calculation, not an ESP-NOW throughput measurement. Shared distribution and higher viable PHY rates can improve the result; range-oriented lower rates can worsen it.

Prefer distributing each encoded source efficiently, then decoding and mixing at recipients. Relays should forward encoded packets rather than repeatedly decode and re-encode them. A central mixer can reduce endpoint decoder count but adds a processing bottleneck, personalized mixes and a dependency on one device. Neither design should be selected from nominal radio bitrate alone.

For initial engineering targets, aim for <=150 ms one-way mouth-to-ear latency at the 95th percentile in the intended four-user scenario, with a separately measured 99th percentile and gap distribution. An illustrative budget is 20 ms frame collection + 5–15 ms processing + 10–40 ms networking + 20–40 ms jitter buffer + 5–10 ms output = 60–125 ms. Codec lookahead, front-end buffering and contention can add more. Measure total acoustic latency; a network ping cannot validate it.

Use sequence numbers and sender sample timestamps, a separate jitter buffer per source, packet loss concealment, and correction for independent audio-clock drift. A permanent sample-rate mismatch eventually drains or fills buffers even on a lossless network. Opus FEC may require waiting for a following packet, trading latency for repair; tune it against the delay budget.

For IP operation, S3 supports Wi-Fi access categories and mapping socket IP precedence to voice priority. This can help contention priority but is not bandwidth reservation or a TDMA schedule. Verify treatment through the selected mesh path. [Wi-Fi QoS](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/wifi-driver/wifi-mac-protocols.html)

Application time slots can pace ESP-NOW traffic, but Wi-Fi contention, driver queues and task scheduling still influence actual transmission time. Treat such slots as soft scheduling until measured. If precise scheduled airtime is a hard requirement, select a radio/stack that explicitly exposes and supports it rather than relying on raw-frame APIs.

## Reconnection, motion and interference

For an ordinary Wi-Fi station, disconnection events and retry policy require software handling; current documentation describes calling connect again after disconnection. A mesh package may supply this above the driver. Network reconnection still leaves us responsible for restoring media and membership state. [Station reconnect documentation](https://docs.espressif.com/projects/esp-idf/en/stable/esp32s3/api-guides/wifi-driver/station-scenarios.html)

Test A–B–C–D with the middle link removed. A useful product should let A/B and C/D continue local conversations if that is the intended partition policy. Restoring the link must merge the group without duplicate speakers, route loops or manual re-pairing. A fixed-root example does not establish this behavior.

Test radio loss separately from application recovery: record last good audio, failure detection, new route, first received packet, and first intelligible playback. Reparenting, address changes and channel scans can break audio even after a basic connectivity test succeeds.

Channel selection is another coordinated state machine. Scanning on a single radio interrupts listening on the working channel. A node switching alone can strand itself; a disconnected node needs a way to rediscover the group's channel. Begin with a fixed legal channel, then evaluate coordinated switching and partition merge under interference. Two independent groups, hidden transmitters and body shadowing belong in the test matrix.

Do not promise a distance from generic ESP-NOW or Wi-Fi marketing. Antenna placement on a worn device, bodies, walls, terrain and interference determine the usable voice link. The S3 is a 2.4 GHz Wi-Fi device; its integrated radio is not HaLow. Espressif long-range modes are separate proprietary features, not a software conversion to sub-GHz. Any low-rate mode must be retested for four-stream airtime and chip/stack compatibility. [Chip comparison](https://docs.espressif.com/projects/esp-idf/en/v5.0/esp32/hw-reference/chip-series-comparison.html), [protocol support overview](https://docs.espressif.com/projects/esp-techpedia/en/latest/esp-friends/advanced-development/wifi-protocol-support.html)

## Hardware and migration to integration

Start with ESP32-S3 plus 8 MB PSRAM. An S3-Korvo-2 audio board is a reasonable reference for audio validation; use the matching board support and validate its actual audio paths. For modular transport testing, an S3 development board with external I2S audio is also suitable. The DevKitC-1 N8R8 family provides 8 MB flash and 8 MB PSRAM variants. [Audio board guidance](https://docs.espressif.com/projects/esp-sr/en/latest/esp32s3/getting_started/readme.html), [development board variants](https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32s3/esp32-s3-devkitc-1/user_guide_v1.0.html)

This is a starting recommendation, not a claim that S3 is the only capable ESP32. Newer Wi-Fi generations do not automatically solve mobile mesh or make a chip a drop-in replacement for our tested audio stack. P4 is not a single-chip Wi-Fi alternative: it needs an external wireless SoC. [P4 documentation](https://docs.espressif.com/projects/esp-idf/en/release-v6.0/esp32p4/get-started/index.html)

For the integrated device, preserve the same module class, PSRAM configuration, audio interfaces and supported antenna arrangement where practical. Add the battery/charger, regulator peak-current capacity, microphone bias or digital mic supply, speaker/headphone stage, buttons and enclosure. Continuous receive and relay duties are not deep-sleep workloads; measure whole-device consumption during conversation, silence, recovery and relay operation.

Keep the firmware split into board audio, speech processing, codec, media/session logic and transport. A common packet should carry protocol version, group and source identity, stream epoch, sequence, sample timestamp and payload. Transport-specific next-hop and hop-limit fields belong in the networking layer. Reset or rotate epochs safely across reboot, including security nonce handling.

The reusable core should be portable C/C++ with explicit timing and allocation interfaces. Supply UDP/IP and ESP-NOW adapters. Protocol definitions, codec settings, mixing rules, test recordings and network simulation cases can carry into integrated hardware or Linux. ESP-IDF drivers, FreeRTOS scheduling, AFE binaries and ESP-NOW do not automatically port to Linux. ESP-NOW would require a compatible radio peer or gateway; a Linux Wi-Fi interface cannot be presumed to support it as a normal socket transport.

## Prototype plan and decision gates

1. **Four-node audio baseline.** On a controlled Wi-Fi network, run real microphones and playback with all four streams active. Measure one encode plus three decodes per node, mixing, echo behavior, acoustic latency, internal heap, PSRAM, stack use and CPU margin. A wired headset is useful for isolating transport issues, followed by the intended acoustics.
2. **Routerless Mesh-Lite trial.** Prove four-node traffic, then forced two- and three-hop routes. Test branch-to-branch media, root loss, non-root loss, partition and merge. If fixed-root behavior is unacceptable, stop treating this as the final network.
3. **ESP-NOW comparison.** Reuse the same media core. Begin with one hop, then bounded relaying. Measure loss versus queue age, per-neighbor delivery, duplicate traffic, and airtime under four active streams. Implement explicit rediscovery rather than relying on association behavior.
4. **Wearable and interference tests.** Test people turning, walking apart and returning, hidden nodes, another intercom group and busy Wi-Fi. Repeat with the intended battery, antenna and enclosure geometry.
5. **Integration decision.** Select ESP32-only networking only after both conversational quality and recovery policy pass. Otherwise preserve the audio/core work and continue with compatible Linux/OpenWrt mesh radios.

Proposed initial gates, to be confirmed against product expectations:

| Gate | Proposed criterion |
|---|---|
| Overlapping conversation | All four users can transmit and receive continuously; no forced talk token |
| Latency | <=150 ms p95 mouth-to-ear under the target topology; report p99 and burst gaps |
| Recovery | Aim for <=1 s restoration after a usable alternate link exists; report detection and media restart separately |
| Partitions | Both connected subgroups remain useful; rejoin automatically when links return |
| Compute margin | Aim for >=25% headroom on the busiest core; no growing queues or audio underruns |
| Stability | Initial one-hour four-stream soak; no resets, memory decline or progressive clock drift |
| Power and range | Measure against the eventual runtime and terrain requirements; no numerical promise yet |

The recovery and latency gates are our proposed product goals, not vendor guarantees. A prototype might demonstrate automatic recovery while failing the one-second target; record that honestly rather than calling every reconnection a pass.

**Recommendation:** proceed with an S3 audio prototype and a tightly scoped comparison of Mesh-Lite and ESP-NOW. Keep four simultaneous speakers as the acceptance load. Existing components cover much of audio and radio access; we still own the group intercom experience, and potentially the entire mobile mesh layer. Avoid committing the final board until range, recovery and sustained four-stream load have been measured.
