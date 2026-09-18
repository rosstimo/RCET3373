# RCET 3373 - Logic Levels, Noise Margin, and Loading Self-Learning Guide

## What you should be able to do

After working through this guide, you should be able to:

- identify guaranteed HIGH, LOW, and undefined voltage regions from a datasheet;
- distinguish a measured voltage from its interpreted logic state;
- calculate basic HIGH and LOW noise margins;
- use input/output current specifications to make a basic loading judgment;
- explain why matching supply voltages do not automatically guarantee a valid interface.

## Digital states are implemented with analog voltages

A logic HIGH is not a magical symbol. It is a voltage that must fall inside the receiving device's guaranteed HIGH region under the actual load.

A typical datasheet separates the input-voltage range into:

- a guaranteed LOW region;
- an undefined or not-guaranteed region;
- a guaranteed HIGH region.

When you measure a voltage, record the measurement first and then interpret it using the applicable specification.

Good engineering language looks like this:

```text
Measured: 4.82 V
Interpretation: guaranteed HIGH under the cited input specification
```

## Use the guaranteed limits

For an interface, useful datasheet quantities include:

- `VIH` minimum: the minimum input voltage guaranteed to be read as HIGH;
- `VIL` maximum: the maximum input voltage guaranteed to be read as LOW;
- `VOH` minimum under a stated load;
- `VOL` maximum under a stated load;
- source/sink current limits and their test conditions.

Do not use absolute-maximum ratings as normal design targets.

## Noise margin

HIGH noise margin:

```text
NMH = guaranteed output HIGH - required input HIGH
```

LOW noise margin:

```text
NML = maximum input LOW - guaranteed output LOW
```

Example:

A source guarantees at least 3.7 V for HIGH and the receiver requires at least 3.5 V.

```text
NMH = 3.7 V - 3.5 V = 0.2 V
```

A positive margin means the interface has some tolerance for variation or voltage loss. A negative margin means the interface is not guaranteed by those specifications.

## Loading is a second check

Voltage compatibility is not enough. The source must also be able to drive the connected load.

For several inputs driven by one output:

1. verify that the loaded output voltage still satisfies the receiver threshold;
2. add the input-current demand and compare it with the source's guaranteed current capability.

Avoid memorizing one universal fan-out number when the actual current data is available.

## Practice

1. A device guarantees `VOH >= 3.7 V`. The receiver requires `VIH >= 3.5 V`. Find the HIGH noise margin.
2. A measured output is 3.0 V and the receiver requires 3.5 V for guaranteed HIGH. Is the interface guaranteed HIGH?
3. Four identical inputs are connected to one output. What two checks must be made before calling the interface valid?
4. Why is "both devices use a 5 V supply" not enough information to prove logic compatibility?
5. Why should an absolute-maximum current rating not be used as the intended operating current?

## Answer key

1. `NMH = 0.2 V`.
2. No. The measured 3.0 V is below the receiver's 3.5 V guaranteed-HIGH threshold.
3. Check the loaded output voltage against the input threshold and check total input-current demand against the source's guaranteed drive capability.
4. Logic compatibility depends on guaranteed output levels, required input thresholds, and loading, not supply voltage alone.
5. Absolute-maximum ratings describe damage limits, not recommended or guaranteed normal operation.
