# LoRa position beacon v1 (reserved)

Reserved for a small, sealed position beacon sent over LoRa (915 MHz US ISM) by the trail
device, so that someone at home, or a companion on the trail, can see where the device was last
heard. Planned in [docs/trail-device-plan.md §9](../../../docs/trail-device-plan.md#9-lora-off).

> **Deferred, 2026-10-25.** The owner chose not to use the trail device's LoRa radio at all, and to
> hold it powered off to save battery. This directory stays reserved so the rules below are not
> lost; nothing is scheduled.

**Nothing is designed here.** A position broadcast over radio is a location leak to anyone with a
receiver, so the threat model gets a section for it first — what is sent, to whom, sealed to which
key, how a replayed or forged beacon is recognised — and only then a format.

Three rules are fixed already, because the rest of Cairn depends on them:

- **A beacon is provisional.** It is never written to the analytical store as a position and
  never joined into an activity. The bundle is the record; a beacon is a *last seen* marker with
  its age.
- **A beacon authorises nothing.** No acknowledgement received over LoRa can cause a bundle to be
  pruned — the same reason a digest can never be the cellular path.
- **Off by default**, and the device's display says when it is on.

- **Producer:** the trail device ([cairn-trail-device-firmware](https://github.com/ParkWardRR/cairn-trail-device-firmware)).
- **Consumer:** a home LoRa gateway forwarding to the server over the tailnet.
- **Status:** reserved, deferred.
