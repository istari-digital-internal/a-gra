# Istari Contributions to A-GRA

This folder holds Istari Digital's proposed additions to the Autonomy Government Reference
Architecture (A-GRA). It is maintained on the `istari-digital-internal/a-gra` fork; nothing
here modifies the upstream ASK 5.0a interface definitions — these are *complementary* proposals
for discussion with the A-GRA maintainers (aflcmc.aflcmc.agra@us.af.mil).

## Contents

- **`A-GRA_Assurance_Annex_Proposal_v0.1.md`** — proposes three additions that sit alongside the
  existing Interface Volumes:
  1. an **Autonomy Assurance Volume** — DAL (DO-178C/ARP4754A) allocation for MA/FA functions,
     traced to a platform FMECA (MIL-HDBK-1629A);
  2. a **Vehicle-Interface failsafe-coverage conformance profile** — makes "FA always retains
     control" testable (off-nominal trigger → defined safe response, MA-command rejection);
  3. a **FlightCapability provenance binding** — so an MA knows whether an advertised
     `MA_FlightCapabilityMT` limit is flight-validated or a placeholder.

Each proposal is backed by a working reference implementation in Istari's digital-thread
UAS-design pipeline (DAL engine + FMECA + A-GRA conformance posture + failsafe-coverage check),
run against a real ~67 kg fixed-wing UAS design point.

## Why

A-GRA 5.0a standardizes the autonomy *interfaces* and is explicit about the safety-critical
MA/FA boundary, but does not define how an ACP's assurance level is *established*, *tested under
off-nominal conditions*, or *substantiated per advertised value*. These proposals fill that gap
from the airworthiness/digital-thread side.
