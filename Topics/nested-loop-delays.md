# RCET3373 - Software Delays and Nested Loops - Self-Learning Guide

# What you should be able to do

After working through this guide, you should be able to:

- convert a PIC oscillator frequency into instruction-cycle time;
- trace `DECFSZ` correctly through its final skip;
- calculate a one-byte software delay;
- explain why a loaded zero can represent 256 loop iterations;
- calculate a two-counter nested delay from the code path;
- determine the timing step caused by changing either counter;
- choose counter values for a target delay;
- explain why direct read-modify-write operations on `PORTB` require care.

# 1. Start with the clock

The PIC16F883 classic mid-range core divides the oscillator by four for instruction timing.

```text
TCY = 4 / FOSC
```

For the 4 MHz crystal used in class:

```text
FOSC = 4,000,000 Hz
TOSC = 1 / FOSC = 250 ns
TCY  = 4 * 250 ns = 1 us
```

That makes the first timing exercises convenient: one instruction cycle corresponds to one microsecond.

Do not assume this remains true if the oscillator changes. The cycle-count algebra remains the same, but the time represented by each cycle changes.

# 2. `DECFSZ` does not take the same time on every pass

Consider:

```asm
    decfsz  count, F
    goto    delay
```

For a normal iteration where the decremented result is not zero:

```text
DECFSZ = 1 cycle
GOTO   = 2 cycles
total  = 3 cycles
```

When the decrement produces zero, `DECFSZ` skips the following instruction. Microchip documents this as a two-cycle `DECFSZ` because the already-fetched next instruction is discarded and a NOP occupies the second cycle.

So the final pass is:

```text
DECFSZ successful skip = 2 cycles
GOTO                    = skipped
```

This final-path difference is the source of many off-by-one timing errors.

# 3. A one-byte delay

Use this code:

```asm
count   EQU     0x20

    movlw   n
    movwf   count

delay:
    decfsz  count, F
    goto    delay
```

Define the timed interval from the first `MOVLW` through completion of the final successful `DECFSZ`.

The two setup instructions cost two cycles.

For `n` effective loop iterations:

```text
first n-1 iterations = 3(n-1)
final iteration       = 2
setup                 = 2
```

Therefore:

```text
Tsingle = 2 + 3(n-1) + 2
        = 3n + 1 cycles
```

## Worked example: n = 10

```text
T = 3(10) + 1
  = 31 cycles
```

At 4 MHz:

```text
T = 31 us
```

# 4. Why `0x00` means 256 effective decrements

An 8-bit register contains values from `0x00` through `0xFF`.

If the counter begins at `0x00`, the first decrement wraps it to `0xFF`. It then continues downward until a later decrement produces `0x00` again. That zero-result decrement is the one that causes the skip.

For this countdown structure, use:

```text
loaded 0x01 -> 1 effective decrement
loaded 0x02 -> 2 effective decrements
...
loaded 0xFF -> 255 effective decrements
loaded 0x00 -> 256 effective decrements
```

For the one-byte delay:

```text
Tmax = 3(256) + 1
     = 769 cycles
     = 769 us at 4 MHz
```

# 5. Timing boundaries matter

A function name or label does not define what the oscilloscope measures.

Suppose RB0 changes state, then the program runs a delay, does some other work, and later changes RB0 again. The measured pulse width includes every executed instruction between those two output changes.

Before doing arithmetic, write the boundary in words.

Example:

> Timing begins immediately after RB0 changes state and ends at the next RB0 state change.

Then count only instructions that execute inside that interval.

A useful bookkeeping model is:

```text
Ttotal = Tuseful_work + Tdelay_setup + Tdelay_loop + Tbranching
```

# 6. Toggling one bit with XOR

The XOR identities are:

```text
x XOR 0 = x
x XOR 1 = NOT x
```

A mask with one `1` bit can therefore toggle a selected read bit.

For RB0:

```text
mask = 0000 0001
```

Conceptually:

```asm
    movlw   b'00000001'
    xorwf   PORTB, F
```

The Boolean reasoning is correct, but there is a hardware warning. The PIC16F883 data sheet says writes to PORT registers are read-modify-write operations. Reading `PORTB` reads actual pin states. If a pin state differs from the output latch value because of loading, analog configuration, or mixed input/output use, the write-back can unintentionally alter another output latch bit.

For a simple classroom waveform on a known configuration, direct XOR is useful. For a general design, maintain the intended output byte in a GPR shadow register and write the complete intended value to the port.

# 7. Build a nested delay from the code, not from a shortcut

Use two counters:

```asm
outer_count EQU     0x20
inner_count EQU     0x21

    movlw   N
    movwf   outer_count

outer_delay:
    movlw   I
    movwf   inner_count

inner_delay:
    decfsz  inner_count, F
    goto    inner_delay

    decfsz  outer_count, F
    goto    outer_delay
```

The key structural fact is:

> Every outer iteration reloads and runs the complete inner delay.

That is why the two counts multiply, but setup and outer-control costs still have to be included.

# 8. Derive the nested timing inside out

## Inner loop itself

For `I` effective decrements:

```text
Tinner_loop = 3(I-1) + 2
            = 3I - 1
```

## Inner setup plus inner loop

Each outer iteration executes:

```text
MOVLW I       1 cycle
MOVWF inner   1 cycle
inner loop    3I - 1 cycles
```

So:

```text
Tinner_per_outer = 3I + 1
```

## Outer control

Across all `N` outer iterations:

```text
first N-1 outer passes = 3(N-1)
final outer DECFSZ      = 2
```

Therefore:

```text
Touter_control = 3N - 1
```

## Add the one-time outer setup

```text
outer setup = 2 cycles
```

Total:

```text
Tnested = 2 + N(3I + 1) + (3N - 1)
        = N(3I + 4) + 1 cycles
```

This formula belongs to this exact code and timing boundary. If you move an instruction, add a NOP, preload W differently, or include surrounding `Main` work, derive the new expression from the changed path.

# 9. Minimum, maximum, and step size

## Minimum

Set both effective counts to one:

```text
T = 1(3*1 + 4) + 1
  = 8 cycles
  = 8 us at 4 MHz
```

## Maximum with two byte counters

A loaded zero represents 256 effective decrements:

```text
T = 256(3*256 + 4) + 1
  = 197633 cycles
  = 197.633 ms at 4 MHz
```

## Change the outer count

Hold `I` fixed. Increasing `N` by one adds:

```text
Delta_N = 3I + 4 cycles
```

Example with `I=10`:

```text
Delta_N = 34 cycles = 34 us
```

## Change the inner count

Hold `N` fixed. Increasing `I` by one adds:

```text
Delta_I = 3N cycles
```

Example with `N=20`:

```text
Delta_I = 60 cycles = 60 us
```

If both counts use the same value `x`:

```text
T = x(3x + 4) + 1
  = 3x^2 + 4x + 1
```

That is quadratic growth, not exponential growth.

# 10. Worked target: exactly 1 ms

At 4 MHz:

```text
1 ms = 1000 us = 1000 instruction cycles
```

We want:

```text
1000 = N(3I + 4) + 1
```

One exact solution is:

```text
I = 11
N = 27
```

Check it:

```text
T = 27(3*11 + 4) + 1
  = 27(37) + 1
  = 999 + 1
  = 1000 cycles
  = 1.000 ms
```

The important habit is to verify the result by substituting the chosen counts back into the equation.

# 11. When software delay loops stop being a good tool

A blocking software delay makes the CPU spend its time counting instead of doing other work.

Software delays are useful for:

- learning instruction timing;
- very small simple delays;
- controlled demonstrations;
- situations where blocking is acceptable.

Move toward timers, interrupts, or sleep when:

- the delay becomes long;
- other work must occur during the wait;
- timing must remain accurate while code changes;
- power consumption matters;
- several independent time intervals must be managed.

# Practice problems

## 1. Instruction cycle

A PIC16F883 is running from a 4 MHz oscillator. What is `TCY`?

## 2. Single delay

Using `Tsingle = 3n + 1`, calculate the delay for `n=25` at 4 MHz.

## 3. Zero preload

What effective count should be used in the timing equation when the byte counter is loaded with `0x00`?

## 4. Hit 50 us

For `Tsingle = 3n + 1`, choose the largest integer `n` that does not exceed 50 us at 4 MHz. How many additional one-cycle instructions are needed to reach exactly 50 us?

## 5. Nested delay

For the explicit nested code in this guide, calculate the delay for:

```text
N = 3
I = 4
```

## 6. Outer step size

If `I=12`, how much does the nested delay change when `N` increases by one?

## 7. Inner step size

If `N=18`, how much does the nested delay change when `I` increases by one?

## 8. Growth type

Set `N=I=x`. Write the timing expression in terms of `x`. Is it linear, quadratic, or exponential?

## 9. Maximum

Calculate the maximum delay when both byte counters are loaded with `0x00` at 4 MHz.

## 10. Hardware reasoning

Why can `XORWF PORTB,F` modify an output bit you did not intend to toggle even if the XOR mask contains a zero for that bit?

# Answer key

## 1

```text
TCY = 4/FOSC = 4/4 MHz = 1 us
```

## 2

```text
T = 3(25) + 1 = 76 cycles = 76 us
```

## 3

256 effective decrements.

## 4

```text
3n + 1 <= 50
n <= 16.333...
```

Use `n=16`:

```text
T = 49 us
```

Add one one-cycle instruction, such as a `NOP`, inside the defined timing boundary to reach 50 us.

## 5

```text
T = N(3I + 4) + 1
  = 3(12 + 4) + 1
  = 49 cycles
  = 49 us
```

## 6

```text
Delta_N = 3I + 4 = 3(12) + 4 = 40 cycles = 40 us
```

## 7

```text
Delta_I = 3N = 54 cycles = 54 us
```

## 8

```text
T = x(3x + 4) + 1
  = 3x^2 + 4x + 1
```

Quadratic.

## 9

```text
T = 256(3*256 + 4) + 1
  = 197633 us
  = 197.633 ms
```

## 10

On this device, PORT writes are read-modify-write. Reading `PORTB` reads the actual pin states. The value read from a loaded or differently configured pin can differ from the intended output latch state, and the modified byte is then written back to the PORT latch.

# What to be able to explain without notes

- why `FOSC/4` matters;
- why the final `DECFSZ` is different;
- why count zero means 256 in this loop;
- how to derive `3n+1` rather than memorize it;
- how to derive a nested delay from the inside outward;
- why the explicit two-loop code gives `N(3I+4)+1`;
- how to separate delay time from surrounding `Main` time;
- why direct PORT read-modify-write operations can surprise you;
- why a timer eventually becomes preferable to a long blocking delay.

# References used to build this guide

Microchip PIC16F882/883/884/886/887 Data Sheet, DS40001291H  
https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf

PICmicro Mid-Range MCU Family Reference Manual  
https://ww1.microchip.com/downloads/en/DeviceDoc/33023a.pdf

RCET3373 Fall 2026 teaching record:

- W02D02 timing and first software-delay session;
- W02D03 nested-loop session and post-class accuracy review.
