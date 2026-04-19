# Beckhoff CX5140 PLC – Cereal Box Filler Control System

**A sequential process-control application implemented on a Beckhoff Windows-based industrial PC using TwinCAT 3 ladder logic, including logical-operation, timer, and state-machine exercises that build up to a full cereal box filling machine.**

---

## Project Overview

This project is an EE 5340 graduate laboratory exercise (Lab 6 – *Beckhoff CX5140*) focused on programming a modern Windows-based PLC in the IEC 61131-3 Ladder Diagram language. The lab walks through the full tool-chain for a Beckhoff controller — network setup, I/O scanning, variable linking, program organization, and online debugging in TwinCAT 3 — and culminates in a complete cereal box filling station controller written as a latched, step-based sequencer with stop/resume and reset handling. The work was completed as a two-person group assignment.

The deliverable demonstrates practical fluency with the Beckhoff/TwinCAT ecosystem, the IEC 61131-3 programming model (POUs, tasks, typed variables, and ladder networks), and the classic set/reset step-sequence pattern that is the workhorse of industrial batch and discrete control.

## Learning Objectives

The lab was structured to build working knowledge of the Beckhoff CX5140 embedded PC platform and TwinCAT 3 engineering environment, hands-on use of Ladder Diagram programming under the IEC 61131-3 standard, implementation of logical-AND and logical-OR circuits from physical switches to a light load, on-delay timer (TON) programming for timed transitions, and finally a full sequential controller for a cereal-filling process with pause/resume, reset, and independent conveyor-motor behavior. Alongside the coding work, the lab also built operational familiarity with configuring the processor over Ethernet (ADS routing, fingerprint verification), scanning EtherCAT Terminals, and forcing physical inputs for online testing.

## Hardware Platform

The target controller is a **Beckhoff CX5140** — a DIN-rail-mounted embedded Industrial PC running the TwinCAT real-time runtime alongside a standard Windows image. The CX5140 integrates directly with Beckhoff's EtherCAT Terminal bus, and in this lab it was populated with digital input and digital output terminals corresponding to the switch panel and lamp load, respectively. Physical I/O mapping used Beckhoff's IEC-style channel addressing:

| Variable | Channel | Address | Direction |
|---|---|---|---|
| `Start_PB` | Terminal input 1 | `%IX40.0` | Input |
| `Stop_PB` | Terminal input 2 | `%IX40.1` | Input |
| `LS` (box-in-position limit switch) | Terminal input 3 | `%IX40.2` | Input |
| `WT_SENSE` (weight sensor) | Terminal input 4 | `%IX40.3` | Input |
| `Reset_PB` | Terminal input 5 | `%IX40.4` | Input |
| `M1` (box conveyor) | Terminal output 1 | `%QX26.0` | Output |
| `M2` (product feed) | Terminal output 2 | `%QX26.1` | Output |
| `FILL` (fill solenoid) | Terminal output 3 | `%QX26.2` | Output |
| `LA4` | Terminal output 4 | `%QX26.3` | Output |
| `LA5` | Terminal output 5 | `%QX26.4` | Output |

The controller was networked over Ethernet to the engineering PC, with the TwinCAT ADS route established using IP `131.151.52.144` (lab bench controller `CX0728E8E`), secured with a fingerprint-verified ADS route and an authenticated administrator session.

## Engineering Environment & Workflow

All programming was done in **TwinCAT XAE Shell** (TwinCAT 3) using the Visual Studio-based engineering environment. The project used the standard PLC project template, with a custom **POU (Program Organization Unit)** created as type *Program* and language *Ladder Logic Diagram (LD)*. The POU was attached to the default `PlcTask` so the runtime scheduler would execute it cyclically alongside the rest of the system.

Variables were declared between `VAR` / `END_VAR` with IEC 61131-3 typing. Physical I/O was declared using the `AT %I*` / `AT %Q*` placeholder syntax and then **linked** to real EtherCAT Terminal channels through TwinCAT's I/O tree after the project was first built. This two-step approach — symbolic declaration first, hardware binding after — is the Beckhoff convention and makes the program portable across terminal reshuffles.

The workflow for each exercise followed the same loop: author the ladder, *Build Solution*, *Activate Configuration* (generating trial licenses if prompted), go **online** via `PLC → Login`, and then exercise the program by toggling physical switches and observing the live ladder where energized elements highlight in blue. When a physical switch wasn't convenient — for example the non-existent `Reset_PB` — inputs were **forced** online from the I/O tree's *Online* tab, using the *Force* / *Release* workflow on the input channel.

## Exercises

### II. Series Operation (Logical AND)

The first rung implemented a two-input AND: `LS` and `WT_SENSE` placed in series drive the `LA5` lamp. This exercise doubled as the platform-familiarization task, walking through POU creation, variable declaration, I/O linking, build, activate, login, and online test.

### III. Parallel Operation (Logical OR)

The ladder was modified offline (in *Config Mode*) to place `LS` and `WT_SENSE` on parallel branches driving `LA5`, exercising the Branch Start/End toolbox element and reinforcing the offline-edit → rebuild → re-download cycle.

### IV. Timer Operation

A TON (on-delay timer) was added on a new network so that closing `LS` immediately energized `LA4` and, five seconds later, also energized `M2`. This demonstrated the TON function block — its `IN` enable, `PT` preset time (specified as `T#5s`), `Q` output, and the `ET` elapsed-time output tied to a `TIME` variable for live monitoring.

### V. Ladder Editing

The program was annotated with network comments (enabled via *Tools → Options → TwinCAT → PLC Environment → FBD/LD and IL Editors → Show network comment*) and additional internal coils were declared between the `VAR` blocks to support the final state-machine logic.

### VI. Cereal Box Filler (primary deliverable)

The culminating task was a complete cereal box filling station controller.

**Process description.** Motor `M1` runs the conveyor until an empty box arrives at the fill position, detected by the limit switch `LS`. The controller then dwells for 2.0 seconds, opens the `FILL` solenoid until the weight sensor `WT_SENSE` confirms the box is full, dwells another 3.0 seconds for settling, and then indexes the next box. Pressing `Stop_PB` at any point pauses the machine at the current step — both motors and the fill solenoid are de-energized, but the active step and its dwell timers remain valid so `Start_PB` cleanly resumes the suspended step. The `Reset_PB` clears all steps and timers and returns the machine to its initial state, but is intentionally ignored while the machine is running. Motor `M2` (the product feed) runs continuously whenever the machine is not paused.

### Implemented Program Structure

The final POU (`Series_Circuit` in the submitted program) uses the canonical IEC latched step-sequence pattern built entirely with set/reset coils:

**Variables.** Five physical inputs (`Start_PB`, `Stop_PB`, `LS`, `WT_SENSE`, `Reset_PB`), five physical outputs (`M1`, `M2`, `FILL`, `LA4`, `LA5`), a master `RUN` latch, five internal step bits (`STEP1` through `STEP5`), and two TON function-block instances (`TON_1`, `TON_2`) with their associated elapsed-time variables.

**Run / pause supervisor.** Network 1 latches the `RUN` bit on `Start_PB` and breaks it on `Stop_PB` via a standard seal-in circuit. This single bit gates every step transition and every output in the program, so pausing really does freeze the machine while preserving the state.

**Step-activation default.** Network 2 sets `STEP1` whenever `RUN` is true and no other step is active — effectively the "no-steps-active" condition that guarantees the machine always lands in the initial conveyor step after a clean start.

**Main sequence.** Networks 3–7 implement the production cycle. `STEP1` runs the conveyor until `LS` transitions `STEP1` → `STEP2`. `STEP2` enables `TON_1` with `PT = T#2s`; when the dwell completes, the step advances to `STEP3`. `STEP3` opens the fill solenoid until `WT_SENSE` transitions to `STEP4`. `STEP4` enables `TON_2` with `PT = T#3s` for the settling dwell, which advances to `STEP5`. `STEP5` indexes the conveyor until `LS` clears (new box leaves the position), after which the sequence wraps back to `STEP1`.

**Reset path.** Network 8 reacts to `Reset_PB` only while `RUN` is false (the lab requirement) — in that case every step bit (`STEP1` through `STEP5`) is reset and `STEP1` is re-set, guaranteeing a clean re-entry regardless of where the cycle was suspended.

**Outputs.** Networks 9–11 drive physical actuators directly off the step bits: `M1` runs while `STEP1 OR STEP5` is active (i.e., whenever the conveyor needs to move); `M2` runs whenever `RUN` is true; and `FILL` energizes only during `STEP3`. Because every output is conjoined with `RUN` (either directly or via the step bits, which themselves only advance while `RUN` is true), pressing `Stop_PB` instantly de-energizes the actuators while leaving the step latches and dwell timers intact — exactly the pause-and-resume behavior the specification requires.

## Technical Accomplishments

The finished program correctly implements every requirement in the lab specification: a complete pause/resume supervisor built on a single `RUN` latch, a five-step production sequence with two independent on-delay timers, a reset path that respects the *"ignored while running"* rule, and differentiated motor behavior (`M1` only during conveyor moves, `M2` continuous, `FILL` only during the weigh step). Beyond the code itself, the exercise produced working fluency with TwinCAT 3's ADS routing, EtherCAT I/O scanning, variable linking, online editing, and input forcing — the operational skills that separate someone who can program a PLC on a simulator from someone who can bring one up on real bench hardware.

## Skills Demonstrated

On the programming side the project covers IEC 61131-3 Ladder Diagram authoring, POU creation and task assignment, typed variable declaration, on-delay timer (TON) function blocks, set/reset coil-based state machines, and seal-in latching circuits. On the platform side it covers Beckhoff TwinCAT 3 engineering workflow in Visual Studio XAE Shell, ADS route creation with fingerprint verification, EtherCAT Terminal scanning, symbolic-to-physical I/O linking, online activation, online input forcing for test, and bench-level Ethernet networking to an embedded CX-series controller. On the control-system-design side it covers sequential process control, pause/resume without loss of state, reset handling under operational constraints, and clean separation between a sequencer and its actuator outputs.

## Tools & Technologies

**Controller:** Beckhoff CX5140 Windows-based embedded PC with EtherCAT Terminal I/O.

**Engineering software:** TwinCAT 3 (XAE Shell on Visual Studio), with the PLC runtime, Ladder Diagram (LD) and Structured Text (ST) editors, and TwinCAT ADS.

**Languages & standards:** IEC 61131-3 Ladder Diagram, IEC variable and address syntax (`%IX`, `%QX`, `T#` time literals), TON function block.

**Communication:** EtherCAT for I/O, Ethernet/ADS for engineering and on-line access.

## Outcome

The laboratory produced a working cereal-filler control program that was demonstrated online to the instructor on the bench hardware, together with a printed ladder listing documenting all networks. The deeper outcome was end-to-end familiarity with a second major PLC ecosystem (Beckhoff / TwinCAT 3 alongside prior coursework on Rockwell and Siemens), and mastery of the latched-step sequencer — a pattern that generalizes directly to the vast majority of discrete-manufacturing, packaging, and batch-process applications in industry.
