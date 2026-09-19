<a id="top"></a>

# RCET 3373 — Instruction Timing and Software Delays

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

Software timing is only meaningful when you know how oscillator time becomes instruction time and exactly which instructions execute inside the interval you are measuring.

The PIC16F883 is especially useful for learning this because its classic mid-range timing model is simple enough to derive by hand:

```text
oscillator -> instruction cycle -> executed path -> predicted time
```

This guide establishes the timing model used by later delay, subroutine, lookup-table, interrupt, and timer work.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- convert oscillator frequency to oscillator period;
- calculate PIC16F883 instruction-cycle time;
- explain the basic fetch/execute pipeline model;
- explain why control-flow and skip behavior can require an extra cycle;
- trace the two timing cases of `DECFSZ`;
- derive the exact one-byte delay formula used here;
- explain why a preload of zero represents 256 decrements in this countdown;
- calculate minimum, maximum, and count-step resolution;
- define the timing boundary before applying a formula.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Digital Timing](digital-timing.md#period-frequency);
- [PIC16F883 Architecture](pic16f883-architecture.md#instruction-format).

Related topics:

- [Nested-Loop Delays](nested-loop-delays.md#core-model);
- [Subroutines and Return Stack](subroutines-return-stack.md#callable-delay);
- [Measurement Strategy and C Timing](measurement-c-timing.md#measurement-strategy);
- [PIC16F883 Timers](timers.md#core-model).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="instruction-cycle"></a>
### Oscillator period and instruction-cycle time

The course PIC16F883 hardware uses a 4 MHz external crystal for the introductory timing work.

```text
FOSC = 4 MHz

TOSC = 1 / FOSC
     = 0.25 us
```

For the classic mid-range PIC core:

```text
TCY = 4 × TOSC
    = 4 / FOSC
```

At 4 MHz:

```text
TCY = 1 us
```

That convenient equality means 31 instruction cycles correspond to 31 us **only when the executed path actually contains 31 instruction cycles**.

<a id="timing-boundary"></a>
### A formula belongs to a defined interval

Before counting anything, state the beginning and end of the timed interval.

Example:

> Start with the first `MOVLW`. End after the final `DECFSZ` completes and skips the following `GOTO`.

Without that sentence, two correct cycle counts can describe two different intervals.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="pipeline"></a>
### Fetch/execute overlap

The PIC overlaps instruction fetch and execution.

A simplified model is:

```text
cycle k:     execute A, fetch B
cycle k+1:   execute B, fetch C
```

Most straight-line instructions therefore complete in one instruction cycle after the pipeline is active.

<a id="control-flow-extra-cycle"></a>
### Why control flow can take an extra cycle

If instruction B changes the Program Counter, the sequential instruction already fetched after B is no longer the correct next instruction.

That fetched instruction is discarded and the new target path must refill.

This produces the documented two-cycle behavior for instructions such as `GOTO`.

The source code did not literally gain a `NOP`; “NOP-like” is only a mental model for the lost pipeline slot.

**Official visual reference:** In the [PICmicro Mid-Range MCU Family Reference Manual](https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf), use the architecture/instruction-flow material to follow instruction fetch, execute, and pipeline flush behavior.

<a id="decfsz"></a>
### `DECFSZ` has two timing cases

`DECFSZ f,F` decrements the file register and skips the next instruction when the result is zero.

For:

```asm
delay:
    decfsz  d1,F
    goto    delay
```

the cases are:

| Result after decrement | `DECFSZ` | `GOTO` | Total |
| --- | ---: | ---: | ---: |
| nonzero | 1 cycle | 2 cycles | 3 cycles |
| zero | 2 cycles | skipped | 2 cycles |

The final pass is therefore different from every ordinary pass.

<a id="single-delay"></a>
### Derive the one-byte delay

Use:

```asm
    movlw   n
    movwf   d1
delay:
    decfsz  d1,F
    goto    delay
```

For the timing boundary defined in this guide:

**Setup**

```text
MOVLW = 1
MOVWF = 1
setup = 2 cycles
```

**First `n - 1` decrements**

```text
DECFSZ + GOTO = 3 cycles each
```

**Final decrement**

```text
DECFSZ with skip = 2 cycles
```

Therefore:

```text
Tcycles = 2 + 3(n - 1) + 2
        = 3n + 1
```

At 4 MHz:

```text
T = (3n + 1) us
```

<a id="zero-means-256"></a>
### Why loading zero means 256 decrements

The counter is eight bits.

If it begins at `0x00`, the first decrement produces `0xFF`, not the terminating zero.

```text
0x00 -> 0xFF -> ... -> 0x01 -> 0x00
```

The countdown therefore requires 256 decrements before `DECFSZ` sees a zero result.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="n10-example"></a>
### Worked example: `n = 10`

**Known**

```text
FOSC = 4 MHz
TCY  = 1 us
n    = 10
```

**Reasoning**

```text
Tcycles = 3(10) + 1
        = 31 cycles
```

Expanded check:

```text
setup                    2
9 ordinary iterations   27
final skip                2
                         --
total                    31 cycles
```

**Result**

```text
T = 31 us
```

<a id="range-resolution"></a>
### Worked example: range and resolution

Minimum effective count:

```text
n = 1
T = 3(1) + 1 = 4 cycles = 4 us
```

Maximum effective count:

```text
n = 256
T = 3(256) + 1 = 769 cycles = 769 us
```

Changing the effective count by one changes the delay by:

```text
T(n+1) - T(n) = 3 cycles = 3 us
```

So this exact delay skeleton has 3 us count-step resolution at 4 MHz.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="waveform-boundary"></a>
### Delay-routine time is not automatically waveform time

Suppose a pin toggles, then code executes:

- delay setup;
- the delay loop;
- two other instructions;
- another pin toggle.

The oscilloscope measures **all executed time between the two edges**.

Do not use `3n + 1` as if it describes the entire waveform path unless the waveform boundaries match the exact delay skeleton.

<a id="timing-analysis-checklist"></a>
### Timing-analysis checklist

For a new software timing problem:

1. identify `FOSC`;
2. calculate `TCY`;
3. write the exact executed code;
4. define the start and stop events;
5. identify instructions with alternate cycle counts;
6. count the actual execution path;
7. derive the equation;
8. substitute test values;
9. measure the same boundary;
10. explain any disagreement.

For scope/counter selection and evidence quality, see [Measurement Strategy and C Timing](measurement-c-timing.md#measurement-strategy).

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. At 4 MHz, calculate `TOSC` and `TCY`.
2. Why does `GOTO` require two instruction cycles on this core?
3. For `n = 5`, calculate the exact delay interval defined in this guide.
4. Expand the `n = 5` count into setup, ordinary iterations, and final skip.
5. What effective count results from loading `0x00` into the byte counter?
6. What are the minimum and maximum delays for this skeleton at 4 MHz?
7. What is its count-only timing resolution?
8. A scope pulse begins before `MOVLW` and ends after two instructions following the final `DECFSZ`. Can `3n + 1` alone equal the measured pulse width?
9. Write a timing-boundary sentence for a new loop.

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. `TOSC = 0.25 us`; `TCY = 1 us`.
2. Changing the PC invalidates the sequentially prefetched instruction, so the target stream requires the documented extra cycle.
3. `T = 3(5) + 1 = 16 us`.
4. Setup = 2 cycles; four ordinary iterations = 12; final skip = 2; total = 16.
5. 256 decrements.
6. Minimum = 4 us; maximum = 769 us.
7. 3 us per count step.
8. No. The measured pulse includes instructions outside the formula's defined interval.
9. A correct answer names the exact first and last instruction/event included in the measurement.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- derive `TCY = 4/FOSC`;
- explain the pipeline reason for two-cycle control-flow behavior;
- trace both `DECFSZ` timing cases;
- derive `3n + 1` from the code path;
- explain why a zero preload means 256 decrements;
- calculate minimum, maximum, and step resolution;
- define a timing boundary before doing arithmetic;
- explain why routine timing and measured waveform timing can differ.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, oscillator/CPU timing and Instruction Set Summary — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 instruction-cycle relationship and documented instruction cycle counts.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A, architecture/instruction-flow material — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: fetch/execute pipeline and classic mid-range instruction timing context.

[Back to top](#top) · [Topics index](README.md)
