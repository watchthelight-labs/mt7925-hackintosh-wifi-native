# MtkWlanNative: the native Wi-Fi menu for the MediaTek MT7925 on macOS Hackintosh

> **Heads up:** a good portion of this code is AI generated. I do understand how it works,
> and I'm happy to explain any of it. If something looks wrong or you have a question,
> open an issue or ping me on Discord (`watchthelight`).

The native Wi-Fi front end for [MtkWlan](https://github.com/watchthelight-labs/mt7925-hackintosh-wifi), the from-scratch macOS driver
for the MediaTek MT7925 (PCI `14C3:7925`). MtkWlan alone shows macOS an Ethernet card. This second kext is an
`IO80211Controller` plus an `IO80211InfraProtocol` interface built against Apple's IO80211 and Skywalk families, so
macOS sees a real Wi-Fi card: the Wi-Fi menu lists networks, macOS runs the scans and the joins, saves the network, sets
a per-network private address, hands the data path to Skywalk. The MtkWlan core still owns the hardware (firmware, DMA,
scan, association, WPA2) and this kext talks to it through a C ops table (`MtkCoreApi.h`), not through C++ linking.

## Status

Tested on one laptop (Legion 5 15IAX10, MT7925, OpenCore 1.0.8) on macOS 26.7.1 (25G241) with MtkWlan 0.11.0.

| Version | What it does | Result |
|---|---|---|
| 0.4.8 | beacon-counter events once a second while the core holds the link, for the Family's missed-beacons check | being tested: 0.4.6 lost the link about 60 s after every join to the Family's "missed beacons timeout" fault; 0.4.7 fixed it except on channels where its cache-age gate starved |
| 0.4.6 | the join lands on the access point macOS chose (the candidate BSSID at +0x21c) | confirmed; 0.4.4 and 0.4.5 left and re-joined every 15 to 60 s because the core joined its own favourite BSS |
| 0.4.4 | CoreCapture pipes started after the core binding (an earlier order lost the core's 15 s front-end race) | macOS joined the network by itself at boot, lease, real default route, internet for apps |
| 0.4.3 | CoreCapture data pipe and stream started (a completed fault report panicked in `CCDataPipe::enqueueBlob` on an unstarted pipe) | no panic |
| 0.4.1 | the join path: `WCL_LOW_LATENCY_INFO`, `SET_MAC_ADDRESS` (private address into the firmware), `WCL_ASSOCIATE`, the link status / assoc / join / first beacon / connect complete events, BSS info and extended BSS info answers, link state update, link up done | macOS-driven join of an open network, DHCP on the private address, network saved and auto-joined on later boots |
| 0.3.1 | scan requests answered with one scan-result event per BSS in the 25G241 layout | airportd parsed 48 networks with security, channel, phy and RSSI |
| 0.2.1 | the station address follows the interface (macOS sets a private address per network; the firmware must answer to it) | DHCP works on the native data path |
| 0.1.0 | controller and interface come up, Skywalk packet pool and queues, data path bridged to the core | packets flow |

**Not there yet:** WPA2-PSK through the Wi-Fi menu is untested on air (the core's WPA2 code passes its host tests; the
only network here is open). No roaming (the Family asks us to associate to one access point and we do exactly that),
no 802.11ax/be, no sleep, no AirDrop/Continuity (no AWDL interface), no WPA3-SAE, no enterprise, Country Code is reported
as XZ. The captive-portal flow is macOS's own: on a portal network the Captive Network Assistant opens and the interface
is ranked "Never" until you log in, exactly as on a real Mac.

## Requirements

- MtkWlan 0.11.0 or later installed and working with its Ethernet front end first (core ops table v4).
- macOS 26.7.1 build 25G241. The kext is built against that build's class layouts and vtable slots and **refuses to
  load on any other build** (`MtkNativeAbi` in `Info.plist` lists the builds it was generated for). For another build,
  see "Building".

## Installation

1. Put `MtkWlanNative.kext` next to `MtkWlan.kext` in your OpenCore `Kexts` folder and add it to `config.plist` after
   MtkWlan (it links against MtkWlan's core).
2. Add `mtkfe=native mtknativeif=1` to `boot-args`. Without them MtkWlan keeps its Ethernet front end and this kext
   stays a controller without an interface (safe to leave installed).
3. Reboot. The Wi-Fi menu appears; join your network from it. If the native front end does not register within 15 s
   the core falls back to the Ethernet front end by itself, so a broken build still leaves you with network.

If the network interface for the card was created while it was an Ethernet port, its service in System Settings may
still be of type Ethernet ("Network Service Inactive" in the Wi-Fi system profile). Remove that service and let macOS
detect the port again (`networksetup -detectnewhardware`).

## Diagnostics

`ioreg -r -c MtkWlanNative -l -w0` holds the controller's state: `MtkNativeTrace` (what happened, in order),
`MtkNativeCalls` (every request the Family made, by slot or ioctl code, and every event we posted), `MtkNativeScan`,
`MtkNativeJoin`, `MtkNativeMacFollow`, the data path counters `MtkNativeTx` / `MtkNativeRx`, and `MtkNativeAbi`. The
Family's own view: `log show --last 5m --info --predicate 'process == "airportd" OR process == "kernel"' | grep -E
'IOC DEBUG|AUTO-JOIN'`. `docs/NOTES-NATIVE.md` is the lab notebook: what each version found in the 25G241 binaries.

## Other MediaTek cards: collecting logs and reporting back

The core driver has run on one card (MT7925 `14c3:7925`, subsystem `17aa:e0ff`). This front end only needs the core,
so it is the core that decides whether your card works; the front end then depends on the macOS build. If you try it:

1. Get MtkWlan working with its Ethernet front end first and report there if it does not.
2. Install this kext, add the boot-args, reboot, wait a minute.
3. Run `bash tools/mtk-report.sh` (no sudo). It writes `~/Desktop/mtk-report-<date>.tar.gz` with the PCI ids, the
   loaded kexts, every `ioreg` property of both kexts, interfaces and routes, the Wi-Fi system profile, the kernel log
   lines of the drivers and the IO80211 family, the Wi-Fi daemons' log, and the last kernel panic reports. MAC addresses
   and SSIDs are replaced by placeholders (`--no-redact` keeps them). Nothing in it is a key or passphrase.
4. Open an issue at github.com/watchthelight-labs/mt7925-hackintosh-wifi-native/issues with the archive and three lines: the card,
   the macOS build (`sw_vers`), and what you saw (did the Wi-Fi menu list networks, did a join get a lease, did the
   machine panic). A panic file in the report tells me where.

## Building

`./build.sh` compiles against the Command Line Tools SDK with the ABI headers in `abi/25G241/`. Those headers are
generated from a dump of the macOS build's kernel collection (class sizes, vtables, the 200 request slots the Family
routes ioctls to); `tools/kcabi/dump.py` makes that dump on the target machine and `tools/kcabi/genhdr.py gen` turns it
into headers for another build. The build script then checks that every override sits on Apple's slot and that every
undefined symbol is exported by the running kernel collection (`tools/kcabi/linkcheck.py`).

## How it works

`docs/HOW-IT-WORKS.md`, and `docs/NOTES-NATIVE.md` for the details per version.

## Credits

- [ga403wm-hackintosh](https://github.com/thanhnndev/ga403wm-hackintosh) (BSD-3-Clause-Clear, Copyright (c) 2026
  thanhnndev and contributors): the shape of a native front end for a MediaTek core on macOS 15, the interface start
  sequence, the join event sequence and several payload layouts, re-read against the 25G241 binaries here. See
  `NOTICE.md`.
- [AirportMtkWlan](https://github.com/mnnobre/AirportMtkWlan) and laobamac's itlwm / SimpleMtkWlan (GPL-2.0): read as
  references for what macOS expects of a Wi-Fi driver; no code from them is in this repository.
- Apple's IO80211Family, IOSkywalkFamily and corecapture, read in disassembly to learn the ABI. No Apple code is in this
  repository; the generated headers hold class and method names and slot numbers only.

## Licence

ISC, see `LICENSE`. `LICENSES/` and `NOTICE.md` cover the adapted parts.
