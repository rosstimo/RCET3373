<a id="top"></a>

# RCET 3373 — ADC and DAC Foundations

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

Embedded systems often sit between two kinds of information:

- physical quantities that vary continuously;
- digital codes that processors can store and manipulate.

An **analog-to-digital converter (ADC)** maps an analog input into a digital code.

A **digital-to-analog converter (DAC)** maps a digital code into an analog output quantity.

Both are finite-resolution devices. They cannot represent infinitely many analog values, so converter reasoning always includes range, number of codes, quantization, reference conditions, and real-world error.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- distinguish ADC and DAC signal direction;
- calculate the number of codes represented by an N-bit converter;
- distinguish number of codes from maximum unsigned code;
- calculate the course ideal step size from full-scale range and bit depth;
- explain quantization;
- distinguish resolution from accuracy;
- explain the role of a reference voltage/range;
- identify why a higher bit count does not automatically guarantee a better real measurement.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Digital Representation](digital-representation.md#core-model);
- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md#core-model).

Related topics:

- [Address, Data, and Control Buses](buses-adc-dac.md#core-model);
- [PIC16F883 ADC](pic16f883-adc.md#core-model) for device-specific acquisition, register, and timing work once that guide is reached in the course.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

For an ideal N-bit converter:

```text
number of codes = 2^N
maximum unsigned code = 2^N - 1
```

For the course's introductory ideal-resolution model:

```text
ideal step ≈ full-scale range / 2^N
```

Examples:

| Resolution | Number of codes | Maximum unsigned code |
| ---: | ---: | ---: |
| 8 bit | 256 | 255 |
| 10 bit | 1024 | 1023 |
| 12 bit | 4096 | 4095 |

<a id="resolution"></a>
### Resolution

**Resolution** describes how finely the converter's digital code space divides the selected analog range in the ideal model.

<a id="accuracy"></a>
### Accuracy

**Accuracy** describes how close the actual conversion result is to the intended or true quantity after real error sources are considered.

More bits improve the ideal code granularity. They do not remove reference error, offset, gain error, nonlinearity, noise, source-impedance problems, or other system errors.

<a id="quantization"></a>
### Quantization

Quantization maps a continuous range of possible analog values into a finite set of digital codes or output steps.

Different analog values can therefore map to the same digital code.

That is a fundamental consequence of finite resolution.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="adc-direction"></a>
### ADC: analog input to digital code

An ADC performs this conceptual direction:

```text
analog quantity -> sampling/conversion -> digital code
```

The selected reference range defines what input span maps into the converter's code range.

A real ADC also has timing requirements, acquisition behavior, input-source requirements, and error specifications.

<a id="dac-direction"></a>
### DAC: digital code to analog output

A DAC performs the opposite conceptual direction:

```text
digital code -> conversion -> analog output
```

The output relationship depends on the DAC architecture, reference, output stage, and device-specific transfer function.

Do not assume every real DAC reaches exactly the reference voltage at its maximum code. Use the device's transfer equation.

<a id="reference-range"></a>
### The reference defines scale

A converter code has meaning only relative to the selected reference/range.

For example, a 10-bit code of 512 represents a different voltage if the converter operates over 0–5 V than if it operates over 0–3.3 V.

<a id="adc-vs-dac"></a>
### ADC and DAC share ideas but solve opposite problems

| Concept | ADC | DAC |
| --- | --- | --- |
| Input | Analog quantity | Digital code |
| Output | Digital code | Analog quantity |
| Finite resolution | Yes | Yes |
| Reference/range matters | Yes | Yes |
| Real errors beyond bit count | Yes | Yes |

**Official visual reference:** Microchip's [Understanding A/D Converter Performance Specifications, AN693](https://ww1.microchip.com/downloads/jp/AppNotes/00693A.pdf) includes ADC transfer-function and error diagrams.

Use those figures to distinguish the ideal staircase/code model from offset, gain, linearity, and quantization effects.

For an official DAC example, see the [MCP4921 product page](https://www.microchip.com/en-us/product/mcp4921), which links the 12-bit SPI DAC data sheet and its device-specific transfer behavior.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="eight-bit-example"></a>
### Worked example: 8-bit converter over 0–5 V

**Known**

```text
N = 8
range = 5 V
```

**Find**

Number of codes and ideal introductory step size.

**Reasoning**

```text
codes = 2^8 = 256

ideal step ≈ 5 V / 256
           ≈ 19.53 mV/count
```

**Result**

- 256 possible codes;
- maximum code 255;
- ideal introductory step ≈ 19.53 mV/count.

<a id="ten-bit-example"></a>
### Worked example: 10-bit converter over 0–3.3 V

```text
codes = 2^10 = 1024

ideal step ≈ 3.3 V / 1024
           ≈ 3.22 mV/count
```

The ideal code spacing is finer than the 8-bit example, but that alone does not establish total measurement accuracy.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

When converter results look wrong, do not begin by blaming “the ADC” or “the DAC.”

Check:

1. actual reference voltage/range;
2. configured resolution;
3. analog source range;
4. whether the code is being interpreted at the correct width;
5. whether the system has enough acquisition/settling time;
6. source impedance and loading;
7. noise and grounding;
8. offset/gain/linearity specifications;
9. whether the expected formula matches the actual converter's transfer function.

For the PIC16F883, device-specific ADC setup and acquisition timing belong in the [PIC16F883 ADC guide](pic16f883-adc.md#core-model).

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. How many codes does an 8-bit converter have?
2. What is the maximum unsigned code of a 10-bit converter?
3. How many codes does a 12-bit converter have?
4. Using the course ideal model, calculate the step size of an 8-bit 0–5 V converter.
5. Using the course ideal model, calculate the step size of a 10-bit 0–3.3 V converter.
6. Why are 256 levels numbered 0 through 255?
7. What is the difference between resolution and accuracy?
8. Why can two analog input values produce the same ADC code?
9. Why does a DAC's real transfer equation need to come from its data sheet?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. 256.
2. 1023.
3. 4096.
4. `5 V / 256 ≈ 19.53 mV/count`.
5. `3.3 V / 1024 ≈ 3.22 mV/count`.
6. Zero is one of the codes, so 256 distinct values run from 0 through 255.
7. Resolution is ideal code granularity; accuracy includes how close the real system result is to the intended/true value.
8. Finite-resolution quantization maps a continuous input range into a finite set of codes.
9. Real DAC architectures and output ranges differ; the actual device transfer function defines the relationship between code and output.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- state the direction of ADC and DAC conversion;
- calculate `2^N` codes and `2^N - 1` maximum code;
- calculate the course ideal step size;
- explain quantization;
- distinguish resolution from accuracy;
- explain why the reference/range matters;
- explain why bit depth alone does not guarantee total system accuracy.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *Understanding A/D Converter Performance Specifications*, AN693 — https://ww1.microchip.com/downloads/jp/AppNotes/00693A.pdf
  - Used for: ADC transfer functions, resolution/accuracy/error concepts, and converter-performance terminology.

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, A/D Converter and Electrical Specifications material — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 ADC context and device-specific reminder that real converter behavior is governed by its data sheet.

- Microchip Technology Inc., *MCP4921 — 12-Bit Single Output DAC with SPI* — https://www.microchip.com/en-us/product/mcp4921
  - Used for: official example of a real voltage-output DAC and device-specific transfer-function documentation.

### Further reading

- Microchip Technology Inc., *Key Parameters for Selecting a Digital-to-Analog Converter (DAC) IC* — https://www.microchip.com/en-us/about/media-center/blog/2024/understanding-key-parameters-for-selecting-a-dac-ic
  - Supplemental overview of DAC resolution, speed, settling, and other selection parameters.

[Back to top](#top) · [Topics index](README.md)
