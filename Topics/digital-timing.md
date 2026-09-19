# RCET 3373 - Digital Timing Self-Learning Guide

## What you should be able to do

After working through this guide, you should be able to:

- identify propagation delay, setup time, hold time, pulse width, period, and active edges;
- convert between frequency and period using engineering notation;
- explain why a logically correct circuit can still fail because of timing;
- distinguish synchronous from asynchronous interaction;
- read a basic timing diagram and decide what must be measured.

## A truth table does not tell you when

A truth table tells you the logical relationship between signals. Real hardware also has to satisfy timing requirements.

If an input changes and the output is supposed to become HIGH, the next question is:

> When does the output become HIGH?

That delay between cause and observed response is part of the circuit behavior.

## Propagation delay

Propagation delay is a measurable interval between a specified input event and the corresponding output event. The exact reference points come from the applicable device specification.

When a maximum propagation delay is specified, compare the measured or calculated delay with that limit.

## Setup and hold

For data captured on an active clock edge:

- **setup time** is the required interval that data must already be valid before the edge;
- **hold time** is the required interval that data must remain valid after the edge.

A logically correct data value can still be captured incorrectly if the timing requirement is violated.

## Frequency and period

```text
f = 1 / T
T = 1 / f
```

Examples:

```text
1 MHz  -> 1 us period
4 MHz  -> 250 ns oscillator period
10 kHz -> 100 us period
```

Be careful not to confuse one pulse width with a complete waveform period.

## Synchronous and asynchronous events

A synchronous event is aligned to a clock relationship used by the system.

An asynchronous event is not inherently aligned to the receiver's clock. A pushbutton transition is a useful example.

Asynchronous does not mean random or unreliable. It describes the timing relationship.

## Predict before measuring

Before using an oscilloscope, write down:

- expected frequency or period;
- expected pulse width;
- the edge or event relationship that matters;
- the observation that would confirm or contradict the prediction.

This makes the measurement evidence rather than just a screenshot.

## Practice

1. Find the period of a 2 MHz signal.
2. Find the frequency of a waveform with a 50 us period.
3. A signal is HIGH for 20 us and LOW for 20 us. What is the full period and frequency?
4. Why can a circuit satisfy its truth table but fail at higher speed?
5. Is a pushbutton event necessarily synchronous to the processor clock?

## Answer key

1. `T = 1 / 2 MHz = 0.5 us`.
2. `f = 1 / 50 us = 20 kHz`.
3. Full period = 40 us, so `f = 25 kHz`.
4. Propagation, setup, hold, or other timing requirements may be violated even though the static logic relationship is correct.
5. No. A pushbutton event normally arrives independently of the processor clock and is therefore asynchronous to it.
