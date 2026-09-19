<a id="top"></a>

# RCET 3373 — Embedded-System Integration and Troubleshooting

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

Individual peripherals are easier to understand in isolation than in a real embedded system.

A complete design may need to:

- sample analog inputs;
- maintain a control loop;
- communicate over UART, I2C, SPI, or CAN;
- update outputs;
- preserve settings;
- respond to interrupts;
- survive reset or faults;
- meet timing deadlines.

Integration problems usually appear at the boundaries between otherwise-correct subsystems.

A useful system-level question is:

> What shared resource, timing assumption, electrical interface, or state transition connects the two things that appear to be failing?

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- build a timing and resource budget for a small embedded system;
- identify shared MCU resources that can create peripheral conflicts;
- separate hardware event timing from software response timing;
- explain reset, watchdog, brown-out, and startup-state concerns;
- distinguish transient faults from persistent design/configuration errors;
- identify persistent-state risks across reset or power loss;
- design a layered verification strategy;
- troubleshoot from evidence instead of changing several variables at once;
- explain tradeoffs among CPU time, latency, memory, pins, timer resources, and communication bandwidth.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

This guide integrates material from the rest of the course.

Useful prerequisites include:

- [PIC16F883 Architecture](pic16f883-architecture.md#architecture-overview)
- [Interrupts and Context Saving](interrupts-context-saving.md#core-model)
- [PIC16F883 Timers](timers.md#core-model)
- [Nonblocking Timing and State Machines](nonblocking-state-machines.md#core-model)
- [PIC16F883 ADC](pic16f883-adc.md#core-model)
- [UART](uart-asynchronous-serial.md#core-model)
- [I2C](i2c.md#core-model)
- [SPI](spi.md#core-model)
- [PIC16F883 Data EEPROM](pic16f883-data-eeprom.md#core-model)
- [Feedback Control and PID](feedback-control-pid.md#core-model)

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="resource-budget"></a>
### Resource budget

An embedded design consumes finite resources.

Typical categories include:

| Resource | Example questions |
| --- | --- |
| CPU time | How much time is spent in ISRs, control calculations, and polling? |
| Timers | Which subsystem owns Timer0, Timer1, Timer2, CCP/PWM timebases? |
| Pins | Are alternate functions competing for the same physical pin? |
| RAM | Are buffers, state variables, and ISR temporaries within available space? |
| Program memory | Does code/data fit with margin? |
| Interrupt latency | Can urgent events be serviced before their deadline? |
| Bus bandwidth | Can serial traffic complete at the required rate? |
| Power/current | Can outputs and external loads operate safely together? |

A subsystem can work alone and fail after integration because both subsystems assumed they owned the same resource.

<a id="timing-budget"></a>
### Timing budget

For a periodic function, break the requirement into pieces.

```text
period
-> hardware event
-> interrupt latency
-> ISR work
-> foreground work
-> communication or conversion delay
-> deadline margin
```

The total must fit the application requirement.

The correct question is not only:

> How long does my function take?

It is also:

> What else can delay this function from running when it needs to?

<a id="fault-layers"></a>
### Fault layers

Troubleshooting is easier when failures are classified by layer:

1. power/reset/clock;
2. electrical I/O;
3. pin multiplexing/configuration;
4. peripheral configuration;
5. interrupt/event routing;
6. application state/timing;
7. protocol/data interpretation;
8. physical-system behavior.

Start at the lowest layer that can explain the evidence.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="shared-resources"></a>
### Shared resources create hidden coupling

Examples of shared resources include:

- one pin with several alternate functions;
- one timer used as the timebase for more than one peripheral;
- one interrupt vector serving many sources;
- one MSSP peripheral switching between SPI and I2C roles;
- one CPU servicing several real-time tasks;
- one power rail supplying several loads.

A configuration change made for one subsystem can therefore alter another subsystem without changing its source file.

Document ownership of shared resources explicitly.

<a id="reset-startup"></a>
### Reset and startup are system states

Reset does more than send execution to `0x0000`.

A robust design asks:

- What caused reset?
- What are the documented register reset states?
- Which outputs must remain safe during startup?
- Which persistent values should be loaded?
- Which stale flags must be cleared?
- When is it safe to enable interrupts or actuators?

The PIC16F883 includes reset-related features such as Power-on Reset, Brown-out Reset, MCLR, and Watchdog Timer behavior. Use the device data sheet to determine the exact cause/status behavior and startup sequence.

<a id="watchdog"></a>
### Watchdog is not a substitute for fixing software

A watchdog can recover a system that stops servicing it within the configured interval.

That can improve fault recovery, but it does not make a logically incorrect design correct.

A useful watchdog strategy includes:

- a reasoned timeout;
- servicing only after required system work is known to be healthy;
- a way to identify/watch reset cause;
- safe startup after watchdog reset;
- logging or preserving diagnostic state when practical.

If code blindly clears the watchdog inside every fast loop path, the watchdog may prove very little about system health.

<a id="persistent-state"></a>
### Persistent state needs a reset policy

Persistent state can outlive the software execution that created it.

For each EEPROM-backed value, define:

- default value when no valid record exists;
- validation method;
- version/format expectations;
- what happens after partial/corrupt update;
- whether reset should reload it immediately;
- whether the stored value can command unsafe hardware.

Persistent storage is part of system startup, not an isolated memory exercise.

<a id="concurrency"></a>
### Concurrency can exist without an RTOS

Even a simple PIC application has concurrent-looking behavior because:

- hardware timers run while code executes;
- peripherals receive bytes independently;
- interrupts can preempt foreground code;
- ADC conversions can complete later;
- physical events occur asynchronously.

That means shared state must have clear ownership.

Questions to ask:

- Who writes this variable?
- Who reads it?
- Can an interrupt change it between two foreground instructions?
- Is the access atomic at this MCU width?
- Does a flag represent an event or a current state?
- Can events be lost if two occur before software clears the flag?

<a id="verification-layers"></a>
### Verification should be layered

A good verification path moves from simple evidence to integrated evidence:

1. **static review:** register values, equations, pin maps;
2. **build evidence:** warnings, map/listing, generated code;
3. **simulation/debugger:** state transitions and control flow;
4. **electrical measurement:** voltage, clock, waveforms;
5. **protocol decode:** UART/I2C/SPI/CAN transaction evidence;
6. **system behavior:** complete functional requirement;
7. **fault tests:** reset, timeout, disconnect, invalid input, power cycle.

Passing a high-level test does not prove every lower layer is healthy, but lower-layer evidence makes failures much easier to isolate.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="timer-conflict-example"></a>
### Worked example: two features want Timer2

Suppose:

- a state machine uses Timer2 for a 1 ms system tick;
- a new PWM design also assumes Timer2 is available as its timebase.

Each feature may work independently.

Together, changing Timer2's prescaler or `PR2` for PWM changes the system tick.

The bug is not in either high-level algorithm. The integration failure is **shared timer ownership**.

Possible responses include:

- redesign the common timebase;
- move one function to another timer;
- derive both requirements from one compatible timer configuration;
- use a different peripheral/resource.

<a id="latency-example"></a>
### Worked example: ISR load reduces foreground margin

Suppose:

```text
system period = 1 ms
timer ISR = 80 us
UART ISR worst case = 120 us
control update = 350 us
foreground housekeeping = 250 us
```

Worst-case occupied time:

```text
80 + 120 + 350 + 250 = 800 us
```

Nominal margin:

```text
1000 - 800 = 200 us
```

That 200 us margin must also absorb variation, additional interrupts, and execution paths not included in the estimate.

A design that “usually fits” can still miss deadlines under worst-case overlap.

<a id="reset-example"></a>
### Worked example: safe actuator startup

Suppose an output eventually controls a motor driver.

A safe startup plan can be:

```text
reset
-> keep output driver disabled
-> establish clock/configuration
-> load/validate persistent settings
-> initialize timer/control state
-> clear stale flags
-> establish safe command value
-> enable interrupts
-> enable actuator last
```

The exact hardware sequence depends on the circuit, but the principle is to make unsafe output impossible during incomplete initialization.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="troubleshooting-method"></a>
### Evidence-driven troubleshooting method

When a system fails:

1. write the expected behavior;
2. identify the earliest observable point where reality differs;
3. measure or inspect that layer;
4. change one variable at a time;
5. keep notes on what the evidence ruled out;
6. move upward only after the lower layer is verified.

Avoid changing:

- clock settings;
- code;
- wiring;
- compiler optimization;
- peripheral configuration

all at once. That can make the system start working without teaching you what was wrong.

<a id="integration-checklist"></a>
### Integration checklist

Before declaring a design complete, verify:

- clock source and frequency;
- configuration/reset behavior;
- pin multiplexing;
- electrical loading;
- timer ownership;
- interrupt routing and ISR duration;
- ADC/reference/acquisition assumptions;
- serial bus bandwidth and error handling;
- state-machine timing;
- persistent-data validity;
- watchdog/reset strategy;
- RAM/program-memory use;
- worst-case rather than only average timing;
- behavior after disconnect, bad data, reset, and power cycle.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Why can two peripherals work separately but fail when enabled together?
2. Name five finite MCU resources that should be budgeted.
3. Why is worst-case interrupt overlap more important than average ISR time for a hard deadline?
4. What is the first question to ask when a shared pin stops behaving after another peripheral is enabled?
5. Why should an actuator often be enabled late in the startup sequence?
6. What does a watchdog prove if it is serviced unconditionally in every fast loop?
7. Why can a multi-byte foreground variable shared with an ISR require an atomic-access strategy?
8. Give one example of a protocol-level failure and one example of an electrical-layer failure.
9. Why should persistent EEPROM data have a validity/version strategy in an important application?
10. A system misses a 1 ms deadline only when serial traffic is heavy. What categories would you investigate first?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. They may compete for a timer, pin, interrupt path, CPU time, memory, bus, or power resource.
2. Examples: CPU time, timers, pins, RAM, program memory, interrupt latency, bus bandwidth, output current.
3. A deadline fails when the worst allowed combination exceeds the available time, even if average behavior is fast.
4. Check pin multiplexing and which peripheral currently owns/configures that physical pin.
5. It prevents incomplete initialization or reset defaults from producing unsafe physical motion/output.
6. Very little about application health; the loop may still be logically broken while continuing to clear the watchdog.
7. The ISR can update part of the value while foreground code is reading another part.
8. Protocol: wrong UART baud or missing I2C ACK. Electrical: invalid logic voltage, missing pull-up, excessive load, poor bus termination.
9. Power loss, partial writes, or software revisions can otherwise make old/corrupt state appear valid.
10. ISR duration/frequency, buffering, foreground scheduling, bus throughput, and overall timing margin.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- create a resource and timing budget;
- identify shared-resource conflicts;
- explain startup/reset as a system state;
- describe a useful watchdog strategy;
- identify concurrency even without an RTOS;
- describe layered verification;
- troubleshoot from the earliest failed observable layer;
- defend system tradeoffs using evidence and resource limits.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Reset, Watchdog, Interrupt, Timer, peripheral, and Electrical Specifications sections — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883-specific reset/watchdog/resource behavior.

- Microchip Developer Help, *8-bit PIC MCU Design Recommendations* — https://developerhelp.microchip.com/xwiki/bin/view/products/mcu-mpu/8bit-pic/design-recommendations/
  - Used for: system-level hardware design, reset, programming/debug, oscillator, and I/O considerations.

- Microchip Technology Inc., *Compiled Tips 'N Tricks Guide*, DS01146B — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/SupportingCollateral/01146B.pdf
  - Used for: practical embedded state-machine, watchdog, software, and integration patterns.

- Microchip Technology Inc., *Hardware Techniques for PICmicro Microcontrollers*, AN234 — https://www.microchip.com/en-us/application-notes/an234
  - Used for: practical hardware integration and interface design context.

[Back to top](#top) · [Topics index](README.md)
