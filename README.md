# MtkWlanNative: the native Wi-Fi menu for the MediaTek MT7925 on macOS Hackintosh

> **Heads up:** a good portion of this code is AI generated. I do understand how it works,
> and I'm happy to explain any of it. If something looks wrong or you have a question,
> open an issue or ping me on Discord (`watchthelight`).

The native Wi-Fi menu for the MediaTek MT7925 (Filogic 360, Wi-Fi 7, PCI `14C3:7925`, AMD RZ717 `14C3:0717`) on a
Hackintosh. [mt7925-hackintosh-wifi](https://github.com/watchthelight-labs/mt7925-hackintosh-wifi) is the driver; on
its own it shows macOS an Ethernet port and you join networks with a tool, like SimpleMtkWlan does for the MT7921 and
MT7922. This second kext is an `IO80211Controller` plus an `IO80211InfraProtocol` interface built against Apple's
IO80211 and Skywalk families, so **macOS sees a real Wi-Fi card: the Wi-Fi icon in the menu bar, the Wi-Fi pane in
System Settings, scans and joins run by macOS, saved networks, auto-join at boot, a private address per network, the
data path on Skywalk.** No client app, no command line. The driver still owns the hardware (firmware, DMA, scan,
association, WPA2) and this kext talks to it through a C ops table (`MtkCoreApi.h`), not through C++ linking.

It is built against one macOS build, Tahoe 26.7.1 (25G241), and refuses to load on any other, so a wrong build can
never take your network away: the driver falls back to its Ethernet front end by itself.

## The MT7925 set

| Repo | What it does |
|---|---|
| [mt7925-hackintosh-wifi](https://github.com/watchthelight-labs/mt7925-hackintosh-wifi) | the driver (required); its README has the cards table and the comparison with SimpleMtkWlan and AirportMtkWlan |
| mt7925-hackintosh-wifi-native (this one) | the native Wi-Fi menu on top of it |
| [mt7925-hackintosh-bluetooth](https://github.com/watchthelight-labs/mt7925-hackintosh-bluetooth) | the Bluetooth firmware loader for the same card |

The bundle zip on the driver's [releases page](https://github.com/watchthelight-labs/mt7925-hackintosh-wifi/releases)
carries both Wi-Fi kexts with an `INSTALL.md`. A complete EFI that uses the set:
[Legion-5-15IAX10-OpenCore](https://github.com/watchthelight-labs/Legion-5-15IAX10-OpenCore).

## Status

Tested on one laptop (Legion 5 15IAX10, MT7925, OpenCore 1.0.8) on macOS 26.7.1 (25G241) with MtkWlan 0.11.0.

| Version | What it does | Result |
|---|---|---|
| 0.4.9 | link quality update events (0x27) once a second, which refresh the timestamps the Family's assoc timer checks | **stable**: macOS auto-joins at boot, holds the link (ten quiet minutes, zero leaves), DHCP, real default route, internet for apps. 0.4.7 and 0.4.8 posted beacon counters only and the Family still raised "missed beacons timeout" 60 s after each join |
| 0.4.8 | beacon-counter events while the core holds the link | still lost the link every 60 s: the beacon counter feeds a different check |
| 0.4.6 | the join lands on the access point macOS chose (the candidate BSSID at +0x21c) | confirmed; 0.4.4 and 0.4.5 re-joined every 15 to 60 s because the core joined its own favourite BSS |
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

- MtkWlan 0.11.1 or later (0.11.0 works but can lose the start-up race, see the changelog) installed and working with its Ethernet front end first (core ops table v4).
- macOS 26.7.1 build 25G241. The kext is built against that build's class layouts and vtable slots and **refuses to
  load on any other build** (`MtkNativeAbi` in `Info.plist` lists the builds it was generated for). For another build,
  see "Building".

## Installation

1. Put `MtkWlanNative.kext` next to `MtkWlan.kext` in your OpenCore `Kexts` folder and add it to `config.plist` after
   MtkWlan (it links against MtkWlan's core).
2. Add `mtkfe=native mtknativeif=1` to `boot-args`. Without them MtkWlan keeps its Ethernet front end and this kext
   stays a controller without an interface (safe to leave installed).
3. Reboot. The Wi-Fi menu appears; join your network from it. If the native front end declines, or claims the core
   and still has not registered after 90 s (15 s with MtkWlan before 0.11.1), the core falls back to the Ethernet front
   end by itself, so a broken build still leaves you with network.

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

The driver has run on one card (MT7925 `14c3:7925`, subsystem `17aa:e0ff`) and matches the AMD RZ717 (`14c3:0717`).
This front end only needs the driver, so it is the driver that decides whether your card works; the front end then
depends on the macOS build. The MT7921 and MT7922 are not supported by the driver yet (the plan and the call for
testers are in [its issue #2](https://github.com/watchthelight-labs/mt7925-hackintosh-wifi/issues/2)). If you try it:

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
