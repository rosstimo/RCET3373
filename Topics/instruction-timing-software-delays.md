# RCET3373 W02D02 - Self-Learning Guide

# What you should be able to do

Calculate the PIC16F883 instruction-cycle time from the oscillator, explain basic pipeline timing, count a one-byte `DECFSZ` software delay, and define exactly what interval your timing equation describes.

# 1. Start with oscillator frequency

The course hardware uses a 4 MHz external crystal.

```text
FOSC = 4 MHz = 4,000,000 Hz
```

Period is the reciprocal of frequency:

```text
TOSC = 1 / FOSC
     = 1 / 4,000,000 s
     = 0.25 us
```

That is the period of one oscillator cycle, not one complete PIC instruction cycle.

# 2. One instruction cycle uses four oscillator periods

For the PIC16F883 mid-range core:

```text
TCY = 4 * TOSC
```

At 4 MHz:

```text
TCY = 4 * 0.25 us
    = 1 us
```

The data-sheet timing model describes four phases, Q1 through Q4, within the instruction cycle.

This 4 MHz clock is convenient because instruction-cycle counts and microseconds have the same numeric value:

```text
31 instruction cycles = 31 us
```

provided the counted path really contains 31 instruction cycles.

# 3. Why most instructions are one cycle

The PIC overlaps instruction fetch and execution. While the current instruction executes, the next sequential instruction is fetched.

A simplified model is:

```text
instruction cycle k:     execute A, fetch B
instruction cycle k + 1: execute B, fetch C
```

This overlap permits normal straight-line instructions to complete at one instruction per instruction cycle after the pipeline is active.

# 4. Why `GOTO` takes the extra cycle

Suppose B is a `GOTO`. The core has already fetched the next sequential instruction C, but B changes the program counter to a different target. The prefetched sequential instruction is discarded and the target stream has to refill.

That refill/discard produces the documented two-cycle behavior.

It is reasonable to imagine the discarded slot as "NOP-like time" while learning the pipeline, but the source code did not literally acquire a NOP instruction.

# 5. `DECFSZ` has two timing cases

`DECFSZ f,F` decrements the file register and skips the next instruction if the result is zero.

For the delay loop:

```text
result is nonzero -> 1 cycle, next GOTO executes
result is zero    -> 2 cycles, next GOTO is skipped
```

That final skip changes the loop arithmetic.

# 6. Build the one-byte delay

Use this exact skeleton:

```asm
    movlw   n
    movwf   d1
delay:
    decfsz  d1,F
    goto    delay
```

Before counting cycles, define the interval:

> Start with the first `MOVLW`. End after the final `DECFSZ` has completed and skipped the `GOTO`.

That definition is part of the formula.

# 7. Derive the formula instead of memorizing it

Setup:

```text
MOVLW = 1 cycle
MOVWF = 1 cycle
setup = 2 cycles
```

For the first `n - 1` decrements, the result is not zero:

```text
DECFSZ = 1
GOTO   = 2
total  = 3 cycles each
```

On the final decrement, the result becomes zero and `GOTO` is skipped:

```text
final DECFSZ = 2 cycles
```

Total:

```text
T = 2 + 3(n - 1) + 2
  = 3n + 1 instruction cycles
```

At 4 MHz:

```text
T = (3n + 1) us
```

# 8. Worked example: n = 10

```text
T = 3(10) + 1
  = 31 us
```

Check by expanding the path:

```text
setup                    2
9 ordinary iterations   27
final skip                2
                         --
total                    31 cycles
```

# 9. Minimum, maximum, and resolution

## Minimum

With `n = 1`:

```text
T = 3(1) + 1 = 4 us
```

## Why a literal zero count means 256

The counter is 8 bits. If it begins at `0x00`, the first decrement produces `0xFF`, not the final zero condition. It must pass through the complete 8-bit range and return to zero.

```text
0x00 -> 0xFF -> ... -> 0x01 -> 0x00
```

That takes 256 decrements.

## Maximum

```text
n = 256
T = 3(256) + 1
  = 769 us
```

## Resolution

Changing the effective count by one changes the delay by three cycles:

```text
T(n+1) - T(n) = 3 us
```

So this exact loop can adjust delay in 3 us increments by changing only the count.

# 10. Do not confuse delay-routine time with waveform time

Suppose the code toggles a pin, loads the counter, delays, then executes other instructions before toggling the pin again.

The oscilloscope pulse width includes **every executed instruction between the two edges**, not only the countdown loop.

Therefore:

1. mark the first measured edge/instruction boundary;
2. mark the second;
3. count every instruction on the path;
4. use `3n + 1` only for the exact delay skeleton interval;
5. add the surrounding instruction cycles separately.

The live W02D02 discussion briefly mixed two boundaries and produced 48/49 us wording. The reusable rule is to define the boundary first rather than choose one of those values out of context.

# 11. Why we need nested loops next

One 8-bit counter reaches at most 769 us with this exact setup. Longer delays need another counter or another timing mechanism.

W02D03 adds an outer counter. The same discipline still applies: write the explicit code, define start and stop, count every execution path, then derive the equation.

# Practice problems

1. At 4 MHz, calculate `TOSC` and `TCY`.
2. Why does `GOTO` require two instruction cycles on this core?
3. For `n = 5`, calculate the exact delay interval defined in this guide.
4. Expand the `n = 5` cycle count into setup, ordinary iterations, and final skip.
5. What effective count is produced by loading `0x00` into an 8-bit `DECFSZ` countdown?
6. What are the minimum and maximum delays for this skeleton at 4 MHz?
7. What is its count-only timing resolution?
8. A scope pulse begins before `MOVLW` and ends after two instructions that follow the final `DECFSZ`. Can `3n + 1` alone equal the measured pulse width? Explain.
9. Write the timing-boundary sentence you would record before analyzing a new loop.

# Answer key

1. `TOSC = 0.25 us`; `TCY = 4TOSC = 1 us`.
2. The sequentially prefetched instruction is discarded when the PC changes, so the target path requires the documented refill/extra cycle.
3. `T = 3(5) + 1 = 16 us`.
4. Setup 2 cycles; four ordinary iterations at 3 cycles = 12; final skip 2; total 16.
5. 256 decrements.
6. Minimum 4 us at effective `n = 1`; maximum 769 us at effective `n = 256`.
7. 3 us per count step.
8. No. The pulse includes instructions outside the interval defined by `3n + 1`; their cycles must be counted too.
9. A valid answer explicitly names the first instruction/event and the last instruction/event included in the interval.

# What to be able to explain without notes

- 4 MHz -> 0.25 us oscillator period -> 1 us instruction cycle;
- the pipeline reason for two-cycle control-flow/skip behavior;
- the full derivation of `3n + 1`;
- why zero represents 256 decrements in this loop;
- minimum, maximum, and resolution;
- why every timing formula must be attached to a defined code/measurement interval.

# Reference

Microchip DS40001291H, using the oscillator/CPU/instruction-cycle and instruction-set timing information for the PIC16F883.
