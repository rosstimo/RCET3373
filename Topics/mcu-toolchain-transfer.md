<a id="top"></a>

# RCET 3373 — Approaching an Unfamiliar Microcontroller and Toolchain

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

The PIC16F883 is a teaching platform, not the final answer to every embedded problem.

Professional embedded work requires moving among:

- different processor architectures;
- different board vendors;
- different programming/debug interfaces;
- different IDEs and command-line tools;
- bare-metal C/C++;
- vendor SDKs and HALs;
- interpreted runtimes such as MicroPython or CircuitPython.

The transferable skill is not memorizing one menu layout.

It is being able to answer:

> What is this hardware, how does code get onto it, what software stack runs it, and what evidence proves the first program is really executing?

A successful first exploration should end with a simple observable program such as blinking an LED and a correct explanation of the complete path from source text to target behavior.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- distinguish an MCU/SoC from a development board;
- identify the target architecture and core family;
- locate the MCU data sheet/reference manual and board documentation;
- identify supply and GPIO logic voltage;
- identify the programming/debug path;
- distinguish an external probe, on-board probe, bootloader, USB mass-storage bootloader, and interpreter/runtime workflow;
- identify the host tool, SDK/framework, language, runtime, and deployed artifact;
- bring an unfamiliar supported platform to a minimal observable program using a verified recipe;
- explain how the new workflow differs from the PIC16F883/PICkit/MPLAB baseline;
- avoid claiming a board/toolchain recipe is supported until it has been tested end to end.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [PIC16F883 Architecture](pic16f883-architecture.md#architecture-overview);
- [MPLAB X, PIC Assembler, PSECTs, and Vectors](mplab-pic-as-psects-vectors.md#tool-roles);
- [Embedded C on the PIC16F883](embedded-c-pic16f883.md#build-path);
- [Processor Datapaths, Cycles, and Pipelining](processor-datapath-pipelining.md#isa-microarchitecture).

This guide teaches the **transfer method**.

Specific board recipes should be published separately only after the exact hardware/toolchain combination has been verified.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="layers"></a>
### Identify the layers separately

For every unfamiliar platform, name these layers when they exist:

| Layer | Question |
| --- | --- |
| MCU / SoC | What silicon actually executes the program? |
| Development board | What supporting hardware surrounds the MCU? |
| Processor architecture | What core/ISA is inside? |
| Programming/debug interface | ICSP, SWD, JTAG, UART bootloader, USB DFU, native USB, etc.? |
| Host tool | IDE, programmer utility, CLI, editor, REPL client? |
| SDK / HAL / framework | Vendor SDK, HAL, Arduino core, Pico SDK, ESP-IDF, etc.? |
| Language | Assembly, C, C++, Python-family runtime, Rust, other? |
| Runtime | Bare-metal firmware, RTOS, MicroPython, CircuitPython, etc.? |
| Artifact | HEX, ELF-derived image, BIN, UF2, source file, other? |
| Observable proof | LED, serial message, scope trace, debugger halt, other? |

Do not collapse all of those into the word “IDE.”

<a id="programming-path"></a>
### Programming path is not always the same as the USB cable you see

A host USB cable may connect to:

- an external or on-board debug probe;
- a USB-to-UART bridge feeding a serial bootloader;
- a native USB bootloader in the target MCU;
- a USB mass-storage bootloader;
- an interpreter/runtime exposing a filesystem or REPL.

The visible connector does not tell you the complete programming architecture.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="external-probe"></a>
### External programmer/debugger

A dedicated probe connects to the target's programming/debug interface.

PIC baseline:

```text
MPLAB X / IPE
-> PICkit
-> ICSP
-> PIC16F883 program memory
```

Other ecosystems may use SWD or JTAG with probes such as vendor debuggers, CMSIS-DAP devices, or J-Link-class tools.

The host builds a firmware image, then the probe programs/debugs the target.

<a id="onboard-probe"></a>
### On-board programmer/debugger

Some development boards include a second device that acts as the debug/programming probe.

The student may see only one USB cable, but internally the path may still be:

```text
host USB
-> on-board probe
-> SWD/JTAG
-> target MCU
```

Distinguish the target MCU from the helper/debug MCU on the board.

<a id="bootloader"></a>
### Bootloader or device-firmware-update path

A bootloader is already present in ROM or Flash and accepts new firmware through a supported transport.

Possible transports include:

- UART serial;
- native USB;
- USB DFU;
- other device-specific channels.

The important distinction is that the target's bootloader performs the programming rather than an external hardware debug probe.

<a id="mass-storage"></a>
### USB mass-storage firmware transfer

Some devices enter a boot mode that appears as a removable drive.

A firmware image is copied to that drive, then the device programs/reboots.

Example ecosystem: Raspberry Pi RP2040/RP2350 BOOTSEL with UF2 firmware images.

The file copied is a **firmware image**, not the ordinary source-code project.

<a id="runtime"></a>
### Interpreter/runtime plus REPL or source filesystem

Some boards run an interpreter such as MicroPython or CircuitPython.

Possible workflow:

```text
install runtime firmware
-> connect to REPL or mounted source filesystem
-> execute/transfer source
-> runtime interprets source on the MCU
```

This differs fundamentally from rebuilding and flashing a new native firmware image for every source edit.

A REPL is a **Read-Evaluate-Print Loop** that can execute commands interactively on the target runtime.

<a id="bringup-contract"></a>
### Standard bring-up contract

For a supported platform recipe:

1. identify the exact MCU and board;
2. find the MCU and board documentation;
3. identify supply and GPIO logic voltage;
4. locate an on-board LED or design a safe external LED;
5. identify the programming/debug path;
6. identify the host toolchain and language/runtime;
7. install or verify required software;
8. create/open the minimal project or runtime;
9. build/compile or connect to the interpreter;
10. program/transfer the program;
11. blink the LED at a stated period;
12. explain the full source-to-execution path;
13. record one piece of evidence;
14. state one important difference from the PIC16F883 workflow.

A recipe is not considered supported merely because a vendor says the board/toolchain should work. The exact course path must be exercised and documented.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="pic-baseline-example"></a>
### Worked example: PIC16F883 baseline

A course PIC assembly path is:

```text
main.S
-> preprocessor
-> PIC Assembler
-> linker
-> HEX/program image
-> MPLAB programming/debug operation
-> PICkit 3
-> ICSP
-> PIC16F883 Flash
-> reset vector
-> executing instructions
-> observable GPIO behavior
```

This is the comparison baseline for later platforms.

<a id="uf2-example"></a>
### Worked example: firmware image versus source file

Suppose a board enters a mass-storage bootloader and accepts a `.uf2` file.

Correct model:

```text
source project
-> compiler/linker/tool
-> UF2 firmware image
-> copy image to bootloader volume
-> bootloader programs target
-> reboot
```

Incorrect model:

> The MCU is directly executing my C source file from the USB drive.

The copied artifact is generated firmware.

<a id="runtime-example"></a>
### Worked example: interpreted source workflow

A MicroPython-style path may be:

```text
runtime firmware already installed
-> host opens serial/USB REPL
-> command entered
-> interpreter executes it on target
```

A persistent script can then be stored according to that runtime's filesystem/startup rules.

This separates:

- installing the runtime;
- interacting with the runtime;
- storing application source.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="exploration-checklist"></a>
### Exploration checklist

When you receive an unfamiliar board:

1. read the exact part number from the MCU and board;
2. identify the core/architecture;
3. locate the official board page;
4. locate schematic/pinout;
5. identify power and I/O voltage;
6. identify USB connector function;
7. determine whether a debug probe is on-board;
8. determine whether a bootloader exists;
9. identify the recommended host tool/SDK;
10. locate the simplest official blink or GPIO example;
11. determine the generated/deployed artifact;
12. build and program;
13. verify the actual LED pin/output;
14. measure the blink period if timing is part of the claim;
15. document the programming path in your own words.

<a id="troubleshooting"></a>
### Troubleshoot by layer

If “upload failed,” determine which layer failed:

```text
source/build?
-> generated image?
-> USB/driver?
-> programmer/debugger?
-> bootloader mode?
-> target power?
-> target reset?
-> debug/program interface?
-> application pin/configuration?
```

Do not reinstall an IDE to fix a wiring/power problem or rewrite source to fix an unrecognized USB probe until the failing layer is identified.

<a id="verification-boundary"></a>
### Tested-recipe boundary

A course recipe should say what was actually verified:

- exact board revision/model;
- host OS when relevant;
- tool/version range if it materially affects success;
- cable/probe path;
- source/example used;
- build result;
- program/transfer result;
- observable proof.

If that path has not been verified, present it as an exploration candidate, not a guaranteed student procedure.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Distinguish MCU from development board.
2. What is the difference between an external debug probe and a bootloader?
3. Why can one USB cable hide an on-board SWD/JTAG probe?
4. What is a UF2 file in a mass-storage bootloader workflow: source code or a generated firmware artifact?
5. What is a REPL?
6. Why is MicroPython source transfer different from compiling C into a native firmware image?
7. Name the nine layers in the course exploration model.
8. A board powers up but the host cannot see its programmer/debugger. Which troubleshooting layer should you investigate before changing GPIO code?
9. Why should a published course recipe name the exact hardware/toolchain path it actually tested?
10. What observable result makes a minimal bring-up more useful than “Build Successful” alone?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. MCU is the target silicon; the development board adds power, connectors, LEDs, debugger, headers, and other support.
2. A probe programs/debugs through a hardware interface; a bootloader is target-resident software/ROM that accepts a firmware image through a supported transport.
3. The cable can connect to a separate on-board debug device, which then talks to the target over SWD/JTAG.
4. Generated firmware artifact.
5. Read-Evaluate-Print Loop, an interactive interface to a target runtime/interpreter.
6. The interpreter/runtime is already installed and executes source through its runtime model; native C is compiled/linked into target machine code before deployment.
7. MCU/SoC, board, architecture, programming/debug interface, host tool, SDK/framework, language, runtime, artifact, plus observable proof as the verification layer.
8. USB/probe/power/interface detection, not the application GPIO source.
9. Students need a reproducible supported path rather than an untested collection of theoretically compatible parts/tools.
10. It proves the built/transferred program reached the target and affected real hardware behavior.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- name the layers of an embedded development ecosystem;
- distinguish programmer/debugger and bootloader paths;
- distinguish firmware-image transfer from interpreter/source deployment;
- trace the PIC16F883 baseline source-to-execution path;
- investigate an unfamiliar board systematically;
- troubleshoot at the correct layer;
- explain why exact recipes must be verified before they become course-supported instructions.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *MPLAB X IDE* — https://www.microchip.com/en-us/development-tool/MPLAB-X-IDE
  - PIC/MPLAB baseline development environment.

- Arm, *CMSIS-DAP* — https://arm-software.github.io/CMSIS_5/DAP/html/index.html
  - Example of a standardized debug-access probe model used in Arm ecosystems.

- Raspberry Pi, *Microcontroller Documentation* — https://www.raspberrypi.com/documentation/microcontrollers/
  - Official RP2040/RP2350, C/C++ SDK, MicroPython, and BOOTSEL/UF2 ecosystem documentation.

- Espressif Systems, *ESP-IDF Programming Guide — Get Started* — https://docs.espressif.com/projects/esp-idf/en/latest/esp32s3/get-started/
  - Official example of a modern vendor SDK/toolchain path.

- Arduino, *UNO R4 Minima* — https://docs.arduino.cc/hardware/uno-r4-minima/
  - Official example of a high-level framework/board workflow layered over a microcontroller platform.

- Adafruit, *The CIRCUITPY Drive* — https://learn.adafruit.com/welcome-to-circuitpython/the-circuitpy-drive
  - Public runtime/source-filesystem example for understanding interpreted deployment.

[Back to top](#top) · [Topics index](README.md)
