<a id="top"></a>

# RCET 3373 — MPLAB X, PIC Assembler, PSECTs, and Vectors

*Self-learning guide*

[Topics index](README.md)

<a id="contents"></a>
## Contents

- [1. Why this matters](#why-this-matters)
- [2. What you should be able to do](#learning-outcomes)
- [3. Prerequisites and related topics](#prerequisites)
- [4. Core model and vocabulary](#core-model)
- [5. How it works](#how-it-works)
- [6. Worked examples](#worked-examples)
- [7. Apply, verify, and troubleshoot](#apply-verify-troubleshoot)
- [8. Practice](#practice)
- [9. Answer key](#answer-key)
- [10. What you should be able to explain without notes](#retrieval-check)
- [11. References](#references)

[Back to top](#top) · [Topics index](README.md)

<a id="why-this-matters"></a>
## 1. Why this matters

Writing assembly instructions is only part of creating firmware.

Before a PIC16F883 can execute your program, source code passes through several tools that transform and place it into program memory. Some addresses are dictated by the microcontroller hardware. Other addresses are chosen by the linker according to the sections and placement rules in the project.

Understanding that distinction helps you answer questions such as:

- Why does execution begin at `0x0000`?
- Why is the interrupt entry at `0x0004`?
- Why can ordinary code move without changing the source file?
- What does a PSECT actually control?
- Why can a project build successfully but put code somewhere you did not expect?
- How can you verify what the linker actually produced?

The goal is to understand the complete path from PIC Assembler source to executable code in PIC16F883 program memory.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- identify the roles of MPLAB X, PIC Assembler, the linker, MPLAB IPE, and the PICkit;
- explain why an uppercase `.S` source file is preprocessed before assembly;
- explain what a PSECT represents;
- distinguish hardware-defined vector addresses from linker and course layout choices;
- explain the purpose of the RCET linker-placement options;
- interpret `delta=2` for PIC16F883 CODE PSECTs;
- verify linked code placement using MPLAB X Program Memory;
- diagnose common source-section and linker-placement problems.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [PIC16F883 program memory](pic16f883-architecture.md#program-memory);
- [reset and interrupt vectors](pic16f883-architecture.md#vectors).

Electrical limits and output-loading calculations are intentionally **not** covered here. See:

- [PIC16F883 electrical limits](pic16f883-architecture.md#electrical-limits);
- [PORTB configuration and loading](digital-io-portb-configuration.md#electrical-loading).

For the tested RCET project setup, see the public [MPLAB X + PICkit 3 Setup reference](https://github.com/rosstimo/pic_projects/blob/main/References/MPLAB-PICkit3-Setup.md).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

The important model is the path from source code to programmed memory:

```text
main.S
  ↓
C preprocessor
  ↓
PIC Assembler (pic-as)
  ↓
object code organized into PSECTs
  ↓
linker
  ↓
linked program-memory addresses
  ↓
program/debug tool
  ↓
PIC16F883 program memory
```

The source file describes instructions and sections. The linker decides where relocatable sections are placed unless explicit placement rules constrain them.

<a id="tool-roles"></a>
### Tool roles

| Tool | Primary role |
| --- | --- |
| MPLAB X IDE | Project, editor, build, debug, and device-development environment |
| PIC Assembler (`pic-as`) | Processes PIC assembly source and produces object information for the build |
| Linker | Combines sections and assigns final addresses |
| MPLAB IPE | Programming-focused application |
| PICkit | Hardware interface used to program/debug the target PIC |

Microchip identifies **MPLAB X IDE v6.20 as the final MPLAB X release supporting PICkit 3**. Later MPLAB X releases do not support the PICkit 3.

That is a tool-support constraint, not a property of the PIC16F883 itself.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="preprocessed-source"></a>
### Uppercase `.S` means preprocessed assembly

RCET PIC assembly source uses an uppercase `.S` extension.

That tells the toolchain to run the source through the **C preprocessor** before assembly. This permits preprocessor features in addition to assembler syntax.

The correct term is **C preprocessor**, not “precompiler.”

For RCET source-file conventions, see the [RCET PIC-AS Style Guide](https://github.com/rosstimo/pic_projects/blob/main/RCET_PIC-AS_Style_Guide.md#31-source-file-extension).

<a id="configuration-bits"></a>
### Configuration bits establish device behavior before ordinary code runs

Configuration settings control device-level behavior such as oscillator configuration and watchdog options.

They are not ordinary sequential instructions executed by `main`.

The correct settings depend on the actual target hardware, so configuration values should be checked against the PIC16F883 data sheet instead of copied blindly from an unrelated project.

<a id="psects"></a>
### PSECTs describe sections

A **PSECT** describes a section of program or data to the assembler/linker.

A relocatable CODE PSECT does not mean:

> Put this code at the address where it appears in my source file.

Source-file order and final program-memory placement are different ideas.

Unless a placement rule constrains a relocatable PSECT, the linker assigns it a legal location.

<a id="hardware-vectors"></a>
### Hardware vectors and linker placement are different layers

For the PIC16F883:

| Purpose | Address | Defined by |
| --- | ---: | --- |
| Reset entry | `0x0000` | PIC16F883 hardware |
| Interrupt entry | `0x0004` | PIC16F883 hardware |
| RCET ordinary-code start | `0x0008` | Course/linker convention |

The first two addresses come from the microcontroller architecture.

`0x0008` is useful for the current RCET source layout, but the PIC16F883 does not require all ordinary application code to begin there.

The current RCET linker placement is:

```text
-Wl,-presetVect=0000h,-pisrVect=0004h,-pcode=0008h
```

These options connect the named course PSECTs to the desired addresses.

The **names** `resetVect`, `isrVect`, and `code` are source/course choices.

The **addresses** `0x0000` and `0x0004` are hardware facts.

<a id="delta-2"></a>
### Why `delta=2` appears

PIC16F883 program memory is word-addressed and uses 14-bit instruction words.

Object-file storage is represented in whole bytes.

For the mid-range CODE PSECT usage in this course, `delta=2` maps two object bytes to one program-memory address.

This setting belongs to the assembler/linker representation. It does not mean the PIC suddenly executes 16-bit instructions.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="vector-placement-example"></a>
### Worked example: placing reset and ordinary code

**Known**

The PIC16F883 hardware begins reset execution at `0x0000`.

The project defines the PSECTs `resetVect`, `isrVect`, and `code`.

The RCET layout intends:

```text
resetVect -> 0x0000
isrVect   -> 0x0004
code      -> 0x0008
```

**Find**

What connects the source section names to those addresses?

**Governing information or model**

PSECT names identify sections. The linker determines final placement.

**Reasoning**

The linker must receive placement rules for the named sections:

```text
-Wl,-presetVect=0000h,-pisrVect=0004h,-pcode=0008h
```

**Result**

The build system can place the RCET PSECTs at the intended addresses.

**Meaning**

The hardware does not know what `resetVect` means.

The linker does not invent the reset address.

The source defines the section name, the linker connects that section to an address, and the PIC hardware determines where reset execution begins.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="verify-program-memory"></a>
### Verify the linked result

Do not stop at “Build Successful.”

After building:

1. open the MPLAB X Program Memory view;
2. locate the reset-entry instructions;
3. verify that reset code begins at `0x0000`;
4. locate the interrupt entry if the project defines one;
5. verify that it begins at `0x0004`;
6. locate ordinary application code;
7. compare the actual addresses with the intended linker layout.

The linked program is evidence of what the tools actually produced.

<a id="undefined-psect"></a>
### Undefined-PSECT linker error

Suppose a linker option attempts to place `resetVect`, but no linked source defines a PSECT by that name.

The linker cannot place a section that does not exist.

This error is useful evidence. Check the source PSECT name and the linker option rather than changing unrelated device settings.

<a id="programmer-detection"></a>
### Separate build problems from programmer problems

A clean build and PICkit detection are different layers.

If the project builds successfully but the PICkit 3 is unavailable, investigate programmer/USB/tool detection before changing assembler source or linker placement.

That separation is especially important when the development environment includes a virtual machine or USB passthrough.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Which PIC16F883 addresses are hardware-defined for reset and interrupt entry?
2. Is `0x0008` a hardware-required start address for ordinary application code?
3. What job does a PSECT perform?
4. What job does the linker perform?
5. Why does uppercase `.S` matter?
6. What does `delta=2` describe in this CODE PSECT context?
7. Why should Program Memory be inspected after a successful build?
8. A linker option refers to `resetVect`, but no source file defines that PSECT. What kind of failure should you investigate?
9. A project builds correctly but MPLAB X cannot see the PICkit 3. Should your first change be to the PSECT layout? Explain.
10. Explain the difference between “the PIC resets at `0x0000`” and “the linker places `resetVect` at `0x0000`.”

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. Reset entry is `0x0000`; interrupt entry is `0x0004`. These are properties of the PIC16F883 architecture.
2. No. `0x0008` is the current RCET layout convention for ordinary code.
3. A PSECT describes a program or data section for the assembler/linker.
4. The linker combines sections and assigns final addresses according to the target memory model and placement rules.
5. The source passes through the C preprocessor before assembly.
6. For this mid-range CODE PSECT usage, two object bytes correspond to one word-addressed program-memory location.
7. A successful build proves the tools produced output; Program Memory verifies where the linked code actually landed.
8. Investigate the relationship between the PSECT names defined in source and the section names used by the linker options.
9. No. Programmer detection is a separate tool/hardware/USB problem from source/link placement.
10. The first statement describes hardware behavior. The second describes how the build places source code so that it satisfies that hardware requirement.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

Before you consider this topic learned, you should be able to:

- trace the path from `main.S` to linked PIC16F883 program memory;
- distinguish assembler, linker, IDE, programmer, and device responsibilities;
- explain the difference between a PSECT name and its final address;
- identify which vector addresses are hardware-defined;
- explain why the RCET code start at `0x0008` is a convention;
- explain `delta=2` without describing the PIC16F883 as a 16-bit-instruction device;
- verify vector/code placement in MPLAB X;
- separate build/linker failures from programmer-detection failures.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 program-memory organization, reset and interrupt vector addresses, configuration behavior, and device architecture.

- Microchip Technology Inc., *MPLAB XC8 PIC Assembler User's Guide*, DS50002974 — https://ww1.microchip.com/downloads/aemDocuments/documents/DEV/ProductDocuments/UserGuides/MPLAB-XC8-PIC-Assembler-Users-Guide-DS50002974.pdf
  - Used for: PIC Assembler source handling, PSECTs, assembler/linker behavior, options, and diagnostics.

- Microchip Technology Inc., *MPLAB XC8 PIC Assembler User's Guide for Embedded Engineers* — https://onlinedocs.microchip.com/oxy/GUID-205B1F42-0E06-45E1-8D34-E3D05C15710F-en-US-3/
  - Used for: example-driven PIC Assembler project structure, PSECTs, directives, and build workflow.

- Microchip Technology Inc., *MPLAB X IDE* — https://www.microchip.com/en-us/development-tool/MPLAB-X-IDE
  - Used for: MPLAB X IDE information and PICkit 3 support status.

- Microchip Technology Inc., *MPLAB Ecosystem Downloads Archive* — https://www.microchip.com/en-us/tools-resources/archives/mplab-ecosystem
  - Used for: archived MPLAB X IDE v6.20 availability for the current PICkit 3 workflow.

- Microchip Technology Inc., *MPLAB X IDE User's Guide*, DS50002027D — https://ww1.microchip.com/downloads/en/DeviceDoc/50002027D.pdf
  - Used for: project/build/debug workflow and memory-view concepts.

### Related RCET reference

- [MPLAB X + PICkit 3 Setup for RCET PIC16F883 Projects](https://github.com/rosstimo/pic_projects/blob/main/References/MPLAB-PICkit3-Setup.md)
  - Provides the tested RCET project setup, linker-placement options, and Program Memory verification checklist.

[Back to top](#top) · [Topics index](README.md)
