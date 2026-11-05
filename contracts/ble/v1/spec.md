# BLE protocol v1

Custom GATT service between the iPhone (central) and the Cairn ESP32 dongle (peripheral). Typed characteristics, fixed binary layouts, little-endian, no CBOR/JSON/strings on the wire.

This spec is normative here, in the Cairn contracts (`contracts/ble/v1/`). The iOS app tracks it; a protocol change updates this file and the vectors together. Golden byte vectors are in [vectors/golden/golden-vectors.json](vectors/golden/golden-vectors.json). They come from an independent Python implementation of this spec, and the iOS tests replay them.

## Discovery

| Item | Value |
|---|---|
| Service UUID | `A8E3xxxx-4F5B-11EF-A017-325096B39F47` (base; 16-bit suffix per characteristic below. Final values assigned when the spec lands in the firmware repo) |
| Scan filter | Service UUID only. Advertised name `Cairn` is display-only |
| Security | All characteristics require encrypted + authenticated access (`READ_ENC`/`WRITE_ENC` + `AUTHEN`). Unbonded phones can discover but not read, write, or subscribe. (The firmware's dev build flashed for the 2026-10-06 first-trip bring-up drops `AUTHEN` and pairs with Just Works; see the firmware README's "Pairing and access" caveat. This contract describes the shipping target.) |
| Bonding | Static 6-digit passkey from gitignored `secrets.h`. Bond limit 1; new bond replaces old. Bond reset via boot-time button or firmware command |

## Characteristics

| Name | Suffix | Dir | Size | Cadence | Phase |
|---|---|---|---|---|---|
| `GNSS_FIX` | `0001` | phone → device, write w/o response | 28 B | ~1 Hz | 1 |
| `GNSS_QUALITY` | `0010` | device → phone, notify | 8 B | 1 Hz | 1 |
| `COMPANION_STATUS` | `0011` | device → phone, notify | 8 B | 1 Hz | 1 |
| `PROTOCOL_VERSION` | `00F0` | read | 2 B | once | 1 |
| `BARO_ALT` | `0002` | phone → device | 4 B | ~1 Hz | 2 |
| `UTC_SYNC` | `0003` | phone → device | 8 B | connect + 1/min | 2 |
| `ENGINE_DECLARATION` | `0004` | phone → device, write w/o response | 1–31 B | once per bond | 2 |
| `OBD_LIVE` | `0020` | device → phone | 48 B | ~1 Hz | 2 |
| `DEVICE_STATUS` | `0021` | device → phone | 12 B | 1 Hz | 2 |

`PROTOCOL_VERSION`: `u8 version` + `u8 capabilities bitmap`. The app refuses to stream to unknown major versions. Bit 2 = bundle offload ([offload.md](offload.md)); bit 3 = device information ([device-info.md](device-info.md), which also defines the characteristics `0040` to `0044`).

## `GNSS_FIX` (28 bytes)

| Off | Size | Field | Type | Units / sentinel |
|---:|---:|---|---|---|
| 0 | 4 | `lat_e7` | i32 | deg × 10⁷; 0 when position invalid |
| 4 | 4 | `lon_e7` | i32 | deg × 10⁷; 0 when position invalid |
| 8 | 4 | `alt_cm` | i32 | cm above WGS-84 ellipsoid; `0x7FFFFFFF` = invalid |
| 12 | 2 | `speed_cmps` | u16 | cm/s; `0xFFFF` = unavailable (CLLocation.speed < 0) |
| 14 | 2 | `heading_cdeg` | u16 | course over ground, centideg; `0xFFFF` = unavailable (CLLocation.course < 0) |
| 16 | 2 | `h_acc_cm` | u16 | cm; `0xFFFF` = invalid; clamped to 65534 on overflow, never wrapped |
| 18 | 2 | `v_acc_cm` | u16 | cm; `0xFFFF` = invalid; clamped to 65534 |
| 20 | 1 | `fix_type` | u8 | 0 none, 1 2D, 2 3D (mapping below) |
| 21 | 1 | `validity_flags` | u8 | b0 position, b1 altitude, b2 speed, b3 course |
| 22 | 2 | `sample_age_ms` | u16 | `now − CLLocation.timestamp` at write time |
| 24 | 2 | `seq` | u16 | monotonic, wrapping; duplicate detection + ack |
| 26 | 1 | `companion_flags` | u8 | b0 charging, b1 hint; all other bits 0 (below) |
| 27 | 1 | `reserved` | u8 | 0 |

### `companion_flags` (byte 26)

The phone's own state, which the receiver cannot see. Byte 26 was the low half of a two-byte
`reserved`, and both sides were already using it — the app wrote the charging bit and the
firmware read it — so most of this writes down what shipped rather than changing anything on the
wire. A reader that does not know a bit ignores it; a writer leaves unknown bits zero.

| Bit | Name | Meaning |
|---:|---|---|
| 0 | `CHARGING` | The phone is on power. The device reads it as permission to sample harder: in the reference firmware a charging phone during a trip forces the highest sampling dynamics, because the usual reason to hold back is the phone's battery and that reason is gone |
| 1 | `HINT` | **Not a measurement.** A cached fix, sent once per link so the receiver has somewhere to start. See below |
| 2–7 | — | Reserved, zero |

**`HINT` frames.** The dongle's receiver sits under the dash and starts cold. A phone that has
just been in the owner's pocket knows roughly where the car is, and handing the receiver that
position shortens the time to first fix. But the phone's last fix is usually minutes old, and
every other rule in this protocol exists to stop an old fix being mistaken for a current one.
`HINT` is how both can be true:

- A `HINT` frame is **accepted regardless of `sample_age_ms`**, and only for hinting. The
  staleness rule below does not apply to it.
- A `HINT` frame is **never recorded as a position sample** and never becomes `last_gnss`, never
  clears `DEGRADED_GNSS` and never influences trip scoring. It may be recorded as a hint event.
- `sample_age_ms` saturates at `0xFFFF`, which on a `HINT` frame means "65.535 s or older" and
  carries no further information. It is still filled in, because it is the one thing that says
  how little the hint is worth.
- `validity_flags` b0 must be set: a hint with no position is nothing. The other bits are
  whatever the cached fix had, which for a parked phone is usually no speed and no course.

Two different actors, two different rules, which is why they are stated apart:

- **A writer sends at most one `HINT` per link**, as early as it has a cached position, and
  none at all once the device reports a fix. Sending a stream of them would be sending a stream
  of stale positions, which is the thing the rest of this protocol prevents.
- **A device accepts a hint only before its first accepted fix of the current receiver start,
  and may accept more than one.** It is not limited to one: a receiver start can be offered a
  position by the device's own last saved fix, by a Wi-Fi scan and by a phone, and which of
  them the receiver ends up holding should be decided by **stated accuracy**, not by which
  arrived last. A device that takes only the first hint of a start is taking the worst one
  whenever a better one follows; a device that takes the last is letting a phone that happened
  to connect late overwrite a sharper fix.

### `fix_type` mapping

| CLLocation | `fix_type` |
|---|---|
| `horizontalAccuracy < 0` | 0 |
| `horizontalAccuracy ≥ 0`, `verticalAccuracy < 0` | 1 (2D) |
| both ≥ 0 | 2 (3D) |

Core Location exposes no DGPS/RTK state, DOP, satellite count, or constellations. `sats_used`, `sats_visible` → `0xFF`; `hdop_e2` → `0xFFFF` in the stored sample.

### Reference systems

- Position: WGS-84 (matches bundle format v2).
- Altitude: height above the WGS-84 ellipsoid, as the bundle format specifies, not MSL.
- `heading_cdeg`: course over ground, not compass heading. `CLHeading` is a Phase 2 enrichment.

## `BARO_ALT` (4 bytes, Phase 2, phone → device)

| Off | Size | Field | Type | Units / sentinel |
|---:|---:|---|---|---|
| 0 | 4 | `rel_alt_cm` | i32 | cm relative to where `CMAltimeter` updates started; `0x7FFFFFFF` = invalid; clamped to `0x7FFFFFFE` |

Relative barometric altitude, not MSL and not the GNSS ellipsoid. The dongle should not mix it with `alt_cm` without a reference.

## `UTC_SYNC` (8 bytes, Phase 2, phone → device)

| Off | Size | Field | Type | Units / sentinel |
|---:|---:|---|---|---|
| 0 | 8 | `unix_ms` | u64 | phone wall clock, Unix milliseconds; sent on connect and about once a minute |

The app sends both only when the dongle exposes the characteristic, using write-without-response when offered, otherwise a confirmed write.

## `ENGINE_DECLARATION` (1–31 bytes, Phase 2, phone → device)

A UTF-8 engine profile identifier (e.g. `bmw-n20`, `bmw-b58`). Written without response, no NUL terminator, 1–31 bytes. The dongle persists the latest declaration in NVS and uses it in the engine-discovery chain (compile-time → **BLE declaration** → VIN pattern → VIN engine codes → default). A profile the firmware does not have compiled in is accepted and logged, but will not change the active profile until an image with that profile is flashed.

The phone writes this at most once per bond, after the user picks a vehicle whose profile differs from what the dongle would auto-select. It is not an override of a running profile mid-trip: the firmware uses it only at the next OBD identification pass. The server does not read it directly — the vehicle record's `engine_code` on the server is independent and used for display and query filtering.

Writes larger than 31 bytes or empty are refused silently and logged on the device as `engine declaration: bad length`.

## Staleness and timing

| Rule | Where | Value |
|---|---|---|
| Discard cached/old locations before encoding | iOS | `abs(now − location.timestamp) > 2 s`, **except a `HINT` frame**, which is cached on purpose |
| Reject late arrivals | firmware | `sample_age_ms > CAIRN_PHONE_GNSS_STALE_MS` (3000), **except a `HINT` frame**, which is used for hinting and never stored |
| Timestamp mapping | firmware | `fact.monotonic_ms = millis() − sample_age_ms` |
| Mapping error bound | | BLE latency (~10–30 ms) + phone measurement-to-send delay |

BLE receipt time is not measurement time. A cached fix arriving after reconnect must not become `last_gnss`, clear `DEGRADED_GNSS`, or influence trip scoring. A `HINT` frame is the one way to send a cached fix deliberately, and it is subject to exactly the same prohibition: it may steer the receiver and it may be logged, and it may do nothing else.

## `GNSS_QUALITY` (8 bytes, internal receiver)

Proposed layout (fields are fixed by the plan; offsets are not yet frozen): `fix_type` u8, `sats_used` u8, `hdop_e2` u16, `fix_age_ms` u32. Lets the app show both receivers side by side.

## `COMPANION_STATUS` (8 bytes, firmware acceptance feedback)

Proposed layout: `last_accepted_seq` u16, `accepted_count` u16, `rejected_count` u16, `queue_drop_count` u16.

The app's "Streaming" indicator means *the device recently acknowledged acceptance*, not that the phone called `write`.

## Firmware rejection rules

Reject and count when: payload ≠ 28 B, `sample_age_ms` over threshold, `validity_flags` b0 clear, duplicate `seq`, or fact queue full (counted as drop). Callbacks are bounded: validate, copy, non-blocking `xQueueSend`, return.

## Session semantics

- On disconnect: firmware clears all phone GNSS state (last fix, seq, counters). No replay.
- On reconnect: device accepts the phone's new `seq` without continuity from the prior session.
- Unknown protocol versions, characteristics, and flag bits: ignore flags, refuse unknown major versions.

## iOS write flow control

Use `canSendWriteWithoutResponse` and `peripheralIsReady(toSendWriteWithoutResponse:)`. Check `maximumWriteValueLength(for: .withoutResponse) ≥ 28` at connect. Flow control does not prove the firmware queue accepted the sample; `COMPANION_STATUS` does.
