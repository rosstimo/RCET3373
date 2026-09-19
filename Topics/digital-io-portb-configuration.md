# RCET3373 W01D05 - Self-Learning Guide

# What you should be able to do

Use the PIC16F883 data sheet to determine what must be configured for PORTB to operate as eight digital outputs, explain which associated registers do or do not matter to that function, and implement an observable counter without relying on a port read-modify-write instruction.

# 1. Begin with the function

The requested behavior is:

> Use all eight PORTB pins as digital outputs and repeatedly display an incrementing 8-bit value.

Do not begin with "clear a bunch of registers." Begin by asking what must be true for that behavior to exist.

# 2. Find the pins and alternate functions

Use the PIC16F883 pin diagram and PORTB section. PORTB pins have alternate functions. The presence of an alternate-function label does not automatically mean the alternate peripheral is active, but it tells you what other chapters/registers may affect the pin.

Your first questions should always include:

- Which pins?
- Which alternate functions?
- Which SFRs?
- Which reset values?
- Which other enables can claim or alter the pin?

# 3. Direction and digital mode are different controls

`TRISB` controls direction:

```text
TRIS bit = 1 -> input / output driver disabled
TRIS bit = 0 -> output driver enabled
```

After reset, `TRISB` is all ones, which keeps the output drivers disabled.

That does **not** mean every PORTB pin is already configured for ordinary digital input. On this device, the implemented `ANSELH` bits reset to analog mode. RB0 through RB5 therefore need their analog function disabled for normal digital I/O. RB6 and RB7 are not controlled by those ANSELH analog-channel bits.

For eight digital outputs, both questions must be satisfied:

1. Are the pins in digital mode?
2. Are the output drivers enabled?

# 4. Read reset-state diagrams literally

The data-sheet reset legend matters.

- `0` and `1` are defined reset states.
- `x` means unknown.
- `u` means unchanged in the reset-state notation used by the data sheet.

An `x` in a PORT value diagram does not mean active-high or active-low. It means software must not assume a defined reset value for that bit.

# 5. Classify associated registers before writing code

For every register mentioned by the port section, ask which bucket it belongs in.

## Required to change

For this exercise, digital/analog selection and direction clearly matter. The output value also has to be established.

## Already acceptable at reset

If the data sheet shows that a control already has the required state, leaving it alone may be the correct design decision.

## Alternate peripheral only

A selector for a disabled peripheral is not automatically a problem. For example, a Timer1 gate-source selector does not itself enable Timer1 gating. Check the actual enable path before changing it.

This is a better rule than clearing every register that mentions PORTB.

# 6. Bank selection should be intentional

The STATUS bank-select bits have documented reset values. The starting bank is therefore not a mystery after the reset conditions considered here.

Still, select the required bank explicitly before accessing a banked SFR. That documents intent and prevents a later edit from silently changing which register the instruction reaches.

# 7. Initialize before enabling the drivers

A clean bring-up order is:

1. leave `TRISB` as inputs while configuring;
2. disable the analog functions required for digital PORTB operation;
3. place a known value in the desired software/output state;
4. write that value to `PORTB`;
5. set `TRISB` for outputs;
6. begin normal updates.

This reduces surprises when the external drivers become active.

# 8. Do not increment `PORTB` directly

A tempting loop is:

```asm
loop:
    incf    PORTB,F
    goto    loop
```

Do not use this as the reusable model. `INCF PORTB,F` is a read-modify-write operation. The read can reflect physical pin levels. A heavily loaded or slow output can therefore change the value that is read, modified, and written back.

Keep the desired value in a GPR instead:

```asm
    clrf    portShadow
    movf    portShadow,W
    movwf   PORTB

loop:
    incf    portShadow,F
    movf    portShadow,W
    movwf   PORTB
    goto    loop
```

Now the arithmetic operates on software state, then a whole byte is explicitly copied to the output port.

# 9. Know what `INCF` does to STATUS

`INCF` affects the Zero flag. If an 8-bit value increments from `0xFF` to `0x00`, the result is zero and the Zero flag is set. `INCF` does not update Carry.

# 10. Watchdog timer mental model

The WDT is an independent timer when enabled. Correctly running software services it at deliberate points. If the program stops reaching that service point before timeout, the WDT can cause its configured response.

It does not inspect the program counter and decide whether the address is correct.

# 11. Hardware inputs still need electrical reasoning

A high-impedance pin is not actively driving an output level, but that alone does not guarantee a safe or useful state. Floating CMOS inputs can behave unpredictably and consume unnecessary current. Use defined states compatible with the actual board and alternate functions.

# Worked reasoning example

**Goal:** PORTB is an 8-bit digital output.

**Known from reset:** output drivers begin disabled by `TRISB`; several PORTB pins default to analog mode through `ANSELH`.

**Required decisions:**

1. disable the relevant analog functions;
2. initialize the intended output state;
3. enable the PORTB output drivers;
4. update output through a shadow GPR;
5. leave unrelated alternate peripherals alone unless their enable state creates a real conflict.

The data sheet, not memory or a copied initialization block, justifies each decision.

# Practice

1. Why is `TRISB = 0xFF` after reset useful during bring-up but insufficient to say PORTB is ready as eight digital inputs?
2. What is the purpose of `ANSELH` in this exercise?
3. Why is explicit bank selection useful even when the reset bank is documented?
4. A Timer1 gate-source selector names RB5, but Timer1 gating is disabled. Must the selector necessarily be changed for basic PORTB output use? Explain.
5. Why is `INCF PORTB,F` risky?
6. What is a shadow register and how does it remove that particular risk?
7. Which STATUS flag does `INCF` affect when the result wraps to zero?
8. What does reset symbol `x` mean?
9. What does the WDT actually monitor?

# Answer key

1. `TRISB = 0xFF` disables the output drivers, but RB0-RB5 can still be configured for analog input by their reset `ANSELH` state. Direction and analog/digital selection are separate controls.
2. It controls analog selection for the analog-capable PORTB pins; clear the required implemented bits for digital operation.
3. It documents the bank expected by the following access and prevents surrounding code changes from redirecting the access.
4. No. Source selection and gate enable are separate. An alternate function should be changed only when its enabled state actually conflicts with the requested port function.
5. It reads physical PORTB pin states, modifies the result, and writes it back, so readback affected by load/timing can corrupt intended output bits.
6. A GPR stores the intended output byte. Arithmetic updates that GPR and `MOVWF PORTB` copies the complete intended value to the port.
7. Zero. `INCF` does not update Carry.
8. Unknown.
9. It is an independent timeout mechanism. Software demonstrates progress by servicing it before timeout when the design uses that scheme.

# What to be able to explain without notes

- the peripheral-interrogation checklist;
- why TRIS and analog selection are separate;
- which configuration you must change versus which you should leave alone;
- why reset-state notation matters;
- why a shadow register is safer than `INCF PORTB,F`;
- how you would prove the output configuration works on actual hardware.

# Reference

Microchip DS40001291H, especially the PORTB, I/O pin, register-summary/reset, instruction-set, and electrical-characteristics material relevant to the PIC16F883.
