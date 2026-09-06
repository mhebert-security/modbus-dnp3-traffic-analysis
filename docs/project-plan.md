# Project plan

Honest tracking of where this project stands. Anything marked done has an
artifact in the repo that rebuilds it; nothing is marked done on intent.

## Phase 0. Reproducible lab baseline

Goal: normal Modbus and DNP3 traffic I can generate, capture, and rebuild
from a script.

- [ ] Modbus master and slave simulation with pymodbus on loopback
- [ ] A DNP3 outstation and master on loopback
- [ ] Capture scripts (tshark) that record a defined run
- [ ] Lab notes that let a clean machine reproduce the run

## Phase 1. Normal as a written model

Goal: turn the capture into expectations a rule can check.

- [ ] Baseline of function codes and unit identifiers in the run
- [ ] Baseline of coil and register addresses touched
- [ ] Cadence model: what polls when, and how often
- [ ] Written profile of the normal link, not a scatter of packets

## Phase 2. Anomaly detection and documentation

Goal: flag what does not fit the profile, and explain each case.

- [ ] Detect a write from a unit identifier outside the baseline
- [ ] Detect an unscheduled write or read outside the cadence window
- [ ] Detect a function code the baseline never uses
- [ ] One writeup per scenario: the packet, the expectation, the why

## Phase 3. DNP3 depth

Goal: apply the same method where DNP3 adds structure.

- [ ] Application layer confirmations and object addressing in the baseline
- [ ] Unsolicited responses as a normal shape, not an error
- [ ] DNP3 specific anomaly scenarios

## Phase 4. Findings outward

Goal: the documented anomalies feed the OT and ICS CVE research roadmap,
and each scenario is honest about what it proves.

## Notes

The detection starts from the physics of the place. A write that changes a
setpoint is normal at shift change and alarming at 3 a.m. on a Sunday. The
baseline exists to make that distinction concrete instead of intuitive.
