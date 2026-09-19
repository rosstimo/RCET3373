# RCET3373 Topics

These are the public student-facing self-learning and reference materials for **RCET 3373: Advanced Computer Architecture and Embedded Systems**.

Use the topic that matches the concept you are reviewing. Practice quizzes and instructor feedback may link directly to a specific page or section here.

The weekly grouping below is an **ideal learning sequence**, not a rigid daily schedule. Topic pages keep stable URLs so they can also be used independently for review and remediation.

## Week 1 · Digital foundations

- [Digital Representation](digital-representation.md) — binary, hexadecimal, bit fields, masks, and representation.
- [Two's Complement](twos_complement_notes.md) — signed fixed-width integers and subtraction.
- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md) — electrical meaning of digital HIGH/LOW and interface checks.
- [Digital Timing](digital-timing.md) — propagation delay, setup/hold, frequency/period, and synchronous/asynchronous events.
- [Address, Data, and Control Buses](buses-adc-dac.md) — bus roles, address spaces, shared-bus ownership, and memory-mapped I/O.
- [ADC and DAC Foundations](adc-dac-foundations.md) — resolution, codes, quantization, references, and converter direction.
- [Memory Systems](memory-systems.md) — organization, SRAM, DRAM, nonvolatile memory, timing, and expansion.

## Week 2 · PIC16F883 architecture and I/O

- [PIC16F883 Datasheet and Architecture](pic16f883-architecture.md) — CPU, program/data memory, vectors, stack, instruction format, oscillator, and electrical boundaries.
- [MPLAB X, PIC Assembler, PSECTs, and Vectors](mplab-pic-as-psects-vectors.md) — tool roles, source processing, sections, linker placement, and program-memory verification.
- [Data Memory, SFRs, and Banking](data-memory-sfrs-banking.md) — SFR/GPR distinction, bank selection, register maps, and reset-state reasoning.
- [PORTB Datasheet-Driven Configuration](digital-io-portb-configuration.md) — multifunction pins, direction/digital mode, initialization, read-modify-write risk, and loading.

## Week 3 · Instruction timing, delays, subroutines, and measurement

- [Instruction Timing and Software Delays](instruction-timing-software-delays.md) — FOSC/4, instruction cycles, DECFSZ timing, and exact delay boundaries.
- [Nested-Loop Software Delays](nested-loop-delays.md) — nested timing derivation, range, resolution, and target-delay design.
- [Subroutines and the PIC16F883 Return Stack](subroutines-return-stack.md) — CALL/RETURN, hardware stack, PCLATH, timing, and simulator reasoning.
- [Timing Measurement and Compiled-Code Verification](measurement-c-timing.md) — scope versus counter, prediction/measurement workflow, disagreement diagnosis, and compiled-C timing.

## Week 4 · Lookup tables and dynamic timing

- [Lookup Tables, Computed GOTO, and Dynamic Timing](lookup-tables-dynamic-timing.md) — RETLW tables, PCL/PCLATH, table-placement boundaries, selectable timing, and whole-path timing.

## Week 5 · Interrupts

- [Interrupts and Context Saving](interrupts-context-saving.md) — event/flag/enable routing, vectoring, source identification, context save/restore, and latency.

## Week 6 · Hardware timers and nonblocking timing

- [PIC16F883 Timers](timers.md) — Timer0, Timer1, Timer2, prescalers, preloads, PR2/postscaler, interrupts, selection, and troubleshooting.
- [Nonblocking Timing and Embedded State Machines](nonblocking-state-machines.md) — periodic ticks, elapsed-time scheduling, explicit states, ISR boundaries, and multiple timed behaviors.

## Week 7 · ADC and sensor conditioning

- [ADC and DAC Foundations](adc-dac-foundations.md) — converter model, resolution, quantization, references, and accuracy.
- [PIC16F883 ADC and Sensor Conditioning](pic16f883-adc.md) — channels, acquisition time, conversion clock, result handling, source impedance, conditioning, and measurement.

## Week 8 · UART / asynchronous serial

- [UART and Asynchronous Serial Communication](uart-asynchronous-serial.md) — framing, baud generation, TX/RX paths, buffering, errors, and measurement.

## Week 9 · Persistent state

- [Memory Systems](memory-systems.md) — nonvolatile-memory foundations.
- [PIC16F883 Data EEPROM and Persistent State](pic16f883-data-eeprom.md) — read/write sequencing, protected writes, completion, endurance, and integrity.

## Week 10 · I2C

- [I2C Communication](i2c.md) — open-drain bus behavior, pull-ups, START/STOP, addressing, ACK/NACK, transactions, MSSP, and troubleshooting.

## Week 11 · Feedback and control

- [Feedback Control and PID Foundations](feedback-control-pid.md) — closed-loop model, P/I/D terms, sampling, saturation, windup, noise, and embedded implementation limits.

## Week 12 · SPI, CAN, and protocol selection

- [SPI Communication](spi.md) — synchronous shifting, chip select, clock phase/polarity, MSSP, and waveform verification.
- [CAN Bus Foundations](can-bus.md) — controller versus transceiver, differential bus, termination, arbitration, frames, CAN FD, and robotics use.
- [Embedded Communication Protocol Selection](communication-protocol-selection.md) — compare UART, I2C, SPI, and CAN by topology, timing, wiring, robustness, and application fit.

## Week 13 · Embedded C

- [Embedded C on the PIC16F883](embedded-c-pic16f883.md) — XC8 build model, headers, fixed-width types, SFR access, volatile, interrupts, optimization, and generated code.
- [Timing Measurement and Compiled-Code Verification](measurement-c-timing.md) — open the abstraction when exact generated timing matters.

## Week 14 · Processor architecture depth

- [Processor Datapaths, Cycles, and Pipelining](processor-datapath-pipelining.md) — ISA versus microarchitecture, datapaths, single/multi-cycle designs, pipelines, hazards, and branch flushes.

## Week 15 · MCU and toolchain transfer

- [Approaching an Unfamiliar Microcontroller and Toolchain](mcu-toolchain-transfer.md) — MCU versus board, programming/debug paths, SDK/runtime layers, bring-up contract, and verification boundaries.

Device-specific alternate-MCU recipes should be treated as supported course procedures only after that exact hardware/toolchain path has been tested end to end.

## Week 16 · System integration and troubleshooting

- [Embedded-System Integration and Troubleshooting](embedded-system-integration-troubleshooting.md) — timing/resource budgets, shared-peripheral conflicts, reset/watchdog behavior, persistent state, concurrency, and layered verification.

## Study pattern

For calculation-heavy topics:

1. identify the governing clock, data width, or physical range;
2. write the governing relationship before inserting numbers;
3. calculate or predict the expected result;
4. implement or trace the mechanism;
5. measure or inspect evidence when applicable;
6. explain any disagreement instead of forcing the numbers to match.

For device-specific behavior, use the current official device documentation as the technical authority and use these guides to organize the reasoning.
