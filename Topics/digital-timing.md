<a id="top"></a>

# RCET 3373 — Digital Timing

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

A truth table tells you **what** logical relationship should exist. Real hardware must also satisfy **when** that relationship becomes valid.

A circuit can be logically correct and still fail because:

- an output changes too late;
- input data changes too close to a clock edge;
- a pulse is too short;
- one subsystem assumes a different timing relationship than another;
- an asynchronous event arrives near a sampling edge.

Digital timing turns those ideas into quantities you can predict, measure, and compare with specifications.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- identify period, frequency, pulse width, duty cycle, and active edges;
- distinguish propagation delay, setup time, and hold time;
- convert between frequency and period using engineering notation;
- distinguish synchronous from asynchronous events;
- read a basic timing diagram as a set of relationships between signals;
- identify the correct two events to measure for a specified delay;
- explain why a static logic result can be correct while a real circuit still fails at speed.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md#core-model).

Related topics:

- [Instruction Timing and Software Delays](instruction-timing-software-delays.md#instruction-cycle);
- [Measurement Strategy and C Timing](measurement-c-timing.md#measurement-strategy);
- [PIC16F883 Timers](timers.md#core-model).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="period-frequency"></a>
### Period and frequency

Frequency and period are reciprocals:

```text
f = 1 / T
T = 1 / f
```

Examples:

```text
1 MHz  -> 1 us period
4 MHz  -> 250 ns period
10 kHz -> 100 us period
```

<a id="pulse-width"></a>
### Pulse width and duty cycle

A pulse width is the time a signal remains in one state.

A full period includes the complete repeating waveform.

For a repetitive digital waveform:

```text
duty cycle = high time / period
```

Do not confuse a HIGH pulse width with the full period.

<a id="timing-relationships"></a>
### Cause, reference edge, and response

A timing specification normally relates two defined events.

Examples:

- input edge -> output edge;
- data valid -> active clock edge;
- active clock edge -> data no longer required to remain valid;
- control assertion -> valid bus data.

Always identify the reference event and the response/constraint event before measuring.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="propagation-delay"></a>
### Propagation delay

Propagation delay is the interval between a specified input event and the corresponding output event.

The exact reference points come from the device specification. A timing diagram may define the measurement from one 50% voltage crossing to another, or use some other stated reference.

Do not invent the measurement points.

<a id="setup-hold"></a>
### Setup and hold time

For data captured on an active clock edge:

- **setup time** is how long the data must already be valid before the edge;
- **hold time** is how long the data must remain valid after the edge.

A correct data value can still be captured incorrectly if either requirement is violated.

<a id="synchronous-asynchronous"></a>
### Synchronous and asynchronous events

A **synchronous** event is aligned to a timing relationship shared with the receiving system.

An **asynchronous** event has no guaranteed phase relationship to the receiver's clock.

A pushbutton transition is a useful example of an asynchronous event relative to a processor clock.

Asynchronous does not mean random or invalid. It describes the timing relationship.

<a id="timing-diagrams"></a>
### Read timing diagrams as relationships

A useful timing-diagram reading process is:

1. identify the signals;
2. identify active edges or levels;
3. find arrows/dimension lines between events;
4. match each dimension label to the specification table;
5. determine whether the value is a minimum, maximum, or typical value;
6. decide what evidence would show that the requirement is satisfied.

**Visual reference:** The [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf) includes device timing diagrams in its **Electrical Specifications** material.

Use those diagrams to practice matching waveform edges to the corresponding timing parameter names and limits rather than reading only the numeric table.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="frequency-example"></a>
### Worked example: frequency from period

**Known**

```text
T = 50 us
```

**Find**

Frequency.

**Reasoning**

```text
f = 1 / T
  = 1 / 50 us
  = 20 kHz
```

**Result**

```text
f = 20 kHz
```

<a id="full-period-example"></a>
### Worked example: pulse widths versus full period

**Known**

A signal is HIGH for 20 us and LOW for 20 us.

**Find**

Full period and frequency.

**Reasoning**

```text
T = 20 us + 20 us
  = 40 us

f = 1 / 40 us
  = 25 kHz
```

**Result**

The full period is 40 us and the frequency is 25 kHz.

<a id="setup-example"></a>
### Worked example: interpret a setup requirement

**Known**

A receiver requires data to be valid at least 10 ns before a rising clock edge.

In the observed waveform, data becomes valid 14 ns before that edge.

**Result**

The setup requirement is satisfied with 4 ns of additional timing margin.

This does not prove the hold-time requirement; that must be checked separately.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="predict-before-measuring"></a>
### Predict before measuring

Before using an oscilloscope or logic analyzer, write down:

- expected frequency or period;
- expected pulse width or duty cycle;
- the edge/event relationship that matters;
- the acceptable limit;
- what observation would confirm or contradict the prediction.

This makes the instrument trace evidence rather than decoration.

<a id="timing-failure-checklist"></a>
### Timing failure checklist

If the logical function appears correct but behavior fails at speed:

1. verify clock/frequency assumptions;
2. check propagation delays;
3. check setup/hold requirements;
4. check minimum pulse widths;
5. verify whether the source and receiver are synchronous;
6. inspect loading/signal integrity if edges are slow or distorted;
7. compare measured timing with the actual specification, not a remembered rule of thumb.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Find the period of a 2 MHz signal.
2. Find the frequency of a waveform with a 50 us period.
3. A signal is HIGH for 20 us and LOW for 20 us. Find the full period and duty cycle.
4. A device specification gives a maximum propagation delay of 30 ns. You measure 24 ns. Is the measured value within that limit?
5. Why can a circuit satisfy its truth table but fail at higher speed?
6. Is a pushbutton event necessarily synchronous to the processor clock?
7. Data becomes valid 8 ns before a clock edge, but the receiver requires 12 ns setup time. What is wrong?
8. Why should you identify the exact reference edges before measuring propagation delay?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. `T = 1 / 2 MHz = 0.5 us`.
2. `f = 1 / 50 us = 20 kHz`.
3. Period = 40 us; HIGH duty cycle = 20/40 = 50%.
4. Yes. 24 ns is below a 30 ns maximum.
5. Propagation, setup, hold, pulse-width, or related timing requirements may be violated even when the static logic relationship is correct.
6. No. It normally arrives independently of the processor clock.
7. The data provides only 8 ns of setup time, 4 ns less than required.
8. A timing specification is defined between particular events. Measuring different points can produce a number that does not correspond to the specification.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- convert frequency and period in both directions;
- distinguish pulse width from period;
- explain propagation delay;
- explain setup and hold requirements;
- distinguish synchronous and asynchronous events;
- read a simple timing diagram;
- plan a timing measurement before connecting an instrument;
- explain why a logically correct circuit can still fail because of timing.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Electrical Specifications and timing-diagram material — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: device-specific examples of propagation, timing limits, clock relationships, and timing-diagram interpretation.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: clock/instruction-cycle context and general mid-range timing relationships.

[Back to top](#top) · [Topics index](README.md)
