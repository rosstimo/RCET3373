# RCET 3373 - Data Memory, SFRs, and Banking Self-Learning Guide

## What you should be able to do

After working through this guide, you should be able to:

- distinguish Special Function Registers (SFRs) from General Purpose Registers (GPRs);
- explain why banked data memory exists on the PIC16F883;
- use `BANKSEL` conceptually and intentionally;
- trace a short register-initialization sequence;
- build a register bit map from the datasheet;
- use reset/default values as part of a configuration plan.

For the broader device model, see [PIC16F883 Datasheet and Architecture](pic16f883-architecture.md).

## SFRs and GPRs are both data-memory locations

A GPR is general storage for program data.

An SFR is an addressable data-memory location whose bits control or report hardware behavior.

```text
GPR count  -> ordinary program working data
SFR TRISB  -> bits affect PORTB pin direction
```

Both are data memory, but writing them can have very different consequences.

## Why banking exists

The instruction format cannot directly name every possible data-memory location using an arbitrarily large address field.

The PIC therefore organizes data memory into banks and maintains bank-selection state that affects which physical register a file-register access reaches.

In readable source, use `BANKSEL` to make that intent explicit.

## Trace an initialization sequence

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
2. What value is written?
3. What hardware behavior changes?
4. What could happen if the wrong bank were active?

`BANKSEL` does not write the target register's value. It establishes the bank-selection state needed for the following file-register access.

## Build register maps from the datasheet

A useful working table is:

| Bit | Name | R/W | Reset/default | Planned value | Reason |
| --- | --- | --- | --- | --- | --- |
| | | | | | |

Do not stop at copying a register diagram. Connect every planned value to the behavior your design needs.

## Reset state is part of the design

Before writing setup code, ask what the device already does after reset:

- does the pin begin as input or output?
- is an analog function enabled?
- is a peripheral enabled or disabled?
- what is the register's reset value?

Initialization is a transition from the documented default state to your intended application state.

## Practice

Choose an SFR from the PIC16F883 datasheet that is not the one used in the class example.

1. Locate the register and its bank/address information.
2. Identify each relevant bit.
3. Record the reset value.
4. Choose an intended configuration.
5. Explain the reason for each non-default bit choice.
6. Write the minimal `BANKSEL` plus register-write sequence needed for that configuration.

## Self-check

You should be able to explain why each statement is wrong:

- "SFRs are just normal variables with unusual names."
- "`BANKSEL TRISB` sets TRISB to a particular value."
- "Once the correct bank is selected, it stays correct forever regardless of later code."
- "Reset values do not matter because setup code will overwrite everything."

For a concrete I/O example, continue with [PORTB Datasheet-Driven Configuration](digital-io-portb-configuration.md).
