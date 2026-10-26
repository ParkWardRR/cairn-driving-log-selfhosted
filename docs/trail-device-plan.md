# Trail device plan

**Status: Designed.** Nothing is built and nothing has run. The board has not been powered on
for Cairn. The firmware is **Rust on `esp-hal` with Embassy** (the owner's choice, 2026-10-25).
This document is the design; the schedule is
[ROADMAP.md, T1–T8](../ROADMAP.md#t1t8--the-trail-device--designed). The firmware lives in its
own repository,
[cairn-trail-device-firmware](https://github.com/ParkWardRR/cairn-trail-device-firmware).

Added 2026-10-25 at the owner's request: *a physical device for hiking and walking, and sometimes
driving, without the OBD-II dongle.*

**Contents:**
[1. What it is](#1-what-it-is) ·
[2. The hardware](#2-the-hardware) ·
[3. What carries over from the dongle](#3-what-carries-over-from-the-dongle) ·
[4. What is new](#4-what-is-new) ·
[5. Who carries it: the carrier question](#5-who-carries-it-the-carrier-question) ·
[6. Activities: hike, walk, drive](#6-activities-hike-walk-drive) ·
[7. Power](#7-power) ·
[8. Getting bundles home](#8-getting-bundles-home) ·
[9. LoRa](#9-lora-off) ·
[10. Display and buttons](#10-display-and-buttons) ·
[11. Contract changes](#11-contract-changes) ·
[12. Security posture](#12-security-posture) ·
[13. Decisions for the owner](#13-decisions-for-the-owner) ·
[14. Risks](#14-risks)

---

## 1. What it is

A battery-powered recorder you carry. It records where you went — position, elevation, pace,
steps, heading — into the **same sealed, encrypted, receipt-gated v3 bundles the car's dongle
writes**, and hands them to the same server. The server, the dashboard and the phone learn one new
thing: a recording can come from something other than a car.

What it is **not**:

- **Not an OBD logger.** It has no vehicle bus. When it rides in a car it records a GNSS and IMU
  track and nothing about the engine. Where the car's dongle recorded the same window, the server
  can lay the two side by side; the devices never talk to each other.
- **Not an emergency device.** It is not a PLB, it has no satellite uplink, its LoRa radio is
  held off ([§9](#9-lora-off)), and nothing in Cairn will claim it can summon help.
- **Not Meshtastic.** The board is sold as a Meshtastic node. Cairn firmware replaces that
  firmware, and with the LoRa radio off there is nothing for a Meshtastic protocol to run over.

## 2. The hardware

**LILYGO T-Beam Supreme, 915 MHz, u-blox MAX-M10S GNSS, 1.3" SH1106 OLED.** From the vendor's
hardware page; every row is *to be read off the real unit* in T1, because LILYGO ships revisions
with different parts (the magnetometer has three variants).

| Part | Chip | Bus | Notes for Cairn |
|---|---|---|---|
| MCU | **ESP32-S3FN8**, dual-core LX7 | — | 8 MB flash, 8 MB PSRAM. Wi-Fi and BLE 5. **Supports Secure Boot V2 and flash encryption**, which the dongle's rev v1.0 ESP32 cannot |
| GNSS | **u-blox MAX-M10S** | UART GPIO 8/9, PPS GPIO 6; power **ALDO4**, backup from VRTC | Read **UBX-NAV-PVT**, not NMEA, so `h_acc_cm`/`v_acc_cm` are the receiver's own estimates ([format/v3 §4.1](../contracts/format/v3/spec.md)). Configured by UBX-CFG-VALSET only; ships at 9600 baud; hot start needs the 18650 |
| LoRa | **Semtech SX1262** | dedicated SPI (GPIO 10–13), BUSY 4, DIO1 1, RST 5 | **Held off** — never keyed ([§9](#9-lora-off)) |
| Display | **SH1106**, 128×64 monochrome | I²C bus 0 (GPIO 17/18), `0x3C` | Off by default; a screen is the largest discretionary load |
| IMU | **QMI8658**, 6-axis | shared SPI, CS 34, INT 33; on 3V3, so always powered (schematic) | Hardware pedometer and wake-on-motion. **INT is GPIO 33, which is not RTC-capable on the S3** — see [§7](#7-power) |
| Magnetometer | **QMC6310** (U/N) or **QMC6309** | I²C bus 0 | Heading at rest, which GNSS cannot give |
| Environment | **Bosch BME280** | I²C bus 0, `0x77` | Pressure, temperature, humidity. Pressure is the elevation signal |
| RTC | **NXP PCF8563** | I²C bus 1 (GPIO 42/41), `0x51`, INT 14 | **The first Cairn hardware with a real-time clock**, on 3V3 with its own backup cell. `TIME_OBSERVATION` source 4 has been reserved for exactly this |
| PMU | **X-Powers AXP2101** | I²C bus 1, `0x34`, IRQ 40 | Owns every peripheral rail. **Nothing answers until the PMU enables its rail**, so bring-up starts here. Fuel gauge, charge control, the PWR key |
| Storage | microSD | shared SPI, CS 47 | The same role as the dongle's card: a cache of ciphertext |
| Battery | one 18650 | — | Charged over USB-C (3.9–6 V, up to 1 A). The cell also backs the GNSS and the RTC |
| Buttons | PWR (PMU), BOOT (GPIO 0), RST | — | Two usable inputs: BOOT and the PMU's short-press event |
| Expansion | QWIIC, UART1 GPIO 43/44 | — | Unused |

The SD card and the IMU share one SPI bus. The dongle's lesson about a shared, unlocked serial
link ([firmware PR #54](https://github.com/ParkWardRR/cairn-esp32-device-firmware/pull/54)) is
taken as a design rule from the start: **every shared bus has one owner task or one mutex, and
the bring-up tests prove it under load.**

### 2.1 What the board's other firmware taught

On 2026-10-25 the owner collected the board's schematics, its stock images and the source of eight
projects that run on it or do something close — Meshtastic, MeshCore, SoftRF, RNode, a LoRa APRS
tracker, LilyGO's own board support, and three GPS loggers. The findings, with each source's
licence, are in the trail repository's [prior-art.md](https://github.com/ParkWardRR/cairn-trail-device-firmware/blob/main/docs/prior-art.md); the board
facts in [hardware.md](https://github.com/ParkWardRR/cairn-trail-device-firmware/blob/main/docs/hardware.md); the staged procedure in
[bringup.md](https://github.com/ParkWardRR/cairn-trail-device-firmware/blob/main/docs/bringup.md). Most of those projects are GPL-3.0 or unlicensed, so what
carries over is facts and lessons; the only code source is the MIT-licensed XPowersLib and
SensorLib register maps our drivers are written from. What changed the design:

- **The LoRa radio has its own rail (ALDO3)**, so holding it off is one rail never enabled
  ([§9](#9-lora-off)).
- **The board is a core module on a carrier**, and its rails are known from the schematics: ALDO1
  for the display, the I²C sensors *and bus 0's pull-ups*; ALDO4 for GNSS; BLDO1 for the card;
  DCDC1 for the CPU, IMU and RTC. DCDC3–5 feed only the expansion socket and stay off.
- **The M10 keeps another firmware's settings** in battery-backed RAM; Cairn writes its own
  configuration at every boot and clears what Meshtastic left, once.
- **Power figures:** about **105 mA** for the whole board with the M10 tracking (SoftRF) and about
  **1.45 mA** in deep sleep (LilyGO) — both shape [§7](#7-power).
- **The PMU's interrupt misfires** with no battery or a full one: read the PWR key while awake;
  wake on BOOT or a timer.
- **No project fuses barometric and GNSS altitude or computes climb.** Cairn's elevation is its own
  design ([§4](#4-what-is-new)).

## 3. What carries over from the dongle

The point of putting this device in Cairn rather than running Meshtastic is that the hard parts
already exist and are proven on hardware. What carries over is mostly **design, contracts and
vectors**, not code: the trail firmware is Rust ([§3.1](#31-the-firmware-stack-rust-esp-hal-embassy)),
and the dongle is C and C++. The exception is `format/v3`, which already has a Rust implementation
([§3.2](#32-how-the-code-is-shared)).

| From the dongle | Carried over as | Why it carries over unchanged |
|---|---|---|
| `format/v3` sealing — AEAD per frame, CRC chain, Merkle root, signed manifest | The bundle format, byte for byte | Nothing in it is about cars except the *name* of one header field ([§5](#5-who-carries-it-the-carrier-question)) |
| `K_root` generated on the hardware RNG and kept in flash, escrowed through `enrolment/v1` | Key management | The same blob, sealed to the same server key. Flash storage is `sequential-storage`, not NVS, so the durability rules are re-proven rather than assumed |
| The device counter, reserved and durable before the first header exists | Counter handling | Same rule, same reasons |
| The receipt-gated prune, receipts outside the bundle | Deletion | [Invariant 2](../ROADMAP.md#guiding-invariants) is not negotiable on a hiking device either |
| `ble/v1` offload — `LIST`, `GET_MANIFEST`, `READ`, `PUT_RECEIPT` | The phone path | The iPhone app and `cmd/cairn-phone` carry ciphertext they cannot read; they do not care what produced it |
| `TIME_OBSERVATION`, the plausibility floor (invariant 6) | Time | GNSS and the RTC both lie in ways that parse |
| The GNSS plausibility gate ([firmware PR #53](https://github.com/ParkWardRR/cairn-esp32-device-firmware/pull/53)) | Phantom-fix refusal | Re-tuned: walking speeds, no OBD speed to cross-check |
| One transition controller owns all state; sensing only reports facts ([Phase 6](../ROADMAP.md#era-1--the-v2-core-data-path)) | The task architecture | It maps directly onto Embassy tasks and channels |
| Secure OTA — separate update key, verify-before-download, hash-from-flash, set the boot slot last | Updates | The S3 has the same A/B partition model |

### 3.1 The firmware stack: Rust, esp-hal, Embassy

**Decided by the owner, 2026-10-25:** the trail firmware is **Rust on `esp-hal` with Embassy**, not
the dongle's Arduino-on-ESP-IDF C and C++. Reasons that hold up on their own: the bundle format
already exists in Rust; a device that parses radio and serial input (BLE, Wi-Fi, a GNSS UART) benefits from
memory safety more than most; async tasks fit a device that should sleep between samples; and the
ecosystem's licences (MIT / Apache-2.0) sit cleanly beside Cairn's Blue Oak.

| Need | Crate | Notes |
|---|---|---|
| HAL | `esp-hal` 1.x | The stable core; some drivers sit behind its `unstable` feature |
| Async runtime | `embassy-executor`, `embassy-time`, `embassy-sync`, through `esp-rtos` | `esp-rtos` provides Embassy support for `esp-hal` and the scheduler `esp-radio` needs |
| Wi-Fi and BLE controller | `esp-radio` | Needs a heap and `esp-rtos`; `unstable` |
| BLE host and GATT | `trouble-host` | The `ble/v1` offload service is re-implemented on it |
| Networking, TLS | `embassy-net`; `embedded-tls` (TLS 1.3 client) or `esp-mbedtls` | For `uplink/v1` over home Wi-Fi; chosen in T6 |
| Heap | `esp-alloc`, on the 8 MB PSRAM | |
| Keys and counter in flash | `esp-storage` + `sequential-storage` | Replaces NVS |
| Bootloader, partitions, OTA | `esp-bootloader-esp-idf` | The ESP-IDF second-stage bootloader, the app descriptor and the OTA slots |
| SD card, FAT | `embedded-fatfs` or `embedded-sdmmc` | **A risk:** sealing is an atomic directory rename ([format/v3 §2.1](../contracts/format/v3/spec.md)), so the FAT layer must rename and flush in a known order. Chosen in T2 by test, not by README |
| SD + IMU on one SPI bus | `embassy-embedded-hal` shared bus behind a mutex | The shared-bus rule ([§2](#2-the-hardware)) enforced by the type system |
| GNSS | `ublox` | UBX-NAV-PVT parsing, `no_std` |
| Display | `sh1106`, `embedded-graphics` | |
| Barometer | `bme280` (async) | |
| PMU, IMU, magnetometer, RTC | **Written here**: small register drivers for the AXP2101, QMI8658, QMC6310/6309 and PCF8563 | The AXP2101 driver is T1's first job, from the register map in X-Powers' MIT-licensed XPowersLib |
| Cryptography | RustCrypto `chacha20poly1305` (XChaCha20), `hkdf`, `sha2`, `ed25519-dalek`, `x25519-dalek` | The crates the emulator already uses against the vectors. The S3's SHA accelerator only if a measurement asks for it |
| Randomness | `esp-hal`'s TRNG | Its entropy source must be running; like the dongle, the device **refuses to enrol** without hardware randomness |
| Logs | `defmt` over the S3's USB-Serial-JTAG | |

**Tasks.** One task owns each bus — the GNSS UART, I²C bus 0 (display, magnetometer, BME280), I²C
bus 1 (PMU, RTC) and the shared SPI bus (SD, IMU) — and reports facts
over `embassy-sync` channels to **one capture controller task that owns all state** and decides
every transition. The storage task alone writes the card. The radio SPI bus has no owner: the
LoRa chip is sent to sleep once at boot and never addressed again. That is the dongle's Phase 6 rule, and
in Embassy it is the natural shape rather than a discipline.

**Toolchain.** The S3 is Xtensa, which upstream Rust does not target: `espup` installs Espressif's
toolchain fork, and `espflash` flashes and monitors. The logic crates — `cairn-format`, activity
detection, the plausibility gate — are `no_std` and run their tests on stable Rust on the host,
which is where the vectors are replayed. CI builds the firmware image and runs the host tests on
the Podman Linux runners; there is no hardware in CI while there is one board.

### 3.2 How the code is shared

The emulator in the dongle's repository is already a **Rust implementation of `format/v3`**
(`emulator/src/format`, about 4,400 lines with tests): 33 of 33 vectors, a 20-row fault matrix
against a live v3 server, mutations each caught. It is `std`-based today. It is extracted into a
**`no_std` + `alloc` crate, `cairn-format`**, which the emulator and the trail firmware both
depend on, pinned by git revision in each `Cargo.lock`. A fix lands in the crate, and each
consumer takes it by bumping the revision.

What that changes, stated plainly:

- **The emulator stops being the lagging implementation.** Today a format change that reaches
  the C writer but not the Rust emulator turns firmware CI red for days. Once the Rust code ships
  on a device, the same omission breaks a device — so it can no longer be left for later.
- **The count of independent `format/v3` implementations stays three** — Go, C and Rust — not
  four. The emulator and the trail device share one. A shared implementation proves nothing new
  about the spec; it is counted once.
- **`enrolment/v1` and `ble/v1` offload get a new, independent implementation.** No Rust version
  of the enrolment producer or the offload server exists, so the trail device's are written from
  the specs and replay the existing vectors — enrolment goes from two implementations to three.
- **The C lessons survive as tests, not code.** The ROM CRC wrapper that dropped its xorout
  ([Phase 12](../ROADMAP.md#era-1--the-v2-core-data-path)) is why `cairn-format` uses a software
  CRC and the device runs the vectors in a self-test build on its own silicon.

## 4. What is new

| New | Where it shows up |
|---|---|
| **Barometric elevation** | A `BARO_SAMPLE` record; the server fuses it with GNSS altitude into elevation gain and loss. GNSS altitude alone is noisy by tens of metres; pressure is smooth and drifts with weather, and only together do they give a defensible climb |
| **Battery state** | A `POWER_STATUS` record: cell voltage, state of charge, charging, USB present. On a dongle `DEVICE_HEALTH.battery_mv` is the car's rail; here it is the cell |
| **Steps, cadence, heading** | An `ACTIVITY_SUMMARY` record per window: steps from the IMU's pedometer, cadence, the device's own guess at the mode, and magnetic heading |
| **Marks** | A button press writes a `TRIP_EVENT` of a new type, `MARK`, at the current position — a waypoint, a view, a junction, a trail-sign photo taken on the phone |
| **A real-time clock** | `TIME_OBSERVATION` source 4, `RTC`, finally used. Set from GNSS; refused as a basis whenever the PCF8563's voltage-low flag says it lost time |
| **A display** | Status, the current activity, a bearing back to the start ([§10](#10-display-and-buttons)) |
| **Start and stop without an ignition** | Recording starts on sustained movement or a button, and stops on sustained stillness or a button ([§6](#6-activities-hike-walk-drive)) |

## 5. Who carries it: the carrier question

Every v3 bundle carries `vehicle_id` and `assignment_id` in its segment headers and its manifest,
and both are **mandatory** ([format/v3 §3.1, §5.1](../contracts/format/v3/spec.md)). Intake
authorises a bundle against them before it decrypts anything. A hiking device has no vehicle.

**Recommendation: a carrier, without a format change.** The server's vehicle registry gains a
`kind` — `car` or `person` — and the trail device is assigned to a `person` carrier. The 16 bytes
in the header keep their name and their layout (the format is frozen and stable); the
specification gains a sentence saying the field identifies *what carried the device*, which for
every bundle written so far was a car. Everything already scoped by `vehicle_id` — the store, the
sync API, assignments by counter order, the garage — keeps working, and a person's hikes stay
apart from a car's drives because they have a different id.

Alternatives considered:

| Option | Why not |
|---|---|
| A `format/v4` with a `carrier_id` | A breaking format for a rename. Three implementations, every vector, the dongle reflashed over USB — to change a word |
| Assign the trail device to the car | Every walk would appear as a drive of the 428i |
| A synthetic "vehicle" called *On foot* with no `kind` | Works on day one and then every car-shaped view — drive summary, trims, boost — has to guess which rows to skip |

**Driving with the trail device.** It stays assigned to the person. The device classifies the
activity as `drive` from speed and the absence of a step cadence ([§6](#6-activities-hike-walk-drive));
where the car's dongle recorded an overlapping window, the server links the two by time. No radio
link, no pairing between devices, no change to the dongle.

## 6. Activities: hike, walk, drive

**An activity is the trail device's trip**: one outing, start to finish, possibly across several
boots (a battery swap, a reset). It reuses the server's trip builder, which already treats a trip
as an outing rather than a power cycle (`store/v1.4`, `v_trip_leg`).

**Starting and stopping.** No ignition and no supply rail, so the evidence is different and the
rule is the same one the dongle learned the hard way — start on positive evidence with
hysteresis, stop only on positive evidence of stopping:

| Event | Evidence |
|---|---|
| Start | Steps above a cadence floor for a sustained window, **or** GNSS speed above walking pace with IMU motion agreeing, **or** a button |
| Pause | Stillness: no steps, IMU at its idle floor, GNSS speed under the noise floor. A pause is a leg boundary, not an end — lunch at the summit is part of the hike |
| Stop | Stillness for longer than the stop dwell (designed: 20 minutes), **or** a button. The pre-roll ring keeps the first minute of movement before a start is confirmed |

**The mode.** The device writes its own guess (`hike`, `walk`, `run`, `drive`, `unknown`) and a
confidence into each `ACTIVITY_SUMMARY`, and a `MODE_CHANGE` event when it changes its mind. The
**server decides**, from the same records: step cadence, speed, climb rate and the overlap with a
dongle trip. *Hike* versus *walk* is a terrain and climb distinction the server can make better
with the whole track than the device can in a window. A user can override it later; the override
is an annotation, never an edit to a sealed bundle (invariant 1).

**Sampling.**

| Mode | GNSS | IMU | Baro | Why |
|---|---|---|---|---|
| Recording, on foot | 1 Hz | summaries every 1 s, pedometer continuous | 1 Hz | A trail switchback is 10–20 m; 1 Hz at walking pace is a point every 1–1.5 m |
| Recording, drive | 5 Hz (as the dongle) | 1 s summaries, event windows | 1 Hz | Matches the dongle's track density |
| Endurance | 0.1 Hz, M10 power-save | pedometer only | 0.1 Hz | Multi-day. Honest gaps over invented continuity (invariant 4) |
| Idle | off, last fix saved for warm start | wake-on-motion only | off | — |

## 7. Power

**Designed targets, all unmeasured:** at least **24 hours** of 1 Hz on-foot recording from one
3000 mAh 18650 with the display mostly off, and at least **5 days** in endurance mode. The real
numbers come from a meter in T5, not from datasheet arithmetic — the dongle's parked current is
still unmeasured three weeks in, and this is a device where that number *is* the product.

Two figures from other firmware for this board set expectations: SoftRF quotes about **105 mA**
for the whole board with the M10 tracking continuously, which empties 3000 mAh in under 29 hours
before the display, Wi-Fi or BLE draw anything — so **the 24-hour target depends on the
receiver's power save** (cyclic tracking, which on the M10 rules out SBAS and BeiDou B1C); and
LilyGO measures about **1.45 mA** in deep sleep on this board, three times its smaller boards, so
endurance mode gets its days by cutting rails — GNSS, the display and sensors, the card — not by
sleeping the CPU alone.

What drives it: the M10 tracking continuously is the largest steady load; the ESP32-S3 must
light-sleep between samples, not idle awake; the SD card is written in batched flushes (the
dongle's aggressive-flush lesson is about not losing data on a power cut; a trail device's power
cut is a flat cell, which the PMU announces in advance, so it can seal before it dies); the
display times out; the LoRa radio is held off from boot ([§9](#9-lora-off)).

**Wake sources.** The S3's RTC-capable GPIOs are 0–21. The IMU interrupt is on **GPIO 33**, so
wake-on-motion cannot wake the chip from deep sleep — the same shape of problem that ruled out deep
sleep on the dongle. The RTC interrupt (GPIO 14) and BOOT (GPIO 0) *are* RTC-capable, and the
PMU IRQ (GPIO 40) is not. So: light sleep with the IMU interrupt while an activity might start;
deep sleep with an RTC-timer or button wake when the device has been still for hours. **To be
confirmed on the board in T1**, including whether the PMU can gate a rail to work around it.

**A low cell seals, it does not truncate.** At a PMU threshold the device seals the open bundle
and stops recording, leaving enough charge for the RTC and for a later boot to report the state.
`LOW_POWER` in the degraded bitmap means the cell, on this device.

## 8. Getting bundles home

The order a bundle leaves in, and none of these is new code:

1. **BLE to the phone**, `ble/v1` offload, exactly as the dongle — the phone relays ciphertext,
   the server signs a receipt, the device prunes on a receipt it verified itself. This is the path
   that matters on a trail, and it waits on the same iPhone work the dongle does
   ([Phase 29](../ROADMAP.md#phase-29--the-phone-becomes-the-app--in-progress)); until then,
   `cmd/cairn-phone` on a Mac carries it.
2. **Wi-Fi at home**, by `uplink/v1` from the start. The trail device has no legacy mTLS listener
   to migrate from, so it can be the **first implementer** of the device-signed uplink, and the
   dongle follows ([Phase 28](../ROADMAP.md#phase-28--the-networked-dongle-finished--in-progress)).
3. **The SD card**, read on a Mac and pushed with `cairn-push`. The fallback that always works.

There is **no LTE** on this board and no plan to add one.

## 9. LoRa: off

**Decided by the owner, 2026-10-25: the LoRa radio is not used for anything, and is held off to
save battery.** The SX1262 is on the board and nothing in the firmware will key it. What that
takes in practice:

- **The radio is put in its lowest-power state at every boot, before anything else draws power.**
  If it has a PMU rail of its own, that rail stays disabled; if it shares a rail, the chip is
  sent its sleep command (cold start, no retention) and its pins are left in a state that does
  not back-power it. Which of the two applies is a T1 finding, read off the schematic and the
  board — not assumed. Power-on leaves an SX1262 in standby, drawing on the order of half a
  milliamp by its datasheet — over a day, a real slice of the battery budget, so *doing nothing* is not the same as *off*.
- **T1 proves it with a meter**: the board's current with the radio held off is recorded beside
  its current with the radio merely uninitialised.
- **No LoRa crate, task or SPI traffic** beyond that one sleep command. The radio SPI bus has no
  owner task.
- **The antenna stays fitted.** An SX1262 that is ever keyed without one can be damaged; with the
  radio off this is a precaution rather than a risk, and it keeps the board able to run its stock
  firmware again.

The earlier design for a *last seen* beacon — provisional, sealed, never a record, never a reason
to prune — stays recorded in [`beacon/v1`](../contracts/beacon/v1/README.md), which remains
reserved and is now **deferred**. Nothing is designed there, and nothing is scheduled.

## 10. Display and buttons

128×64 monochrome, off unless woken. Four screens, cycled with a short press of BOOT:

| Screen | Shows |
|---|---|
| Status | Recording or not, and how the device knows; fix and satellites; battery %; bundles awaiting a receipt |
| Activity | Elapsed and moving time, distance, elevation gain, pace |
| Back | Bearing and straight-line distance to where the activity started, from the magnetometer and GNSS |
| Clock | Time, its source (GNSS, RTC, phone), sunset |

| Input | Action |
|---|---|
| BOOT, short | Next screen; wakes the display |
| BOOT, long (2 s) | **Mark** — a `MARK` event at the current position, confirmed on screen |
| PWR, short | Wake the display |
| BOOT + PWR, long | Start or stop an activity by hand |
| PWR, long (hardware) | Power off, handled by the PMU — the device seals first if it gets the warning |

The recording indicator follows the iPhone app's CarPlay rule: **it claims recording only on
fresh evidence**, and says so when it cannot confirm. A logger that cannot say "I don't know" is
not trustworthy.

## 11. Contract changes

All **Designed**, none merged into a protocol's spec. Each one follows the
[contract process](../contracts/README.md#how-a-contract-changes): spec and implementation
together, candidate vectors, an independent consumer validating them, then a maintainer's tag.
And a format change has to reach **all three** implementations — Go, the C dongle and the Rust
`cairn-format` crate the emulator and the trail device share — in the same release.

| Protocol | Change | Kind |
|---|---|---|
| `format/v3` | Record type `0x0C` **`BARO_SAMPLE`**, 12 bytes: `pressure_pa` u32, `temp_cc` i16 (°C × 100), `humidity_e2` u16 (%RH × 100), 4 reserved. Sentinels for an absent reading, as every other record | additive — old readers skip unknown types |
| `format/v3` | Record type `0x0D` **`POWER_STATUS`**, 8 bytes: `cell_mv` u16, `soc_pct` u8 (`0xFF` unknown), `flags` u8 (b0 charging, b1 USB present, b2 PMU low-battery warning), `charge_ma` i16 (negative = discharging), `pmu_temp_c` i8, 1 reserved | additive |
| `format/v3` | Record type `0x0E` **`ACTIVITY_SUMMARY`**, 12 bytes: `window_ms` u16, `steps` u16, `cadence_spm` u16, `mode` u8 (0 unknown, 1 still, 2 walk, 3 hike, 4 run, 5 drive), `mode_confidence` u8 (0–100), `heading_cdeg` u16 (magnetic, `0xFFFF` unknown), 2 reserved | additive |
| `format/v3` | `TRIP_EVENT` types **9 `MARK`**, **10 `MODE_CHANGE`** (detail: the new mode), **11 `USER_START`**, **12 `USER_STOP`** | additive — §4.6 already requires an unknown type to be kept |
| `format/v3` | `TIME_OBSERVATION` source **4 `RTC`** moves from reserved to defined, with the rule that an RTC reporting lost integrity (PCF8563 `VL`) is not written at all | clarifying |
| `format/v3` | Prose: `vehicle_id` identifies what *carried* the device; `DEVICE_HEALTH.battery_mv` is the device's supply — the car's rail on a dongle, the cell on a self-powered device | clarifying |
| `format/v3` | Manifest key **30 `device_class`**, optional text: `freematics-one-plus-b`, `lilygo-tbeam-supreme`. Absent means a dongle, which is what every existing bundle is | additive — optional keys sort into place, as key 29 did |
| `ble/v1` | Device-info: a `device class` record and capability bits for barometer, display and self-powered; `info_minor` raised | additive |
| `store/v1` | Tables `baro`, `power`, `activity`; views `v_activity_summary` (distance, moving time, fused elevation gain and loss, pace, steps, mode) and `v_marks`; `kind` on `v_vehicles` | additive |
| `sync/v1` | `kind` on the vehicle entity, optional | additive; `sync/v1` is still draft |
| `enrolment/v1` | **No change.** The same blob and console protocol. The board's USB-serial reset differs; that is a tool concern | — |
| `uplink/v1` | **No change.** The trail device is meant to be its first implementer | — |
| `beacon/v1` | **Reserved and deferred**: a sealed LoRa position beacon. The radio is held off (owner, 2026-10-25), so nothing is designed or scheduled | reserved |

## 12. Security posture

Inherited from the dongle and the [trust model](trust-model-v3.md): every frame AEAD-encrypted, the
card a cache of ciphertext, the root escrowed, deletion only on a verified receipt, signed OTA.

What is different, and better: **the ESP32-S3 supports Secure Boot V2 and flash encryption.**
[Phase 34](../ROADMAP.md#phase-34--chip-hardening-on-replacement-hardware--planned-gated) was
reassigned to "whatever hardware replaces this dongle"; this is the first Cairn hardware on which
it *can* run. The same prerequisites apply unchanged — a sacrificial unit through the whole
procedure in development mode first, the OTA path proven, and the bootloader's secure-boot and
flash-encryption configuration checked in. Doing it with an `esp-hal` image — the ESP-IDF
bootloader from `esp-bootloader-esp-idf`, the application signed with `espsecure` — has not been
done in this project and is part of what T8 proves. A T-Beam Supreme costs about as much as a tank of fuel, which
makes the sacrificial unit an easy decision rather than a blocker.

What is different, and worse: **it is carried, so it is lost.** A dongle stays bolted in a car; a
hiking device is left on a summit cairn. Revocation of a lost device has to be a working button
before the device is used for real, not a later phase.

## 13. Decisions for the owner

| # | Decision | Recommendation |
|---|---|---|
| 1 | **Repository name.** `cairn-trail-device-firmware` was used for the new repository | Keep. A cairn is a trail marker; the name was always waiting for this device |
| 2 | **The carrier model** ([§5](#5-who-carries-it-the-carrier-question)) | A `kind` on the vehicle registry; no format change |
| — | **Firmware language** | **Decided 2026-10-25: Rust, `esp-hal` + Embassy** ([§3.1](#31-the-firmware-stack-rust-esp-hal-embassy)) |
| 3 | **Where `cairn-format` lives** ([§3.2](#32-how-the-code-is-shared)): the dongle repository's emulator workspace, the trail repository, or a repository of its own | **The emulator workspace in the dongle repository**, where the code and its fault matrix already are. Move it out only if a third consumer appears |
| 4 | **Meshtastic.** | **Moot while the radio is off** (owner, 2026-10-25). If LoRa ever returns: Meshtastic and its protocol definitions are GPLv3 and Cairn is Blue Oak, so implementing it is a licensing question first |
| 5 | **How many boards.** One unit to use, one to sacrifice | **Two**, before T8 — and the second one is the first chance in the project to exercise power-loss-during-OTA without risking the only unit |

## 14. Risks

| Risk | Mitigation |
|---|---|
| A board revision differs from the vendor page (three magnetometer variants already) | T1 reads every part off the real unit and records it in the trail repository's `docs/hardware.md`; the firmware probes rather than assumes |
| `cairn-format` drifts between the emulator and the device | One crate, pinned by revision in both `Cargo.lock` files, replaying the same vectors in both CIs |
| Automatic light sleep under Embassy on `esp-hal` is immature | T5 budgets for explicit light sleep around timer waits; measure before assuming the executor's idle is cheap |
| `esp-radio` is `unstable` and changes between minor releases | Exact versions in `Cargo.lock`, bumped deliberately, never by a floating range |
| The FAT layer cannot rename a directory atomically, or reorders writes | T2 tears power mid-seal on the board and checks the recovery scan, before the crate choice is final |
| Wi-Fi and BLE coexistence under `esp-radio` — the dongle's [#31](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/31) on Arduino | T6 runs offload and upload in one boot, repeatedly |
| Secure Boot V2 with a Rust image is unproven here | T8, on the sacrificial unit only |
| Battery life far below the target | T5 is a phase of its own with a meter, before anything is called usable on a multi-day trip |
| Phantom fixes under tree cover and in canyons | The plausibility gate re-tuned for walking, with the IMU and pedometer as the cross-check OBD speed was in the car; the server's `v_position_trusted` already exists |
| The LoRa radio draws power although unused | Held off from boot — rail disabled or chip in cold sleep — and measured in T1 and T5 |
| Work on the trail device starves the dongle's critical path | T-phases sit beside phases 27–29, never ahead of them; the shared-core rule means a fix found on the trail device lands in the dongle first |
