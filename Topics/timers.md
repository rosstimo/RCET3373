<a id="top"></a>

# RCET 3373 — PIC16F883 Timers

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

Software delay loops make instruction timing visible, but they keep the CPU busy.

Hardware timers solve a different problem:

> Count time or external events independently while the processor performs other work.

The PIC16F883 provides three timers with different architectures:

- **Timer0:** simple 8-bit overflow counter with a shared prescaler;
- **Timer1:** wider 16-bit counter with internal/external clock options;
- **Timer2:** 8-bit period-register timer with prescaler and postscaler.

The important skill is not memorizing which timer is “best.” It is translating a timing requirement into a clock chain, count range, event, flag, and software response.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- derive a timer tick from `FOSC`, source selection, and prescaling;
- calculate Timer0 full-count intervals and preloads;
- explain the Timer0 flag/enable path;
- calculate and split a Timer1 preload;
- explain why Timer1's 16-bit range can simplify longer intervals;
- calculate Timer2 match periods using `PR2 + 1`;
- distinguish Timer2 match period from postscaled interrupt period;
- distinguish a hardware timer event from a later ISR/output transition;
- diagnose common timer configuration failures;
- select Timer0, Timer1, or Timer2 from an application requirement.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Instruction Timing and Software Delays](instruction-timing-software-delays.md#instruction-cycle);
- [Interrupts and Context Saving](interrupts-context-saving.md#event-flag-enable);
- [Digital Timing](digital-timing.md#period-frequency).

Related topics:

- [Timing Measurement](measurement-c-timing.md#measurement-strategy);
- future PWM/control topics that use a timer as a hardware timebase.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="clock-chain"></a>
### Start with the clock chain

For the course 4 MHz oscillator:

```text
FOSC = 4 MHz
FCY  = FOSC / 4 = 1 MHz
TCY  = 1 us
```

When a timer uses internal `FOSC/4`, `TCY` is the starting tick before any timer prescaler.

A general timer reasoning chain is:

```text
clock source
-> prescaler
-> timer count
-> overflow or compare/match
-> flag
-> interrupt gating or polling
-> software response
```

Keep the **hardware event** separate from what software does after the event.

<a id="preload-math"></a>
### Preload math

For an N-state up-counter that overflows after its maximum value:

```text
counts remaining = modulus - preload
```

For an 8-bit timer:

```text
modulus = 256
preload = 256 - desired counts
```

For a 16-bit timer:

```text
modulus = 65536
preload = 65536 - desired counts
```

The requested time must first be converted into timer counts using the actual timer tick.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="timer0"></a>
### Timer0: 8-bit overflow timing

Timer0 counts from its current value through `0xFF`, then rolls over to `0x00`.

Starting at zero gives 256 increments.

At 4 MHz, internal clock, no prescaler:

```text
timer tick = 1 us
full-count interval = 256 × 1 us
                    = 256 us
```

With a 1:8 prescaler:

```text
timer tick = 8 us
full-count interval = 256 × 8 us
                    = 2.048 ms
```

The Timer0/WDT prescaler is shared. The `PSA` setting determines which resource receives it.

**Official visual reference:** In the Timer0 chapter of the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf), inspect the Timer0/WDT prescaler block diagram.

Follow the selected clock source through the prescaler assignment into `TMR0`, and note that the prescaler resource is shared rather than independently duplicated.

<a id="timer0-preload"></a>
### Timer0 preload

For a 100-count interval with a 1 us timer tick:

```text
preload = 256 - 100
        = 156
        = 0x9C
```

Starting `TMR0` at `0x9C` gives an ideal 100 us hardware count to overflow.

A software write to `TMR0` affects subsequent increment timing. The device data sheet documents a two-instruction-cycle increment inhibition after writing `TMR0`.

Do not hide that device behavior inside a universal “ISR correction.” Treat it as one specific part of the implemented timing path.

<a id="timer0-interrupt"></a>
### Timer0 event and interrupt path

Separate the concepts:

```text
Timer0 rollover
-> T0IF set
-> T0IE source enable
-> GIE global enable
-> interrupt service
```

`T0IE` does not start or stop Timer0.

Software can:

- poll `T0IF`; or
- enable interrupt service and respond in an ISR.

The same timer hardware event occurs either way.

Timer0 uses the direct interrupt path and does not require `PEIE`.

<a id="timer1"></a>
### Timer1: 16-bit range

Timer1 uses the 16-bit pair:

```text
TMR1H:TMR1L
```

Important controls include:

- `TMR1CS`: internal `FOSC/4` or external `T1CKI` source;
- Timer1 prescaler;
- `TMR1ON`: start/stop;
- `TMR1IF`: interrupt flag;
- `TMR1IE`: source enable;
- `PEIE` and `GIE`: interrupt gates for the peripheral path.

A 16-bit timer can represent much longer intervals than Timer0 before overflow.

**Official visual reference:** In the Timer1 chapter of the PIC16F883 data sheet, inspect the Timer1 block diagram.

Focus on clock-source selection, prescaler, `TMR1H:TMR1L`, and how Timer1's interrupt flag connects to the peripheral-interrupt system.

<a id="timer1-preload"></a>
### Timer1 preload

At 4 MHz, internal `FOSC/4`, 1:1 prescale:

```text
timer tick = 1 us
desired interval = 20,000 us
counts = 20,000

preload = 65,536 - 20,000
        = 45,536
        = 0xB1E0
```

Split the 16-bit value:

```text
TMR1H = 0xB1
TMR1L = 0xE0
```

A deliberate stop/load/start sequence is easy to reason about when learning reload timing. Keep ideal timer-count math separate from software overhead.

<a id="timer2"></a>
### Timer2: period register and postscaler

Timer2 uses a different model:

```text
FOSC/4
  -> prescaler
  -> TMR2
  -> compare with PR2
  -> postscaler
  -> TMR2IF
```

The period-register relationship is:

```text
match period = (PR2 + 1) × base timer tick × prescale
```

The `+ 1` matters.

If `PR2 = 99`, the period contains 100 count states.

**Official visual reference:** In the Timer2 chapter of the PIC16F883 data sheet, inspect the Timer2 block diagram and `T2CON` register description.

Follow `FOSC/4` through the prescaler and `TMR2/PR2` comparator, then distinguish the comparator event from the postscaled `TMR2IF` event.

<a id="timer2-period"></a>
### Timer2 match period versus interrupt period

At 4 MHz, 1:1 prescale:

```text
timer tick = 1 us
PR2 = 99
match period = (99 + 1) × 1 us
             = 100 us
```

With a 1:5 postscale:

```text
TMR2/PR2 match = every 100 us
TMR2IF         = every 500 us
```

The postscaler changes when the interrupt flag is asserted. It does not change the underlying `TMR2 = PR2` match period.

<a id="timer2-side-effects"></a>
### Timer2 configuration writes have side effects

The PIC16F883 data sheet documents that:

- writing `T2CON` clears the Timer2 prescaler and postscaler counters;
- writing `TMR2` clears the prescaler and postscaler counters;
- writing `T2CON` does not itself clear `TMR2`.

This matters in real code.

If a fast main loop repeatedly rewrites `T2CON`, a postscaler may repeatedly reset before it reaches its intended count.

Configure the timer when configuration needs to change. Do not continuously rewrite an unchanged configuration without reason.

<a id="polling-vs-interrupt"></a>
### Polling versus interrupt response

A timer flag can be used in two broad ways.

**Polling**

```text
main code repeatedly checks flag
-> when set, software handles event
```

**Interrupt**

```text
flag + enables request interrupt
-> CPU vectors to ISR
-> software handles event
```

Polling and interrupts do not create different hardware timer periods. They change how software notices/responds to the event.

<a id="hardware-vs-observed"></a>
### Hardware event time versus observed software edge

Do not assume a GPIO transition inside an ISR occurs at the exact timer overflow or match.

The hardware event happens first.

Then the executed path may include:

```text
interrupt latency
-> vector branch
-> context save
-> source identification
-> handler instructions
-> diagnostic GPIO update
```

Therefore:

- calculate the timer's hardware interval;
- calculate/trace the software response separately;
- measure the observable edge separately.

Do not apply one universal ISR-latency correction to every design.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="timer0-example"></a>
### Worked example: Timer0 full count with 1:256 prescaler

**Known**

```text
FOSC = 4 MHz
internal timer source
TCY = 1 us
prescale = 256
```

**Reasoning**

```text
timer tick = 1 us × 256
           = 256 us

interval = 256 counts × 256 us
         = 65,536 us
         = 65.536 ms
```

**Result**

Timer0 full-count overflow interval = 65.536 ms.

<a id="timer1-example"></a>
### Worked example: Timer1 20 ms preload

At a 1 us tick:

```text
counts = 20,000
preload = 65,536 - 20,000
        = 45,536
        = 0xB1E0
```

Load:

```text
TMR1H = 0xB1
TMR1L = 0xE0
```

This is the ideal hardware-count preload. Software reload overhead is a separate path.

<a id="timer2-example"></a>
### Worked example: Timer2 with prescale and postscale

Given:

```text
FOSC = 4 MHz
PR2 = 99
prescale = 1:4
postscale = 1:5
```

Base tick:

```text
1 us
```

Prescaled tick:

```text
4 us
```

Match period:

```text
(99 + 1) × 4 us
= 400 us
```

Interrupt-flag period:

```text
400 us × 5
= 2.000 ms
```

<a id="timer2-maximum-example"></a>
### Worked example: maximum Timer2 interrupt interval in this configuration model

At 4 MHz:

```text
PR2 = 255
prescale = 1:16
postscale = 1:16
```

```text
interval = 256 × 16 us × 16
         = 65.536 ms
```

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="timer-selection"></a>
### Choose the timer from the requirement

| Timer | Useful architectural feature | Typical reason to choose it |
| --- | --- | --- |
| Timer0 | simple 8-bit overflow/preload, shared prescaler | short/simple overflow timing or external count use |
| Timer1 | 16-bit range, internal/external source | longer intervals that fit naturally in one 16-bit count |
| Timer2 | `PR2` period register + prescaler + postscaler | repeated period generation / regular peripheral timebase |

This table is a reasoning aid, not a rule that forbids other uses.

Always compare:

- required interval/range;
- resolution;
- source clock;
- prescaler choices;
- reload/period mechanism;
- interrupt path;
- whether another peripheral needs the timer;
- software overhead and response requirements.

<a id="timer-troubleshooting"></a>
### Timer troubleshooting checklist

If the expected timer event is missing or wrong:

1. verify `FOSC`;
2. verify timer clock-source selection;
3. verify prescaler assignment/value;
4. verify preload or `PR2`;
5. verify timer enable/start state;
6. verify the flag directly;
7. if interrupts are used, verify the source enable and correct `PEIE/GIE` path;
8. verify the flag-clearing method;
9. check whether software repeatedly rewrites timer registers/configuration;
10. distinguish the hardware event from a later diagnostic output edge;
11. measure the actual tick/period using the appropriate instrument.

<a id="scope-boundary"></a>
### Scope boundary

This guide establishes timer architecture, count math, event/flag behavior, and timer selection.

It does not claim:

- one universal ISR-overhead correction;
- complete CCP/PWM configuration;
- a completed servo implementation.

Those are separate topics built on the timer timebase.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. At `FOSC = 4 MHz`, what are `FCY` and `TCY`?
2. With Timer0 internal clock and no prescaler, what is the interval from `0x00` to overflow?
3. What is that interval with a 1:256 prescaler?
4. With a 1 us Timer0 tick, what preload gives an ideal 100-count interval?
5. What is the ideal Timer1 preload for 20 ms at 4 MHz, internal `FOSC/4`, 1:1?
6. Split that Timer1 preload into high and low bytes.
7. At 4 MHz and Timer2 1:1 prescale, what `PR2` gives a 100 us match period?
8. If that match uses a 1:5 postscale, how often does `TMR2IF` occur?
9. A fast main loop repeatedly writes the same `T2CON` value and a postscaled event never arrives. What should you suspect?
10. Timer1 is counting and `TMR1IF` becomes set, but the ISR is never entered. Which interrupt gates should you inspect?
11. Why is a GPIO toggle inside a timer ISR later than the underlying hardware timer event?
12. Which timer architecture would you examine first for a 20 ms single-overflow interval at a 1 us internal tick, and why?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. `FCY = 1 MHz`; `TCY = 1 us`.
2. 256 us.
3. 65.536 ms.
4. `256 - 100 = 156 = 0x9C`.
5. `0xB1E0`.
6. `TMR1H = 0xB1`, `TMR1L = 0xE0`.
7. `PR2 = 99 = 0x63`.
8. Every 500 us.
9. Writes to `T2CON` reset the prescaler/postscaler counters, so the postscale may never accumulate.
10. `TMR1IE`, `PEIE`, and `GIE`, plus correct source/flag handling.
11. Interrupt latency, vectoring, context save, source checks, and executed handler instructions occur after the timer event.
12. Timer1, because its 16-bit range can represent 20,000 counts directly without repeated 8-bit overflows.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- draw the general timer clock chain;
- calculate an internal timer tick;
- derive an 8-bit or 16-bit preload;
- explain Timer0's flag/enable path;
- explain why Timer1's width changes the range;
- derive Timer2's `(PR2 + 1)` period;
- distinguish Timer2 match and postscaled flag periods;
- explain Timer2 configuration side effects;
- distinguish polling from interrupt response;
- distinguish hardware event time from an ISR-driven output edge;
- select a timer from an engineering requirement.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Timer0, Timer1, Timer2, Interrupts, and Electrical Specifications material — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: timer clock sources, widths, prescalers, `PR2`, postscaler, register side effects, flags/enables, `TMR0` write behavior, and device timing.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A, timer chapters — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: broader classic mid-range timer architecture and timing context.

- [PIC16F883 Interrupt Reference](https://github.com/rosstimo/pic_projects/blob/main/References/PIC16F883-Interrupts.md)
  - Related RCET reference for flag/enable/latency reasoning used when timer events request interrupt service.

[Back to top](#top) · [Topics index](README.md)
