# Proposed A-GRA Addition — Mission/Flight-Autonomy Assurance Annex

**Proposal v0.1 · 14 JUN 2026 · Istari Digital (istari-digital-internal fork of `open-arsenal/a-gra`)**
**Against:** A-GRA ASK 5.0a (AFLCMC, 21 APR 2026)
**Contact:** cbenson@istaridigital.com · upstream feedback: aflcmc.aflcmc.agra@us.af.mil

> Status: working proposal for discussion. Nothing here changes the ASK 5.0a interface
> definitions; it proposes a *new, complementary* assurance volume + two conformance-profile
> additions that sit alongside the existing Interface Volumes.

---

## 1. Why — the gap

A-GRA 5.0a rigorously standardizes the **interfaces** between Mission Autonomy (MA) and the
rest of the system, and is explicit about the MA/FA boundary: *"FA always retains control but
'listens' to MA based on mission state"* (Start Here Guide; Vehicle Interface Volume). The
Vehicle Interface even names the FA's responsibilities "with regard to safety-criticality."

But ASK 5.0a stops at the interface. It does **not** define:

1. **How an ACP's autonomy functions get an assurance level.** "FA retains control / safety-
   criticality" is asserted, never substantiated. There is no method to allocate a Design
   Assurance Level (DAL, per ARP4754A/DO-178C) to MA vs FA functions, and no link from a
   platform safety analysis to those levels. Two ACPs could both be "L1 compliant" with wildly
   different actual assurance.
2. **A safety/off-nominal conformance test.** The L1 Compliance Document's harness is
   *sequence/message-level* ("Interactions ... must use the correct messages and populate
   required fields with valid data"). It verifies that messages are well-formed — not that the
   FA does the safe thing when the link drops, the battery is critical, or MA issues an
   out-of-envelope command. For "FA always retains control" to be more than a slogan, it must
   be *testable*.
3. **Provenance for the values an FA advertises.** `MA_FlightCapabilityMT` carries
   `AirspeedLimit`, `AccelerationLimit`, `OrientationRateLimits`, and Waypoint/Curve-following
   performance profiles (L1 Compliance Doc Tables A-4-1..A-4-10). The schema does not bind those
   numbers to substantiating analysis. An MA cannot tell whether an advertised limit is a
   flight-validated value or a placeholder — yet it plans against them.

These three gaps are exactly where a digital-thread / airworthiness pipeline has something to
contribute. The proposals below are each backed by a *working implementation* we run against a
real ~67 kg fixed-wing UAS design (see §5).

---

## 2. Proposal A — Autonomy Assurance Volume (DAL allocation tied to a platform FMECA)

Add an **Assurance Volume** (peer to the six Interface Volumes) that standardizes how an ACP
allocates assurance levels to its autonomy functions.

- **Severity-driven DAL.** Each MA and FA function is assigned a DAL (A–E) from the severity of
  the failure condition its malfunction causes, per the standard ARP4754A/DO-178C mapping
  (5 Catastrophic→A … 1 No-effect→E).
- **Traceable to a platform FMECA.** Severities are not asserted; they are taken from a
  MIL-HDBK-1629A FMECA of the platform, and each function names the failure modes it
  contributes to. This makes the allocation auditable and prevents the "everything is DAL A" or
  "everything is DAL D" failure modes.
- **MA/FA split is the natural assurance boundary.** A-GRA already separates the safety-critical
  FA from mission MA; the Assurance Volume formalizes that FA functions inherit the highest DALs
  (the FA *retains control*) while MA functions are assured to the level their worst credible
  effect demands. This lets best-of-breed MA be procured at a *known, bounded* assurance level —
  directly serving the "Mission Autonomy Sold Separately" strategy.
- **Honest debt, not a tick-box.** The Volume distinguishes *allocation* (what level each
  function must meet) from *evidence* (DO-178C life-cycle artifacts). An ACP reports its
  allocation **and** its evidence gap, so integrators see the real assurance posture.

*Worked example included (§5): an 8-function allocation for our UAS FA — attitude
stabilization / EKF / propulsion control / failsafe logic at DAL A, stall protection DAL B,
nav/telemetry DAL C, logging DAL E — every level traced to FMECA modes.*

## 3. Proposal B — Vehicle-Interface Failsafe-Coverage conformance profile

Add a **safety-conformance profile** to the L1 compliance harness for the Vehicle Interface
that makes "FA always retains control" testable. The profile asserts that for every off-nominal
trigger, the FA exhibits a defined safe response **regardless of MA commands**:

| Trigger | Required FA behaviour (example) |
|---|---|
| Lost C2 / RC link | Defined failsafe (e.g. RTL after hold) — independent of MA |
| Low / critical battery | Staged failsafe → RTL → land |
| GPS / EKF loss | Revert to a degraded-but-safe mode; reject MA nav commands that require lost state |
| Geofence breach | Fence action regardless of MA route |
| Out-of-envelope MA command | FA validates and **rejects** (the VI already defines accept/reject) |

The harness would drive these conditions and confirm (a) the FA's `MA_FlightCommandStatusMT`
rejects unsafe commands, and (b) the documented failsafe fires. Coverage = triggers-with-a-
defined-response / total triggers; conformance requires 1.0. This upgrades the compliance story
from "messages are well-formed" to "the FA is safe when things go wrong."

## 4. Proposal C — FlightCapability provenance binding

Add an optional **provenance attribute** to the `MA_FlightCapabilityMT` capability fields (and,
more broadly, to advertised performance values): a small structure naming the *source of
substantiation* (e.g. analysis ID, value class: MEASURED / VALIDATED / DERIVED / PLACEHOLDER)
and optionally a confidence/last-validated stamp. An MA consuming `AirspeedLimit` /
`AccelerationLimit` / `OrientationRateLimits` can then distinguish a flight-validated envelope
from a paper estimate and plan accordingly (e.g. apply margin to DERIVED limits). This is the
"the thread is the source of truth" principle applied to the interface: every number an ACP
advertises carries where it came from.

*(Candidate for the future L2 component-level interfaces too: a units + provenance convention
for intra-MA capability data.)*

---

## 5. Reference implementation (evidence this is buildable, not just paper)

These proposals are extracted from a working digital-thread UAS-design pipeline that runs all
three against a real design point. The artifacts:

- **DAL allocation engine** — `scripts/software_assurance.py`: derives required DAL per function
  from FMECA severity; emits the allocation + the honest assurance-debt count. Policy in a
  rank-0 decision record `DESIGN-SWAL-A`.
- **FMECA engine** (the severity source) — `scripts/fmeca.py` (MIL-HDBK-1629A), policy
  `DESIGN-FMECA-A`.
- **A-GRA conformance posture + VI FlightCapability field-mapping** — same module's
  `agra_conformance` block; maps `MA_FlightCapabilityMT` fields → substantiating analysis
  (AirspeedLimit/AccelerationLimit ← V-n diagram), reports coverage + the OrientationRateLimits
  gap. Policy `DESIGN-AGRA-A`.
- **Failsafe-coverage check** — the `failsafe_triggers` matrix in `DESIGN-SWAL-A`, graded to a
  coverage ratio (the Proposal-B profile in miniature).
- Each is gated by a deterministic test + a semantic review guide and is carried in a digital
  thread with per-value provenance "atoms" — i.e. Proposal C already in practice internally.

We are happy to contribute these as a starting point for an A-GRA Assurance Volume and to
work the details with the A-GRA maintainers.

---

### Changelog
- **v0.1 (14 JUN 2026)** — initial proposal: Assurance Volume (DAL-from-FMECA), VI
  failsafe-coverage conformance profile, FlightCapability provenance binding.
