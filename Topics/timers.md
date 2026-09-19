# RCET 3373 - PIC16F883 Timers Self-Learning Guide

## What you should be able to do

After working through this guide, you should be able to:

- calculate the timer tick from the oscillator and prescaler;
- calculate Timer0 overflow intervals and preloads;
- explain the Timer0 event/flag/enable path;
- choose Timer1 when its 16-bit range fits an interval more naturally;
- calculate and split a Timer1 preload;
- calculate Timer2 match and interrupt periods using `PR2 + 1`, prescale, and postscale;
- distinguish a hardware timer event from a later ISR diagnostic/output transition;
- diagnose common timer configuration mistakes;
- choose among Timer0, Timer1, and Timer2 from the timing requirement rather than from a memorized preference.

This guide builds on [Instruction Timing and Software Delays](instruction-timing-software-delays.md) and [Interrupts and Context Saving](interrupts-context-saving.md).

## Start with the PIC instruction clock

For the course's 4 MHz oscillator example:

```text
FOSC = 4 MHz
FCY  = FOSC / 4 = 1 MHz
TCY  = 1 us
```

When Timer0 or Timer1 uses the internal `FOSC/4` source, this instruction-cycle timing becomes the starting timer tick before prescaling.

## Timer0: simple 8-bit overflow timing

Timer0 is an 8-bit count. Starting at `0x00`, a complete count to the next overflow uses 256 timer increments.

At 4 MHz with the internal clock and no Timer0 prescaler:

```text
timer tick = 1 us
interval   = 256 * 1 us
           = 256 us
```

### Prescaler

With a 1:8 prescaler:

```text
timer tick = 1 us * 8
           = 8 us

full-count interval = 256 * 8 us
                    = 2048 us
                    = 2.048 ms
```

The prescaler changes the timer input rate. It does not change the oscillator.

The Timer0/WDT prescaler is shared. The `PSA` setting determines which resource receives it.

### Preload

A preload reduces the number of counts remaining before overflow.

For a 100-count interval with a 1 us timer tick:

```text
preload = 256 - 100
        = 156
        = 0x9C
```

Starting `TMR0` at `0x9C` gives an ideal 100 us hardware count to overflow before software-response overhead is considered.

For exact timing after a software write to `TMR0`, remember that the device behavior taught in Week 4 includes a two-instruction-cycle increment inhibition after the write.

## Timer0 event, flag, and software response

The timer hardware can keep counting while the main program does other work.

The important concepts are separate:

- the timer overflows;
- `T0IF` records the event;
- `T0IE` controls whether that source may request interrupt service;
- `GIE` is the global interrupt gate;
- software must clear the flag as part of handling the event.

`T0IE` does not start or stop Timer0. It controls the interrupt response.

Polling `T0IF` and servicing `T0IF` in an ISR can use the same timer configuration. The conceptual change is how software waits for or responds to the event.

## Hardware event time versus observed software edge

Do not assume that a pin transition performed inside an ISR occurs at exactly the hardware overflow instant.

The timer event happens first. Vectoring, source checks, context handling, and the diagnostic instruction occur later.

Therefore:

- calculate the hardware timer interval;
- measure the actual observable edge separately;
- treat the difference as part of the executed software path.

Do not use one universal ISR-latency correction number unless the exact implementation has been verified.

## Timer1: 16-bit range

Timer1 uses the 16-bit `TMR1H:TMR1L` count.

Important controls in the taught foundation include:

- `TMR1CS`: internal `FOSC/4` or external `T1CKI` source selection;
- Timer1 prescaler choices;
- `TMR1ON`: start/stop control;
- interrupt path through `TMR1IF`, `TMR1IE`, `PEIE`, and `GIE`.

### Example: ideal 20 ms preload

At 4 MHz, internal `FOSC/4`, 1:1 prescale:

```text
TCY = 1 us
desired interval = 20,000 us
counts = 20,000

preload = 65,536 - 20,000
        = 45,536
        = 0xB1E0
```

Split the preload:

```text
TMR1H = 0xB1
TMR1L = 0xE0
```

For a deliberate reload, stop/load/restart is a clear classroom pattern. Keep software-overhead compensation separate from the ideal hardware count.

Timer1 is often a more natural choice than Timer0 when the requested interval fits its wider 16-bit range without awkward repeated overflow bookkeeping.

## Timer2: period register plus postscaler

Timer2 uses a different model.

The taught timing path is:

```text
FOSC/4
  -> prescaler
  -> TMR2 count
  -> comparison with PR2
  -> postscaler
  -> TMR2IF
```

The match-period model is:

```text
Timer2 match period =
    (PR2 + 1) * timer tick * prescale
```

The `+ 1` matters because `PR2 = 99` represents 100 count states in the period.

### Example: 100 us match period

At 4 MHz and 1:1 prescale:

```text
timer tick = 1 us
PR2 + 1 = 100
PR2 = 99 = 0x63
```

### Add the postscaler

If the match period is 100 us and the postscale is 1:5:

```text
TMR2/PR2 match = every 100 us
TMR2IF event   = every 500 us
```

The postscaler changes the interrupt-flag period, not the underlying TMR2/PR2 match period.

### Worked example

Given:

```text
FOSC = 4 MHz
PR2 = 99
prescale = 1:4
postscale = 1:5
```

Then:

```text
base tick = 1 us
prescaled tick = 4 us
match period = (99 + 1) * 4 us
             = 400 us
interrupt period = 400 us * 5
                 = 2.000 ms
```

### Maximum taught Timer2 interval

At 4 MHz:

```text
PR2 = 255
prescale = 1:16
postscale = 1:16
```

gives:

```text
256 * 16 us * 16 = 65.536 ms
```

per `TMR2IF` event.

## Timer2 configuration side effects matter

A useful Week 4 troubleshooting result was caused by repeatedly rewriting timer configuration in a fast main loop.

The taught device behavior is:

- writing `T2CON` clears the Timer2 prescaler/postscaler counters;
- writing `TMR2` also clears those scaler counters;
- writing `T2CON` does **not** clear `TMR2` itself.

If software expects a 1:10 postscaled event but continuously writes `T2CON`, the postscaler may never accumulate ten match events.

Configure the timer at the appropriate event instead of rewriting the same setup continuously without need.

## Choosing among the timers

Use the application requirement.

### Timer0

Useful when an 8-bit overflow/preload model is sufficient and when the Timer0 source/prescaler arrangement fits the job.

### Timer1

Useful when a longer 16-bit count range makes the requested interval fit naturally.

### Timer2

Useful when the `PR2` period model and prescaler/postscaler structure fit a repeated shorter event or one-shot/event-driven design.

The goal is not to memorize a universally "best" timer. Compare width, range, clocking, period mechanism, and interrupt behavior against the requirement.

## Practice

1. At `FOSC = 4 MHz`, what is `TCY`?
2. With Timer0 internal clock and no prescaler, what is the interval from `0x00` to overflow?
3. What is that Timer0 full-count interval with a 1:256 prescaler?
4. With a 1 us Timer0 tick, what preload gives an ideal 100-count interval?
5. What is the ideal Timer1 preload for 20 ms at 4 MHz, internal `FOSC/4`, 1:1?
6. Split that Timer1 preload into high and low bytes.
7. At 4 MHz, Timer2 1:1 prescale, what `PR2` gives a 100 us match period?
8. If that 100 us match uses a 1:5 postscale, how often does `TMR2IF` occur?
9. A fast main loop repeatedly writes the same `T2CON` value and a postscaled Timer2 interrupt never arrives. What should you suspect?
10. Timer1 is counting and `TMR1IF` becomes set, but the ISR is never entered. Which interrupt gates should you inspect?

## Answer key

1. `TCY = 1 us`.
2. `256 us`.
3. `65.536 ms`.
4. `256 - 100 = 156 = 0x9C`.
5. `0xB1E0`.
6. `TMR1H = 0xB1`, `TMR1L = 0xE0`.
7. `PR2 = 99 = 0x63`.
8. Every `500 us`.
9. Repeated writes to `T2CON` clear the prescaler/postscaler counters, preventing the postscale count from accumulating.
10. Inspect `TMR1IE`, `PEIE`, and `GIE` in addition to the flag/handler logic.

## Scope boundary

This guide does not claim a universal exact ISR-latency correction, a completed servo implementation, or dedicated CCP/PWM configuration. Those were not verified as completed Week 4 teaching content.

## Official reference

The Week 4 course material used the **PIC16F882/883/884/886/887 Data Sheet**, including Timer0, Timer1, Timer2, and interrupt register documentation.

https://ww1.microchip.com/downloads/en/DeviceDoc/41291F.pdf
