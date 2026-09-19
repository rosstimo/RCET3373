<a id="top"></a>

# RCET 3373 — PIC16F883 ADC and Sensor Conditioning

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

A microcontroller cannot directly reason about an arbitrary analog voltage. The PIC16F883 ADC converts a selected analog input into a 10-bit digital result that software can store, compare, display, log, or use for control.

A correct conversion depends on more than setting one start bit.

The complete path includes:

```text
physical quantity
-> sensor / conditioning
-> analog pin
-> input multiplexer
-> sample-and-hold acquisition
-> ADC conversion clock
-> 10-bit result
-> software interpretation
```

If the signal range, reference, source impedance, acquisition time, or conversion clock is wrong, the ADC can return a perfectly valid-looking number that is not a trustworthy measurement.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- identify the PIC16F883 ADC data path and major control registers;
- configure an analog-capable pin for analog input;
- select an ADC channel and voltage references;
- distinguish acquisition time from conversion time;
- select an ADC conversion clock from the data-sheet requirements;
- start a conversion and determine when it has completed;
- interpret the 10-bit result in `ADRESH:ADRESL`;
- relate an ADC code to an input voltage using the selected reference range;
- explain how source impedance affects acquisition;
- identify basic sensor-conditioning needs such as scaling, offset, filtering, and protection;
- verify ADC behavior with independent voltage measurement.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [ADC and DAC Foundations](adc-dac-foundations.md#core-model);
- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md#core-model);
- [PIC16F883 Architecture](pic16f883-architecture.md#data-memory);
- [Data Memory, SFRs, and Banking](data-memory-sfrs-banking.md#core-model).

Related topics:

- [Interrupts and Context Saving](interrupts-context-saving.md#event-flag-enable) if conversion-complete interrupts are used;
- [Timing Measurement](measurement-c-timing.md#prediction-evidence) for verifying signal and timing assumptions.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="adc-data-path"></a>
### ADC data path

The PIC16F883 ADC is a successive-approximation converter with a sample/hold front end.

A useful model is:

```text
ANx pin
  -> analog multiplexer
  -> holding capacitor acquires input voltage
  -> conversion process
  -> 10-bit result
  -> ADRESH:ADRESL
```

The selected input must remain connected long enough for the internal holding capacitor to settle sufficiently before conversion begins.

**Official visual reference:** In Section 9.0, **Analog-to-Digital Converter (ADC) Module**, of the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf), inspect the ADC block diagram and analog-input model.

Focus on the channel multiplexer, holding capacitor, reference inputs, and result registers.

<a id="adc-registers"></a>
### Major ADC registers and fields

| Register | Important role |
| --- | --- |
| `ANSEL`, `ANSELH` | select pins for analog/digital behavior |
| `ADCON0` | ADC clock selection, channel selection, `GO/DONE`, ADC enable |
| `ADCON1` | result justification and voltage-reference selection |
| `ADRESH:ADRESL` | 10-bit conversion result |
| `PIR1.ADIF` | conversion-complete interrupt flag |
| `PIE1.ADIE` | ADC interrupt enable |

Use the data sheet for exact bit positions and reset states.

<a id="acquisition-conversion"></a>
### Acquisition and conversion are different intervals

**Acquisition time**

Time allowed for the ADC's internal holding capacitor to charge toward the selected input voltage.

**Conversion time**

Time required after sampling to produce the digital result.

A design can have a correct conversion clock and still be inaccurate if acquisition time is too short.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="pin-channel"></a>
### Configure the pin and select the channel

For an analog input:

1. identify the physical pin and its `ANx` channel;
2. configure the applicable `ANSEL/ANSELH` bit for analog operation;
3. configure the pin as an input;
4. select the channel with the ADC channel-select field;
5. choose the voltage-reference arrangement.

A pin can be physically wired correctly and still fail as an ADC input if its digital/analog configuration does not match the intended function.

<a id="reference-result"></a>
### Reference range gives the code scale

For the normal ideal model with `VREF-` and `VREF+`:

```text
input span = VREF+ - VREF-
codes = 1024
```

An ideal approximate code relationship is:

```text
code ≈ (VIN - VREF-) / (VREF+ - VREF-) × 1024
```

Clamp interpretation to the converter's valid 10-bit code range and use the data sheet's transfer-function details for precision work.

`ADFM` controls whether the 10-bit result is left- or right-justified in `ADRESH:ADRESL`.

Right justification is convenient when software wants to treat the pair as a conventional 10-bit unsigned value.

<a id="adc-clock"></a>
### Choose a valid conversion clock

The ADC requires a conversion clock period, commonly called `TAD`, within the device's specified range for accurate conversion.

The selected ADC clock comes from the available ADC clock choices in `ADCON0`.

Do not choose a divider merely because it “works in simulation.”

Use:

1. actual `FOSC`;
2. the data-sheet ADC clock options;
3. the specified `TAD` requirements.

Then verify that the selected clock is valid for the operating conditions.

<a id="acquisition-time"></a>
### Allow the input to acquire

Changing the selected channel connects a different analog source to the ADC's internal sample/hold network.

Before starting conversion, allow enough acquisition time for the holding capacitor to settle.

Required acquisition time depends on the device input network and the external source impedance.

A high-impedance sensor or resistor network may require more acquisition time than a low-impedance source.

**Official visual reference:** Microchip Developer Help, [ADC Acquisition Time](https://developerhelp.microchip.com/xwiki/bin/view/products/data-converters/adc-specs/acquisition-time/), shows a typical SAR ADC input model.

Use that figure to identify the source resistance, internal switch/multiplexer resistance, and holding capacitor that create the settling requirement. Use the PIC16F883 data sheet for the device-specific calculation and limits.

<a id="conversion-sequence"></a>
### Conversion sequence

A reusable reasoning sequence is:

```text
configure analog pin
-> enable ADC
-> select channel/reference/clock
-> wait required acquisition time
-> set GO/DONE
-> conversion occurs
-> GO/DONE clears when complete
-> ADIF records completion
-> read ADRESH:ADRESL
```

If interrupts are used, `ADIE`, `PEIE`, and `GIE` participate in the interrupt path. The hardware conversion itself does not require the CPU to sit in a polling loop.

<a id="sensor-conditioning"></a>
### Sensor conditioning happens before the ADC

A real sensor signal may need conditioning before it is safe and useful.

Common goals include:

- **scaling:** keep the signal inside the ADC reference range;
- **offset:** move a bipolar or shifted signal into the allowed input range;
- **buffering:** reduce the effective source impedance seen by the ADC;
- **filtering:** reduce unwanted noise or bandwidth before sampling;
- **protection:** limit voltage/current during abnormal conditions;
- **ground/reference integrity:** ensure the measured voltage has a meaningful reference.

Do not “fix” an unsuitable analog signal only in software if the electrical interface violates the ADC input requirements.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="code-voltage-example"></a>
### Worked example: ideal 2.50 V input on a 0–5.00 V range

**Known**

```text
VREF- = 0 V
VREF+ = 5.00 V
VIN   = 2.50 V
resolution = 10 bit
```

**Reasoning**

```text
code ≈ 2.50 / 5.00 × 1024
     ≈ 512
```

**Result**

The ideal code is approximately midscale, around 512.

**Meaning**

A code near 512 is plausible for a 2.5 V input on a 0–5 V range. It is not proof of accuracy; reference error, input settling, noise, and converter error still matter.

<a id="voltage-from-code"></a>
### Worked example: estimate voltage from code

**Known**

```text
code = 768
range = 0–5.00 V
```

**Reasoning**

Using the introductory ideal model:

```text
VIN ≈ 768 / 1024 × 5.00 V
    ≈ 3.75 V
```

**Result**

Estimated input ≈ 3.75 V.

<a id="source-impedance-example"></a>
### Worked example: why source impedance matters

Suppose two sources both produce 2.5 V:

- Source A has low output impedance;
- Source B reaches the ADC through a very large resistance.

The final DC voltage can be the same, but Source B may charge the internal holding capacitor more slowly.

If conversion begins too soon after channel selection, Source B can produce a larger settling error.

The engineering response is to use the device acquisition-time model, extend acquisition time, and/or reduce/buffer the source impedance.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="adc-checklist"></a>
### ADC bring-up checklist

When a conversion is wrong or unstable:

1. measure the actual analog voltage with an appropriate instrument;
2. verify the physical pin/channel;
3. verify `ANSEL/ANSELH`;
4. verify pin direction;
5. verify channel selection;
6. verify reference selection and actual reference voltage;
7. verify `FOSC` and ADC clock/`TAD`;
8. verify acquisition time after channel/source changes;
9. verify result justification;
10. verify software combines `ADRESH:ADRESL` correctly;
11. check source impedance and conditioning;
12. check noise, grounding, and reference stability.

<a id="measurement-proof"></a>
### Compare ADC result with independent evidence

A useful verification record contains:

```text
measured VIN
selected references
predicted ideal code
observed ADC code
difference
possible error sources
```

Do not compare an ADC code with an assumed sensor voltage when the voltage can be measured directly.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. How many possible codes does the PIC16F883 10-bit ADC provide?
2. What is the difference between acquisition time and conversion time?
3. Why must an analog-capable pin be configured appropriately before using it as an ADC input?
4. What registers hold the 10-bit result?
5. What does `GO/DONE` indicate?
6. Why does source impedance affect the required acquisition time?
7. For an ideal 0–5 V range, approximately what code corresponds to 1.25 V?
8. For an ideal 0–3.3 V range, estimate the voltage represented by code 512.
9. Give three reasons a sensor output might need analog conditioning before the ADC.
10. If the ADC code is wrong, why should you measure the actual pin voltage before changing the conversion formula?
11. Which interrupt gates apply if ADC conversion completion is handled by an ISR?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. 1024 codes, numbered 0 through 1023.
2. Acquisition lets the sample/hold network settle to the input; conversion produces the digital result after sampling.
3. The pin's analog/digital configuration controls whether the intended analog path is active.
4. `ADRESH:ADRESL`.
5. It is set to start conversion and clears when conversion is complete.
6. Higher source impedance increases the RC settling time of the ADC input/holding network.
7. `1.25/5 × 1024 ≈ 256`.
8. `512/1024 × 3.3 V ≈ 1.65 V`.
9. Examples: scaling, offset, buffering, filtering, protection, reference/ground conditioning.
10. The ADC may be correctly converting a physical voltage different from the assumed one; independent measurement separates analog-interface errors from conversion/software errors.
11. `ADIE`, `PEIE`, and `GIE`, plus correct `ADIF` handling.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- trace the analog input through the ADC data path;
- identify the major PIC16F883 ADC registers;
- distinguish acquisition and conversion;
- select a conversion clock from `FOSC` and data-sheet limits;
- estimate voltage from code and code from voltage;
- explain why source impedance matters;
- identify basic sensor-conditioning functions;
- outline a measurement-driven ADC troubleshooting process.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Section 9.0 "Analog-to-Digital Converter (ADC) Module" and associated electrical specifications — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 ADC architecture, registers, channel/reference selection, result format, acquisition/conversion requirements, and interrupt behavior.

- Microchip Developer Help, *Analog-to-Digital Converter (ADC) Acquisition Time* — https://developerhelp.microchip.com/xwiki/bin/view/products/data-converters/adc-specs/acquisition-time/
  - Used for: sample/hold acquisition model and source-impedance explanation.

- Microchip Technology Inc., *Understanding A/D Converter Performance Specifications*, AN693 — https://ww1.microchip.com/downloads/jp/AppNotes/00693A.pdf
  - Used for: converter transfer functions, quantization, resolution, accuracy, and error terminology.

[Back to top](#top) · [Topics index](README.md)
