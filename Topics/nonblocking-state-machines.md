<a id="top"></a>

# RCET 3373 — Nonblocking Timing and Embedded State Machines

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

A blocking delay answers:

> Do nothing else until this time has passed.

That is acceptable for some simple demonstrations, but it scales poorly when an embedded system must scan inputs, update outputs, communicate, service several timed behaviors, or respond quickly to events.

Nonblocking timing asks a different question:

> Has enough time passed for this state to advance?

A hardware timer supplies a timebase while the application keeps doing useful work. This is the bridge from delay-loop thinking to event-driven embedded software.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- distinguish blocking and nonblocking timing;
- explain the difference between timer tick, application interval, and state duration;
- represent behavior as explicit states and transitions;
- use a periodic timer event to advance software time;
- keep ISR work small and move application behavior into main-level logic when practical;
- build several longer intervals from one regular system tick;
- explain rollover-safe elapsed-time reasoning conceptually;
- identify race/shared-state concerns between ISR and main code;
- choose between polling a timer flag, ISR tick generation, and direct hardware output depending on the requirement.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [PIC16F883 Timers](timers.md#core-model);
- [Interrupts and Context Saving](interrupts-context-saving.md#core-model);
- [Timing Measurement](measurement-c-timing.md#prediction-evidence).

Related topics:

- [Feedback Control and PID](feedback-control-pid.md#sampled-control);
- [Embedded C](embedded-c-pic16f883.md#volatile);
- [Embedded-System Integration and Troubleshooting](embedded-system-integration-troubleshooting.md#timing-budget).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="blocking-vs-nonblocking"></a>
### Blocking delay

A blocking sequence might be:

```text
turn LED on
-> wait 500 ms doing nothing else
-> turn LED off
-> wait 500 ms doing nothing else
```

The CPU cannot naturally perform unrelated foreground work during those waits.

<a id="tick-model"></a>
### Nonblocking periodic tick

Use a hardware timer to create a regular event:

```text
timer event every 1 ms
-> update software time or set an event flag
-> return
```

Main code keeps cycling:

```text
scan inputs
process communication
update state machines
perform ready work
repeat
```

Application actions occur when their deadlines or elapsed intervals are satisfied.

<a id="state-machine"></a>
### State machine

A state machine makes persistent behavior explicit.

```text
STATE_ON
  if 500 ms elapsed -> LED off, enter STATE_OFF

STATE_OFF
  if 500 ms elapsed -> LED on, enter STATE_ON
```

The state variable remembers where the application is between main-loop passes.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="timer-tick"></a>
### One timer tick can support many application intervals

Suppose Timer2 produces a 1 ms periodic event.

Software can build:

```text
10 ticks   -> 10 ms
50 ticks   -> 50 ms
100 ticks  -> 100 ms
500 ticks  -> 500 ms
1000 ticks -> 1 s
```

The timer's hardware interval stays 1 ms. Different application counters or deadlines create the longer behaviors.

<a id="isr-boundary"></a>
### Keep the ISR's job clear

A simple timer ISR might:

1. verify the timer source;
2. clear or acknowledge the timer event correctly;
3. increment a software tick counter or set a flag;
4. return.

Application work can then happen outside the ISR.

A small ISR usually gives lower latency for other events, easier timing analysis, less shared-state complexity, and easier debugging. Immediate ISR work can still be justified when the requirement truly needs it.

<a id="elapsed-time"></a>
### Track elapsed time rather than waiting

Conceptual C-style logic:

```c
if ((uint16_t)(now - lastChange) >= interval)
{
    lastChange = now;
    advanceState();
}
```

The important model is:

```text
current time - saved event time -> elapsed time
```

With unsigned modular counters, carefully constructed elapsed-time comparisons can continue to work across counter rollover as long as the interval and counter-width assumptions are valid.

<a id="multiple-state-machines"></a>
### Several state machines can share one timebase

One main loop can service:

```text
LED state machine
motor-sequence state machine
button state
serial timeout state
sensor-sampling state
```

Each keeps its own state and timing information while sharing one hardware tick. This scales much better than independent busy-wait delays.

<a id="events"></a>
### Time is only one kind of event

A transition can depend on:

- elapsed time;
- button/input;
- UART byte;
- ADC threshold;
- CAN message;
- completion flag;
- fault condition.

A useful embedded model is:

```text
current state + event or condition -> action + next state
```

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="blink-example"></a>
### Worked example: nonblocking 500 ms blink

**Known**

```text
system tick = 1 ms
desired half-cycle = 500 ms
```

Store:

```text
state = ON or OFF
lastChange = tick value when state last changed
```

Main-loop logic:

```text
if elapsed >= 500 ticks:
    toggle output
    remember new lastChange
    update state
```

Between changes, main can keep scanning inputs and handling communication.

<a id="two-task-example"></a>
### Worked example: two independent rates

One 1 ms tick supports:

```text
sensor sample every 20 ms
status LED toggle every 500 ms
```

These do not require two hardware timers. Each task tracks its own elapsed interval, and neither task blocks the other while waiting.

<a id="isr-load-example"></a>
### Worked example: ISR execution budget

Suppose:

```text
tick period = 1 ms
ISR execution = 40 us
```

The ISR consumes:

```text
40 us / 1000 us = 4%
```

of processor time if it executes once per tick.

If the ISR grows to 600 us, it consumes 60% of the processor time and leaves much less time for foreground work.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="state-machine-checklist"></a>
### State-machine checklist

For each behavior:

1. list the states;
2. define what each state means physically;
3. define events or conditions that cause transitions;
4. define actions on transition or while in state;
5. define required timing;
6. identify persistent variables;
7. identify data shared with ISR or hardware;
8. define a safe initial/reset state;
9. define fault or timeout behavior;
10. decide how to prove each transition.

<a id="nonblocking-debug"></a>
### Common nonblocking-timing failures

- timer tick is not the expected period;
- timer flag is never cleared;
- ISR increments the wrong or shared variable;
- shared C state is missing required `volatile` treatment;
- a multi-byte time value is read inconsistently across an ISR update;
- comparison uses the wrong time units;
- the state variable is reinitialized every main-loop pass;
- a deadline is reset continuously before it expires;
- long ISR or blocking code prevents timely foreground work;
- output transition is timed from software response instead of the underlying hardware event.

Use the timer and interrupt guides to debug those lower layers first.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Distinguish blocking and nonblocking timing.
2. With a 2 ms system tick, how many ticks represent 100 ms?
3. Why can one hardware timer support several software intervals?
4. What information must persist between main-loop passes for a state machine?
5. Why is a short ISR usually easier to integrate than a very large ISR?
6. A state variable is recreated with its initial value every function call. Why can that break a state machine?
7. A 1 ms ISR takes 250 us. What fraction of CPU time is consumed if it runs every tick?
8. Name three non-time events that could cause a state transition.
9. Why can reading a multi-byte tick counter shared with an ISR require extra care on an 8-bit MCU?
10. Why is “toggle output, delay, toggle output, delay” difficult to extend to simultaneous UART and sensor work?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. Blocking code waits and prevents ordinary foreground progress; nonblocking code checks elapsed time or events while continuing other work.
2. 50 ticks.
3. Software can maintain separate counters or deadlines derived from the common tick.
4. Current state and any timing/event history needed to decide the next transition.
5. It consumes less CPU time and latency, reduces shared-state complexity, and keeps application behavior easier to reason about.
6. The program loses memory of the previous state and effectively restarts the state machine each call.
7. 25%.
8. Examples: button event, UART data, ADC threshold, CAN message, fault flag, operation complete.
9. The ISR can change one byte between foreground reads of the bytes; an atomic snapshot or critical-section strategy may be needed.
10. Busy waits prevent the foreground from servicing those other tasks while time passes.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- distinguish blocking waits from elapsed-time scheduling;
- derive application intervals from a periodic tick;
- describe an embedded state machine;
- keep ISR work intentionally bounded;
- run several timed behaviors from one timebase;
- identify shared-state and rollover concerns;
- move from delay-loop thinking to event-driven application structure.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Timer and Interrupt chapters — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: hardware timer and interrupt behavior underlying the software timebase.

- Microchip Technology Inc., *Compiled Tips 'N Tricks Guide*, DS01146B — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/SupportingCollateral/01146B.pdf
  - Used for: practical embedded-software state-machine and timing design patterns.

- [PIC16F883 Timers](timers.md#core-model) and [Interrupts and Context Saving](interrupts-context-saving.md#core-model)
  - RCET prerequisite guides for the hardware event sources used by this state-machine model.

[Back to top](#top) · [Topics index](README.md)
