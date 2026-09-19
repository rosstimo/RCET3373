# RCET3373 Topics

These are the public student-facing self-learning and reference materials for RCET3373.

Use the topic that matches the concept you are reviewing. Practice quizzes and instructor feedback may link directly to a specific page or section here.

## Digital foundations

- [Digital Representation](digital-representation.md) - binary, hexadecimal, bit fields, masks, and signed interpretation.
- [Two's Complement Notes](twos_complement_notes.md) - additional worked two's-complement examples.
- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md) - electrical meaning of digital HIGH/LOW and interface checks.
- [Digital Timing](digital-timing.md) - propagation delay, setup/hold, frequency/period, and synchronous/asynchronous events.
- [Buses, ADC, and DAC](buses-adc-dac.md) - address/data/control roles, address spaces, converter resolution, and accuracy.
- [Memory Systems](memory-systems.md) - capacity, transactions, SRAM, DRAM, nonvolatile memory, and expansion.

## PIC16F883 architecture and I/O

- [PIC16F883 Datasheet and Architecture](pic16f883-architecture.md) - program/data memory, vectors, stack, banks, instruction fields, oscillator, and electrical limits.
- [MPLAB, PIC Assembler, PSECTs, and Vectors](mplab-pic-as-psects-vectors.md) - tool roles, configuration bits, sections, vectors, and program-memory verification.
- [Data Memory, SFRs, and Banking](data-memory-sfrs-banking.md) - SFR/GPR distinction, BANKSEL, register maps, and reset-state reasoning.
- [PORTB Datasheet-Driven Configuration](digital-io-portb-configuration.md) - multifunction pins, direction/digital mode, initialization, bank selection, and loading.

## Timing, control flow, and measurement

- [Instruction Timing and Software Delays](instruction-timing-software-delays.md) - FOSC/4, instruction cycles, DECFSZ timing, and basic delay math.
- [Nested-Loop Delays](nested-loop-delays.md) - nested timing derivation, range, resolution, and precision delay design.
- [Subroutines and Return Stack](subroutines-return-stack.md) - CALL/RETURN behavior, hardware stack, PCLATH, and simulator reasoning.
- [Measurement Strategy and C Timing](measurement-c-timing.md) - scope versus counter, prediction/measurement workflow, disagreement diagnosis, and compiled-C timing.
- [Lookup Tables and Dynamic Timing](lookup-tables-dynamic-timing.md) - computed GOTO, RETLW, PCL/PCLATH, table boundaries, and selectable delays.
- [Interrupts and Context Saving](interrupts-context-saving.md) - flags/enables, vectoring, shared ISR structure, context save/restore, and latency.
- [PIC16F883 Timers](timers.md) - Timer0, Timer1, Timer2, prescalers, preloads, PR2/postscaler math, interrupts, selection, and troubleshooting.

## Study pattern

For calculation-heavy topics:

1. identify the governing clock or data width;
2. write the governing relationship before inserting numbers;
3. calculate or predict the expected result;
4. implement or trace the mechanism;
5. measure or inspect evidence when applicable;
6. explain any disagreement instead of forcing the numbers to match.

For device-specific behavior, use the PIC16F883 datasheet as the technical authority and use these notes to organize the reasoning.
