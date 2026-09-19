<a id="top"></a>

# RCET 3373 — Data Memory, SFRs, and Banking

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

PIC16F883 data memory contains both ordinary working RAM and registers connected directly to hardware.

The same file-register instructions can access both, but the consequences are very different:

- writing a GPR changes software data;
- writing an SFR can change pin direction, start a peripheral, clear a flag, or alter system state.

The classic mid-range PIC also uses **banked data memory**, so the register reached by an instruction depends on both the instruction's file-register field and the currently selected bank.

Understanding that model prevents a common class of bugs: perfectly valid instructions that access the wrong physical register.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- distinguish SFRs from GPRs;
- read the PIC16F883 data-memory map;
- explain why banked data memory exists;
- determine the bank and low file-register offset from a full register address;
- determine the `RP1:RP0` state needed for direct addressing;
- explain what `BANKSEL` does and does not do;
- explain why `BANKMASK()` is useful in PIC Assembler source;
- use register reset values as part of a configuration plan;
- trace a short initialization sequence and identify wrong-bank errors.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [PIC16F883 Architecture](pic16f883-architecture.md#data-memory);
- [Digital Representation](digital-representation.md#bit-fields).

Related topics:

- [MPLAB X, PIC Assembler, PSECTs, and Vectors](mplab-pic-as-psects-vectors.md#tool-roles);
- [PORTB Configuration](digital-io-portb-configuration.md#core-model).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="sfrs-and-gprs"></a>
### SFRs and GPRs share the data-memory address space

A **General Purpose Register (GPR)** is ordinary working RAM.

A **Special Function Register (SFR)** is an addressable data-memory location whose bits control or report hardware behavior.

Examples:

| Register/location | Category | Meaning |
| --- | --- | --- |
| `counter` in allocated RAM | GPR | software variable |
| `STATUS` | SFR | flags + bank-selection state |
| `PORTB` | SFR | port data/readback |
| `TRISB` | SFR | PORTB direction control |
| `TMR0` | SFR | Timer0 count/state |

Both are addressed as file registers, but only SFRs have device-defined hardware meaning.

<a id="full-address-offset"></a>
### Full address versus instruction offset

The data sheet may show:

```text
PORTC = 0x07
TRISC = 0x87
```

The low seven address bits of both are:

```text
0x07
```

The instruction's file-register field cannot distinguish those two locations by itself.

Bank-selection state supplies the missing part of the address.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="bank-selection"></a>
### Direct bank selection with `RP1:RP0`

For direct file-register addressing, STATUS bits `RP1:RP0` select the bank:

| `RP1:RP0` | Bank |
| --- | ---: |
| `00` | 0 |
| `01` | 1 |
| `10` | 2 |
| `11` | 3 |

The conceptual address is:

```text
bank-selection state + 7-bit file-register field -> target register
```

That is why `PORTC = 0x07` and `TRISC = 0x87` can share the same low seven-bit offset while remaining different registers.

**Official visual reference:** In Section 2.0, **Memory Organization**, of the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf), inspect the data-memory maps for the PIC16F883.

Focus on the repeated low offsets across banks, mirrored/common registers, and the SFR/GPR regions.

<a id="status-register"></a>
### STATUS is special because banking depends on it

The STATUS register contains the direct-address bank-select bits.

It is available through mirrored locations so software can reach it while different banks are active.

Relevant bits include:

```text
bit 6  RP1
bit 5  RP0
```

The same STATUS register also contains arithmetic/status flags, so bank selection and arithmetic state coexist in one SFR.

<a id="banksel"></a>
### `BANKSEL` makes bank intent explicit

Readable PIC Assembler source normally uses:

```asm
BANKSEL TRISB
CLRF    TRISB
```

`BANKSEL TRISB` does **not** clear or configure `TRISB`.

It causes the assembler to generate the bank-selection sequence required so that the following file-register access reaches `TRISB`.

The CPU does not execute one native opcode named `BANKSEL` on this classic core.

<a id="bankmask"></a>
### `BANKMASK()` fits a full symbol into the file-register field

A register symbol can represent the full banked address.

A classic file-register instruction has only the low file-register field available in the instruction word.

`BANKMASK()` removes the bank portion from the operand after the correct bank has been selected.

Example:

```asm
BANKSEL TRISC
BSF     BANKMASK(TRISC), 2
```

The hardware model remains:

```text
selected bank + low register offset -> target SFR
```

Assembler helpers make that model safer to express; they do not change the silicon architecture.

<a id="reset-state"></a>
### Reset values are part of configuration

Before writing initialization code, check what the device already does after reset.

A useful register-planning table is:

| Bit/field | Meaning | Reset state | Required state | Need to change? | Reason |
| --- | --- | --- | --- | --- | --- |
| | | | | | |

Do not clear every nearby register “just to be safe.”

Initialization should be a reasoned transition from documented reset state to intended application state.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="trisc-example"></a>
### Worked example: locate `TRISC`

**Known**

```text
TRISC = 0x87
```

**Find**

Bank and low file-register offset.

**Reasoning**

`0x87` lies in Bank 1.

Its low seven bits correspond to `0x07`.

**Result**

```text
Bank 1 + offset 0x07 -> TRISC
```

Therefore direct access requires `RP1:RP0 = 01`.

<a id="manual-bank-example"></a>
### Worked example: manually select Bank 2

Bank 2 requires:

```text
RP1:RP0 = 10
```

Since STATUS is at low offset `0x03`, one manual sequence is:

```asm
BCF 0x03, 5    ; RP0 = 0
BSF 0x03, 6    ; RP1 = 1
```

After both instructions, Bank 2 is selected.

This is useful for understanding the hardware. In normal source, prefer `BANKSEL` when appropriate.

<a id="initialization-trace"></a>
### Worked example: trace initialization intent

```asm
BANKSEL ANSELH
CLRF    ANSELH

BANKSEL TRISB
CLRF    TRISB

BANKSEL PORTB
CLRF    PORTB
```

For each pair, ask:

1. Which register is intended?
2. Which bank must be active?
3. What value is written?
4. What hardware behavior changes?
5. Is that write justified by the documented reset state and application need?

The bank-selection line prepares addressing. The register-write line changes the target.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="wrong-bank-bug"></a>
### Wrong-bank bug

Suppose code executes:

```asm
BCF 0x05, 0
```

The programmer intends to clear `PORTA` bit 0.

If `RP1:RP0 = 01`, Bank 1 is active.

The instruction is legal, but the target is the Bank 1 register at low offset `0x05`, not Bank 0 `PORTA`.

This is why bank state is part of program correctness.

<a id="bank-debug-checklist"></a>
### Banking debug checklist

When an SFR write appears to do nothing:

1. confirm the target symbol/address in the data sheet;
2. confirm the expected bank;
3. inspect STATUS `RP1:RP0`;
4. inspect the actual assembled/disassembled instruction sequence;
5. verify that a skip instruction did not accidentally skip only part of a generated bank-selection sequence;
6. verify that later code did not change bank state before the access;
7. use `BANKSEL` intentionally rather than relying on an old bank assumption.

For course style rules around `BANKSEL`, see the [RCET PIC-AS Style Guide](https://github.com/rosstimo/pic_projects/blob/main/RCET_PIC-AS_Style_Guide.md#10-banking-rules).

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. In one sentence, distinguish an SFR from a GPR.
2. Why can `PORTC = 0x07` and `TRISC = 0x87` share the same low file-register offset?
3. Which `RP1:RP0` values select Banks 0, 1, 2, and 3?
4. Which bank contains `TRISC = 0x87`?
5. What low file-register offset is used for `TRISC`?
6. What does `BANKSEL TRISB` accomplish?
7. Does `BANKSEL TRISB` write a value to `TRISB`?
8. Why is `BANKMASK(TRISC)` useful?
9. Why do reset values belong in an initialization plan?
10. If `BCF 0x05,0` executes while Bank 1 is selected, why might the code be valid yet wrong?
11. Why is explicit bank selection usually safer than assuming a previous routine left the correct bank active?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. An SFR has device-defined hardware control/status meaning; a GPR is ordinary software working RAM.
2. The instruction field carries only the low offset; separate bank-selection state identifies the bank.
3. Bank 0 = `00`, Bank 1 = `01`, Bank 2 = `10`, Bank 3 = `11`.
4. Bank 1.
5. `0x07`.
6. It causes the assembler to establish the bank needed to access `TRISB`.
7. No. A later instruction performs the actual register write/read.
8. It removes the bank portion of a full symbol so the operand fits the file-register field after bank selection.
9. They tell you which configuration is already present and which bits actually require changes.
10. The instruction operates on low offset `0x05` in the currently selected bank, which may not be `PORTA`.
11. Bank state is shared execution state and may be changed by surrounding code; explicit selection documents and restores intent.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- distinguish SFRs and GPRs;
- explain full register address versus low file-register offset;
- derive bank selection from a full register address;
- state the `RP1:RP0` bank table;
- explain what `BANKSEL` does;
- explain what `BANKMASK()` does;
- use reset values to justify minimal initialization;
- identify a wrong-bank bug from code and STATUS state.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Section 2.0 "Memory Organization" and STATUS/register-map material — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: SFR/GPR maps, bank addresses, STATUS bits, reset values, and device-specific register locations.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A, Memory Organization material — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: classic mid-range banked data-memory architecture.

- Microchip Technology Inc., *MPLAB XC8 PIC Assembler User's Guide*, DS50002974 — https://ww1.microchip.com/downloads/aemDocuments/documents/DEV/ProductDocuments/UserGuides/MPLAB-XC8-PIC-Assembler-Users-Guide-DS50002974.pdf
  - Used for: `BANKSEL`, `BANKMASK()`, symbols, and assembler behavior.

- [RCET PIC-AS Style Guide](https://github.com/rosstimo/pic_projects/blob/main/RCET_PIC-AS_Style_Guide.md#10-banking-rules)
  - Used for: current course source-style expectations around explicit banking.

[Back to top](#top) · [Topics index](README.md)
