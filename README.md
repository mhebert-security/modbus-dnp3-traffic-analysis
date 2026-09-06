# modbus-dnp3-traffic-analysis

> The write that can stop a line.

Modbus and DNP3 still move control traffic in the clear, built for a time
when the network was a locked room. This project builds a baseline of normal
traffic on those links and flags the message that does not belong, so the
anomaly shows up while it is still a packet and not yet an outage. The
detection starts from the physics of the place, not from a list of known
attack names.

## Status

Building. This repo is the plan and the scaffold. The first artifact is a
reproducible lab baseline, and it is not written yet. There are no captures
and no detection rules here on purpose: a baseline that cannot be rebuilt is
a story, not evidence. The work lives in `docs/` until the lab runs.

## Why these protocols

Neither Modbus nor DNP3 authenticates or encrypts by default. They were
designed for serial links inside a locked room, and decades later they still
drive pumps, breakers, and valves on networks that are no longer locked. A
single write to the wrong coil can stop a physical line, so the traffic that
reaches a controller deserves the same scrutiny as the traffic that reaches
a database.

Normal is learnable. A link that polls a handful of unit identifiers on a
fixed cadence has a shape, and a shape can be described. The anomaly is
anything that does not fit: a new unit, an unscheduled write, a function
code that never appears in the baseline. That is the detection model this
project is building, documented per scenario rather than tuned in the dark.

## Plan

Phases are tracked in [docs/project-plan.md](docs/project-plan.md). The
short version:

1. Stand up a reproducible Modbus and DNP3 lab and capture normal traffic.
2. Write the baseline down as concrete expectations, not vibes.
3. Detect and document anomalies against that baseline, one scenario at a
   time.
4. Let the findings feed the OT and ICS CVE research roadmap.

## Repository layout

```
docs/      Plans, baseline notes, and anomaly documentation
lab/       pymodbus and DNP3 simulation config and capture scripts
analysis/  Capture analysis and baseline tooling
```

`lab/` and `analysis/` are empty until the first reproducible run exists.

## Author

Matt Hébert · [mhebert.dev](https://mhebert.dev) · matt@mhebert.dev
