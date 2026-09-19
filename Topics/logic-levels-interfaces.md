<a id="top"></a>

# RCET 3373 — Logic Levels, Noise Margin, and Loading

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

Digital circuits use analog voltages to represent logical states.

A signal is not a valid logic HIGH merely because it is “close to 5 V,” and two devices are not automatically compatible because they use the same supply voltage.

A reliable interface must satisfy two separate questions:

1. Does the source produce voltages the receiver is guaranteed to interpret correctly?
2. Can the source drive the connected electrical load while maintaining those voltages?

These checks become important whenever a microcontroller pin drives another logic device, sensor input, LED interface, transistor stage, or longer interconnect.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- distinguish measured voltage from interpreted logic state;
- identify `VIH`, `VIL`, `VOH`, and `VOL` in technical documentation;
- calculate HIGH and LOW noise margins;
- distinguish guaranteed operating specifications from absolute-maximum limits;
- compare total input-current demand with output-drive capability;
- decide whether an interface is guaranteed by the available specifications;
- identify when a level translator, buffer, transistor stage, or other interface circuit may be required.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Digital Representation](digital-representation.md#core-model) for bit/state notation.

Related topics:

- [PORTB configuration and loading](digital-io-portb-configuration.md#electrical-loading);
- [PIC16F883 electrical limits](pic16f883-architecture.md#electrical-limits);
- [Digital Timing](digital-timing.md#core-model) for timing requirements that must be checked in addition to voltage compatibility.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

The source and receiver each contribute part of the interface specification.

| Quantity | Meaning | Belongs to |
| --- | --- | --- |
| `VOH(min)` | Lowest output voltage guaranteed for HIGH under stated load | Source |
| `VOL(max)` | Highest output voltage guaranteed for LOW under stated load | Source |
| `VIH(min)` | Lowest input voltage guaranteed to be read as HIGH | Receiver |
| `VIL(max)` | Highest input voltage guaranteed to be read as LOW | Receiver |

The region between `VIL(max)` and `VIH(min)` is not a guaranteed logic region.

A measured value can be recorded independently from its interpretation:

```text
Measured: 4.82 V
Interpretation: guaranteed HIGH under the cited input specification
```

That wording separates evidence from conclusion.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="guaranteed-regions"></a>
### Guaranteed logic regions

For an input:

- at or below `VIL(max)`, LOW is guaranteed;
- at or above `VIH(min)`, HIGH is guaranteed;
- between those limits, the logic interpretation is not guaranteed by that specification.

Do not invent a midpoint threshold unless the documentation actually defines one.

<a id="noise-margin"></a>
### Noise margin

HIGH noise margin:

```text
NMH = VOH(min) - VIH(min)
```

LOW noise margin:

```text
NML = VIL(max) - VOL(max)
```

Positive margin means the guaranteed source output still clears the receiver requirement.

Negative margin means the interface is not guaranteed by those limits.

<a id="loading"></a>
### Loading changes the output condition

Output-voltage guarantees are normally tied to stated source/sink currents.

As load current increases, the output pin may no longer maintain the same voltage.

For several loads connected to one output:

1. determine the total current demand;
2. verify that the source remains within its allowed operating range;
3. verify that the resulting guaranteed output voltage still satisfies the receiver threshold.

Do not use one memorized “fan-out” number when the actual current specifications are available.

<a id="absolute-maximum"></a>
### Absolute maximum is not a design target

Absolute-maximum ratings describe stress boundaries beyond which damage or reliability problems may occur.

They are not:

- normal operating points;
- guaranteed output-drive conditions;
- recommended LED currents;
- proof that a voltage/current is appropriate for continuous use.

Use the DC/electrical characteristics and design recommendations for normal operation.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="high-margin-example"></a>
### Worked example: HIGH noise margin

**Known**

A source guarantees:

```text
VOH(min) = 3.7 V
```

The receiver requires:

```text
VIH(min) = 3.5 V
```

**Find**

The HIGH noise margin.

**Reasoning**

```text
NMH = VOH(min) - VIH(min)
    = 3.7 V - 3.5 V
    = 0.2 V
```

**Result**

```text
NMH = 0.2 V
```

**Meaning**

The guaranteed HIGH output exceeds the receiver's guaranteed-HIGH requirement by 0.2 V.

<a id="invalid-interface-example"></a>
### Worked example: measured voltage below the guaranteed HIGH threshold

**Known**

```text
measured output = 3.0 V
VIH(min)        = 3.5 V
```

**Result**

The measured voltage is below the guaranteed-HIGH threshold.

It may sometimes be interpreted as HIGH by a real device, but the specification does not guarantee that behavior.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="interface-checklist"></a>
### Interface checklist

For a digital connection, check:

1. source supply and receiver supply;
2. `VOH(min)` and `VOL(max)` at the relevant load;
3. `VIH(min)` and `VIL(max)`;
4. HIGH and LOW noise margins;
5. total source/sink current;
6. per-pin and total-device current limits;
7. whether a pin is actually configured for digital operation;
8. timing requirements if the signal is clocked or edge-sensitive.

<a id="electrical-loading"></a>
### Physical loads need more than logic-threshold analysis

An LED, relay driver, transistor, or other load may draw far more current than a logic input.

For those cases, logic compatibility is only one part of the design. Use the device's output-current/voltage specifications and the external component ratings.

For PIC16F883 output-pin configuration, continue with [PORTB configuration and loading](digital-io-portb-configuration.md#electrical-loading).

**Official table reference:** In the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf), use the **Electrical Specifications** chapter and its DC-characteristics tables for the actual input thresholds, output levels, and current conditions.

Focus on the test conditions attached to each value. A voltage specification without its stated current/load condition can be misleading.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. A source guarantees `VOH >= 3.7 V`. The receiver requires `VIH >= 3.5 V`. Find the HIGH noise margin.
2. A source guarantees `VOL <= 0.25 V`. The receiver guarantees LOW for `VIL <= 0.8 V`. Find the LOW noise margin.
3. A measured output is 3.0 V and the receiver requires 3.5 V for guaranteed HIGH. Is HIGH guaranteed?
4. Four identical inputs are connected to one output. What two broad electrical checks must be made?
5. Why is “both devices use a 5 V supply” insufficient to prove logic compatibility?
6. Why should an absolute-maximum current rating not be used as the intended operating current?
7. A logic threshold check passes, but the source voltage droops when an LED is attached. What category of specification should you inspect next?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. `NMH = 3.7 - 3.5 = 0.2 V`.
2. `NML = 0.8 - 0.25 = 0.55 V`.
3. No. The measured 3.0 V is below the guaranteed-HIGH threshold.
4. Check that the loaded output voltage still satisfies the receiver threshold and that total input-current demand is within the source's guaranteed drive capability.
5. Compatibility depends on output levels, input thresholds, current/load, and sometimes timing—not supply voltage alone.
6. Absolute-maximum ratings are stress limits rather than guaranteed normal operating conditions.
7. Inspect the source's output-drive/DC-characteristic specifications and the load current requirements.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- distinguish `VIH`/`VIL` from `VOH`/`VOL`;
- calculate HIGH and LOW noise margin;
- explain the undefined/not-guaranteed input region;
- separate a measured voltage from its logic interpretation;
- explain why load current changes the interface analysis;
- explain why absolute-maximum ratings are not normal design targets;
- outline a datasheet-driven compatibility check between two digital devices.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Electrical Specifications chapter — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 input thresholds, output-voltage/current conditions, current limits, and absolute-maximum versus DC-characteristic distinctions.

- Microchip Technology Inc., *Common 8-Bit PIC Microcontroller I/O Pin Issues*, TB3009 — https://www.microchip.com/en-us/application-notes/tb3009
  - Used for: practical I/O-pin behavior and interface troubleshooting context.

- Microchip Technology Inc., *Hardware Techniques for PICmicro Microcontrollers*, AN234 — https://www.microchip.com/en-us/application-notes/an234
  - Used for: practical output/load and interface examples.

[Back to top](#top) · [Topics index](README.md)
