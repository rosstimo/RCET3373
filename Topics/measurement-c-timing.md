# RCET 3373 - Measurement Strategy and C Timing Self-Learning Guide

## What you should be able to do

After working through this guide, you should be able to:

- choose an oscilloscope or frequency counter based on the measurement question;
- distinguish a calculated prediction from measured evidence;
- identify common timing-measurement limitations;
- troubleshoot disagreement between prediction and measurement in an orderly way;
- explain why C source alone does not determine exact PIC instruction timing.

## Choose the instrument for the question

An oscilloscope is useful for:

- pulse width;
- duty cycle;
- edge shape;
- sequence relationships;
- transient behavior;
- visually comparing a waveform with a prediction.

A frequency counter is useful when the primary question is a stable repetition frequency measured precisely over many cycles.

Neither instrument is universally better.

## Record the prediction first

Before measuring a timing result, write down:

- expected pulse width;
- expected full period;
- expected frequency;
- relevant tolerance;
- the exact source or instruction path used for the calculation.

Only then measure.

This keeps the instrument result tied to a falsifiable prediction.

## Measurement limitations matter

Possible limitations include:

- oscilloscope sample interval or resolution;
- time-base accuracy;
- trigger stability;
- probe attenuation/setup;
- measuring too few cycles;
- rounded display values;
- oscillator tolerance.

A screenshot should include enough settings and context for another person to understand what was measured.

## Troubleshoot disagreement in order

Use this sequence before inventing a correction factor:

```text
Is FOSC actually what was assumed?
-> Is the intended build/source actually programmed?
-> Was the executed path counted correctly?
-> Was pulse width confused with full period?
-> Are instrument/probe/time-base settings correct?
-> Only then investigate subtler device/hardware effects.
```

Do not change the formula merely to force agreement with a measurement.

## Why C source does not give exact cycle timing

Consider:

```c
for (int i = 0; i < n; i++)
{
    ;
}
```

The exact instruction-cycle count is not determined by these source lines alone. It depends on the generated machine instructions, including compiler output, optimization, target architecture, integer width, and generated control flow.

Cycle-accurate timing belongs to the instructions that actually execute.

That is one reason hardware timers become preferable for many precise or nonblocking timing jobs.

## Practice

1. You need to verify HIGH pulse width and duty cycle. Which instrument is the natural first choice?
2. You only need a precise stable repetition frequency over many cycles. Which instrument is a natural choice?
3. Your measured frequency is exactly twice the predicted value. What common waveform-calculation mistake should you check early?
4. Why is "one C statement equals one PIC instruction" not a valid timing rule?
5. A calculated and measured delay disagree. Should you first add an unexplained correction factor or verify clock/build/path/measurement assumptions?

## Answer key

1. Oscilloscope.
2. Frequency counter.
3. Check whether a half-cycle or pulse width was mistaken for the full period.
4. The compiler translates source into machine instructions, and the generated path depends on compiler and target details.
5. Verify the clock, programmed build, executed path, waveform interpretation, and instrument setup first.
