# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A KiCad 10 hardware project (`unila.*`), not software. It is a DEC double-height flip-chip module that does two jobs:

1. **Unibus terminator** compatible with the DEC M930: 56 bus lines each have 180 Ω to +5 V (discrete 1206, R59–R114, pin 1 on `+5V`) and 390 Ω to GND (14 × 4-element 0603 arrays RN1–RN14, pins 5–8 on `GND`). +5 V comes from fingers AA2/BA2 directly (no TERMPWR jumper; the board is always terminating). Decoupling: 7 × 100 nF + 1 × 22 µF on +5 V. PA0/PA1 are probed but not terminated, as on the M930.
2. **Probe adapter for an HP 16702A frame with a 16556A state/timing card**: four 2×20 pod headers (J2, J3, J4, J8) that take the HP 40-pin pod cable directly. Each probed signal goes through its own HP isolation network (91 kΩ ∥ 8.2 pF, 0603) next to the edge fingers. J8 (pod 4) is optional and intended to stay unpopulated unless needed.

It works in both normal Unibus and MUD (Modified Unibus) slots. Pins that are GND on normal Unibus carry MUD signals here, and the 20 V/BAT pins are left open. J6/J7 strap NPG/BG7 in from AU1/AV1 and must be left open in MUD slots (20 V would reach the analyzer).

## Editing rules

- **Never edit `*.kicad_sch`, `*.kicad_pcb`, `*.kicad_pro` with text tools.** All modifications go through the Konnect MCP server (configured in `.mcp.json`; see the `konnect` skill). If Konnect isn't connected, stop and say so rather than falling back to file edits.
- KiCad is often open on this project (`~unila.kicad_pro.lck`); the user edits interactively between requests, so re-export before every review instead of trusting earlier results.

## Checking the design (read-only, via kicad-cli)

Write outputs to the scratchpad, not the repo.

```sh
kicad-cli sch export netlist --format kicadxml -o OUT/unila.xml unila.kicad_sch
kicad-cli sch erc -o OUT/erc.rpt unila.kicad_sch
kicad-cli pcb drc --schematic-parity --severity-all -o OUT/drc.rpt unila.kicad_pcb
kicad-cli pcb render --side top|bottom --width 1400 --height 2000 -o OUT/x.png unila.kicad_pcb
```

The `pcbnew` Python module (KiCad 10) is available for PCB queries (track widths, zones, footprint positions).

Known benign noise: ERC `lib_symbol_mismatch` and one `unconnected_wire_endpoint` per 8.2p capacitor (58 wire stubs). DRC with parity should be fully clean (0 violations, 0 unconnected). ERC does **not** catch the errors that have actually occurred here (misnamed labels splitting a net, networks fanned out to two pods), so always verify from the netlist.

## Netlist invariants to verify

- Bus nets are named `/<SIGNAL>_L` or `/<SIGNAL>_H` and connect to `J1` (`CONN-DEC-DOUBLE`, pins `AA1`…`BV2`).
- Every bus net except `/PA0_L`/`/PA1_L` has exactly one 180R (other end on `+5V`) and one 390R array pin (array pins 5–8 on `GND`).
- Probe-side nets are prefixed `Q` (e.g. `/QD00`, `/QBR7_L`). Each `Q` net must have exactly one 91k, one 8.2p and **one** pod pin. One isolation network must never drive two analyzer inputs, and no bus net may connect to a pod pin directly.
- Pod pinout used: pin 3 = clock, pins 7…37 (odd) = D15…D0, all even pins GND, pins 1/5/39 unconnected (1 and 39 are +5 V on the HP pod).
- On the 16556A, clock inputs (J/K/L/M) can also be recorded as data in timing and state mode, except in Range terms. A single card's four pods share one clock system.

## PCB notes

- Edge-finger area must be kept free of copper: the rule area there must be on F.Cu/B.Cu (not a courtyard layer) with tracks, vias and pours disallowed.
- GND is poured on both layers (`GND_F`, `GND_B`) and stitched with a via row at y = 94 mm; keep stitching vias clear of the mounting holes H1–H4 (y = 95.25 mm). +5 V runs on a F.Cu bar zone (`+5V_BAR`) plus 1 mm tracks. Min track 0.2 mm, netclass clearance 0.2 mm, copper-to-edge 0.5 mm.
- Konnect cannot delete vias, move footprint text fields, or set copper-to-edge clearance; ask the user to do those in pcbnew. Run DRC with `--refill-zones` (or Konnect `run_drc` with `refill_zones`), otherwise stale fills hide pour problems.
- `production/` (BOM, positions, gerbers) is gitignored and regenerated from KiCad. Check it's current before any fab order.
