<a id="top"></a>

# RCET 3373 — SPI Communication

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

SPI is a synchronous serial interface widely used between microcontrollers and nearby peripherals such as:

- sensors;
- ADCs and DACs;
- displays;
- Flash memory;
- radio modules;
- shift registers.

Unlike UART, SPI normally includes a clock signal.

Unlike I2C, ordinary SPI does not define a shared device-addressing transaction on the data lines. A controller typically selects the intended peripheral with a dedicated chip-select signal.

SPI is useful because it is simple, fast, and exposes the relationship among clock phase, data direction, and device selection very clearly.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- identify SCK, SDO/MOSI, SDI/MISO, and SS/CS roles;
- explain synchronous full-duplex shifting;
- distinguish controller/master and peripheral/slave roles;
- explain why chip-select signals are needed;
- interpret clock polarity and clock phase conceptually;
- distinguish the four common SPI modes;
- identify the PIC16F883 MSSP resources used in SPI mode;
- explain why a transmit operation also receives bits;
- estimate transfer time from clock rate and number of bits;
- verify SPI traffic with a logic analyzer or oscilloscope.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Digital Timing](digital-timing.md#timing-diagrams);
- [Address, Data, and Control Buses](buses-adc-dac.md#core-model);
- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md#core-model).

Related topics:

- [I2C](i2c.md#core-model);
- [UART](uart-asynchronous-serial.md#core-model);
- [CAN](can-bus.md#core-model);
- [Communication Protocol Selection](communication-protocol-selection.md#core-model).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="spi-signals"></a>
### Typical SPI signals

| Generic role | Common label | PIC16F883 MSSP label |
| --- | --- | --- |
| Serial clock | SCK | `RC3/SCK/SCL` |
| Controller-to-peripheral data | MOSI / SDO | `RC5/SDO` |
| Peripheral-to-controller data | MISO / SDI | `RC4/SDI/SDA` |
| Peripheral select | CS / SS | `RA5/SS/AN4` in slave use |

Different vendors use different naming conventions. Read the target device's pin names rather than assuming MOSI/MISO terminology is universal.

<a id="shift-model"></a>
### SPI is a shift-register exchange

A useful model is two shift registers connected in a loop:

```text
controller shift register ----> peripheral shift register
          ^                              |
          |                              v
          +------------------------------+
```

Each clock edge moves bits.

During an eight-clock transfer:

- the controller sends eight bits;
- the peripheral can simultaneously send eight bits back.

That is why a “write” operation still produces received bits, even if the application ignores them.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="chip-select"></a>
### Chip select identifies the intended peripheral

A controller may share SCK and data lines with several SPI devices.

A separate select line tells which peripheral should participate.

Conceptually:

```text
CS low/active
-> clocks occur
-> bits shift
-> CS returns inactive
```

The exact active level and transaction boundary come from the peripheral data sheet.

Some devices require CS to remain active for an entire multi-byte command. Others use it differently. Do not generalize beyond the target documentation.

<a id="clock-mode"></a>
### Clock polarity and phase

SPI devices must agree on:

- **clock polarity (CPOL):** idle clock level;
- **clock phase (CPHA):** which edge is used to change/sample data.

The four combinations are often called SPI modes 0–3.

The PIC16F883 MSSP expresses these choices through its SPI control/status bits rather than by using one universal “mode number” register.

Translate the peripheral's required timing into the PIC's documented settings.

**Official visual reference:** In Section 13.3, **SPI Mode**, of the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf), inspect Figure 13-1, the MSSP SPI block diagram, and the SPI timing diagrams.

Focus on SCK, SDO, SDI, shift-register behavior, and which clock edge changes or samples data.

<a id="mssp-spi"></a>
### PIC16F883 MSSP in SPI mode

Important MSSP resources include:

- `SSPBUF`: software-visible transmit/receive buffer;
- internal shift register;
- `SSPCON`: mode/control settings;
- `SSPSTAT`: status and timing-related settings;
- `PIR1.SSPIF`: transfer-complete interrupt flag;
- SPI pins SCK, SDI, SDO;
- SS when configured for slave behavior.

A software write to `SSPBUF` begins the hardware path that shifts bits according to the configured SPI mode.

At completion, received data is available through the documented buffer path and `SSPIF` records the event.

<a id="controller-peripheral"></a>
### Controller and peripheral roles

In controller/master mode:

- PIC generates SCK;
- PIC controls the transaction timing;
- PIC typically controls chip-select GPIO signals.

In peripheral/slave mode:

- external controller supplies SCK;
- PIC shifts data according to that clock;
- SS may participate in selection depending on configuration.

Both ends must agree on mode and word/transaction meaning.

SPI itself does not define what command byte `0x9F` or register address `0x12` means. The peripheral's protocol defines that higher layer.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="transfer-time-example"></a>
### Worked example: byte transfer time

**Known**

```text
SPI clock = 1 MHz
transfer = 8 bits
```

**Reasoning**

```text
clock period = 1 us
8 clocks × 1 us = 8 us
```

**Result**

One 8-bit transfer takes 8 us of clocking, not including chip-select setup/hold or software gaps between bytes.

<a id="command-read-example"></a>
### Worked example: command followed by read data

Suppose a sensor requires:

```text
CS active
-> send register-read command
-> send dummy byte while sensor shifts result back
-> CS inactive
```

The dummy byte is not wasted at the electrical level. Eight clock cycles are required so the sensor can shift eight result bits back.

The exact command and dummy value come from the sensor data sheet.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="spi-checklist"></a>
### SPI bring-up checklist

If communication fails:

1. verify supply/logic-level compatibility;
2. verify SCK/SDI/SDO wiring;
3. verify chip-select pin and active level;
4. verify controller/peripheral roles;
5. verify clock frequency;
6. verify polarity/phase;
7. verify bit order if the device specifies one;
8. verify command framing and CS boundaries;
9. inspect `SSPIF` and buffer/status behavior;
10. verify the peripheral is powered/reset/ready;
11. compare raw waveform with the peripheral timing diagram.

<a id="spi-measurement"></a>
### Measure all relevant signals together

For SPI troubleshooting, capture:

- CS;
- SCK;
- controller-to-peripheral data;
- peripheral-to-controller data.

A decoded byte stream is useful, but the raw timing relationship answers questions such as:

- Was CS active before the first clock?
- Is the idle clock level correct?
- Is data stable at the sampling edge?
- Does the peripheral ever drive the return-data line?

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. What are the jobs of SCK, SDO/MOSI, SDI/MISO, and CS/SS?
2. Why can SPI send and receive at the same time?
3. At 2 MHz, how long does an 8-bit transfer take?
4. What two clock properties define the common SPI modes?
5. Why might two devices both say “SPI” yet fail to communicate when first connected?
6. Does SPI itself define a standard device address like I2C?
7. Why might software send a dummy byte while reading from a peripheral?
8. Which PIC16F883 peripheral implements SPI hardware?
9. Why should CS be included in a logic-analyzer capture?
10. What does `SSPIF` tell software?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. Clock, controller-to-peripheral data, peripheral-to-controller data, and device/transaction selection.
2. Both devices contain shift paths; each clock moves a bit in both directions.
3. `8 / 2 MHz = 4 us`.
4. Clock polarity and clock phase.
5. They may disagree on clock mode, frequency, chip-select behavior, bit order, voltage levels, or higher-level command format.
6. No. Device selection is normally provided by chip-select lines and application wiring.
7. Clock pulses are still required to shift the peripheral's return bits to the controller.
8. MSSP.
9. CS defines which device/transaction is active and often frames the command.
10. A documented MSSP transfer event/completion has occurred and requires the appropriate software handling.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- identify the four common SPI signal roles;
- explain the shift-register model;
- explain chip selection;
- explain clock polarity/phase;
- relate a peripheral timing diagram to PIC MSSP settings;
- calculate transfer time;
- explain why read operations still require transmitted clocked bits;
- troubleshoot SPI with a four-channel waveform.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Section 13.3 "SPI Mode", including Figure 13-1 and associated timing diagrams — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 MSSP SPI signals, modes, registers, shift behavior, and timing.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A, SSP/MSSP serial-peripheral material — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: broader synchronous-serial architecture context.

[Back to top](#top) · [Topics index](README.md)
