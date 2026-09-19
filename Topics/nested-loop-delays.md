<a id="top"></a>

# RCET 3373 — Nested-Loop Software Delays

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

A single 8-bit countdown can only create a limited blocking delay.

Nested loops extend that range by repeating an entire inner delay for every outer-loop pass.

The important lesson is not to memorize one equation. It is to learn how to derive timing **inside out** from the exact code path.

That same habit applies later to:

- callable delay routines;
- lookup-table paths;
- interrupt latency;
- state-machine timing;
- timer service code.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- derive a two-counter nested delay from explicit code;
- account for the final `DECFSZ` skip in both loops;
- explain why each outer iteration reloads and executes the complete inner loop;
- calculate minimum and maximum delay;
- calculate the timing step caused by changing either counter;
- solve for counter values near a target delay;
- explain why equal inner/outer counts produce quadratic growth;
- recognize when software delay loops should give way to hardware timers.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Instruction Timing and Software Delays](instruction-timing-software-delays.md#single-delay);
- [timing-boundary discipline](instruction-timing-software-delays.md#timing-boundary).

Related topics:

- [Subroutines and Return Stack](subroutines-return-stack.md#callable-delay);
- [PIC16F883 Timers](timers.md#core-model);
- [PORTB read-modify-write behavior](digital-io-portb-configuration.md#read-modify-write).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

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

Therefore the timing contains a multiplicative term, but setup and outer-control overhead still matter.

At the course 4 MHz clock:

```text
TCY = 1 us
```

The formulas in this guide are written in instruction cycles first. Multiply by `TCY` if the clock changes.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="inner-loop"></a>
### Derive the inner loop

For `I` effective decrements:

```text
first I - 1 passes:
    DECFSZ = 1
    GOTO   = 2
    total  = 3 each

final pass:
    DECFSZ with skip = 2
```

Therefore:

```text
Tinner_loop = 3(I - 1) + 2
            = 3I - 1 cycles
```

<a id="inner-per-outer"></a>
### Add inner setup

Each outer iteration executes:

```text
MOVLW I       1 cycle
MOVWF inner   1 cycle
inner loop    3I - 1 cycles
```

So:

```text
Tinner_per_outer = 3I + 1 cycles
```

<a id="outer-control"></a>
### Add outer control

Across `N` outer iterations:

```text
first N - 1 outer passes:
    DECFSZ + GOTO = 3 each

final outer pass:
    DECFSZ with skip = 2
```

Thus:

```text
Touter_control = 3(N - 1) + 2
               = 3N - 1 cycles
```

<a id="nested-formula"></a>
### Combine the terms

The one-time outer setup is:

```text
MOVLW N = 1
MOVWF outer = 1
total = 2 cycles
```

Total:

```text
Tnested = 2 + N(3I + 1) + (3N - 1)
        = N(3I + 4) + 1 cycles
```

This formula belongs to **this exact code and timing boundary**.

Move an instruction, add a `NOP`, change the preload sequence, or include surrounding code and the expression must be re-derived.

<a id="zero-preload"></a>
### A loaded zero still means 256 decrements

For either 8-bit counter:

```text
loaded 0x01 -> 1 effective decrement
...
loaded 0xFF -> 255 effective decrements
loaded 0x00 -> 256 effective decrements
```

Use the effective count 256 in the timing equation when a byte counter is loaded with zero.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="three-four-example"></a>
### Worked example: `N = 3`, `I = 4`

**Known**

```text
Tnested = N(3I + 4) + 1
```

**Reasoning**

```text
T = 3(3×4 + 4) + 1
  = 3(16) + 1
  = 49 cycles
```

At 4 MHz:

```text
T = 49 us
```

<a id="one-ms-example"></a>
### Worked example: exactly 1 ms

At 4 MHz:

```text
1 ms = 1000 instruction cycles
```

We want:

```text
1000 = N(3I + 4) + 1
```

One exact integer solution is:

```text
I = 11
N = 27
```

Check:

```text
T = 27(3×11 + 4) + 1
  = 27(37) + 1
  = 1000 cycles
  = 1.000 ms
```

<a id="range-step-example"></a>
### Worked example: range and step size

Minimum:

```text
N = 1
I = 1

T = 1(3×1 + 4) + 1
  = 8 cycles
```

Maximum with both byte counters loaded with `0x00`:

```text
N = 256
I = 256

T = 256(3×256 + 4) + 1
  = 197633 cycles
  = 197.633 ms at 4 MHz
```

Change outer count by one while holding `I` fixed:

```text
Delta_N = 3I + 4 cycles
```

Change inner count by one while holding `N` fixed:

```text
Delta_I = 3N cycles
```

If `N = I = x`:

```text
T = x(3x + 4) + 1
  = 3x^2 + 4x + 1
```

The growth is quadratic, not exponential.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="solve-target"></a>
### Choose counts for a target

A practical process is:

1. convert the desired time to instruction cycles;
2. write the exact timing equation;
3. choose a practical inner count;
4. solve for the outer count;
5. round only when necessary;
6. substitute the chosen counts back into the equation;
7. calculate residual error;
8. decide whether a few one-cycle instructions can trim the residual;
9. measure the same defined interval.

<a id="when-not-to-use-delay-loops"></a>
### Know when a timer is the better tool

Blocking software delay loops are useful for:

- learning instruction timing;
- very simple short delays;
- controlled demonstrations;
- cases where the CPU genuinely has nothing else to do.

Move toward [hardware timers](timers.md#core-model) when:

- the wait is long;
- other work must continue;
- timing should remain stable when code changes;
- several time intervals must coexist;
- power matters;
- events should be scheduled rather than blocked.

<a id="common-errors"></a>
### Common nested-delay errors

Watch for:

- forgetting the final skip cost;
- counting inner setup only once instead of once per outer pass;
- treating a zero preload as zero iterations;
- assuming routine timing equals waveform timing;
- substituting literal register values instead of effective counts;
- using a memorized formula after changing the code.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. For `N = 3`, `I = 4`, calculate the delay at 4 MHz.
2. If `I = 12`, how much does the delay change when `N` increases by one?
3. If `N = 18`, how much does the delay change when `I` increases by one?
4. Set `N = I = x`. Write the timing expression and identify its growth type.
5. Calculate the maximum two-byte nested delay at 4 MHz when both counters are loaded with `0x00`.
6. Why is the inner-loop setup multiplied by `N`?
7. Why does a changed code path require a new timing derivation?
8. A target is 5 ms. Describe the process you would use to select `N` and `I`; do not guess counts.
9. When is a hardware timer preferable to another level of software nesting?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. `T = 3(3×4 + 4) + 1 = 49 cycles = 49 us`.
2. `Delta_N = 3(12) + 4 = 40 cycles = 40 us`.
3. `Delta_I = 3(18) = 54 cycles = 54 us`.
4. `T = 3x^2 + 4x + 1`; quadratic.
5. `197633 us = 197.633 ms`.
6. Every outer iteration reloads and executes the inner loop from the beginning.
7. The formula is a count of the exact instructions actually executed; changing the path changes the count.
8. Convert 5 ms to cycles, use `N(3I+4)+1`, choose/solve integer counts, substitute to verify, then calculate residual error and measure the same boundary.
9. When the CPU should not be blocked, timing must survive code changes, waits become long, or several independent time intervals must coexist.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- derive the nested equation from the inside outward;
- explain every term in `N(3I + 4) + 1`;
- explain why zero means 256 effective decrements;
- calculate minimum, maximum, and count-step changes;
- solve and verify target-delay counter values;
- explain why equal counters create quadratic growth;
- identify the point where a timer is a better engineering choice.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Instruction Set Summary — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: `DECFSZ`, `GOTO`, and documented instruction cycle counts.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: pipeline/skip/control-flow timing context.

- [Instruction Timing and Software Delays](instruction-timing-software-delays.md#single-delay)
  - RCET prerequisite derivation for the one-byte `3n + 1` delay used as the starting point for this guide.

[Back to top](#top) · [Topics index](README.md)
