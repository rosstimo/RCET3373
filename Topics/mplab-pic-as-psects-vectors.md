# RCET3373 W01D04 - Self-Learning Guide

# What you should be able to do

Explain the first PIC16F883 build/link/startup path, create a PIC Assembler project, distinguish fixed hardware vector addresses from linker/course choices, and read electrical tables without treating damage limits as design goals.

# 1. Know the tool roles

- **MPLAB X IDE:** project/editor/build/debug environment.
- **PIC Assembler (`pic-as`):** assembles PIC source and participates in the build/link flow.
- **MPLAB IPE:** programming-focused environment separate from the full IDE.
- **PICkit:** programming/debug hardware between the computer and target PIC.

For the course PICkit 3 environment, the classroom record used MPLAB X IDE v6.20, the final MPLAB X release supporting PICkit 3.

# 2. Why `main.S` is uppercase

Uppercase `.S` is preprocessed assembly source. It passes through the C preprocessor before assembly, allowing preprocessor constructs and C-style comments in addition to normal assembler `;` comments.

Say **C preprocessor**, not "precompiler."

# 3. Configuration bits happen before ordinary program logic

Configuration bits choose important startup/device behavior such as oscillator mode and watchdog-related options. In this course the 4 MHz external crystal uses the appropriate XT oscillator configuration. Development-friendly settings are deliberate choices for this lab, not universal production recommendations.

# 4. What a PSECT does

A PSECT describes a section of program/data to the assembler/linker. A relocatable CODE PSECT does not automatically mean "put this at the address where it appears in my source file."

The linker chooses a legal location unless a placement rule constrains it.

# 5. Hardware vector addresses versus course layout

For PIC16F883:

```text
Reset entry       0x0000   hardware-defined
Interrupt entry   0x0004   hardware-defined
Ordinary code     0x0008   course convention, not hardware-required
```

If a relocatable reset PSECT contains the reset entry code, it must be linked to `0x0000` so hardware arrives at the intended instructions.

Ordinary code can be elsewhere as long as the reset path branches to the linked label correctly.

# 6. Why `delta=2` appears

PIC16F883 program memory is word-addressed. Its instruction word is 14 bits, while object files represent data in whole bytes. In this Mid-range CODE PSECT, `delta=2` maps two object bytes to one program-memory address.

# 7. Verify with program memory

Do not trust your mental picture alone.

1. Build the project.
2. Open the program-memory view/map.
3. Find reset code and labels.
4. Compare actual addresses with your intended layout.
5. Change one placement rule if needed.
6. Rebuild and verify again.

An undefined-PSECT linker error is useful evidence: a placement option cannot refer to a section that has not been defined in the linked program.

# 8. Electrical tables have different purposes

## Absolute maximum

A damage/stress boundary. Do not design to it.

Examples from the class review include the 25 mA per-pin source/sink absolute maximum and the 6.5 V VDD-to-VSS absolute maximum. These are not targets.

## Guaranteed electrical characteristics

Specifications such as output voltage at stated current/supply conditions. These tell you what behavior Microchip guarantees inside defined conditions.

## Design target

What you choose with margin after considering the PIC, external load, resistor/LED ratings, total supply current, and device dissipation.

For output-driver dissipation, Microchip's relationship uses the voltage drop across the output drivers, not simply `VDD * pin current` for every sourced output.

# Practice

1. What is hardware-defined at `0x0000` and `0x0004`?
2. Is `0x0008` a hardware-required address for ordinary code?
3. Why must a relocatable reset PSECT be explicitly placed at the reset entry in this design?
4. What does uppercase `.S` change in the build path?
5. What does `delta=2` mean in this code-PSECT context?
6. Why is a 25 mA absolute maximum not an LED design target?
7. Why is linker placement better described as deterministic than random?

# Answer key

1. Reset entry at `0x0000`, interrupt entry at `0x0004`.
2. No. It is a course layout convention.
3. Hardware begins execution at `0x0000`; the linker must put that reset-entry code there.
4. The source is processed by the C preprocessor before assembly.
5. Two object bytes correspond to one word-addressed program-memory location.
6. Absolute maximum ratings are stress limits; design should satisfy guaranteed characteristics with margin and all other current/dissipation constraints.
7. The linker follows allocation rules to choose legal sections/addresses; surprising placement is not randomness.

# References

See the data-sheet, MPLAB X, and PIC Assembler references in the package README.
