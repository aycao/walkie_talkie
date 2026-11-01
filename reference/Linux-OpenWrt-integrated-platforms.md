# Integrated Linux/OpenWrt platforms for the mesh intercom

Research date: October 31, 2026. Documentation-based assessment; no boards or radio combinations have been tested. Requirements carried forward: at least four nodes, overlapping hands-free conversation, infrastructure-free mesh, automatic reconnection, and a credible route from prototype to integrated hardware.

## Bluetooth requirement update

**Updated requirement: Bluetooth must receive phone music/audiobooks concurrently with mesh intercom operation, as well as provide phone configuration.** This requires Classic A2DP sink support in the initial design; the previously proposed nRF52840 BLE-only module is insufficient for media. Favor wireless CM4 plus a qualified mesh radio for the first concurrent-audio prototype, with TI WL1837-based Linux hardware as an integrated alternative. Carambola/Lima needs an added Classic audio solution and audio-capacity validation. Read the [Bluetooth + Wi-Fi platform study](Bluetooth-WiFi-OpenWrt-platform-study.md) for the revised shortlist, TI WL1837 and CompuLab options, BLE integration, Raspberry Pi considerations and HaLow plan. Carambola 3, Lima and VoCore2 require an added Bluetooth controller/module. Wireless CM4 includes Bluetooth, but both its OpenWrt Bluetooth setup and a separate qualified mesh radio need validation. Mesh support varies within mt76; do not assume MT7921/MT7922 combo radios support 802.11s.

## Recommendation

**Integrated Linux + Wi-Fi platforms exist. A Raspberry Pi with a USB dongle is not our only path.** The closest product category is a solderable wireless system-on-module (SoM): Linux processor, RAM, flash, radio and RF circuitry on a small module. It still needs a carrier PCB for power, audio, controls and the intended antenna arrangement.

Shortlist:

1. **8devices Carambola 3 / Lima + BLE MCU module:** compact OpenWrt modules with ath9k Wi-Fi; add phone control through a carrier-mounted BLE module over UART. Limited CPU performance is the principal audio risk.
2. **VoCore2:** very small MediaTek module; use upstream mt76 for mesh investigation. Audio driver integration and CPU headroom need early proof.
3. **Wireless Compute Module 4 + PCIe mesh radio:** built-in Bluetooth/BLE and more flexibility for Linux audio, at the cost of a larger carrier and separate mesh radio. Qualify Bluetooth under the chosen OpenWrt build. Use ath9k or an explicitly mesh-capable AP-oriented mt76 radio; onboard Pi Wi-Fi is not the mesh baseline.
4. **8devices Cherry:** a compact modern ARM/Wi-Fi alternative, conditional on confirmation of the exact supported driver, kernel and secure mesh configuration. Vendor OpenWrt branding is insufficient evidence of upstream or mesh support.

Do not select a product solely because its CPU and radio are on the same chip. **Driver architecture, lifecycle, audio support and power are more important than silicon count.**

## What degree of Wi-Fi control is realistic?

Linux offers a useful split: our intercom uses ordinary IP packets, while the kernel and Wi-Fi stack manage mesh links and forwarding. On suitable hardware, we can inspect neighbors and paths, configure channel and rate policies, monitor traffic, inspect driver statistics and modify open source code.

SoftMAC generally puts more protocol work in the host, using Linux mac80211. FullMAC generally places more in radio firmware. This is a spectrum: a mac80211 driver can still rely on substantial closed firmware. It does not imply unrestricted PHY access, raw IQ samples, arbitrary legal transmit power or deterministic TDMA. [mac80211 documentation](https://wireless.docs.kernel.org/en/latest/en/developers/documentation/mac80211.html)

| Driver family | Why it matters | Qualification |
|---|---|---|
| ath9k | Mature open driver, mesh-point and monitor support; strong choice for MAC investigation | Older Wi-Fi 4 silicon and lifecycle concerns |
| mt76 | Open mac80211 driver family covering integrated and discrete MediaTek radios | Exact chip, firmware, bus and release still matter |
| ath10k | Linux mesh support exists | More firmware dependence; select and test firmware carefully |
| ath11k / newer Qualcomm | Modern radio and processor platforms | Do not infer secure mesh behavior from AP/STA operation |
| brcmfmac | Useful conventional Wi-Fi on many compact computers | Firmware-centric; not our default for open MAC/802.11s experiments |

Linux Wireless explicitly lists mesh-point mode for ath9k. For ath10k, its documentation specifies firmware feature requirements for mesh and security. OpenWrt warns that some drivers advertise mesh yet behave poorly, and calls out ath10k-ct issues on particular hardware. [ath9k](https://wireless.docs.kernel.org/en/latest/en/users/drivers/ath9k.html), [ath10k mesh](https://wireless.docs.kernel.org/en/latest/en/users/drivers/ath10k/mesh.html), [OpenWrt mesh guide](https://openwrt.org/docs/guide-user/network/wifi/mesh/802-11s)

## Integrated platforms

Dimensions below refer to modules, not their development kits or complete intercoms. No candidate is yet qualified for four-person audio.

| Module | Processor and memory | Wi-Fi/control path | Module size | Assessment |
|---|---|---|---|---|
| Carambola 3 | QCA4531, 650 MHz MIPS; current detailed specifications list 128 MB RAM / 32 MB NOR | 2.4 GHz 1x1 Wi-Fi 4, ath9k | 28 x 38 mm | First compact radio-control candidate |
| Lima | QCA4531, 650 MHz MIPS; current vendor wiki lists 128 MB RAM / 32 MB flash | 2.4 GHz 2x2 Wi-Fi 4, ath9k family; also exposes PCIe | 25 x 35 mm | Similar CPU; more radio/expansion options |
| VoCore2 | MT7628, 580 MHz MIPS; 128 MB RAM; flash depends on revision | 2.4 GHz Wi-Fi 4; upstream mt76 path | 25.6 x 25.6 mm | Particularly compact; audio is an early integration gate |
| Cherry | IPQ5000, dual Cortex-A53 at 1 GHz; 256 MB RAM / 32 MB NOR | 2.4 GHz 2x2 Wi-Fi 6; verify exact software stack | 19.4 x 30.5 mm | Attractive modern module, conditional recommendation |
| Omega2 family | MT7688, 580 MHz MIPS; memory varies by model | 2.4 GHz Wi-Fi 4; distinguish vendor firmware from upstream mt76 | Depends on board/SoM variant | Secondary alternative; prefer sufficient RAM/flash |
| MT7981B + MT7976C platform | ARM processor plus companion Wi-Fi radio and external memory | Modern mt76/OpenWrt ecosystem | Board-specific | Good reference architecture, not a tiny single-chip solution |

**Carambola 3 and Lima.** Their product pages advertise solderable modules and OpenWrt support. Carambola 3 is explicitly listed with ath9k in OpenWrt's hardware data. Vendor summaries contain conflicting RAM/flash figures, so specify the exact ordering revision rather than trusting the introductory paragraph. Carambola 3 lists 1.6 W and Lima 3.1 W, without enough common test conditions to predict our runtime. These are not measured intercom power figures. [Carambola 3](https://www.8devices.com/products/carambola-3), [OpenWrt hardware data](https://openwrt.org/toh/hwdata/8devices/8devices%20carambola3), [Lima](https://www.8devices.com/products/lima), [Lima wiki](https://wiki.8devices.com/lima)

For these small MIPS platforms, benchmark a suitable fixed-point Opus build: one encoder, three decoders, mixing and any echo/noise processing alongside forwarding. Clock rate and available RAM do not establish real-time audio capacity. Their advertised serial interfaces do not establish a ready-to-use full-duplex I2S audio path. USB audio could establish a baseline; a soldered USB audio controller remains an integration option if internal USB is acceptable.

**VoCore2.** Its vendor page lists 128 MB RAM and a 2024 change from 16 MB to 32 MB NOR; older tables still list 16 MB. Confirm actual flash and matching firmware layout. The page publishes Wi-Fi standby/full-speed current, but these are not battery-life predictions. Vendor audio guidance describes an MT7628 I2S driver and codec integration requiring custom firmware in the discussed configuration. Avoid assuming the latest upstream image includes a complete microphone/playback path. [VoCore2](https://vocore.io/v2.html), [OpenWrt driver inventory](https://openwrt.org/toh/views/toh_admin_wlandriver), [vendor audio discussion](https://forum.vocore.io/viewtopic.php?f=16&t=4356)

**Cherry.** The vendor lists I2S, PCIe and up to 7 W, but its wiki describes the 7 W figure as development-kit consumption. Request module measurements at our workload. Its small ARM module is compelling, yet this research did not establish an upstream supported board image plus secured 802.11s on its integrated radio. Treat that as an unresolved gate, not as a negative capability claim. [Cherry specifications](https://www.8devices.com/products/cherry), [Cherry wiki](https://wiki.8devices.com/cherry)

**Omega2.** Onion documents an OpenWrt-derived vendor system with an older base, whereas OpenWrt independently lists upstream Omega2 support. Those are different software baselines. Use the exact Omega2/2+/2S variant's memory, pinout, firmware and lifecycle; do not transfer board support between them without checking. [Onion documentation](https://docs.onion.io/omega2-docs/omega2.html), [upstream hardware entry](https://openwrt.org/toh/hwdata/onion/onion_omega2)

**Modern MediaTek reference.** OpenWrt One combines MT7981B with MT7976C, rather than putting the whole RF implementation into one small processor chip. Its 148 x 100.5 mm board is a networking reference, not our final wearable. Its value is a documented OpenWrt platform for network experiments. Shrinking it entails module sourcing or a real processor/RAM/RF board design, not simply removing Ethernet connectors. [OpenWrt One hardware](https://docs.banana-pi.org/en/OpenWRT-One/BananaPi_OpenWRT-One), [mt76 repository](https://github.com/openwrt/mt76)

## Carambola 3 and Lima: controls, audio and Bluetooth

Follow-up findings confirmed October 31, 2026 from manufacturer specifications, module pin tables and Qualcomm's public QCA4531 documentation. These are documented interface capabilities, not a tested intercom implementation. Confirm the exact module revision before designing the carrier: Lima's datasheet includes both QCA4531 and QCA9531 ordering variants.

### What runs on the module versus the carrier

Each module already contains a Linux-capable processor, RAM, flash and Wi-Fi. A second host processor is not inherently required for mesh networking, button handling or the intercom application. The carrier PCB supplies power and connects the module to audio circuitry, controls, antennas and debugging connections. A separate audio processor remains an option if measured performance requires it; four-person audio processing capacity is still unproven.

### Physical controls and digital interfaces

| Function | Intended interface | Implementation notes |
|---|---|---|
| Mute, pairing and volume buttons | GPIO inputs | Linux receives button events; software handles debouncing and application behavior |
| Status LEDs | GPIO outputs | Use appropriate resistors and, where needed, driver circuitry |
| Microphone capture and headphone/speaker playback | External USB audio controller as the initial architecture | Controller provides analog conversion; qualify simultaneous capture/playback and latency |
| General peripherals | SPI and UART where exposed | These are digital interfaces, not direct microphone or speaker connections |
| Debug console | UART | Useful for firmware setup and diagnosis |
| Additional expansion on Lima | PCIe | Exposed by Lima; not a necessary part of the proposed audio path |

GPIO pins read or drive digital logic levels. Some pins have alternate functions, but their existence does not imply an ADC, DAC or I2S controller. Select pins using the module's documented pin functions and voltage specifications. Several pins also set boot configuration; external button circuits must not force an incorrect boot state. Lima's SPI interface is shared with internal flash, so external SPI devices require careful chip-select and bus integration. [Carambola 3 datasheet, pp. 4-6](https://www.8devices.com/media/products/carambola3/downloads/Carambola_3_datasheet.pdf), [Lima datasheet, pp. 4-8](https://www.8devices.com/media/products/lima/downloads/lima_datasheet.pdf)

### Audio: no documented direct analog or I2S path

| Audio capability | Carambola 3 | Lima |
|---|---|---|
| Exposed microphone ADC / analog audio input | Not documented | Not documented |
| Exposed audio DAC / analog audio output | Not documented | Not documented |
| Native I2S digital audio interface on module pins | Not documented | Not documented |
| USB host connection for an external audio controller | Yes | Yes |

The inspected module pin tables do not identify audio ADC/DAC or I2S functions. Qualcomm's public QCA4531 interface list likewise does not advertise these audio facilities. This establishes the documented design options; it is not a claim to have excluded every undocumented chip capability. A microphone or headphones cannot be connected directly to ordinary digital GPIOs. [Carambola 3 pinout](https://www.8devices.com/media/products/carambola3/downloads/Carambola_3_datasheet.pdf), [Lima pinout](https://www.8devices.com/media/products/lima/downloads/lima_datasheet.pdf), [Qualcomm QCA4531 product brief](https://www.qualcomm.com/content/dam/qcomm-martech/dm-assets/documents/qca4531-product-brief.pdf)

Use the following architecture as the practical initial baseline:

```text
Buttons / LEDs -------- GPIO -------- Linux + Wi-Fi module
                                             |
                                            USB
                                             |
                                  External audio controller
                                  /                       \
                    Microphone + ADC               DAC + output stage
                                                          |
                                                  Headphones / speaker
```

The audio controller contains or connects to the ADC for microphone capture and the DAC for playback, with the necessary microphone and output circuitry. This hardware audio codec is distinct from Opus speech compression running in software. Linux accesses the controller through an appropriate audio driver, normally exposing it through ALSA.

For a prototype, a supported USB audio device can establish the baseline. In the integrated product, place a suitable USB audio controller directly on the carrier and connect its USB data lines internally. No removable dongle or external USB connector is inherently required. Verify simultaneous microphone/playback operation, sample rates, buffering, clock behavior, drivers and total acoustic latency before selecting the controller.

USB is not theoretically the only possible transport: an external ADC/DAC or audio processor could use another documented digital interface with suitable buffering and driver development. Such a design is additional engineering, not a ready-made audio capability established for these modules. Do not assume I2S support from Carambola 2 examples: that predecessor advertised I2S, while Carambola 3's inspected pin tables do not. Physical footprint compatibility does not establish equivalent peripheral functions. [Carambola 2 interfaces](https://www.8devices.com/products/carambola-2)

### Bluetooth

**Neither Carambola 3 nor Lima includes built-in Bluetooth Classic or Bluetooth Low Energy (BLE).** Both manufacturer product tables explicitly list Bluetooth Standards as N/A. [Carambola 3 specifications](https://www.8devices.com/products/carambola-3), [Lima specifications](https://www.8devices.com/products/lima)

Bluetooth phone pairing or wireless headset connectivity would require a separate Bluetooth controller/module and appropriate Linux software. BLE capability alone must not be treated as proof of headset audio support; qualify the required audio profiles or LE Audio features on the actual controller and software stack. If Bluetooth is added over USB alongside USB audio, the carrier may need a USB hub depending on the selected components. Account for the extra radio, antenna arrangement, power and coexistence with 2.4 GHz Wi-Fi.

## Lifecycle findings

The live manufacturer pages now mark **Carambola 2, Jalapeno and Habanero as EOL and no longer sold**, even though cached search results and some store references still look active. Do not select these as the final design baseline. They can remain useful examples of the architecture or existing bench hardware. [Carambola 2](https://www.8devices.com/products/carambola-2), [Jalapeno](https://www.8devices.com/products/jalapeno), [Habanero](https://www.8devices.com/products/habanero)

Carambola 3 has a separate published 2026 product-change notice. This is evidence of ongoing revision activity, not a lifetime guarantee. Before purchase, obtain lifecycle and availability information for the exact SKU, plus its hardware revision and current memory configuration. [Carambola 3 change notice](https://www.8devices.com/carambola-3-to-receive-qol-improvement)

## Separate processor and radio: integration without a dongle

There are two separate questions: the electrical host bus, and the module's physical packaging. USB does not inherently require a removable dongle. A soldered radio module can use short USB traces entirely within the product. However, if the goal is to avoid USB electrically as well as mechanically, PCIe is the leading route to the open-driver radios below.

| Interface | Integration method | Project implication |
|---|---|---|
| PCIe | Internal Mini PCIe/M.2 card, solderable radio module, or custom radio circuitry | Strong route to ath9k/mt76; requires host PCIe, suitable power and high-speed PCB layout |
| SDIO | Compact soldered Wi-Fi module | Good physical integration; driver/firmware mesh support must be verified independently |
| USB | Soldered module or controller on carrier | Removes the dongle; retains USB transport and its supported driver |
| SPI | Suitable low-throughput radio/module with explicit Linux support | Useful for certain HaLow designs; not a generic replacement bus for PCIe Wi-Fi chips |

Mini PCIe and M.2 describe form factors and connectors, not a guarantee of the required electrical bus. An M.2 slot wired for an SSD cannot be assumed to accept a Wi-Fi module without the right pinout and adapter. PCIe Wi-Fi and Bluetooth USB may also be separate functions on one card.

### Radio shortlist for a Linux host

| Chip/driver | Concrete module examples | Role |
|---|---|---|
| AR9280 / ath9k | Compex WLE200NX, Mini PCIe | Open-driver research reference; confirm current availability and lifecycle |
| ath9k PCIe radio | Silex SX-PCEAN2 | Manufacturer documents ath9k evaluation; verify exact current part and lifecycle |
| AR9380 / ath9k | jjPlus JWX6083, Mini PCIe | Alternative reference; old silicon and three RF chains may be unnecessary |
| QCA9882 / ath10k | jjPlus JWX6055, Mini PCIe | Wi-Fi 5 option, more firmware dependence than ath9k |
| MT7915 / mt76 | AsiaRF AW7915-NPD Mini PCIe or AW7915-AED M.2 | Modern AP-oriented test option; likely excessive radio/power for a small wearable |
| MT7612/MT7662 family / mt76 | Source an exact PCIe module rather than substituting a USB version | Potential older alternative, not a procurement-ready module recommendation |

Compex lists AR9280 for WLE200NX; its hardware guide gives a 2.4 W maximum. Silex explicitly documents its PCIe radio with ath9k. These demonstrate credible internal-radio architectures, not guaranteed current volume availability. [Compex catalogue](https://compex.com.sg/wp-content/uploads/2019/03/wle-catalogue-240-1.pdf), [WLE200NX guide](https://compex.com.sg/wp-content/uploads/2018/10/wle200nx-hw-guide-v1.pdf), [Silex evaluation](https://www.silextechnology.com/sx-pcean2-evaluation-pcie)

jjPlus lists the exact AR9380 and QCA9882 modules above. Linux Wireless documents the corresponding driver families; qualification still requires the actual card and firmware. [jjPlus module catalogue](https://www.jjplus.com/wifi-modules-2/), [ath10k hardware](https://wireless.docs.kernel.org/en/latest/en/users/drivers/ath10k.html)

AsiaRF documents PCIe MT7915 modules in Mini PCIe and M.2 packaging. Its AW7915-NPD page specifies average consumption of 4–8 W and 9 W maximum, plus substantial 3.3 V supply requirements. This is a useful modern Linux networking reference but a poor default wearable radio without workload power measurements. Do not select it merely because it supports Wi-Fi 6. [Module variants](https://asiarf.com/asiarf-wi-fi5-6-6e-7-modules-installation-guide/), [AW7915-NPD power](https://asiarf.com/product/wifi-6-11ax-dbdc-1800mbps-mini-pcie-module/), [MediaTek Linux driver mapping](https://wireless.docs.kernel.org/en/latest/en/users/drivers/mediatek.html)

Changing from a prototype USB radio to a final PCIe radio may change driver and firmware behavior even when both are called MediaTek Wi-Fi. Validate the intended final radio early. Retaining the same radio module on a custom carrier is a much lower-risk transition than jumping from an adapter to a bare RF chip.

## Raspberry Pi choice

For an integrated design needing a PCIe radio, start with **Compute Module 4**, not automatically the smallest Pi board. CM4 is 55 x 40 mm and exposes PCIe Gen 2 x1; the product brief states production through at least January 2034. Its carrier can include the radio, audio codec and battery circuitry. CM5 provides another PCIe-capable option, but extra processing power must justify its power and thermal implications. [CM4 product brief](https://pip-assets.raspberrypi.com/categories/634-raspberry-pi-compute-module-4/documents/RP-008169-DS/cm4-product-brief), [CM5 brief](https://datasheets.raspberrypi.com/cm5/cm5-product-brief.pdf)

Pi Zero 2 W is compact and useful for software experiments, but it does not expose a native PCIe host interface. It is not the convenient starting point for an ath9k PCIe integration. An SDIO or internal USB design is a different path requiring separate validation. [Pi Zero documentation](https://www.raspberrypi.com/documentation/computers/zero.html)

Do not rely on the onboard Pi Wi-Fi as our open mesh radio by default. The Linux driver inventory distinguishes firmware-driven brcmfmac from mac80211 mesh-capable families. An onboard chip working in AP/STA mode does not demonstrate the mode and control we need. [Linux driver capabilities](https://wireless.docs.kernel.org/en/latest/en/users/drivers.html)

## Stack and work remaining

A suitable Linux stack is: audio capture/playback through ALSA; Opus and speech processing; our session/mixing/timing application; UDP media; Linux networking; secured 802.11s peering; mac80211 plus a qualified driver and radio. A small mesh can initially use stable per-device IP addresses, avoiding dependence on one DHCP node. Peer identity should remain independent of IP address.

Linux's mesh-point interfaces can discover peers and forward traffic, with tools for inspecting peer links and paths. This removes much of the custom routing burden of ESP-NOW. It does not supply four-person audio sessions, coordinated channel changes, application recovery, clock correction or acceptable voice interruption during movement. [Linux mesh interfaces](https://wireless.docs.kernel.org/en/latest/en/users/documentation/iw/vif.html), [iw tools](https://wireless.docs.kernel.org/en/latest/en/users/documentation/iw.html)

OpenWrt and ordinary Linux can share the networking and audio application architecture. The exact board still needs supported device configuration, drivers and packages. Avoid assuming that a vendor fork, firmware with an OpenWrt interface, and an upstream-supported OpenWrt release are interchangeable.

For future HaLow, preserve available SPI/SDIO, interrupts, reset and power-control signals on the host/carrier. Quectel's FGH100M family uses those interfaces and is physically integrable. Its exact regional suffix and driver/kernel support must be qualified; AP/STA documentation is not proof of mesh support. A vendor reply in the inspected FGH100M-H discussion said mesh documentation was not released at that time. That historical reply does not establish present inability; it identifies a validation question. [Quectel family brochure](https://quectel.com/content/uploads/2025/03/Quectel_Product_Brochure_V8.3.pdf), [vendor mesh discussion](https://forums.quectel.com/t/mesh-networking-with-fgh100m-h-m-2/46173)

## Next decision

Prioritize Carambola 3/Lima plus a BLE MCU for compact open-radio operation, and wireless CM4 plus an internal PCIe mesh radio for audio processing headroom. Investigate CompuLab CL-SOM-iMX7 WAB/A as an integrated Linux Wi-Fi/Bluetooth/audio alternative, with an explicit OpenWrt support qualification. See the companion Bluetooth study for the updated comparison. Other candidates require both mesh evidence and a Bluetooth integration plan.

Before committing hardware, prove secured four-node mesh, forced multi-hop paths, overlapping real speech, full-duplex capture/playback, one encode plus three decodes, echo handling, partition/merge and recovery. Measure acoustic delay, CPU margin, whole-device power and body-worn range. Obtain exact SKU pinouts, lifecycle and RF integration documents.

There is no procurement-ready winner yet. The research establishes that a compact integrated Linux/OpenWrt architecture is real and identifies credible routes; selecting it requires the combined network/audio/power tests above.
