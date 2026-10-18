# Changelog

## 0.4.11 (2026-10-18)

- Tx queues restart on the next 2 ms tick after the core frees a slot. They used to wait for the 10 ms control poll, which capped upload near 190 Mbit/s. Frames refused while the link was down are asked for again when it comes up.
- Tx completion moves the pending list once per tick instead of once per packet.
- Measured on the author's laptop with MtkWlan 0.11.3: about 410 Mbit/s of local UDP on VHT 80 MHz 2x2, gateway ping 3-4 ms (5 ms before).

## 0.4.10 (2026-10-18)

- Claims the core at the end of `probe()` (`MTK_CORE_FE_CLAIM`, MtkWlan 0.11.1), before Apple's interface start can outlast the core's 15 s fallback. Before this, a slow `IO80211SkywalkInterface` start (17.2 s on one boot) let the Ethernet front end win by 230 ms and the Wi-Fi menu stayed empty until the next reboot. Every place `start()` gives up now declines, so the fallback still comes at once when the front end really cannot run.
- Debug boot-arg `mtknativedelay=<ms>` (at most 60000) holds the interface start back, to test the late path.

## 0.4.9 (2026-10-18)

- Link quality update events (0x27, 0x1dc bytes) once a second while the link is up. The Family's assoc timer declares "missed beacons" when its beacon and traffic timestamps are both older than 60 s, and only this event refreshes them; the beacon counter of 0.4.7 feeds a different counter. Proven on the author's laptop: ten quiet minutes without a leave.

## 0.4.8 (2026-10-18)

- Beacon-counter events are posted while the core holds the link, not only while its scan cache entry for the access point is fresh (the firmware keeps the associated AP's beacons to itself on some channels, and the Family raised BeaconLoss again).

## 0.4.7 (2026-10-18)

- Beacon-counter events (0xaf, u32 count at +0x54) once a second while the link is up. Without them the Family raises the fault "missed beacons timeout" about 60 s after every join and leaves the network. Under test.

## 0.4.6 (2026-10-18)

- The candidate BSSID is read at +0x21c (0.4.5 read the wrong field). Confirmed on the author's laptop: the join lands on the access point macOS chose.

## 0.4.5 (2026-10-18)

- `setWCL_ASSOCIATE` passes the first candidate's BSSID to the core (`apple80211AssocCandidates`: count at +0x214, entries of 18 bytes from +0x21c, BSSID at +6). 0.4.4 joined the core's own favourite access point and the Family re-joined every 15 to 60 s. Under test.

## 0.4.4 (2026-10-18)

- CoreCapture pipes and streams are started after the interface has bound the core. Starting them earlier took the native front end past the core's 15 s fallback and the Ethernet front end won the boot.

## 0.4.3 (2026-10-18)

- `IOService::start` on the CoreCapture data pipe, data stream and log pipe. The factories only alloc and init; a fault report completing on an unstarted data pipe panics in `CCDataPipe::enqueueBlob`.

## 0.4.2 (2026-10-18)

- One version string (`MTK_NATIVE_VERSION`), shown by the Family as "Firmware Version".

## 0.4.1 (2026-10-18)

- ioctls 0x170 `SET_MAC_ADDRESS` and 0x22d `DBRG_ENTROPY` admitted on the join path; a refused address answers NotPermitted.

## 0.4.0 (2026-10-18)

- The Family drives the association: `WCL_ASSOCIATE`, `WCL_LEAVE_NETWORK`, `WCL_JOIN_ABORT`, the events 0xd8 / 0x4e / 0xd3 / 0xd4 / 0xd5, `WCL_BSS_INFO`, `WCL_EXTENDED_BSS_INFO` (0x214 bytes on 25G241), `WCL_LINK_STATE_UPDATE`, `WCL_LINK_UP_DONE` and the link-path requests. The core's open-network fallback is off in native mode.

## 0.3.1 (2026-10-18)

- The ioctl allow-list admits `WCL_SCAN_REQ` and `WCL_SCAN_ABORT`; every WCL request is gated by `isCommandProhibited` first.

## 0.3.0 (2026-10-18)

- Scan requests run the core's scan; one 0xc9 event per BSS in the 25G241 layout (IEs at +0x44), then 0xed.

## 0.2.1 (2026-10-18)

- The interface follows the BSD link-layer address into the core (`set_mac`, core ops v4): macOS gives the interface a private address per network and the firmware must answer to it.

## 0.1.0 (2026-10-17)

- Controller and interface, Skywalk packet pool and queues, data path bridged to the core, ABI guard per macOS build.
