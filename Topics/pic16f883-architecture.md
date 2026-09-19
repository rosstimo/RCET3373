# RCET 3373 - PIC16F883 Datasheet and Architecture Self-Learning Guide

This guide covers the PIC16F883 architecture concepts introduced immediately after the computer-memory unit. Keep the PIC16F882/883/884/886/887 data sheet open while working through it. The goal is not to memorize a register map. The goal is to be able to find the information and reason from it.

# What you should be able to do

After completing this guide, you should be able to:

- locate PIC16F883-specific information in a family data sheet;
- distinguish program memory from data memory;
- interpret `4K x 14` program-memory notation;
- explain the reset vector, interrupt vector, Program Counter, and hardware return stack at a basic level;
- distinguish Special Function Registers (SFRs) from General Purpose Registers (GPRs);
- explain why PIC16F883 data memory is divided into banks;
- determine the `RP1:RP0` values needed to select a bank;
- explain why a file-register instruction contains only seven address bits;
- interpret `f`, `d`, `b`, and `k` in the instruction-set summary;
- construct simple `BCF` and `BSF` instructions;
- explain the difference between a register's full data-memory address and the address field encoded in a machine instruction;
- identify the documentation needed to configure the oscillator and safely connect hardware.

# 1. Start by using the right document

The PIC16F883 is one member of a device family. Its data sheet also covers the PIC16F882, PIC16F884, PIC16F886, and PIC16F887.

That matters because nearby diagrams can look nearly identical while describing different program-memory sizes, pin counts, or peripheral resources.

When you use the data sheet:

1. read the title of the figure or table;
2. verify that it includes **PIC16F883**;
3. prefer section numbers when recording references in your notes;
4. mark the pages or PDF bookmarks you use repeatedly.

Useful sections for this topic are:

- Section 1.0 - Device Overview
- Section 2.0 - Memory Organization
- Section 2.2.2.1 - STATUS Register
- Section 3.0 - I/O Ports
- Section 4.0 - Oscillator Module
- Section 15.0 - Instruction Set Summary
- Section 17.0 - Electrical Specifications

The **device data sheet** is the primary source for exact PIC16F883 behavior. The **PICmicro Mid-Range MCU Family Reference Manual** is useful when you need a broader explanation of classic PIC architecture. The **MPLAB XC8 PIC Assembler documentation** is the source for current assembler syntax and linker behavior.

# 2. The pinout is physical; the architecture diagram is logical

A microcontroller pin can serve several purposes.

For example, a label such as:

```text
MCLR / VPP / RE3
```

means that the same physical package pin participates in several possible functions:

- reset input (`MCLR`);
- programming voltage (`VPP`);
- an I/O-related function (`RE3`) under the conditions supported by the device.

Other pins can be shared between:

- digital I/O;
- analog inputs;
- serial communications;
- timer or capture/compare functions;
- oscillator functions.

This is why a logical block diagram can appear to show a pin in more than one place. It is showing internal functional connections, not drawing the physical package literally.

## Retrieval check

Why can you not assume that a pin labeled `RA0` will behave as an ordinary digital I/O pin immediately after reset?

Because the same pin may have alternate peripheral or analog functions that must be configured appropriately before ordinary digital I/O behavior is available.

# 3. Find the computer inside the PIC16F883

The device overview contains the same major ideas that appear in a generic computer architecture diagram:

- **program memory** for instructions;
- **data memory / register file** for working state and hardware-control registers;
- **ALU** for arithmetic and logic operations;
- **W register** as the working accumulator used by many instructions;
- **STATUS register** for arithmetic status, reset state, and data-memory bank selection;
- **Program Counter (PC)** for selecting the next instruction address;
- **hardware return stack** for return addresses;
- **clock / oscillator** for timing;
- **I/O ports** for external digital signals;
- **peripherals** such as timers, ADC, serial interfaces, and comparators.

You do not need to understand every block yet. The important step is recognizing the structure.

# 4. Program memory: what does `4K x 14` mean?

The PIC16F883 program memory is organized as:

```text
4K x 14
```

Break that into two questions.

## How many locations?

In digital memory notation:

```text
1K = 1024 = 2^10
4K = 4096 = 2^12
```

Therefore, 12 changing address bits are enough to select 4096 locations.

The valid implemented program-memory addresses are:

```text
0x0000 through 0x0FFF
```

The highest address is `0x0FFF`, which is 4095 decimal.

That does **not** mean there are only 4095 locations.

Count the endpoints:

```text
address 0       = first location
address 1       = second location
...
address 4095    = 4096th location
```

So:

```text
0 through 4095 = 4096 possible addresses
```

## How wide is each location?

Each program-memory location is 14 bits wide.

For this classic PIC architecture, those 14-bit words hold machine instructions.

That is different from a processor whose instruction memory is described purely as 8-bit bytes. Do not automatically convert a 14-bit instruction word into "two bytes." The instruction word is specifically 14 bits wide.

## Worked example 1 - address count

**Known:** program memory is `4K x 14`.

**Determine:** number of locations, number of address bits needed, highest implemented address.

**Reasoning:**

```text
4K = 4 x 1024
   = 4096
   = 2^12
```

So 12 variable address bits can identify all implemented locations.

The largest 12-bit unsigned value is:

```text
1111 1111 1111 binary
= 0xFFF
= 4095 decimal
```

**Result:**

- 4096 locations;
- 12 bits are enough to identify those implemented locations;
- highest implemented address = `0x0FFF`;
- each location = 14 bits wide.

# 5. Why does the data sheet say the Program Counter is 13 bits?

The PIC16F883 has a **13-bit Program Counter**.

At first this can look inconsistent:

```text
4K locations -> 12 bits needed
Program Counter -> 13 bits
```

They are different facts.

The classic mid-range core has a 13-bit PC architecture. This particular device implements 4K words of program memory. The data sheet specifies what happens when program addresses outside the implemented range are accessed.

Do not force the Program Counter width to equal the minimum number of bits required by the installed memory.

# 6. Reset vector and interrupt vector

Two early program-memory addresses have special entry roles:

```text
0x0000  reset vector
0x0004  interrupt vector
```

## Reset vector

When the processor resets, execution begins from the reset vector at `0x0000`.

Reset can be associated with events such as power-up or other reset mechanisms supported by the device.

The important model is:

```text
reset occurs
    ↓
Program Counter is directed to 0x0000
    ↓
code placed there begins the startup path
```

## Interrupt vector

When an enabled interrupt is accepted, the processor redirects execution to `0x0004`.

A common misconception is:

> "If the Program Counter reaches 0x0004 during normal execution, does that cause an interrupt?"

No.

Address `0x0004` is still a normal program-memory location. Its special role is that the interrupt hardware sends execution there when an interrupt is accepted.

Normal execution can pass through that address like any other address.

# 7. The hardware return stack

The PIC16F883 has an 8-level hardware return stack.

Its main job is to hold return addresses when control flow temporarily leaves the current location, such as for subroutine calls and interrupts.

For now, remember:

- it stores return addresses;
- it is separate from ordinary GPR data memory;
- the processor manages it automatically for the relevant control-flow operations;
- it is not a general-purpose RAM buffer for your variables.

The details matter later when you use `CALL`, `RETURN`, and interrupts.

# 8. Program memory and data memory are different structures

A useful simplified comparison is:

| Program memory | Data memory / register file |
|---|---|
| Stores executable instruction words | Stores working data and hardware-control/status registers |
| 14-bit program words | 8-bit file registers |
| Selected by program addressing / PC | Selected by file-register addressing and bank selection |
| Nonvolatile Flash for firmware | GPR portion is working SRAM; SFR addresses connect to hardware state |

This separation is a major feature of the PIC architecture.

# 9. Data memory: SFRs and GPRs

PIC documentation calls many data-memory locations **file registers**.

Two categories matter immediately.

## Special Function Registers (SFRs)

SFRs have defined hardware meanings.

Examples include:

- `STATUS`
- `PORTA`
- `PORTB`
- `PORTC`
- `TRISA`
- `TRISB`
- `TRISC`
- timer registers
- ADC registers
- serial-interface registers

Writing an SFR can configure or control hardware.

For example, a TRIS bit affects whether the associated port pin is configured as an input or output.

## General Purpose Registers (GPRs)

GPRs are ordinary working RAM locations available to your program.

They can hold:

- variables;
- counters;
- intermediate results;
- temporary state.

The hardware does not assign a peripheral-control meaning to an ordinary GPR location.

# 10. Why are there four data-memory banks?

The PIC16F883 data-memory map is divided into four banks.

A simplified view is:

```text
Bank 0
Bank 1
Bank 2
Bank 3
```

The full register map gives addresses such as:

```text
PORTA  = 0x05
TRISA  = 0x85
PORTC  = 0x07
TRISC  = 0x87
```

The upper portion of the full address identifies which bank contains the target register.

Some registers are mirrored so that they can be accessed through addresses in several banks. `STATUS` is an important example because software needs to alter bank selection regardless of the bank currently selected.

There is also a common GPR area whose underlying RAM can be accessed through corresponding addresses in multiple banks.

# 11. The key derivation: `PORTA = 0x05` and `TRISA = 0x85`

This is the part you should be able to reason through without memorizing a rule.

Consider:

```text
PORTA = 0x05
TRISA = 0x85
```

Convert the values to binary.

```text
0x05 = 0000 0101
0x85 = 1000 0101
```

Now look ahead at the PIC instruction format. A file-register instruction only contains **seven bits** for the `f` address field.

The low seven bits are:

```text
PORTA low 7 bits = 0000101
TRISA low 7 bits = 0000101
```

They are the same.

Therefore the seven-bit `f` field by itself cannot distinguish PORTA from TRISA.

Something else must select the bank.

That "something else" is the direct-address bank selection in the STATUS register.

# 12. STATUS contains the bank-select bits

The STATUS register includes:

```text
bit 7  IRP
bit 6  RP1
bit 5  RP0
bit 4  TO
bit 3  PD
bit 2  Z
bit 1  DC
bit 0  C
```

For direct file-register addressing, the important bits are:

```text
RP1 RP0
 0   0  -> Bank 0
 0   1  -> Bank 1
 1   0  -> Bank 2
 1   1  -> Bank 3
```

This gives a useful conceptual model:

```text
RP1:RP0       seven-bit f field
 bank         register offset
   \              /
    \            /
     target file register
```

## Worked example 2 - which bank contains TRISC?

From the register map:

```text
TRISC = 0x87
```

The `0x80` portion indicates Bank 1 in this map.

Therefore:

```text
RP1:RP0 = 01
```

The low seven-bit offset is:

```text
0x87 -> low 7 bits -> 0x07
```

So at the machine-addressing level:

```text
Bank 1 + offset 0x07 -> TRISC
```

# 13. The instruction format confirms the model

The PIC16F883 instruction word is 14 bits wide.

The instruction-set summary groups instructions into three broad forms.

## Byte-oriented file-register operations

Conceptually:

```text
opcode | d | ffffffff?
```

For this architecture the `f` field is seven bits.

- `f` = file-register address field, `0x00` through `0x7F` in the encoded operand;
- `d` = destination choice.

For byte-oriented instructions:

```text
d = 0 -> result goes to W
d = 1 -> result goes back to file register f
```

## Bit-oriented file-register operations

These include:

- a file-register field `f`;
- a bit number `b` from 0 through 7.

Examples include `BCF` and `BSF`.

## Literal/control operations

These use `k` for a literal value or control-flow field, depending on the instruction.

For example:

```asm
MOVLW 0x5A
```

loads the literal value `0x5A` into W.

# 14. First manual bank-selection code

Suppose you want to select **Bank 2**.

From the bank table:

```text
Bank 2 -> RP1:RP0 = 10
```

From the STATUS map:

```text
RP1 = bit 6
RP0 = bit 5
```

A simple manual sequence is:

```asm
BCF 0x03, 5
BSF 0x03, 6
```

Read it literally.

```text
BCF 0x03,5
```

- `BCF` = Bit Clear f;
- `0x03` = STATUS register's low file-register offset;
- `5` = RP0;
- result: RP0 becomes 0.

```text
BSF 0x03,6
```

- `BSF` = Bit Set f;
- `0x03` = STATUS;
- `6` = RP1;
- result: RP1 becomes 1.

After both instructions:

```text
RP1:RP0 = 10 -> Bank 2
```

# 15. A port-bit example

Suppose Bank 0 is selected and you execute:

```asm
BCF 0x05, 0
```

At the register-map level:

```text
Bank 0 + offset 0x05 -> PORTA
```

Therefore the instruction clears bit 0 in PORTA.

Similarly:

```asm
BSF 0x05, 3
```

sets bit 3 in PORTA.

## Important hardware warning

Changing a PORT register bit does not automatically guarantee the external pin behaves the way you imagine.

The pin's behavior also depends on configuration such as:

- TRIS direction;
- analog/digital selection;
- enabled peripheral functions;
- electrical loading.

That is why real I/O initialization requires more than writing a PORT value.

# 16. Full register address versus encoded instruction operand

This distinction is easy to miss.

The data sheet might show:

```text
TRISC = 0x87
```

That is useful because it identifies the register's complete location in the banked data-memory map.

But a classic file-register instruction has only seven bits available in `f`.

At the hardware level:

```text
Bank 1 selected separately
+
0x07 encoded in f
=
TRISC targeted
```

# 17. What the assembler does for you

Modern source code normally uses register symbols rather than raw addresses.

With the XC8 PIC Assembler include file, code can look like:

```asm
#include <xc.inc>

BANKSEL TRISC
BSF     BANKMASK(TRISC), 2
```

There are two separate ideas here.

## `BANKSEL TRISC`

`BANKSEL` is assembler-level help. It generates the bank-selection instruction sequence appropriate for the target register and device.

The CPU does not execute an instruction whose native opcode is literally "BANKSEL" on this classic core.

## `BANKMASK(TRISC)`

The symbol `TRISC` represents its full address. `BANKMASK()` removes the bank portion so the file-register operand fits the instruction field.

Depending on linker settings, PIC Assembler can also be configured to truncate the extra address bits automatically. That is a toolchain choice. It does not change the hardware addressing model.

The safest learning sequence is:

1. understand the full address;
2. understand bank + 7-bit offset;
3. understand what `BANKSEL` and `BANKMASK()` do for source-code readability and tool correctness.

# 18. W and STATUS flags

W is the PIC's working accumulator.

Many instructions use W for:

- loading literals;
- arithmetic;
- logic;
- moving data between registers.

But do not use the overly literal rule "everything passes through W." `BCF` and `BSF`, for example, operate directly on a bit in a file register.

STATUS also contains arithmetic flags:

- `Z` = Zero;
- `DC` = Digit Carry/Borrow behavior;
- `C` = Carry/Borrow behavior.

For now, the most useful arithmetic flag to understand is `Z`.

If an instruction that affects `Z` produces a zero result, the Zero flag is set. Later, code can test that flag to make decisions.

Detailed carry/borrow behavior is best learned while working actual arithmetic examples.

# 19. The oscillator is part of the architecture

The processor cannot execute instructions without a clock.

The PIC16F883 supports internal and external clock sources.

For external quartz crystals or ceramic resonators, the device supports LP, XT, and HS modes. These modes select different oscillator amplifier gain ranges.

In the RCET hardware, a **4 MHz external crystal** is used for the introductory setup.

The data sheet is the primary source for:

- which oscillator pins are used;
- which configuration mode applies;
- the supported frequency range;
- startup behavior.

The crystal manufacturer's data sheet is needed for the crystal's specified electrical parameters, including load capacitance.

# 20. Do not treat a crystal as an ideal 4.000000 MHz source

A real oscillator circuit depends on more than the frequency printed on the crystal.

Important factors include:

- crystal frequency tolerance;
- specified load capacitance;
- PCB and pin parasitic capacitance;
- drive level;
- equivalent series resistance;
- supply voltage;
- temperature;
- board layout.

An external crystal is useful because it can provide a stable, predictable timebase, but "external crystal" does not mean "exact under every condition."

For a production-quality design, oscillator performance should be verified over the expected operating conditions.

# 21. Internal oscillator correction

The PIC16F883 also has internal oscillator resources.

The device includes:

- an 8 MHz high-frequency internal oscillator;
- a 31 kHz low-frequency internal oscillator;
- software-selectable high-frequency divisions including 4 MHz and lower values.

So the internal oscillator is not limited to one 4 MHz operating choice.

This matters later when selecting between board simplicity, pin availability, power, and timing accuracy requirements.

# 22. Before connecting a load, read the electrical specifications

A correct instruction can still damage hardware if the electrical design is wrong.

Before connecting LEDs or other loads, find:

- operating supply-voltage range;
- allowed input voltage ranges;
- input logic thresholds;
- output-high and output-low characteristics;
- maximum source/sink current per pin;
- aggregate current limits;
- total power limits.

## Absolute maximum is not the normal operating target

An **absolute maximum rating** is a stress boundary. It tells you where permanent damage can occur.

It is not the value you should intentionally operate at continuously.

For example, if the absolute maximum says an I/O pin can source or sink up to 25 mA, that does not mean "design every LED for 25 mA."

For normal design, also inspect the DC characteristics that guarantee output voltage at specified current.

# Worked example 3 - select a bank and change a TRIS bit

**Goal:** set bit 2 of TRISC.

**Step 1: Find TRISC.**

From the register map:

```text
TRISC = 0x87
```

**Step 2: Determine the bank.**

`0x87` is in Bank 1.

```text
RP1:RP0 = 01
```

**Step 3: Determine the low file-register offset.**

```text
0x87 -> offset 0x07
```

**Step 4: Select Bank 1 manually.**

```asm
BCF 0x03, 6    ; RP1 = 0
BSF 0x03, 5    ; RP0 = 1
```

**Step 5: Set bit 2 at offset 0x07.**

```asm
BSF 0x07, 2
```

**Meaning:** set TRISC bit 2.

Remember that on PIC TRIS registers, setting a TRIS bit configures the corresponding pin as an input. Clearing the TRIS bit configures it as an output.

**Assembler-oriented version:**

```asm
#include <xc.inc>

BANKSEL TRISC
BSF     BANKMASK(TRISC), 2
```

The second form is much nicer to maintain, but the first form explains what the hardware model is doing.

# Worked example 4 - identify a wrong-bank bug

Suppose the programmer intends to clear PORTA bit 0 and writes:

```asm
BCF 0x05, 0
```

But `RP1:RP0 = 01` at that moment.

The seven-bit offset is still `0x05`, but Bank 1 is selected.

Therefore the processor is not targeting Bank 0 PORTA. It is targeting the Bank 1 register at the same offset.

The instruction can be perfectly legal and still do the wrong thing.

This is why bank state is part of program correctness.

# Practice problems

## Part A - program memory

1. Convert `4K` program-memory locations to an exact decimal count.
2. How many changing address bits are needed to identify 4096 locations?
3. What is the largest 12-bit unsigned binary value in decimal and hexadecimal?
4. Why does an address range of `0x0000..0x0FFF` contain 4096 locations?
5. How many bits are stored in each PIC16F883 program-memory word?
6. What address is the reset vector?
7. What address is the interrupt vector?
8. Does normal execution through the interrupt-vector address itself create an interrupt? Explain.

## Part B - data memory and banks

9. In one sentence, distinguish an SFR from a GPR.
10. Which two STATUS bits select the bank for direct addressing?
11. What `RP1:RP0` values select Bank 0? Bank 1? Bank 2? Bank 3?
12. `PORTC` is at `0x07` and `TRISC` is at `0x87`. What are the low seven address bits of each?
13. Why can a seven-bit `f` field not distinguish those two registers by itself?
14. If `RP1:RP0 = 10`, which bank is selected?
15. Why is it useful for STATUS to be accessible through mirrored addresses?

## Part C - instructions

16. In the instruction-set notation, what does `f` mean?
17. What does `b` mean?
18. What does `d = 0` mean for a byte-oriented instruction?
19. What does `d = 1` mean?
20. What does `k` usually represent?
21. Translate `BCF 0x05,0` into plain language, assuming Bank 0 is selected.
22. Write two manual instructions that leave the processor in Bank 3.
23. Write two manual instructions that leave the processor in Bank 0.
24. Explain why `BANKSEL` is useful even though it is not a native instruction executed as a single PIC opcode.
25. What problem does `BANKMASK()` solve in current PIC Assembler source?

## Part D - oscillator and hardware

26. Which data-sheet section should you search first for PIC16F883 clock modes?
27. Why should you consult the crystal manufacturer's data sheet before choosing load capacitors?
28. Give three real-world factors that can make an oscillator differ from the nominal crystal frequency or fail to start correctly.
29. Why is an absolute-maximum current not a good normal design current?
30. What additional electrical table should you use when deciding how much output current a pin should source or sink while still maintaining valid logic levels?

# Answer key

## 1

`4K = 4 x 1024 = 4096` locations.

## 2

12 bits, because `2^12 = 4096`.

## 3

`2^12 - 1 = 4095 decimal = 0xFFF`.

## 4

The range includes zero. Counting all integer addresses from 0 through 4095 gives 4096 distinct addresses.

## 5

14 bits per program-memory word.

## 6

`0x0000`.

## 7

`0x0004`.

## 8

No. `0x0004` is still a program-memory location. Interrupt hardware redirects execution there when an interrupt is accepted. Merely executing code that happens to reside at address 4 does not generate an interrupt.

## 9

An SFR has a defined hardware control/status function; a GPR is ordinary working RAM available to software.

## 10

`RP1` and `RP0` in STATUS.

## 11

```text
Bank 0 = 00
Bank 1 = 01
Bank 2 = 10
Bank 3 = 11
```

## 12

Both have the same low seven-bit offset corresponding to `0x07`.

## 13

The bank information is not contained in the seven-bit field. The bank must be selected separately.

## 14

Bank 2.

## 15

Software can access STATUS while in any bank, which is especially useful because STATUS contains the direct bank-selection bits.

## 16

The file-register address field used by the instruction.

## 17

A bit number within an 8-bit file register.

## 18

Store the result in W.

## 19

Store the result in file register `f`.

## 20

A literal constant or instruction-specific literal/control field.

## 21

Clear bit 0 of the file register at offset `0x05`. With Bank 0 selected, that is PORTA bit 0.

## 22

Bank 3 requires `RP1:RP0 = 11`.

```asm
BSF 0x03, 6
BSF 0x03, 5
```

## 23

Bank 0 requires `RP1:RP0 = 00`.

```asm
BCF 0x03, 6
BCF 0x03, 5
```

## 24

`BANKSEL` lets the assembler generate the correct bank-selection sequence for a symbol, reducing manual bit-setting errors while still respecting the hardware's banked addressing requirement.

## 25

A register symbol can represent a full banked data-memory address, while a classic file-register instruction has only a limited `f` field. `BANKMASK()` removes the bank portion from the operand so it fits the instruction after the bank has been selected.

## 26

Section 4.0, Oscillator Module.

## 27

The crystal's load capacitance is a crystal-specific requirement. The correct external capacitors depend on that specification plus pin/board parasitics, not merely on the nominal frequency.

## 28

Any three of: crystal tolerance, temperature, supply voltage, load capacitance, parasitic capacitance, drive level, equivalent series resistance, PCB layout, amplifier gain/startup margin.

## 29

Absolute maximum is a stress/damage boundary, not a guaranteed continuous operating point. Good design stays within guaranteed operating specifications with margin.

## 30

Use the DC electrical characteristics, especially the rows specifying output-high/output-low voltage at stated source/sink currents and supply conditions.

# What to be able to explain without notes

Before moving on, you should be able to explain these out loud:

- why `4K x 14` means 4096 instruction locations and a 14-bit word;
- why 4095 is the highest address but not the number of locations;
- what the Program Counter does;
- why `0x0000` and `0x0004` are called vectors;
- the difference between program memory and data memory;
- the difference between an SFR and a GPR;
- why `PORTA = 0x05` and `TRISA = 0x85` expose the need for bank selection;
- how `RP1:RP0` selects the bank;
- why the instruction's `f` field is only seven bits;
- what `f`, `d`, `b`, and `k` mean;
- what the two lines `BCF 0x03,5` and `BSF 0x03,6` do;
- why `BANKSEL` and `BANKMASK()` exist;
- why a crystal oscillator still requires engineering rather than assuming the printed frequency is perfect;
- why the electrical-specification chapter matters before connecting LEDs or other loads.

# References used to build this guide

Microchip Technology, **PIC16F882/883/884/886/887 Data Sheet**, DS40001291H.  
https://ww1.microchip.com/downloads/en/devicedoc/40001291h.pdf

Microchip Technology, **PICmicro Mid-Range MCU Family Reference Manual**, DS33023A.  
https://ww1.microchip.com/downloads/en/DeviceDoc/33023a.pdf

Microchip Technology, **MPLAB XC8 PIC Assembler User's Guide**.  
https://ww1.microchip.com/downloads/aemDocuments/documents/DEV/ProductDocuments/UserGuides/MPLAB-XC8-PIC-Assembler-Users-Guide-DS50002974.pdf

Microchip Technology, **AN849 - Basic PICmicro Oscillator Design**.  
https://www.microchip.com/en-us/application-notes/an849
