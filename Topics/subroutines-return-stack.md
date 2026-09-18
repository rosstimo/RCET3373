# RCET3373 Self-Learning Guide - Subroutines and the PIC16F883 Return Stack

## Goal

You should be able to explain where a PIC16F883 subroutine returns, calculate the timing of a callable delay, determine simultaneous stack depth, and use MPLAB X Simulator to check your reasoning.

## 1. CALL solves the return-path problem

`GOTO` loads a new Program Counter destination but does not automatically remember where execution came from.

`CALL` transfers execution and saves the address of the instruction after the call, `PC + 1`, on the hardware return stack. `RETURN` restores the top saved address into the Program Counter.

A useful analogy is a **bookmark**: the saved address marks where execution resumes.

## 2. Know the actual hardware stack

PIC16F883 provides:

```text
8 levels × 13 bits
hardware return-address stack
```

Rules:

- separate from program/data memory;
- Stack Pointer not readable/writable by software;
- `CALL` and interrupt entry push;
- `RETURN`, `RETLW`, and `RETFIE` pop;
- ninth push overwrites the oldest entry because the stack is circular;
- no overflow/underflow status flag.

The Pez analogy helps with LIFO: the last return address pushed is the first one removed.

### Sequential versus nested

```asm
call A
call A
call A
call A
```

If every call returns before the next begins, maximum depth = **1**.

```text
Main -> A -> B -> C
```

Maximum simultaneous depth = **3**.

## 3. Include interrupts in depth reasoning

PIC16F883 has one interrupt vector at `0x0004` and no hardware interrupt-priority levels. Accepted interrupt service pushes a return address and clears `GIE`, so normal interrupt service does not nest another interrupt. `RETFIE` returns and restores `GIE`.

## 4. PCLATH and larger programs

The Program Counter is 13 bits. `CALL` and `GOTO` carry 11 destination bits; `PCLATH<4:3>` supplies the upper two. Page selection matters when targets cross 2K-word pages.

## 5. Exact callable 50 us delay

At 4 MHz:

```text
TCY = 4/FOSC = 1 us
```

```asm
    call    Delay50us

Delay50us:
    movlw   15
    movwf   count
loop50:
    decfsz  count, F
    goto    loop50
    return
```

Define the interval from **CALL start through RETURN completion**:

```text
CALL              2 cycles
setup             2 cycles
loop            3n - 1 cycles
RETURN            2 cycles
--------------------------
T                3n + 5 cycles
```

Solve `3n + 5 = 50`: `n = 15`. At 1 us/cycle, that is exactly 50 us.

## 6. Build 5 ms from the 50 us routine

For the explicit outer routine:

```text
T = 53N + 5
```

Choose the largest `N` that does not exceed 5000 us:

```text
N = floor((5000 - 5)/53) = 94 = 0x5E
T = 53(94) + 5 = 4987 us
residual = 13 us
```

Use the one-byte setup/loop formula `3n + 1`. `n = 4` gives 13 us. Put that block before the already-counted final `RETURN` to reach exactly 5000 us.

## 7. Delay time is not automatically waveform time

If other instructions between output edges add 6 us:

```text
measured half-period = 5006 us
period = 10012 us
frequency ≈ 99.88 Hz
relative overhead = 6/5000 = 0.12%
```

The oscilloscope measures the entire path between edges.

## 8. Measurement resolution

If one `NOP` changes the prediction by 1 us, ask whether the current instrument configuration can resolve 1 us on a multi-millisecond pulse. A displayed standard deviation is not automatically a complete uncertainty statement.

Record the prediction, method/settings, measurement, and whether the method has enough resolution for the claim.

## 9. MPLAB X Simulator

Useful workflow:

1. select **Simulator** as project tool;
2. halt at reset vector when debugging begins;
3. inspect Program Memory;
4. inspect PC, W, STATUS, and SFRs;
5. single-step;
6. use breakpoints;
7. edit a register such as `PORTB` and observe program behavior;
8. inspect generated instructions around directives such as `BANKSEL`.

Simulation supports code/register reasoning. It does not prove wiring, oscillator/probe loading, analog behavior, or other hardware-dependent claims.

## 10. BANKSEL and timing

`BANKSEL symbol` is an assembler directive that emits bank-selection instructions. In timing-critical code, inspect the listing/disassembly and count what was actually emitted rather than assigning a universal one-cycle cost.

## Review questions

1. What address does `CALL` save?
2. What does `RETURN` restore?
3. How deep is the PIC16F883 return stack?
4. Why can four sequential calls have depth 1 while three nested calls have depth 3?
5. Does PIC16F883 have hardware interrupt priorities?
6. Why must `CALL` and `RETURN` appear in an exact subroutine timing equation?
7. Why is `n = 15` exact for the shown 50 us routine?
8. Why can a 5000 us routine produce a 5006 us measured half-period?
9. What should you inspect before timing `BANKSEL`?
10. What simulator conclusions still need hardware verification?

## Official references

- https://ww1.microchip.com/downloads/en/devicedoc/41291e.pdf
- https://ww1.microchip.com/downloads/en/DeviceDoc/33023a.pdf
- https://onlinedocs.microchip.com/oxy/GUID-4DC87671-9D8E-428A-ADFE-98D694F9F089-en-US-5/index.html
