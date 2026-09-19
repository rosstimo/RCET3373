<a id="top"></a>

# RCET 3373 — Embedded C on the PIC16F883

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

Changing from assembly to C does not change the PIC16F883 hardware.

The same:

- registers;
- pins;
- timers;
- ADC;
- EUSART;
- MSSP;
- interrupts;
- memory limits;
- electrical constraints

still exist.

C changes how the programmer expresses the solution and how much work the compiler performs between source text and machine instructions.

The transferable skill is:

> Read C as a higher-level description, but verify hardware behavior in the data sheet and exact executed behavior in the generated machine code when it matters.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- describe the XC8 C build path from source to programmed image;
- distinguish C language behavior from PIC16F883 hardware behavior;
- explain the roles of `<xc.h>` and standard headers such as `<stdint.h>`;
- use fixed-width integer types when width matters;
- access SFRs using device-header symbols;
- explain why `volatile` matters for hardware/shared state and what it does **not** guarantee;
- explain how optimization can change generated instructions and timing;
- inspect generated assembly/disassembly for a C construct;
- explain why interrupt and peripheral setup sequences still come from device documentation;
- compare a small assembly implementation with its C equivalent.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [PIC16F883 Architecture](pic16f883-architecture.md#architecture-overview);
- [Data Memory, SFRs, and Banking](data-memory-sfrs-banking.md#core-model);
- [Timing Measurement and Compiled-Code Verification](measurement-c-timing.md#compiled-c);
- [Interrupts and Context Saving](interrupts-context-saving.md#context-saving).

Use the relevant peripheral guide for hardware behavior:

- [ADC](pic16f883-adc.md#core-model)
- [UART](uart-asynchronous-serial.md#core-model)
- [I2C](i2c.md#core-model)
- [SPI](spi.md#core-model)
- [Timers](timers.md#core-model)

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="build-path"></a>
### C source still becomes machine instructions

A simplified XC8 build path is:

```text
C source
-> preprocessor
-> compiler
-> generated target code / assembler stages
-> linker
-> executable/program image
-> PIC program memory
```

The processor never executes C source text.

When exact timing, memory placement, or instruction side effects matter, inspect the generated target code.

<a id="authority-layers"></a>
### Keep the authority layers separate

| Question | Primary authority |
| --- | --- |
| What does this C syntax mean? | C/XC8 compiler documentation |
| What symbol names map to device registers? | XC8 device headers + device documentation |
| What does writing `ADCON0` do to hardware? | PIC16F883 data sheet |
| What instruction sequence did the compiler generate? | build listing/disassembly |
| What happened on the physical pin? | measurement + electrical/device documentation |

The compiler cannot redefine the silicon.

The data sheet does not define C language syntax.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="headers"></a>
### `<xc.h>` provides device/toolchain definitions

XC8 projects commonly include:

```c
#include <xc.h>
```

For the selected target device, XC8 headers provide names for device registers, bits, and toolchain-supported definitions.

Examples may include names such as:

```c
PORTB
TRISB
ANSELH
TMR0
INTCON
```

Use the device data sheet to determine what those registers/bits mean.

Do not treat a header name as a substitute for reading the peripheral chapter.

<a id="stdint"></a>
### `<stdint.h>` makes integer width explicit

The standard C header:

```c
#include <stdint.h>
```

provides fixed-width types when the implementation supports them, such as:

```c
uint8_t
int8_t
uint16_t
int16_t
```

This is useful in embedded code because hardware registers, protocol fields, counters, and packed data often have exact widths.

Compare:

```c
unsigned int value;
```

with:

```c
uint16_t value;
```

The second communicates the intended width directly.

<a id="sfr-access"></a>
### C accesses the same SFRs

Assembly:

```asm
BANKSEL TRISB
CLRF    TRISB
```

C conceptually expresses the same desired register state:

```c
TRISB = 0x00;
```

The compiler handles the target instruction sequence and banking details needed for that access.

The hardware effect still comes from the PIC register definition.

This is one of the most important abstraction boundaries:

> C can hide the bank-selection instructions, but it does not remove banked hardware from the device.

<a id="bit-access"></a>
### Symbolic bit access improves intent

Device headers may provide bit-field structures/macros such as:

```c
TRISBbits.TRISB0 = 1;
```

or other toolchain-supported symbolic forms.

These can make intent clearer than manually constructed masks.

For portable logic that is not tied directly to device headers, masks and bitwise operators remain important.

Use the current XC8/device-header syntax rather than copying examples from a different PIC/compiler generation.

<a id="volatile"></a>
### `volatile` means the value can change outside ordinary code assumptions

A C compiler tries to eliminate unnecessary memory reads/writes when optimization proves they are redundant.

A `volatile` object tells the compiler that accesses are observable and the value may change in ways not visible from ordinary C flow.

This matters for things such as:

- memory-mapped hardware registers;
- variables modified by an ISR and read by main code;
- data shared with hardware mechanisms.

But `volatile` does **not** automatically provide:

- atomic multi-byte access;
- mutual exclusion;
- correct interrupt synchronization;
- hardware memory barriers for every architecture;
- thread safety.

It controls compiler treatment of accesses; it is not a complete concurrency mechanism.

<a id="configuration"></a>
### Configuration bits still need deliberate settings

XC8 C commonly expresses configuration settings with compiler-supported configuration pragmas.

The exact syntax/settings come from current XC8 documentation and the selected device.

The engineering sequence remains:

```text
determine required hardware configuration
-> verify device data sheet
-> express it using current compiler syntax
-> build
-> verify programmed behavior
```

Do not copy configuration bits from a project using different clock or watchdog hardware.

<a id="interrupts-c"></a>
### C interrupt functions still obey the PIC interrupt architecture

XC8 provides target-specific interrupt function syntax and compiler-generated entry/exit support.

But the application still must reason about:

- which event sets the flag;
- source enable;
- `PEIE` when applicable;
- `GIE`;
- flag clearing;
- source identification on the shared vector;
- shared variables;
- ISR duration/latency.

C makes the entry/exit mechanics easier to express. It does not change the one-vector PIC16F883 interrupt architecture.

<a id="optimization"></a>
### Optimization changes generated instructions

The compiler may:

- remove unused calculations;
- keep values in registers;
- combine operations;
- reorder operations when language rules permit;
- inline functions;
- choose different branch structures.

Therefore:

```text
same C source + different optimization/tool version
-> potentially different machine code
-> potentially different exact timing/size
```

Use [compiled-code verification](measurement-c-timing.md#compiled-c) when exact instruction timing matters.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="port-example"></a>
### Worked example: PORTB software state

A clear C model keeps desired output state separate from physical readback:

```c
#include <xc.h>
#include <stdint.h>

uint8_t portShadow = 0;

void updatePort(void)
{
    portShadow++;
    PORTB = portShadow;
}
```

This preserves the same reasoning used in the [PORTB guide](digital-io-portb-configuration.md#read-modify-write):

```text
software state -> update -> complete port write
```

<a id="volatile-example"></a>
### Worked example: ISR-updated flag

Conceptually:

```c
volatile uint8_t eventPending = 0;
```

An ISR sets the flag; main code checks and clears it.

`volatile` ensures the compiler continues to perform the required accesses rather than assuming the value cannot change unexpectedly.

It does not make a larger multi-byte data structure automatically safe from interruption halfway through access.

<a id="generated-code-example"></a>
### Worked example: exact timing question

Suppose C contains:

```c
PORTB++;
```

Do not answer:

> That takes one cycle.

Instead:

1. build with the actual XC8 version/settings;
2. inspect the generated instruction sequence;
3. determine whether the access requires banking/read/modify/write behavior;
4. count the actual target path;
5. measure if physical timing matters.

The C operator and the PIC instruction are different abstraction levels.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="c-debug-checklist"></a>
### Embedded-C troubleshooting checklist

If C code does not produce expected hardware behavior:

1. verify the device/peripheral setup from the data sheet;
2. verify the project target device;
3. verify configuration bits;
4. verify header/register names belong to that device;
5. inspect warnings;
6. inspect SFR values in debugger;
7. inspect generated code when abstraction may hide a detail;
8. verify optimization assumptions;
9. identify variables shared with ISR/hardware and their `volatile`/atomicity needs;
10. measure the physical signal.

Do not switch randomly between assembly and C until the layer containing the problem is identified.

<a id="translation-exercise"></a>
### Translate concepts, not source lines

A useful assembly-to-C comparison asks:

- What state is being represented?
- Which hardware register is affected?
- Which operations are compiler bookkeeping?
- Which hardware side effects remain?
- What did the compiler optimize away or add?

The goal is not one-to-one source translation.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Does the PIC16F883 execute C source directly?
2. What is the job of `<xc.h>`?
3. Why might `uint8_t` be preferable to an unspecified-width integer for a byte protocol field?
4. If C writes `TRISB = 0x00`, which document defines the hardware meaning of that write?
5. What does `volatile` tell the compiler?
6. Name two things `volatile` does not guarantee.
7. Why can exact loop timing change when optimization settings change?
8. What should you inspect to determine the actual instructions generated from C?
9. Does using C eliminate the need to understand `ADIF`, `TMR2IF`, or `RCIF`?
10. Why can a C interrupt function still require careful reasoning about shared data and ISR duration?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. No. The toolchain translates C into target machine instructions/program image.
2. It provides target/toolchain definitions, including device-specific register/bit names for the selected MCU.
3. It communicates the intended fixed width and avoids depending on implementation-specific ordinary integer width.
4. The PIC16F883 data sheet.
5. That accesses are observable and the object may change outside ordinary compiler-visible flow, so required reads/writes must not be optimized away improperly.
6. Examples: atomicity, mutual exclusion, interrupt safety for multi-byte data, universal memory barriers.
7. Optimization changes the generated target instruction sequence.
8. Listing/disassembly/generated assembly for the actual build.
9. No. The C program still controls the same hardware flags/registers.
10. Hardware events can occur asynchronously, variables can be shared, and long ISR work affects system latency even when compiler-generated entry/exit code is used.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- trace C source through the build into PIC machine code;
- separate C/compiler authority from device-data-sheet authority;
- explain `<xc.h>` and `<stdint.h>`;
- explain SFR access in C;
- explain `volatile` accurately;
- explain why optimization affects exact code/timing;
- compare assembly and C at the hardware-behavior level;
- inspect generated code when the abstraction must be opened.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *MPLAB XC8 C Compiler User's Guide for PIC MCU*, DS50002737 — https://ww1.microchip.com/downloads/aemDocuments/documents/DEV/ProductDocuments/ReferenceManuals/MPLAB-XC8-C-Compiler-Users-Guide-for-PIC-DS50002737.pdf
  - Used for: XC8 language extensions, device access, interrupts, optimization, headers, configuration, and generated-code/toolchain behavior.

- Microchip Technology Inc., *MPLAB XC8 Documentation and Downloads* — https://www.microchip.com/en-us/tools-resources/develop/mplab-xc-compilers/xc8
  - Used for: current XC8 documentation/release entry point.

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: hardware behavior of the registers/peripherals accessed from C.

- ISO/IEC 9899 C language library concept via implementation header `<stdint.h>`; use the XC8 compiler documentation to verify the supported fixed-width integer types for the selected target.
  - Used for: fixed-width integer-type rationale.

[Back to top](#top) · [Topics index](README.md)
