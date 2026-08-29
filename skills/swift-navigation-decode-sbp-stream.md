---
name: decode-sbp-stream
description: >-
  Turn a Swift Binary Protocol byte stream from a Piksi, Duro or Starling device into validated JSON
  and read the position, integrity and correction-age fields that matter.
api: Swift Binary Protocol (SBP)
generated: '2026-08-29'
method: generated
grounded_in:
  - spec/*.yaml (29 packages, 322 definitions, 242 messages — parsed 2026-08-29)
  - json-schema/*.json (225 JSON Schema draft-06 documents)
  - https://github.com/swift-nav/libsbp#installing-sbp2json-json2sbp-json2json-and-related-tools
operations:
  - MSG_GPS_TIME
  - MSG_UTC_TIME
  - MSG_POS_LLH
  - MSG_POS_LLH_COV
  - MSG_VEL_NED
  - MSG_BASELINE_NED
  - MSG_DOPS
  - MSG_ORIENT_EULER
  - MSG_AGE_CORRECTIONS
  - MSG_DGNSS_STATUS
  - MSG_PROTECTION_LEVEL
---

# Decode an SBP stream

SBP is a binary wire protocol, not an HTTP API: a 6-byte header, a variable payload, and a CCITT
CRC16 (XMODEM) trailer. Every message type has a fixed field layout published in `spec/` and a JSON
Schema in `json-schema/`.

## 1. Get the bytes

Devices expose SBP over serial (`/dev/ttyUSB0`, `/dev/ttyACM0`, `/dev/tty.usbmodem*`, typically
115200 baud) or over TCP — Swift's own client documents `<device-ip>:55556` for Piksi Multi, Duro and
Starling.

## 2. Convert to JSON

```sh
pip3 install sbp            # or: cargo install --git https://github.com/swift-nav/libsbp.git --bins
python3 -m sbp2json < sbp.bin
```

The Rust and Haskell builds of `sbp2json` are substantially faster than the Python one; the provider
says so itself. `json2sbp` reverses it. `json2json` expands the abbreviated JSON the Swift Console
writes into full per-message objects.

## 3. Validate

Each message validates against `json-schema/<MessageName>.json` (draft-06). Two of the 225 published
schemas — `MsgStartup.json` and `MsgCellModemStatus.json` — carry an illegal trailing comma and will
be rejected by strict parsers. Skip those two rather than repairing them; they are upstream defects,
recorded in `json-schema/swift-navigation-json-schema.yml`.

## 4. Read the messages that matter

| Message | What it gives you |
|---|---|
| `MSG_GPS_TIME` | GPS week + time of week; precedes a set of solution messages sharing that time |
| `MSG_UTC_TIME` | UTC, with the leap-second offset applied |
| `MSG_POS_LLH` | Absolute geodetic position + fix mode in `flags` |
| `MSG_POS_LLH_COV` | Same, with the full position covariance matrix |
| `MSG_VEL_NED` | Velocity in the north/east/down frame |
| `MSG_BASELINE_NED` | RTK baseline to the base station |
| `MSG_DOPS` | Dilution of precision |
| `MSG_ORIENT_EULER` | Attitude, on inertial-capable devices only |
| `MSG_AGE_CORRECTIONS` | Age of the differential corrections in use |
| `MSG_DGNSS_STATUS` | Differential solution status |
| `MSG_PROTECTION_LEVEL` | Integrity bounds — read this, not accuracy, for safety decisions |

## Reading `flags` is not optional

`MSG_POS_LLH.flags` carries the fix mode. A single-point (SPP) position and an RTK-fixed position
arrive in the *same message type* and differ by two orders of magnitude in accuracy. Treating every
`MSG_POS_LLH` as centimetre-accurate is the classic SBP integration bug. Cross-check
`MSG_AGE_CORRECTIONS` — a stale correction age means the stream has dropped even though positions
keep arriving.

## Two subtleties the spec states and readers miss

- **`_GNSS` variants exist.** `MSG_POS_LLH` reports GNSS *fused with inertial*; `MSG_POS_LLH_GNSS`
  reports GNSS only. On an inertial device these differ. Pick deliberately.
- **`tow` is usually but not always the time of measurement.** Bit 5 of `flags` is `0` when it is;
  when it is not, `tow` may be a time of arrival or a local system timestamp.

## Deprecated messages

84 of the 322 published definitions carry a `_DEP` / `_DEP_A`…`_DEP_F` suffix. Swift Navigation never
deletes a message or reuses a type id, so an old device may still emit `MSG_OBS_DEP_A` where a
current one emits `MSG_OBS`. Decode them if you must support old firmware; never emit them.
See `lifecycle/swift-navigation-lifecycle.yml` for the full list by package.

## Before you act

Read-only. Nothing in this skill writes to a device. SBP *does* carry write surfaces
(`swiftnav.sbp.settings`, `swiftnav.sbp.flash`, `swiftnav.sbp.bootload`) — those change or erase
device firmware and configuration, have no reversal path documented, and are out of scope here.
