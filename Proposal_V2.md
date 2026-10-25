# MotionScribe — Proposal V2

*A Standalone Embedded Robotic Drawing System*

**Course:** ESE 5190 — Smart Devices, University of Pennsylvania  
**Team:** Fangdong Yuan · Haoyu Shi · Ray Lin  
**Document:** Final Project Proposal — Version 2  
**Date:** October 25, 2026

*Design status: This proposal describes planned work and proposed acceptance targets, not measured results. Hardware specifications, calibration parameters, and drawing performance will be verified during prototyping.*

## 1. Abstract

MotionScribe is an interactive tabletop system that turns touchscreen sketches and predefined patterns into drawings on paper. An STM32F411RE (NUCLEO-F411RE) running FreeRTOS will process user input, convert drawing paths into joint commands, and drive a modified Adeept ADA031-V5.0 robotic arm through hardware-timer PWM. Both drawing modes will share one firmware build and motion pipeline, with no PC required during operation. Our target is recognizable, repeatable drawings with clean separation between strokes, rather than precision-plotter accuracy.

## 2. Motivation

Drawing on a touchscreen is intuitive; reproducing that drawing with a robotic arm is not. Screen coordinates must be mapped to a calibrated physical workspace, converted into reachable joint positions, and executed with appropriate timing and pen contact. Touch input, display updates, motion planning, and servo control must also work together on a resource-constrained microcontroller.

MotionScribe turns these challenges into an interactive demonstration of register-level C programming, FreeRTOS, peripheral drivers, robot kinematics, mechanical design, and hardware–software integration. Rather than relying on the arm manufacturer’s control application, our team will develop and understand the STM32-based drawing and control firmware.

### 2.1 Feature priorities

We group features as **Must** (required for the final system), **Should** (improvements after the baseline works), and **May** (optional extensions). Only Must features are committed to the final demonstration; Sections 5 and 6 define their measurable requirements.

| Priority | Feature | Expected behavior |
| --- | --- | --- |
| Must | Draw What You Draw | Sketch a simple shape on the touchscreen, preview it, and have the arm reproduce it on paper. |
| Must | Choose & Draw | Select a predefined pattern, such as a flower or smiley face, and have the arm draw it. |
| Must | Standalone operation | Run both modes on the STM32 without a connected PC. |
| Must | Physical controls | Use Start/Pause and Cancel buttons, plus an analog knob to select Slow/Normal/Fast drawing speeds. Park is a separate, validated motion state. |
| Must | Stroke separation | Lift the pen during non-drawing moves so distinct strokes are not connected. |
| Must | Bounded motion | Reject invalid paths and keep commanded joint motion within calibrated limits. |
| Should | Drawing-quality refinement | Improve smoothness and repeatability through better calibration and path sampling. |
| Should | Adjustable patterns | Change parameters such as flower size or petal count. |
| May | Pen-contact sensing | Add an FSR to measure pen contact after the passive mount has been validated. |
| May | More drawing inputs | Add text drawing or SVG import. |
| May | AI-assisted input | Explore natural-language or AI-generated drawing paths as an additional input source. |
| May | ROS 2 integration | Explore an interface to an external ROS 2 system without making the baseline device PC-dependent. |
| May | Further hardware and interaction upgrades | Explore a custom PCB, external joint-position sensing, or sound/distance inputs. |

May features are potential future extensions, not dependencies of the standalone STM32 drawing system or requirements for the course demo.

### 2.2 Core engineering challenges

The main challenges are embedded control (timers, interrupts, ADC, SPI/I2C, and FreeRTOS scheduling), robot motion (coordinate mapping, inverse kinematics, and path timing), and physical integration (servo power, calibration, pen contact, and repeatability).

## 3. System Block Diagram

*Figure 1. Proposed hardware architecture.* The STM32 is the sole controller of the servos during normal operation. A PC may be connected for programming or diagnostics but is not part of the runtime system. Exact pin assignments, touch/display interfaces, servo channels, and supply ratings are subject to component verification.

```mermaid
flowchart TB
    TOUCH["Touchscreen + display<br/>Sketch / preview / status<br/>SPI or I2C: confirm controller"]
    BTN["Two buttons<br/>Start/Pause; Cancel"]
    POT["Speed potentiometer<br/>Analog voltage"]
    MCU["NUCLEO-F411RE / STM32F411RE<br/>Register-level C + FreeRTOS<br/>GPIO / EXTI / ADC / SPI or I2C / Timers"]
    SERVO["Servo channels<br/>Joint mapping: to be verified"]
    ARM["Adeept V5.0 robotic arm<br/>Compliant pen holder → paper"]
    MCUPOWER["Standalone MCU/logic supply<br/>Source and rating: to be verified"]
    SERVOPWR["Regulated servo supply<br/>Voltage/current: to be verified"]
    GND["Common ground reference"]
    PC["Optional PC<br/>Programming / UART logs only"]

    TOUCH <-->|"SPI / I2C"| MCU
    BTN -->|"GPIO / EXTI"| MCU
    POT -->|"ADC"| MCU
    MCU -->|"Hardware-timer PWM"| SERVO
    SERVO --> ARM
    MCUPOWER --> MCU
    SERVOPWR --> SERVO
    MCU --- GND
    SERVO --- GND
    MCUPOWER --- GND
    SERVOPWR --- GND
    PC -.->|"Development only"| MCU
```

The design includes at least two inputs (touchscreen, buttons, and a potentiometer), two outputs (display and robotic arm), an analog ADC input, and an SPI/I2C peripheral through the selected display/touch hardware. It also covers at least three course topics: timers/PWM, interrupts, RTOS, and advanced motion planning. We will confirm the potentiometer’s acceptance as the analog input and the display/touch interface with the teaching team.

*Shared software pipeline.* Both drawing modes use the same path representation and control flow.

```mermaid
flowchart LR
    A["Touchscreen strokes<br/>or stored pattern"] --> B["Stroke data<br/>Preserve pen-up boundaries"]
    B --> C["Sampling, scaling<br/>and paper calibration"]
    C --> D["Workspace checks<br/>Pen-up/down trajectories"]
    D --> E["Inverse kinematics<br/>Joint limits + timed targets"]
    E --> F["Bounded motion queue"]
    F --> G["FreeRTOS Motion Task<br/>Sole actuator-command writer"]
    G --> H["Timer registers → PWM<br/>Servo joints"]
```

We plan three FreeRTOS tasks: Input/UI, Drawing/Planner, and Motion. Only the Motion Task updates servo commands; hardware timers generate PWM independently of display refresh or path computation. We will evaluate an initial motion-target update interval of approximately 20 ms. The speed knob will adjust trajectory timing, not the PWM frame frequency. The arm’s supplied controller may help with initial assembly checks, but it will not drive the same servos simultaneously with the STM32.

**Control limits:** The firmware will enforce joint/workspace limits, bounded stroke and queue storage, and explicit Start/Pause/Cancel states. Cancel will invalidate unexecuted targets. Pause is not an emergency stop, and Park is a programmed posture rather than encoder-based homing. Without external position feedback, commanded joint targets must not be presented as measured joint or pen-tip positions.

## 4. Design Sketches

*Figure 2. Conceptual tabletop setup (not to scale).* The robotic arm, paper fixture, touchscreen, physical controls, STM32 board, and separate power supplies are arranged for standalone drawing. The arm base and paper will be securely mounted. Exact positions and electrical ratings will be confirmed during prototyping.

![MotionScribe tabletop setup showing the arm, paper fixture, touchscreen, STM32 board, physical controls, and separate power supplies](assets/figures/figure-2-tabletop-setup.png)

*Figure 3. Passive compliant pen mount (conceptual cross-section, not to scale).* A guided pen carriage and elastic element are intended to absorb small height variations and reduce excessive contact force. The guide may require 3D printing; its travel, preload, friction, and lateral play will be tested.

![Passive compliant pen mount with fixed guide frame, elastic element, sliding pen carriage, and arm Z-motion for pen lift](assets/figures/figure-3-compliant-pen-mount.png)

Our initial pen-up/down design combines the passive mount with arm Z-motion. A separate lift actuator is a fallback if early tests show inadequate clearance, contact consistency, or repeatability; it is not included in the current budget. We will test mount travel, friction, preload, return behavior, and lateral play, and confirm active joints and pen-tip orientation on the V5.0 hardware.

## 5. Software Requirements Specification (SRS)

These are proposed acceptance targets, not measured results. If early tests reveal a hardware constraint, we will document the findings and discuss any requirement changes with the teaching team.

| ID | Software requirement | Verification method |
| --- | --- | --- |
| SRS-01 | The system shall execute touchscreen-sketch drawing and stored-pattern drawing using one firmware build and one shared drawing pipeline, without a runtime PC connection. | Disconnect the PC and complete both drawing workflows from onboard input to physical output. |
| SRS-02 | The software shall preserve separate strokes and generate pen-up travel between them. | Use single- and multi-stroke test paths; inspect generated targets and pen-state transitions. |
| SRS-03 | During drawing, Start/Pause/Cancel events shall reach the command/state handler within 100 ms. Cancel shall invalidate unexecuted targets from the canceled job; 100 ms refers to event handling, not mechanical stopping time. | Timestamp events and state changes under display/planning load; cancel at different queue stages and verify that old targets are not executed. |
| SRS-04 | The software shall reject all cases in a defined invalid-path test set before drawing starts and display an error reason. | Test out-of-workspace paths, unreachable points, and joint-limit violations; confirm rejection before actuator execution. |
| SRS-05 | Both drawing modes shall complete five consecutive trials without detected motion-queue corruption, buffer overrun, or unhandled task failure. | Record repeated trials and run boundary, queue-saturation, and memory/stack checks; drawing trials alone do not prove memory safety. |
| SRS-06 | The Slow, Normal, and Fast settings shall produce three distinct motion-timing profiles without changing the servo PWM frame period. | Compare commanded target intervals or trajectory duration for the same pattern at each setting; capture PWM timing. |

## 6. Hardware Requirements Specification (HRS)

| ID | Hardware requirement | Verification method |
| --- | --- | --- |
| HRS-01 | Each active servo channel shall meet the installed servo’s validated control-signal timing specification. The provisional 50 Hz target is a 20.0 ms ± 0.2 ms PWM period with commanded pulse widths accurate to ±10 µs. The initial plan uses three channels; the final channel count depends on joint/lift tests. | Capture all active channels with an oscilloscope or logic analyzer under simultaneous UI and planning activity. Confirm the servo timing specification before freezing the target. |
| HRS-02 | The system shall accept touchscreen input, two physical buttons, and an ADC-connected speed potentiometer. Each function shall pass 10 repeated trials; the knob shall distinguish Slow/Normal/Fast in 10 selection cycles. | Log touch/button events and ADC values; test every input function and speed setting. |
| HRS-03 | The system shall complete five consecutive drawings without an MCU reset caused by servo-power disturbances. | Monitor logic/servo rails during multi-joint movements and record reset status. |
| HRS-04 | The mechanism shall complete five consecutive pen-up transfers without an unintended connecting line on the calibrated paper setup. | Inspect paper output and record workspace, lift clearance, and settling time. |
| HRS-05 | The arm/pen assembly shall complete a fixed line-and-square test pattern five consecutive times, with all intended strokes visibly present and contained within the calibrated drawing area. | Use a fixed drawing checklist and archived photos; separately measure geometric deviations and repeatability. |

Drawing quality and workspace: We will compare lines, squares, circles, and hearts for recognizable shape, repeatability, and geometric deviation. 50 × 50 mm is the initial calibration area; 75 × 75 mm and 100 × 100 mm are candidate test regions where reachable. We will not claim millimeter-level drawing accuracy before measurement. Electrical PWM timing accuracy does not establish pen-tip positioning accuracy.

## 7. Bill of Materials (BOM)

**Required course BOM Google Sheet:** `TODO — Insert the completed team sheet URL before submission.`

The course budget is approximately $150 per team. These figures are planning estimates, not confirmed quotations; zero-cost items depend on lab availability.

| Component | Qty. | Purpose | Estimated cost | Status / dependency |
| --- | ---: | --- | ---: | --- |
| STM32 NUCLEO-F411RE | 1 | Main microcontroller | $0 | Lab/course loan preferred; availability to be confirmed. |
| Adeept ADA031-V5.0 arm kit | 1 | Arm and servos | $70 | Confirm vendor approval, kit contents, and installed servo specifications. |
| Touch LCD | 1 | Sketch input, preview, and status | $0 | Lab loan assumed; controller, bus interface, and logic levels to be confirmed. |
| Potentiometer | 1 | Analog speed input via ADC | $2 | Select value and wiring. |
| Physical buttons | 2 | Start/Pause and Cancel | $2 | Debounce in firmware. |
| Regulated servo supply | 1 | Separate actuator power | $20 | Confirm voltage and peak current margin. |
| Wiring and connectors | 1 set | Signal/power distribution and common ground | $8 | Confirm connectors and wire ratings. |
| Base and paper fixture | 1 set | Secure drawing setup | $8 | Lab or fabricated materials. |
| Compliant pen-mount materials | 1 set | Guided carriage and elastic element | $8 | May require 3D printing. |
| Fasteners | 1 set | Mechanical assembly | $5 | Check included kit hardware. |
| Lightweight pen | 1 | Drawing tool | $2 | Check fit and contact force. |
| Standalone MCU power supply | 1 | Operate without a PC | $0 | Borrowed USB wall adapter or suitable equivalent assumed. |
| Components subtotal | | | $125 | Assumes MCU, display, and logic supply can be borrowed. |
| Shipping and tax reserve | | | $20 | Final amount to be confirmed. |
| **Planned total** | | | **$145** | $5 below the $150 planning target. |

We will check the course’s parts inventory before ordering and confirm an approved purchasing channel for the arm. The course does not reimburse student purchases. If the screen/MCU cannot be borrowed or a separate lift actuator becomes necessary, we will revise the BOM and discuss an increased budget with the teaching team.

The STM32 and servos will use separate power rails with a common ground; the NUCLEO will not power the servos. We will verify supply ratings, 3.3 V control-signal compatibility, startup transients, and an accessible servo-power disconnect. Software Pause is not an emergency stop; removing servo power may release holding torque.

## 8. Final Demo

The demo will take place on a secured tabletop with fixed paper, a touchscreen, physical controls, and standalone power. It does not require special facilities or a runtime PC.

1. **Start safely.** Check a known safe arm posture, enable servos using a validated startup procedure, and display READY. No external position feedback or automatic homing is assumed.
2. **Demo A — Draw What You Draw.** Draw a heart, smiley face, or simple multi-stroke sketch on the screen; preview it; then have the arm reproduce it on paper. Real-time imitation is not required.
3. **Show physical interaction.** Use Start/Pause/Cancel and the Slow/Normal/Fast knob. Demonstrate controlled handling of canceled motion.
4. **Demo B — Choose & Draw.** Select Flower or Happy Face and draw it using the same firmware and motion engine. The initial library is planned to contain approximately 3–5 simple patterns; a fixed Flower is the baseline, and Rainy Day is a possible addition.
5. **Confirm standalone operation.** Repeat both drawing modes with the PC disconnected and show representative drawing/test evidence.

Project deliverables will include firmware source code, wiring/pin documentation, the BOM, calibration notes, pen-mount design files, operating instructions, measurement records, photos, and demonstration media. Recorded commanded joint angles will not be described as measured arm positions.

## 9. Sprint Planning

Our critical path is power/PWM → joint calibration → reliable pen contact → stored-pattern drawing → touchscreen integration → repeated validation. Physical drawing feasibility takes priority over advanced UI features or optional pattern processing.

| Milestone | Planned work | Completion criterion |
| --- | --- | --- |
| Oct 26–Nov 1: Feasibility & interfaces | Confirm arm, touch LCD, power, lab inventory, and documentation. Define key pins, interfaces, and memory limits; begin peripheral bring-up and FreeRTOS structure. | Main hardware dependencies identified and initial drivers running. |
| Nov 2–8: Power & mechanism | Test independent power and PWM; measure joint geometry and safe limits; evaluate passive pen contact and basic IK. Meet with the Account Manager by Nov 6. | Repeatable safe joint commands and documented pen-contact results. |
| Nov 9–15: First physical drawing | Calibrate paper coordinates, generate line/square paths, implement pen-up/down, and validate rejected paths. | Draw a basic pattern using the STM32-controlled arm. |
| Nov 16–20: MVP integration | Integrate touch/display, both drawing modes, and preliminary SRS/HRS tests. No Detkin parts orders after Nov 16. | MVP Demo: November 20, 2026, during lab hours. |
| After MVP: Reliability & polish | Improve drawing fidelity, interaction, queue behavior, and repeated-trial performance; prioritize measured issues. | All Must features work together on one firmware build. |
| Before final demo: Validation & delivery | Freeze the system, run acceptance tests, document limitations, rehearse, and finalize report/video. | Final demonstration and supporting engineering evidence are ready. |

### 9.1 Team responsibilities

| Workstream | Main responsibility | Coordination |
| --- | --- | --- |
| A — Embedded Firmware & Integration | Register-level drivers, FreeRTOS tasks/queues, command state machine, timer scheduling, and debugging. | Coordinate touch/display driver with C and motion commands with B. |
| B — Robotics & Drawing Engine | Stroke processing, geometry, FK/IK, calibration, waypoint/timing generation, and pattern library. | Measure arm geometry with C and integrate timed targets with A. |
| C — Hardware & UI | Procurement, power/wiring, fixtures, pen mount, UI behavior, and hardware testing. | Develop UI interfaces with A and perform calibration tests with B. |

Assignments are provisional. Fangdong Yuan is proposed for Workstream C; A and B will be assigned by the team before submission. All three members will participate in regular full-system integration and must be able to explain the firmware architecture and critical drivers.

### 9.2 Major risks and response plans

| Risk | Early test / mitigation |
| --- | --- |
| Drawing reachability, pen angle, and contact | Measure the V5.0 linkage and servo limits; start with a 50 × 50 mm area; test passive compliance and pen-up clearance before complex patterns. |
| Touch hardware, memory, and drivers | Confirm SPI/I2C controller chips and pin compatibility early; use partial display updates if a full framebuffer exceeds available SRAM. |
| Power integrity and schedule | Check supply transients and signal levels; complete simple stored-pattern motion first; defer Should/May features if needed. |

### 9.3 Development and validation approach

We will write register-level C peripheral drivers and develop the final application logic ourselves; FreeRTOS provides task scheduling. Adeept examples, geometry references, and manufacturer documentation may inform the design but will not replace our original firmware. Any proposed external library will follow the course's documentation/approval requirements. AI tools will not be used to write the final-project firmware, in accordance with the course manual.

During implementation, we will record test evidence for key requirements and investigate failures in PWM timing, task/queue operation, memory bounds, joint calibration, stroke separation, power integrity, and drawing repeatability. Touch sampling limits and drawing quality will be measured rather than assumed; path resampling cannot recover touch points that were never captured.

## References

- [ESE 5190 Fall 2026 Final Project Manual](https://docs.google.com/document/d/17jUNs98_-TAvaIJuj3ORlqEL54Mr2z-0nuQrpoW8yvw/edit)
- [Adeept ADA031-V5.0 product page](https://www.adeept.com/robotic-arm_p0453.html)
- [Adeept V5.0 resources](https://www.adeept.com/learn/detail-103.html)
- [ST NUCLEO-F411RE board information](https://www.st.com/en/evaluation-tools/nucleo-f411re.html)
- [ST STM32F411RE microcontroller information](https://www.st.com/en/microcontrollers-microprocessors/stm32f411re.html)
