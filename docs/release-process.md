<!-- cairn-nav:start -->
<p align="center"><b>Cairn is a family of six repositories.</b> Each builds, tests and releases on its own; they agree through the shared <a href="https://github.com/ParkWardRR/cairn-driving-log-selfhosted/tree/main/contracts">contracts</a>, and they share one <a href="https://github.com/ParkWardRR/cairn-driving-log-selfhosted/blob/main/ROADMAP.md">roadmap</a>.</p>
<!-- cairn-nav:end -->

# The release process

**Three channels — `dev`, `beta`, `stable` — and a rule that decides which features are in
each one.** Scheduled as
[Phase 36](../ROADMAP.md#phase-36--two-channels-beta-and-stable--planned); this page is the
design, the roadmap is the order of work.

## Why this exists

Cairn was built fast. A great deal of it works, and a great deal of it is *written, host-tested
and never confirmed* — the roadmap keeps those apart on purpose
([Proven / Built / Designed](../ROADMAP.md#how-to-read-this)), and the
[Built, not yet proven](../ROADMAP.md#built-not-yet-proven) list is six entries long. Today
there is no way to act on that distinction: one `git push`, one `deploy-v3.sh`, and the
unproven arrives beside the proven with nothing telling them apart. There is no tag on any
component, no artifact anyone can install, and `deploy-v3.sh` ships the **working tree** — so
uncommitted code can reach the machine the car uploads to.

A channel is how "this part is rough" stops being an apology and becomes a property of the
build. Nothing here is about slowing work down: `dev` is exactly as fast as today. It is about
making one build sayable to a stranger and keeping the other one honest.

## The three channels

| | `dev` | `beta` | `stable` |
|---|---|---|---|
| **What it is** | The working tree | A tagged pre-release | A tagged release |
| **Version** | none | `v0.1.0-beta.1` | `v0.1.0` |
| **Server + dashboard** | The Mac, or `deploy-v3.sh --build-only` on the host | The **beta stack** on the host, own ports and own data | The live stack |
| **Firmware** | Flashed over the cable | Signed image on the `beta` OTA channel | Signed image on the `stable` OTA channel |
| **iOS** | A local build | TestFlight internal | TestFlight external, then the App Store |
| **Features** | Everything, including broken | Everything in the registry | **Only `proven`** |
| **Audience** | Me, now | Me, for a week and a drive | Anyone with a dongle |
| **Promise** | None | It starts, and it will be reverted without ceremony | The gates below were met, and the artifacts are checksummed |

`dev` is not a release and has no manifest. The process below is about the other two.

## What a version means

**Per-repository semver, pre-1.0, plus one manifest that names the set.** Each repository tags
its own `vX.Y.Z` when *it* changes; nothing forces a meaningless tag on a repository that did
not move. What makes a *release* is a **train manifest** in this repository naming every
component's tag and commit, so "what is deployed" is one file and not five `git log`s.

- **`0.y.z` until the install path is something a stranger can follow.** There is no API
  stability promise yet and a version number should not imply one. `1.0.0` is for when
  [Phase 32](../ROADMAP.md#phase-32--release-engineering-flash-it-from-a-browser--planned) has
  closed and a dongle can be flashed from a browser.
- **Pre-releases are `-beta.N`**, numbered per train, never reused.
- **The contracts tag is the spine.** A train pins exactly one `contracts-vX.Y.Z`, and every
  component's `contracts.lock` in that train must match it. This is the existing
  [pin discipline](../ROADMAP.md#contract-pins-and-what-a-bump-unblocks) made a release
  condition instead of a weekly PR.
- A train version is **not** a component version. `cairn-0.1.0-beta.1` may contain
  `cairn-vehicle-server v0.3.1` and `cairn-esp32-device-firmware v0.1.0`.

### The train manifest

`releases/cairn-<version>.yaml`, in this repository, one file per train, never edited after it
is published:

```yaml
train: cairn-0.1.0-beta.1
channel: beta
date: 2026-10-12
contracts: contracts-v0.6.0
components:
  - repo: cairn-vehicle-server
    tag: v0.3.0
    commit: 75ed973...
    artifact: cairn-server-linux-amd64.tar.zst
    sha256: ...
  - repo: cairn-esp32-device-firmware
    tag: v0.1.0
    commit: a9d196e...
    artifact: cairn-firmware-merged.bin
    sha256: ...
  # ...dashboard, iOS, modules
features: contracts/release/v1/features.json@contracts-v0.6.0
gates:
  ci: "all six repositories green, run <url>"
  drive: null          # beta does not require it; stable does
  snapshot: null
  rollback: null
  defects: "no open defect at severity: blocker"
supersedes: null
```

A new tool, `tools/cmd/release-check`, validates it and is the thing CI runs:

1. every `tag` resolves, and resolves to the stated `commit`;
2. every component's `contracts.lock` at that commit pins the train's `contracts` tag — unless the
   train records an explicit **held** pin with a reason, because a consumer that cannot yet honour
   a contract must not declare that it does (the iOS app and `contracts-v0.7.0` are the worked
   example: see [the pin table](../ROADMAP.md#contract-pins-and-what-a-bump-unblocks));
3. every artifact named exists on the release and matches its `sha256`;
4. for `channel: stable`, **every gate has non-null evidence**;
5. no feature with `maturity != proven` is enabled in a `stable` train.

It sits beside `pin-dashboard`, `contract-check` and `link-check` and follows the same rule:
with information missing it says so rather than passing.

## The feature registry

**One list, two channels, no forked code.** The thing that decides whether a feature appears in
`stable` is its maturity, and maturity is the roadmap's existing vocabulary:

`contracts/release/v1/features.json`, pinned and fetched like every other contract:

```json
{
  "id": "gps_compare",
  "surface": "dashboard",
  "title": "Device-vs-phone GPS comparison",
  "maturity": "built",
  "evidence": null,
  "defects": [],
  "notes": "Tested against a synthetic track; the live store holds zero phone positions"
}
```

- `maturity: proven` — **in `stable`.** Requires `evidence`: a date and a named observation,
  the same standard the roadmap's [Proven table](../ROADMAP.md#proven-on-hardware-or-in-the-live-deployment)
  already holds. No evidence, no promotion; this is the whole mechanism.
- `maturity: built` or `designed` — **`beta` only.**
- A feature that is `built` and has an open `blocker` defect is off in `beta` too.

It is a contract and not a config file because four repositories consume it and they must agree
about what is on; a private copy per repository is how the thing drifts.

| Consumer | How the channel reaches it | What it gates |
|---|---|---|
| dashboard | `NUXT_CHANNEL=stable\|beta`, registry read at build time, a `useFeature()` composable | Routes **404 in `stable`** rather than rendering a page that half works; nav entries vanish with them |
| server | `CAIRN_CHANNEL` | The API endpoints behind those pages — a hidden page must not be a bypassable one |
| firmware | `-DCAIRN_CHANNEL=` per PlatformIO env | Beta-only capture paths, and the version string the seal stamps |
| iOS | A per-scheme compile-time flag | Beta-only screens; TestFlight internal builds are `beta` |

## The beta stack

One host, two stacks. Parallel systemd units with their own ports, their own data directory and
their own configuration — the live ledger is never a beta deploy's business.

| | prod | beta |
|---|---|---|
| ingest (device mTLS) | `:8443` | `:9443` |
| app listener (`sync/v1`) | `:8444` | `:9444` |
| app behind Tailscale serve | `127.0.0.1:8445` → `:10000` | `127.0.0.1:9445` → `:10001` |
| local admin API | `127.0.0.1:8446` | `127.0.0.1:9446` |
| analytical store | `:8480` | `:9480` |
| web UI | `:3000`, Caddy at `/` | `:3100`, Caddy at `/beta` |
| data | `/var/lib/cairn` | `/var/lib/cairn-beta` |
| config | `/etc/cairn/` | `/etc/cairn-beta/` |
| units | `cairn-{server,tsdb,ui}` | `cairn-{server,tsdb,ui}-beta` |

Four things that are not obvious:

- **Beta goes under `/beta` on `https://cairn.alpina.casa`, not on a second hostname.** A
  passkey is bound to its origin (scheme, host, port), so `cairn-beta.alpina.casa` would mean
  enrolling every passkey a second time and keeping two credential sets straight. A path keeps
  one origin and one set. The Nuxt side needs `NUXT_APP_BASE_URL=/beta`; `NUXT_AUTH_ORIGINS`
  does not change.
- **Beta is seeded by restore, not by driving.** `deploy/snapshot.sh` and
  `deploy/restore-test.sh` already take a cold snapshot of prod and boot it in a scratch stack —
  that is exactly a beta seed, so beta runs on real shapes of data without the car pointing at
  it.
- **The car must never upload to beta by accident, and the reverse matters more.** The device
  reaches the server by a pinned CA and a configured host; beta gets its **own** device
  enrolment and its own receipt seed. A receipt from the beta stack must not satisfy a prune on
  the real dongle — the prune is the one irreversible thing in the system.
- **The build is capped or it wedges the host.** `deploy-v3.sh` already uses `nice -p 4`
  because the host also runs PostgreSQL and six Podman runners. Two stacks building does not
  change the budget.

## The promotion gates

`beta` → `stable` requires all five, recorded as evidence in the train manifest.
`release-check` refuses a stable train with a null gate.

| Gate | What counts | Today |
|---|---|---|
| **CI green** | Every repository in the train green on the Podman runners, including the server's `tests/interop/run.sh` and the contract and pin checks | Available now |
| **One real drive** | A drive carried end to end on the beta stack: trip visible, receipts verified, **and the date correct** — which closes the open verification on [firmware #38](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/38), [#37](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/37) and [#39](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/39) at the same time | One drive away |
| **Snapshot restore-tested** | `deploy-v3.sh --snapshot` before the prod install: a snapshot that does not restore and boot stops the deploy | Available now |
| **Firmware rollback on hardware** | OTA install, rollback, and power-loss-during-update exercised on a dongle ([firmware #11](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/11)) | **Not available — see below** |
| **No blocking defect** | No open defect at `severity: blocker` in the [defect table](../ROADMAP.md#open-defects) | Needs the severity column, then a triage |

### Severity, and what blocks a public release

The defect table gains a severity, because "no blocking defect" needs a word that was never
written down. Three values:

| Severity | Means |
|---|---|
| `blocker` | `stable` cannot ship with it open |
| `beta-ok` | Ship in `beta`; the feature it touches stays out of `stable` |
| `known` | Documented, shipped, not fixed |

The first triage, from the defects open on 2026-10-10:

| Issue | Severity | Why |
|---|---|---|
| firmware [#5](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/5) — weak BLE passkey, no session authentication | **blocker** | Pairing is Just Works today for test convenience, and `ble_companion.cpp` says in a comment to restore MITM + DISPLAY_ONLY before shipping. A public release makes a security claim; this one is not met |
| firmware [#6](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/6) — a Wi-Fi password may stay readable in flash | **blocker** | eFuse flash encryption is ruled out on this silicon, so this needs rotation or application-layer handling, not a chip feature |
| firmware [#11](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/11) — OTA never run on hardware | **blocker** | It is itself a gate |
| firmware [#38](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/38), [#37](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/37), [#39](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/39) | **blocker**, cleared by the drive gate | Built and unconfirmed. A logger whose timestamps were wrong last week does not go to strangers on a host test |
| firmware [#31](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/31) — Wi-Fi cannot associate while BLE is up | `beta-ok` | Wi-Fi is gated off in production and LTE carries the data. The Wi-Fi uplink is therefore a **beta** feature, stated plainly rather than quietly broken |
| firmware [#34](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/34) — no Wi-Fi slot announced, check-in is a no-op | `beta-ok` | Both are unreleased `ble/v1` fields |
| firmware [#32](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/32), [#33](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/33) — `CAIRN_MAX_CHUNKS`, DRAM shim | `known` | Documented constraints with a stated rule (raise freely, lower never) |
| server [#42](https://github.com/ParkWardRR/cairn-vehicle-server/issues/42) — `cairn-phone` nil dereference | `beta-ok` → **blocker** if `cairn-phone` ships | It is the reference client. Publishing it as a tool makes its crash a shipped crash |
| The seven stranded v2 bundles | `known` | 4.76 MB nothing can move. The prov-console drop command should land before `stable`, so a new owner never meets them |

### The one-dongle problem

The rollback gate cannot be met as things stand. There is **one** dongle, no spare, and the
roadmap already records that no sacrificial unit was ever bought
([Phase 34](../ROADMAP.md#phase-34--chip-hardening-on-replacement-hardware--planned-gated)).
Exercising power-loss-during-update on the only unit in the car is the one test that can brick
the project.

Two honest ways out, and this is a decision rather than an engineering task:

1. **Buy a second dongle.** It unblocks the rollback gate, and it also unblocks Phase 34, which
   has been stuck behind the same missing unit since it was written. The cheaper of the two by
   a wide margin.
2. **Ship `stable` with OTA as a beta-only feature**, and say in `INSTALL.md` that a stable
   install is flashed over a cable. The gate is then "OTA is not in `stable`" rather than "OTA
   was tested", which is true and checkable. It makes the first public release worse in exactly
   the way Phase 32 is meant to fix.

Recommendation: buy the unit. Until one of the two is chosen, **no `stable` train can be
cut** — and that is the process working, not the process being in the way.

## What has actually been released

A **component** release is one repository tagging and shipping itself. A **train** is a set that
was released together, with a manifest and the gates met. There is one component release and no
train yet, and the difference matters: the gates above are a property of a train.

| Component | Version | Channel | Date | Commit | Where |
|---|---|---|---|---|---|
| [iOS app](https://github.com/ParkWardRR/cairn-ios-companion-app) | **0.1.0 (11)** | `stable` | 2026-10-10 | `75f545d` | TestFlight, uploaded through Xcode's account session. Internal testers get it on processing; external distribution needs the build assigned to the external group in App Store Connect, which requires Beta App Review |
| [server](https://github.com/ParkWardRR/cairn-vehicle-server) | **v0.1.0** | `stable` | 2026-10-10 | `fc86c63` | Deployed to the live host from a clean export of the tag, after a snapshot that restore-tested. `/healthz` and `/v1/health` both report `version v0.1.0, channel stable` |
| [firmware](https://github.com/ParkWardRR/cairn-esp32-device-firmware) | **v0.1.3** | `stable` | 2026-10-10 | `6d42713` | Flashed to the car's dongle over the cable, NVS backed up before each flash. Four releases that evening: v0.1.0 the identity, **v0.1.1** a failed pairing recovering its stale bond inside the post-boot window, **v0.1.2** the last `WRITE_AUTHEN` that Just Works can never satisfy, **v0.1.3** a failed pairing that reports the security state it reached. Each one verified on the unit, which is what the channel is for |
| dashboard, modules | — | — | — | — | Never tagged. Deployed from a working tree |

The dashboard is the one deployed component with no version, and it is blocked rather than
skipped: its `bootId → tripId` rename is uncommitted across twenty files, and nothing can be
tagged across a half-landed rename (Phase 36 item 1).

### What the three releases do and do not prove

| Proven on 2026-10-10 | Not proven |
|---|---|
| Every component states its channel, version and commit, and the chain agrees on `v0.1.x` | **A bundle has not moved between this firmware and this server.** BLE pairing does not complete; the dongle-side lockout was fixed in v0.1.1 and the remaining fault is on the central — see [firmware #44](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/44) |
| The public Funnel ingress reaches the server: from outside the tailnet, `cairn.fable-minnow.ts.net:8443` answers with the real `CN=cairn.alpina.casa` certificate from `Cairn Private CA` | The LTE uplink with these builds. It is scheduled on the device, not driven from here |
| The store serves real data through the new trip keying: 2 trips, 1,256 positions, 572 OBD rows, 3 metric samples | A drive **on these builds**. One happened on 2026-10-09 and proved the trip dates, but it predates both releases and left OBD density open — so Phase 27's gate is still open and no release has met it |
| The snapshot restore-tested before the install, and the install kept the previous binaries as `.prev` | A rollback. The mechanism exists and has never been used in anger |

The iOS app is the first component in the project to carry a version, a channel and its own
commit, and `scripts/release-testflight.sh` in that repository is the first release path that
refuses a dirty tree. The shipped binary reads back `CairnChannel = stable` and
`CairnCommit = 75f545d`; a build that cannot say what it is reads as `dev` and never as
`stable`.

**It went out on `stable` at the owner's instruction, ahead of the gates**, which is recorded
rather than smoothed over: no train manifest existed to attach evidence to, and the drive and
rollback gates are unmet. What it does carry is a green test suite (340 tests), CI green on the
macOS runner, and an identity that can be traced back to a commit — which is why a later train
can state honestly what was in it.

## Rolling back

| Component | How | Proven? |
|---|---|---|
| server | `deploy-v3.sh` keeps the previous binaries as `<name>.prev`; restore and restart | Mechanism exists, never used in anger |
| server state | The pre-install snapshot, which was restore-tested before the install touched anything | Proven as a CI job and as a deploy step |
| dashboard | Re-deploy the previous train's tag; `deploy-ui.sh` builds from a clean export of a ref | Mechanism exists |
| firmware | A/B partitions and the bootloader's rollback | **Written, never run on hardware** — the gate above |
| iOS | TestFlight: previous build, re-enable | Trivial |

`stable` must refuse to deploy a dirty tree. Today `deploy-v3.sh` rsyncs the working tree,
which is right for `dev` and wrong for anything with a version number: a release you cannot
reproduce is not one.

## What `stable` needs that `beta` does not

`stable` is public from the first tag, so a release is not finished when the code is:

- **Artifacts, built in CI, checksummed, attached to the GitHub release.** A merged firmware
  image and manifest ([firmware #24](https://github.com/ParkWardRR/cairn-esp32-device-firmware/issues/24)),
  a server tarball, a dashboard build. Nothing published today.
- **`INSTALL.md` true for a stranger.** It currently opens "there are no prebuilt release
  artifacts yet, for any part", which the first `stable` tag makes false
  ([front door #16](https://github.com/ParkWardRR/cairn-driving-log-selfhosted/issues/16)).
- **No host-specific assumption in anything shipped.** The deploy scripts already keep real
  host names out of the repository; the same standard has to hold for certificate paths, the
  Tailnet name and the `cairn.alpina.casa` origin.
- **A changelog per component**, written from the train manifests, which is the one document a
  stranger reads before upgrading.
- **The two security blockers above closed**, because a public release is where a security claim
  stops being internal.

## Deliberately not in this process

- **No release branches.** `main` plus tags. Six repositories and one maintainer; a branch per
  channel is a merge burden bought with nothing.
- **No fork of the code per channel.** Divergence is what the feature registry exists to avoid.
- **No time-based soak.** The gates are evidence, not patience. A train waits for a drive, not
  for a Tuesday.
- **No automatic promotion.** `release-check` can say a train is *eligible*. Cutting `stable`
  stays a decision someone makes.
