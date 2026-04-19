# Beckhoff CX5140 Cereal Box Filler — PLC Sequential Control

**A step-based ladder-logic control system for an automated cereal box filling station, implemented on a Beckhoff CX5140 embedded PC with TwinCAT 3 and demonstrated on real lamp-and-switch hardware.**

---

## Project Overview

This project is a graduate coursework assignment for *EE 5340 — Programmable Logic Controllers*, Lab Exercise 6. The objective was to gain working familiarity with Beckhoff's Windows-based PLC platform and the TwinCAT 3 development environment, and to use that environment to design, download, and commission a sequential control program for a realistic industrial process: a cereal box filling station.

The lab is structured to build up gradually — starting from basic combinational logic (a series AND circuit and a parallel OR circuit), adding a TON timer exercise, and culminating in a full machine-control program that sequences a conveyor motor, a fill solenoid, and a pair of sensors across five discrete production steps with stop / start pause-resume behavior and a manual reset. The deliverable is a running ladder program, commissioned against physical lamps and switches wired into the CX5140's digital I/O terminals.

## Learning Objectives

The lab was designed to develop working competence in PC-based PLC programming on the Beckhoff platform, which differs in several important respects from the Rockwell and Siemens ecosystems. Over the course of the exercise the objectives were to bring up a new TwinCAT XAE project from scratch, connect to the CX5140 target over Ethernet via a Secure ADS route, scan and commission the EtherCAT terminals on the DIN rail, define IEC 61131-3 compliant variables, build and download a ladder program, link declared variables to physical I/O channels, and verify correct operation both on-line (with live tag monitoring) and off-line (using TwinCAT's force function to simulate input conditions). The cereal box filler deliverable was the capstone exercise that exercised all of those skills together in a single sequential-control application.

## System Architecture

The hardware cell is organized around a **Beckhoff CX5140 embedded PC** running TwinCAT 3 runtime, mounted on a DIN rail with a series of EtherCAT bus terminals ("Terms" in the TwinCAT project tree). Digital input and output terminals are scanned into the project automatically and addressed by their IEC physical address format. Five physical pushbuttons and switches and five indicator lamps are wired onto a bench board to simulate the plant signals of the cereal filler.

The I/O map used throughout the lab is:

| Signal | Direction | IEC Address | Role |
|---|---|---|---|
| `Start_PB` | Input | `%IX40.0` | Start pushbutton — latches the `RUN` bit |
| `Stop_PB` | Input | `%IX40.1` | Stop pushbutton — clears `RUN`, pauses the sequence |
| `LS` | Input | `%IX40.2` | Limit switch — box in fill position |
| `WT_SENSE` | Input | `%IX40.3` | Weight sensor — box full |
| `Reset_PB` | Input | `%IX40.4` | Manual reset — clears all steps |
| `M1` | Output | `%QX26.0` | Conveyor motor |
| `M2` | Output | `%QX26.1` | Continuous drive motor |
| `FILL` | Output | `%QX26.2` | Fill solenoid |
| `LA4` | Output | `%QX26.3` | Indicator lamp 4 |
| `LA5` | Output | `%QX26.4` | Indicator lamp 5 |

On the software side, the project was built as a **Standard PLC Project** in TwinCAT XAE. The default Structured Text POU was replaced with a new **Program POU named `Series_Circuit`** authored in **Ladder Diagram (LD)**, added to `PlcTask` for cyclic execution. Variables were declared as `AT %I*` or `AT %Q*` so that the physical links could be set up in the TwinCAT I/O tree after the first successful build, giving a clean separation between logic and I/O mapping.

## Cereal Box Filler — Process Description

The cereal filler operates as a five-step sequence. On initial start-up, conveyor motor `M1` runs until an empty box reaches the fill position, which is detected by limit switch `LS`. The machine then pauses for **2 seconds** to let the box settle before opening the `FILL` solenoid. Fill continues until the weight sensor `WT_SENSE` registers a full box, at which point the solenoid closes and the sequence waits another **3 seconds** before restarting the conveyor to eject the full box and bring the next empty one into position. Motor `M2` runs continuously whenever the machine is in the `RUN` state. If `Stop_PB` is pressed at any time, the machine pauses at the current step — motors and fill solenoid are dropped but the internal timers continue to run, so that pressing `Start_PB` resumes the suspended step cleanly. A separate `Reset_PB` clears all steps so that the next `Start_PB` press begins the cycle from scratch, with no box assumed in position; per the specification the reset input is ignored while the machine is actively running.

## Program Design — Step-Based State Machine

Rather than using nested latches or timer chains, the program was organized as a classic **five-step sequential function model** implemented in ladder. Five Boolean flags (`STEP1`, `STEP2`, `STEP3`, `STEP4`, `STEP5`) mark which phase the machine is in at any given scan, a master `RUN` bit gates all outputs, and transitions between steps are implemented as set/reset pairs driven by sensor inputs and timer `.Q` outputs.

The program layout is:

1. **Run latch.** `Start_PB` latches `RUN`; `Stop_PB` breaks the seal, which gives the specified pause behavior automatically — when `RUN` drops, all outputs go low but the step bits and timers are preserved, so pressing `Start_PB` resumes exactly where the sequence left off.
2. **No-steps-active detection.** A series of negated `STEP1..STEP5` contacts produces a helper bit used when initializing the first step.
3. **STEP1 → STEP2 on limit switch.** When `LS` asserts (box in position), `STEP1` is reset and `STEP2` is set.
4. **STEP2 → STEP3 after 2 s.** `TON_1` with a `T#2s` preset runs while `STEP2` is active; when `TON_1.Q` asserts, `STEP2` is reset and `STEP3` is set, gating the fill.
5. **STEP3 → STEP4 on weight sensor.** When `WT_SENSE` asserts (box full), `STEP3` is reset and `STEP4` is set, dropping the fill solenoid.
6. **STEP4 → STEP5 after 3 s.** `TON_2` with a `T#3s` preset runs while `STEP4` is active; when `TON_2.Q` asserts, `STEP4` is reset and `STEP5` is set, restarting the conveyor.
7. **STEP5 → STEP1 when box clears.** When `LS` deasserts (box has moved off the sensor), `STEP5` is reset and `STEP1` is set, beginning the next cycle.
8. **Reset handling.** `Reset_PB` (with `RUN` off) resets all five step bits, clearing the machine state.
9. **`M1` conveyor output.** `M1` is driven by `(STEP1 OR STEP5) AND RUN` — the conveyor runs during the "search for empty box" phase and the "eject full box" phase.
10. **`M2` continuous output.** `M2 = RUN` — runs continuously while the machine is enabled.
11. **`FILL` output.** `FILL = STEP3 AND RUN` — the solenoid only energizes during the fill phase.

This structure is directly analogous to a PackML-style execute sub-state machine, implemented at the scale of a single unit procedure. Because `RUN` globally gates every output but does not reset the step bits or timers, pause-resume falls out of the design naturally.

## TwinCAT Commissioning Workflow

The lab also served as a walk-through of Beckhoff's commissioning tooling, which differs noticeably from other PLC vendors. The commissioning path followed was:

The development laptop was connected to the CX5140 over Ethernet with the controller's IP address (`131.151.52.144`) entered manually into TwinCAT's "Choose Target" dialog. A **Secure ADS** route was added with the controller's device fingerprint copied via right-click → "Copy Fingerprint to Clipboard" and pasted into the "Compare with" dialog to prevent man-in-the-middle attacks. Default credentials were used for initial login.

With the route established, TwinCAT was placed in Config mode and the I/O tree was populated via **right-click Devices → Scan**. Every EtherCAT terminal on the DIN rail appeared in the project automatically, expandable to individual channels — a workflow similar to Rockwell's RSLogix Ethernet discovery but markedly cleaner because EtherCAT carries device identification natively.

After defining the program and its variables in ST syntax (with `AT %I*` / `AT %Q*` placeholder addresses), an initial build was triggered. TwinCAT returns a warning that variables are not linked to physical I/O; links were then added manually by right-clicking each declared variable in the `PLCTask Inputs` / `PLCTask Outputs` tree and using "Change Link…" to bind them to the appropriate `IX 40.x` or `QX 26.x` channel on `Term 2`.

The final deployment steps were to rebuild the solution, click **Activate Configuration** with **Autostart** checked, accept the trial license prompt (standard for the academic runtime), and log in with **PLC → Login**. Once on-line, energized contacts and coils highlight blue in the ladder view, giving direct visual feedback while the machine runs. For testing without wiggling physical switches, TwinCAT's **Force** feature was used: navigating to `I/O → Devices → Device 1 → Term 1 → Term 2`, expanding a channel, and using the Online tab's "Force" button to drive a Boolean to 0 or 1.

## Key Technical Accomplishments

The completed lab demonstrates a full top-to-bottom ladder-logic project on a modern PC-based PLC, including Ethernet-based commissioning with a secure ADS route, automatic EtherCAT I/O discovery, IEC 61131-3 compliant variable declaration with late-bound I/O mapping, a multi-rung ladder program organized around a clean five-step state pattern, correct use of TON timers for inter-step dwells, and proper implementation of pause-resume semantics via a single gating bit that preserves machine state. On-line verification was performed against real lamps and switches, with additional testing via TwinCAT's force function for inputs that aren't physically available on the lab bench.

## Skills Demonstrated

The project exercised the practical skill set expected of a controls engineer working in a Beckhoff shop. On the platform side it covers TwinCAT XAE project creation, PLC target selection over Ethernet, Secure ADS route configuration, EtherCAT device scanning, license activation, and the build → activate → login deployment flow. On the programming side it covers IEC 61131-3 ladder diagram authoring, program organization units (POUs), task assignment, structured variable declaration with `AT %I*` / `AT %Q*` placeholders, manual I/O linking, TON timer instances with `T#` time literals, set/reset step sequencing, and seal-in logic. On the process-control side it covers translation of a verbal process specification into a step-based state model, pause-resume implementation, manual reset handling, and separation of sequencing logic from output gating.

## Tools & Technologies

**Hardware:** Beckhoff CX5140 embedded PC-based PLC, EtherCAT I/O terminals on DIN rail (digital input and output "Term" modules), bench switch/lamp board for plant I/O simulation.

**Software:** Beckhoff TwinCAT 3 (XAE Shell) on Windows, Ladder Diagram (LD) programming language, Structured Text (ST) for variable declarations, TwinCAT ADS communication, TwinCAT on-line force/monitor for testing.

**Standards & frameworks:** IEC 61131-3 PLC programming standard, EtherCAT fieldbus for terminal I/O, step-based sequential function chart design patterns implemented in ladder.

## Outcome

The final program runs on the CX5140 and correctly sequences all five production steps — conveyor motor until limit switch, 2-second settle dwell, fill until weight sensor, 3-second eject dwell, and return — with correct pause-and-resume behavior on the stop pushbutton and a full state reset via the dedicated reset input. Beyond the mechanics of the specific filler, the lab was a concrete walkthrough of what a typical Beckhoff / TwinCAT commissioning day looks like end-to-end: bring up the project, pick the target, route securely over ADS, scan the EtherCAT terminals, write and link the logic, and go on-line to verify against real hardware. That workflow is directly transferable to any TwinCAT-based machine build.
