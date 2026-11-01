# Hands-Free Mesh Intercom — Project Requirements

Status: Initial requirements baseline for discussion  
Version: 0.1  
Updated: October 31, 2026

This document defines the product intent, use cases and high-level requirements. It records the requirements agreed so far and identifies decisions still needed. Detailed hardware, software, protocol and mechanical specifications will be developed separately. A platform appearing in a feasibility study is not a selected design.

## 1. Product intent

Build a compact, battery-powered group intercom for people participating in outdoor activities together, including skiing, riding and hiking. Users should be able to converse naturally while moving, without a separate Wi-Fi router or reliance on cellular coverage.

Each user carries an intercom device connected to their microphone and listening hardware. A phone provides setup and high-level control, and can stream personal music or audiobooks to the device while the intercom continues working.

The intended experience is: configure the group, start an activity, talk freely and keep listening to personal media. Normal conversation should not require taking out a phone or waiting for another speaker to finish.

## 2. Core use cases

| Use case | Expected experience |
|---|---|
| Set up a device | Pair a phone, identify the device and configure it through the phone interface |
| Form or configure a group | Use the phone to establish network and group settings for participating devices |
| Talk during an activity | At least four users can participate; people may speak over one another without push-to-talk floor ownership |
| Listen to personal media | Play music or an audiobook from a phone through Bluetooth while continuing to send and receive intercom speech |
| Move out of range and return | Reachable users continue communicating; devices automatically reconnect when connectivity returns |
| Operate without the phone connected | Previously configured devices continue intercom operation when a phone is absent or disconnected |
| Evolve the radio platform | Preserve a credible integration path toward a future 802.11ah/HaLow variant |

## 3. Requirement status

- **Required:** part of the current product scope established in discussion.
- **Proposed:** a recommended product behavior or design direction requiring agreement before it becomes a firm requirement.
- **Open:** a target or policy that has not yet been specified.
- **Future:** an anticipated capability to consider in architecture, without requiring it in the initial release.

“Required” does not mean already implemented or demonstrated. Feasibility, implementation and validation status belong in the corresponding design and test documents.

## 4. Functional requirements

| ID | Status | Requirement |
|---|---|---|
| FR-01 | Required | Support a group of at least four intercom devices/users |
| FR-02 | Required | Allow overlapping speech: users can speak and listen freely, without a single active-speaker lock or push-to-talk requirement |
| FR-03 | Required | Establish device-to-device mesh communication without a separate router or cellular service |
| FR-04 | Required | Automatically restore intercom connectivity when previously separated devices become reachable again |
| FR-05 | Required | Provide phone-based network configuration, group configuration and high-level device control through Bluetooth |
| FR-06 | Required | Receive personal phone audio over Bluetooth while microphone transmission and received intercom playback remain operational |
| FR-07 | Required | Provide microphone capture and headphone/speaker playback appropriate for the intercom use case |
| FR-08 | Required | Continue configured intercom operation independently of a live phone connection |
| FR-09 | Proposed | Keep each user's phone media local rather than distributing it to the group |
| FR-10 | Proposed | Make speech intelligible over media by mixing it with controlled gain and automatically lowering media volume during conversation |
| FR-11 | Proposed | Provide directly accessible physical controls for essential actions such as mute, volume and pairing |
| FR-12 | Proposed | Show or announce useful status, including group connection, mute, Bluetooth connection and low battery |

FR-02 describes the user experience, not simultaneous RF transmission. Packet scheduling and radio time sharing are acceptable if they preserve natural overlapping conversation.

FR-06 requires concurrent operation, rather than a mode switch between music and intercom. Media behavior during phone disconnects, incoming calls and reconnects remains to be defined.

## 5. High-level technical requirements

| ID | Status | Requirement or architectural direction |
|---|---|---|
| TR-01 | Required | Use hardware and a software stack that support standard 802.11s mesh for the initial conventional Wi-Fi architecture |
| TR-02 | Required | Support multi-hop forwarding, so communication can traverse an intermediate participating device when a direct link is unavailable |
| TR-03 | Required | Support Bluetooth phone control and phone media reception concurrently with mesh intercom traffic |
| TR-04 | Proposed | Use BLE GATT for configuration/control and Bluetooth Classic A2DP sink for phone media reception; confirm the supported phone compatibility matrix |
| TR-05 | Proposed | Use Linux, with OpenWrt as the preferred baseline; qualify any alternative Linux baseline explicitly before platform selection |
| TR-06 | Required | Provide an integration path to a custom carrier/product assembly; the final architecture must not depend on a loose external USB dongle |
| TR-07 | Required | Evaluate power consumption, mechanical integration and platform support as selection criteria, alongside radio and processor capabilities |
| TR-08 | Proposed | Use a processor and audio stack with sufficient margin for microphone processing, at least one local voice encoder, multiple received voice streams, phone-media decoding and mixing |
| TR-09 | Proposed | Separate phone management from real-time voice operation so configuration traffic and phone disconnects do not unnecessarily interrupt conversation |
| TR-10 | Proposed | Define authenticated device ownership/configuration and protected group communication before releasing a usable product |
| TR-11 | Future | Support a path to 802.11ah/HaLow through a suitable radio module, host interface, power supply and antenna arrangement |

An integrated Bluetooth radio is not sufficient evidence of media support: the controller, firmware and host software must support the required profiles and concurrent operation. Likewise, Wi-Fi or OpenWrt support alone does not establish 802.11s capability on a particular radio.

HaLow is a future radio architecture decision, not a promised firmware upgrade to a conventional Wi-Fi chip. It does not require initial devices to operate both radio systems simultaneously.

## 6. Quality and physical design goals

These goals guide prototypes; numerical acceptance limits remain open.

| Area | Desired outcome | Still to specify |
|---|---|---|
| Conversation quality | Speech remains intelligible and interaction feels natural | End-to-end latency, interruption and audio-quality limits |
| Mobility and recovery | Links recover automatically as users move apart and regroup | Reconnection time and behavior during partial group reachability |
| Range | Useful communication in the intended outdoor activities | Typical separation, total group span, terrain and body/obstruction conditions |
| Battery life | Practical runtime during an activity | Required hours, battery size, charging and standby expectations |
| Mechanical design | Compact, wearable or mountable assembly with secure connections | Weight, dimensions, mounting location and headset/helmet arrangement |
| Environmental robustness | Appropriate operation in the selected outdoor conditions | Temperature range, water/snow exposure, ingress and impact targets |
| Controls | Essential controls are usable during the activity | Glove use, button layout and feedback mechanism |
| Support and lifecycle | Reproducible builds and maintainable hardware/software | Availability horizon, update policy and support commitments |

Do not derive runtime from chip standby figures or power-supply ratings. Compare whole devices under concurrent phone audio, intercom traffic and multi-hop forwarding.

## 7. System responsibilities and boundaries

| Component | High-level responsibility |
|---|---|
| Phone | Setup/control interface and source of personal media |
| Local intercom device | Stored configuration, mesh participation, microphone processing, media reception, local audio playback and recovery |
| Mesh network | Device-to-device links and forwarding between reachable participants |
| Other participants' devices | Their own audio capture/playback and mesh forwarding; no participant's phone is required as the shared voice server |

The initial architecture assumes locally connected microphone and listening hardware. Sending mixed audio onward to a separate Bluetooth headset is a separate feature, rather than an implied consequence of accepting phone media.

## 8. Features not currently required

- Phone calls and Bluetooth HFP call handling.
- Bluetooth headset output, LE Audio or broadcast audio.
- Sharing one user's music with the group.
- Internet access, cloud services, cellular backhaul or remote monitoring.
- A guaranteed group capacity beyond four users.
- Deterministic TDMA, custom MAC firmware or a particular codec as a product requirement.

These are scope boundaries for the current baseline, not statements that the features are impossible or permanently excluded.

## 9. Open decisions

Resolve the first four items before selecting final hardware or freezing a battery/enclosure design.

1. **Range and environment:** required direct-link separation, multi-hop group span and representative terrain.
2. **Battery/runtime:** minimum continuous-use duration and acceptable battery/device mass.
3. **Audio arrangement:** wired headset, helmet speakers/microphone or another local transducer arrangement; resulting echo/noise handling needs.
4. **Software baseline:** whether upstream OpenWrt is mandatory or a maintained vendor Linux distribution is acceptable.
5. **Phone support:** Android, iOS or both; scope of configuration, diagnostics and update functions in the first app.
6. **Media policy:** always mix versus automatic ducking; music behavior during speech, calls and reconnects.
7. **Group operation:** ownership, enrollment/removal, saved groups, maximum eventual size and coordinated settings changes.
8. **HaLow timing:** future product option or first-release requirement; whether it supplements or replaces conventional Wi-Fi.
9. **Commercial targets:** prototype budget, eventual unit cost, production quantity and availability horizon.

## 10. High-level validation scenarios

These scenarios define what the eventual system must demonstrate. Measurable thresholds and test methods will be added after the open targets are resolved.

| Scenario | Evidence expected |
|---|---|
| Four-person conversation | Four configured nodes exchange intelligible overlapping speech without a separate router or single-speaker lock |
| Concurrent phone media | Users receive phone media while sending/receiving mesh speech; exercise BLE control during the same session |
| Forced multi-hop | Communication succeeds over a verified relay path with the direct path unavailable |
| Separation and regrouping | Reachable subsets remain useful and separated nodes reconnect automatically when links return |
| Phone independence | Disconnecting a phone stops its media/control connection but does not stop the configured intercom |
| Configuration changes | Device/group settings are applied predictably; coordinated network changes have a recovery path |
| Activity endurance | Measure whole-device energy, thermal behavior and audio/network stability under a representative outdoor workload |

## 11. Planned lower-level specifications

Expand this baseline into focused specifications for radio/networking, Bluetooth/phone application, audio pipeline, embedded software/recovery, power, mechanical/RF integration and verification. Preserve the requirement IDs so each implementation decision and test can trace back to the product need.

Current feasibility references:

- [ESP32 mesh-intercom feasibility](reference/ESP32-mesh-intercom-feasibility.md)
- [Integrated Linux/OpenWrt platforms](reference/Linux-OpenWrt-integrated-platforms.md)
- [Bluetooth/Wi-Fi platform study and system block diagram](reference/Bluetooth-WiFi-OpenWrt-platform-study.md)

The reference studies compare implementation choices. This document remains the source for product scope and requirement status as it is refined.
