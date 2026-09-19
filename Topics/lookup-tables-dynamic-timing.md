<a id="top"></a>

# RCET 3373 — Lookup Tables, Computed GOTO, and Dynamic Timing

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

A program often needs to map a small input value to a different output value.

Examples include:

- button number -> delay count;
- state number -> constant;
- note index -> tone period;
- menu selection -> configuration value.

A lookup table stores that mapping compactly in program memory.

On the PIC16F883, a classic assembly technique uses the table index to modify the low byte of the Program Counter and then returns the selected value with `RETLW`.

This is useful because it combines several important architecture ideas:

- program memory as data storage;
- W as an index and return-value register;
- `PCL` and `PCLATH`;
- the hardware return stack;
- linker placement;
- complete-path timing.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- distinguish a table index from the value returned by the table;
- preserve a selected value separately from a working countdown register;
- explain the roles of `PCL` and `PCLATH`;
- trace an `ADDWF PCL` + `RETLW` table;
- explain how `RETLW` uses the hardware return stack;
- recognize the 256-word low-PCL boundary hazard;
- distinguish that boundary from the 2K-word `CALL`/`GOTO` page issue;
- verify table placement in Program Memory or a listing/map;
- convert a selected half-cycle delay into square-wave frequency;
- explain why equal delay steps produce unequal frequency steps;
- include lookup, polling, and caller code in complete waveform timing.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Nested-Loop Software Delays](nested-loop-delays.md#nested-formula);
- [Subroutines and the Return Stack](subroutines-return-stack.md#call-return-model);
- [PIC16F883 Program Counter and stack](pic16f883-architecture.md#program-counter-stack).

Related topics:

- [Timing Measurement and Compiled-Code Verification](measurement-c-timing.md#measurement-strategy);
- [PIC16F883 Timers](timers.md#core-model) for the hardware-timer alternative to long software waits.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="persistent-selection"></a>
### Persistent selection versus working countdown

Suppose a keypad selects a delay.

Do not store the selected delay only in the same register that the delay loop destroys while counting down.

Use separate roles:

```text
current_count = persistent selected value
out_count     = working outer countdown
in_count      = working inner countdown
```

Then the selected value can reload the working counter each time:

```asm
half_cycle_delay:
    movf    current_count, W
    movwf   out_count
    ; run delay
    return
```

`MOVF current_count,W` reads a register value into W.

`MOVLW` would load a literal constant instead.

<a id="lookup-table"></a>
### A lookup table maps one value to another

Example mapping:

| Zero-based index | Returned delay count |
| ---: | ---: |
| 0 | 250 |
| 1 | 125 |
| 2 | 12 |

The index and returned value are different quantities.

The table lets a small integer select a constant stored in program memory.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="pcl-pclath"></a>
### Program Counter, `PCL`, and `PCLATH`

The PIC16F883 Program Counter is 13 bits wide.

For a write to `PCL`, think of the destination as:

```text
PC = [ upper 5 bits ][ lower 8 bits ]
          ^                 ^
       PCLATH              PCL
```

More precisely:

- `PCL` exposes the low eight PC bits;
- `PCLATH` is a separate readable/writable holding register;
- when software writes `PCL`, `PCLATH<4:0>` supplies the upper PC bits.

For ordinary `CALL` and `GOTO`, the loading path is different: the instruction contains 11 destination bits and `PCLATH<4:3>` supplies the upper bits.

That creates the separate 2K-word page issue described in the [subroutine guide](subroutines-return-stack.md#pclath-call-goto).

**Official visual reference:** In Section 2.0, **Memory Organization**, of the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf), inspect the Program Counter loading diagram.

Focus on the different paths used when writing `PCL` versus executing `CALL` or `GOTO`.

<a id="computed-goto"></a>
### Computed GOTO with `ADDWF PCL`

A classic table is:

```asm
    movf    button_value, W
    call    lookup_delay
    movwf   current_count

lookup_delay:
    addwf   PCL, F
    retlw   0xFA
    retlw   0x7D
    retlw   0x0C
```

Trace the index:

```text
W = 0 -> first RETLW -> returns 0xFA
W = 1 -> second RETLW -> returns 0x7D
W = 2 -> third RETLW -> returns 0x0C
```

`ADDWF PCL,F` changes the low Program Counter byte, so execution lands on one table entry.

`RETLW k` then:

1. loads literal `k` into W;
2. returns through the hardware return stack.

The caller stores W in `current_count`.

<a id="pcl-boundary"></a>
### The 256-word low-PCL boundary hazard

A table is not safe merely because it contains fewer than 256 entries.

Suppose the table begins near:

```text
... 0x00FE
... 0x00FF
... 0x0100
... 0x0101
```

The low byte rolls from:

```text
0xFF -> 0x00
```

An `ADDWF PCL` operation changes the low byte. It does not automatically perform a full 13-bit integer addition with a carry into the upper PC bits.

If `PCLATH` does not contain the correct upper value, the computed branch can land in the wrong 256-word block.

For introductory RCET work:

> Keep the complete `ADDWF PCL` + `RETLW` table inside one 256-word low-PCL block unless the code explicitly handles the upper-PC requirement.

This 256-word boundary is **not** the same as the 2K-word `CALL`/`GOTO` page boundary.

<a id="location-counter"></a>
### `$` is an assembler location counter

PIC Assembler can use `$` to refer to the current assembly location.

For example:

```asm
    decfsz  count,F
    goto    $-1
```

The assembler resolves `$-1` to the previous instruction-word location and emits an ordinary `GOTO`.

Important distinctions:

- `$` is an assembler concept;
- it is not a runtime read of the Program Counter;
- labels are usually clearer for larger control-flow structures.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="lookup-trace-example"></a>
### Worked example: trace index 2

Given:

```asm
lookup:
    addwf   PCL,F
    retlw   0x20
    retlw   0x40
    retlw   0x60
```

**Known**

```text
W = 2
```

**Reasoning**

The index offsets execution to the third `RETLW`.

`RETLW 0x60` loads W and returns.

**Result**

```text
W = 0x60
```

<a id="dynamic-frequency"></a>
### Worked example: delay selection to frequency

Suppose a selected delay produces a half-cycle of:

```text
t_half = 241 us
```

The full period is:

```text
T = 2 × 241 us
  = 482 us
```

Frequency:

```text
f = 1 / 482 us
  ≈ 2074.69 Hz
  ≈ 2.075 kHz
```

<a id="frequency-resolution"></a>
### Equal time steps do not create equal frequency steps

Suppose a selector changes half-cycle time in equal 20 us increments.

Near short delays, 20 us is a large fraction of the total period, so the frequency change is large.

Near long delays, the same 20 us is a small fraction of the total period, so the frequency change is small.

The reason is the reciprocal relationship:

```text
f = 1 / (2t_half)
```

A constant time resolution is not a constant frequency resolution.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="table-placement-check"></a>
### Verify table placement after linking

Before trusting a computed-GOTO table:

1. build the project;
2. inspect Program Memory or the map/listing;
3. record the address of `ADDWF PCL`;
4. record the address of the final `RETLW`;
5. confirm the complete table remains inside one low-PCL 256-word block unless the code handles upper-PC changes;
6. simulate the first, middle, and last valid indices;
7. test an invalid/out-of-range path if the caller can produce one.

The [RCET computed-GOTO reference](https://github.com/rosstimo/pic_projects/blob/main/References/PIC16F883-Computed-GOTO.md) includes a useful deliberate-failure simulation exercise.

<a id="whole-path-timing"></a>
### The lookup routine is not the whole waveform

A measured output path may include:

```text
input scan
-> branch decisions
-> CALL lookup
-> ADDWF PCL
-> selected RETLW
-> store selected value
-> CALL delay
-> delay body
-> RETURN
-> output update
-> main-loop branch
```

The oscilloscope measures the complete path between output edges.

If polling or branch paths take different amounts of time, that variation can appear as jitter even when the named delay routine itself is perfectly repeatable.

<a id="polling-responsiveness"></a>
### Busy waits reduce responsiveness

While the CPU is inside a software delay, it is not checking a new input unless the design explicitly does so inside that path.

Longer waits therefore increase worst-case polling response time.

That limitation is one reason the course transitions from software delays to [hardware timers](timers.md#core-model) and [interrupts](interrupts-context-saving.md#core-model).

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Why should `current_count` and `out_count` be separate registers?
2. In an `ADDWF PCL` table, what is the difference between the input index and the returned value?
3. What does `PCL` represent?
4. What role does `PCLATH` play when software writes `PCL`?
5. Given the three-entry table in this guide, what value returns for index 2?
6. Why can a 16-entry table still have a low-PCL boundary problem?
7. Why is the 256-word computed-GOTO boundary different from the 2K-word `CALL`/`GOTO` page issue?
8. What does `$` mean to PIC Assembler?
9. A half-cycle is 401 us. Ignoring other overhead, calculate square-wave frequency.
10. Why do equal 20 us half-cycle steps cause larger frequency changes at high frequency than at low frequency?
11. Name four pieces of code outside a delay loop that may still affect measured edge-to-edge timing.
12. How would you prove that the first and last entries of a computed-GOTO table land correctly?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. The working countdown destroys `out_count`; `current_count` preserves the selected value so it can be reused.
2. The index chooses an entry; the `RETLW` literal is the value returned by that entry.
3. `PCL` is the readable/writable low eight-bit portion of the Program Counter.
4. `PCLATH<4:0>` supplies the upper PC bits when software writes `PCL`.
5. `0x60`.
6. Placement, not only table length, matters. A short table beginning near a low-byte `0xFF` boundary can straddle two 256-word blocks.
7. `ADDWF PCL` writes the low eight PC bits; `CALL/GOTO` use an 11-bit destination plus different upper bits from `PCLATH`.
8. The current assembler location; `$-1` refers to the previous instruction-word location.
9. `T = 802 us`, so `f ≈ 1246.88 Hz ≈ 1.247 kHz`.
10. Frequency is reciprocal in period, so the same time change is a larger fraction of a short period.
11. Examples: input scan, conditional branches, lookup `CALL`, `ADDWF PCL`, `RETLW`, result storage, delay `CALL/RETURN`, output update, loop branch.
12. Inspect linked Program Memory/listing and simulate boundary indices while observing the actual PC target and returned W value.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- distinguish persistent selection from working countdown state;
- trace index -> `ADDWF PCL` -> selected `RETLW` -> returned W;
- explain how `PCL` and `PCLATH` form the computed target;
- explain the low-PCL boundary hazard;
- distinguish 256-word and 2K-word control-flow boundaries;
- explain `$` as an assembler location counter;
- convert half-cycle delay to frequency;
- explain nonuniform frequency resolution;
- include caller/polling/lookup overhead in measured waveform timing.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Memory Organization, Program Counter/PCL/PCLATH, and Instruction Set Summary — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PC/PCL/PCLATH behavior, `ADDWF`, `RETLW`, control-flow timing, and computed-GOTO cautions.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: classic mid-range Program Counter and computed-control-flow architecture.

- Microchip Technology Inc., *Implementing a Table Read*, AN556 — https://ww1.microchip.com/downloads/en/AppNotes/00556e.pdf
  - Used for: `ADDWF PCL`, `RETLW`, PCLATH, table placement, and computed-GOTO examples.

- Microchip Technology Inc., *MPLAB XC8 PIC Assembler User's Guide*, DS50002974 — https://ww1.microchip.com/downloads/aemDocuments/documents/DEV/ProductDocuments/UserGuides/MPLAB-XC8-PIC-Assembler-Users-Guide-DS50002974.pdf
  - Used for: assembler location-counter `$` and current toolchain syntax.

- [PIC16F883 Computed GOTO and `RETLW` Tables](https://github.com/rosstimo/pic_projects/blob/main/References/PIC16F883-Computed-GOTO.md)
  - RCET shared implementation/reference material for table placement, simulator verification, and whole-path timing.

[Back to top](#top) · [Topics index](README.md)
