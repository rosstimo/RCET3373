<a id="top"></a>

# RCET 3373 — I2C Communication

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

I2C connects multiple digital devices using only two shared signal lines:

```text
SCL = serial clock
SDA = serial data
```

It is commonly used for sensors, EEPROMs, displays, real-time clocks, configuration devices, and other board-level peripherals.

I2C is especially useful for learning shared-bus behavior because the physical layer and protocol are tightly connected:

- devices share the same wires;
- the wires use pull-up resistors;
- devices normally pull a line LOW rather than actively driving it HIGH;
- addressing selects a device;
- ACK/NACK bits provide per-byte handshaking;
- START and STOP conditions define transactions.

On the PIC16F883, I2C is implemented through the MSSP peripheral.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- explain why SDA and SCL need pull-up resistors;
- explain open-drain/open-collector style shared-line behavior;
- identify START, repeated START, STOP, data bits, and ACK/NACK;
- distinguish an I2C device address from an internal register/memory address;
- trace a simple master-write and master-read transaction;
- explain the roles of controller/master and target/slave;
- identify the major PIC16F883 MSSP resources used for I2C;
- explain why bus capacitance and pull-up resistance affect rise time;
- recognize bus-collision/arbitration and clock-stretching concepts;
- verify I2C traffic with an oscilloscope or logic analyzer.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Address, Data, and Control Buses](buses-adc-dac.md#core-model);
- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md#core-model);
- [Digital Timing](digital-timing.md#timing-diagrams).

Related topics:

- [PIC16F883 Data EEPROM](pic16f883-data-eeprom.md#core-model) for internal nonvolatile storage;
- [UART](uart-asynchronous-serial.md#core-model);
- [SPI](spi.md#core-model);
- [Communication Protocol Selection](communication-protocol-selection.md#core-model).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="shared-lines"></a>
### Shared lines with pull-ups

I2C devices normally do not actively drive the bus HIGH.

Instead:

```text
no device pulls line LOW -> pull-up resistor produces HIGH
one or more devices pull line LOW -> bus is LOW
```

This behavior allows several devices to share SDA/SCL without two ordinary push-pull outputs fighting while trying to drive opposite states.

The pull-ups and total bus capacitance determine how quickly a released line rises.

<a id="transaction-elements"></a>
### Transaction elements

Important protocol elements include:

| Element | Purpose |
| --- | --- |
| START | begins a transaction while bus is idle/owned |
| Address + R/W bit | selects target and transfer direction |
| ACK | receiver acknowledges a byte |
| NACK | receiver does not acknowledge / signals end condition depending on context |
| Repeated START | begins another address phase without releasing the bus |
| STOP | releases/ends transaction |
| SCL | clocks bit timing |
| SDA | carries address/data and special START/STOP transitions |

A byte is transferred most-significant bit first.

An ACK/NACK bit follows each transferred byte.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="start-stop"></a>
### START and STOP are special SDA/SCL relationships

When SCL is HIGH:

- SDA HIGH -> LOW defines START;
- SDA LOW -> HIGH defines STOP.

Ordinary data changes are arranged so they do not accidentally look like START or STOP.

**Official visual reference:** In Section 13.4, **I2C Mode**, of the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf), inspect the I2C START/STOP and data-transfer timing diagrams.

Focus on when SDA is allowed to change relative to SCL and how the ninth clock is used for ACK/NACK.

<a id="addressing"></a>
### Device address versus internal address

A common source of confusion is that two different addresses can appear in one transaction.

Example with an external EEPROM:

```text
I2C device address -> selects the EEPROM chip
EEPROM memory address -> selects a location inside that chip
```

Do not merge those into one concept.

The device's data sheet defines its I2C address format and its internal memory/register addressing.

<a id="master-write"></a>
### Conceptual master write

A common write transaction is:

```text
START
-> target address + write
-> ACK
-> register/memory address
-> ACK
-> data byte
-> ACK
-> STOP
```

Longer writes may transfer additional data bytes according to the target device's rules.

<a id="master-read"></a>
### Conceptual register/memory read

Many devices use a combined transaction:

```text
START
-> target address + write
-> ACK
-> internal register/memory address
-> ACK
-> repeated START
-> target address + read
-> ACK
-> target sends data
-> controller NACKs final byte
-> STOP
```

The exact sequence comes from the target device documentation.

<a id="mssp"></a>
### PIC16F883 MSSP resources

The MSSP peripheral provides hardware support for I2C.

Important resources include:

- `SSPCON` / related control bits;
- `SSPSTAT`;
- `SSPBUF`;
- `SSPADD`;
- `PIR1.SSPIF` and `PIE1.SSPIE`;
- bus-collision status/interrupt resources;
- dedicated `SCL` and `SDA` pins for this device.

Use the data sheet's MSSP/I2C chapter for exact mode values and required sequences.

Do not assume MSSP register layouts from a newer PIC apply unchanged to the PIC16F883.

<a id="pullups-rise-time"></a>
### Pull-ups, capacitance, and rise time

When a device releases SDA or SCL, the pull-up resistor charges the bus capacitance.

Therefore:

```text
larger pull-up resistance -> less sink current, slower rise
larger bus capacitance -> slower rise
smaller pull-up resistance -> faster rise, more LOW-state sink current
```

The correct design must satisfy both:

- LOW-state sink capability;
- required rise time for the selected I2C speed and bus capacitance.

Use the I2C specification and device electrical characteristics rather than choosing a resistor only by habit.

<a id="clock-stretching-arbitration"></a>
### Clock stretching and arbitration

Some I2C targets can hold SCL LOW to delay the controller while they become ready. This is **clock stretching**.

In a multi-controller system, arbitration uses the wired-AND/open-drain nature of SDA so a controller can detect when the bus does not match the bit it intended to release HIGH.

These concepts are part of the reason shared-line electrical behavior matters to the protocol.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="eeprom-write-example"></a>
### Worked example: conceptual external EEPROM byte write

Suppose an EEPROM uses a 7-bit I2C target address and an internal byte address.

A conceptual write to one location is:

```text
START
target address + W
ACK
memory address
ACK
data byte
ACK
STOP
```

After STOP, many EEPROMs perform an internal nonvolatile write cycle before the new value is ready for another write operation.

That internal write-cycle behavior belongs to the EEPROM's data sheet, not the generic I2C protocol.

<a id="read-example"></a>
### Worked example: conceptual register read

A sensor register read may require:

```text
START
sensor address + W
ACK
register address
ACK
repeated START
sensor address + R
ACK
sensor sends byte
controller NACK
STOP
```

The write-direction first phase does not write sensor measurement data. It selects which internal register will be read.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="i2c-checklist"></a>
### I2C bring-up checklist

If the bus does not work:

1. verify common ground/reference;
2. verify SDA/SCL pins;
3. verify pull-up resistors are actually present;
4. measure idle HIGH voltage;
5. inspect rise time;
6. verify target address and R/W interpretation;
7. verify controller clock rate;
8. verify START/STOP generation;
9. check ACK/NACK after each byte;
10. distinguish target address from internal register/memory address;
11. inspect MSSP flags/status and collision conditions;
12. verify target-specific delays or write-cycle behavior.

<a id="logic-analyzer"></a>
### Use decoded traffic and raw waveform together

A logic analyzer can decode:

```text
START
address
R/W
ACK/NACK
data
STOP
```

But also inspect the raw waveform.

A decoded transaction can hide:

- slow rise time;
- ringing/noise;
- incorrect voltage levels;
- marginal timing;
- bus contention.

The protocol decode and electrical waveform answer different questions.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Why do I2C SDA and SCL normally need pull-up resistors?
2. What electrical condition defines START?
3. What electrical condition defines STOP?
4. What happens on the ninth clock after an 8-bit byte?
5. Why can a transaction contain both a device address and a register/memory address?
6. Why is a repeated START useful?
7. What happens to rise time if bus capacitance increases while the pull-up resistor stays the same?
8. Why can making the pull-up resistor too small also be a problem?
9. What is clock stretching?
10. A logic analyzer shows the correct address but every address receives NACK. Name several things you would check.
11. Which PIC peripheral provides I2C hardware support?
12. Why should raw SDA/SCL waveforms still be inspected even when the analyzer successfully decodes bytes?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. Devices normally pull lines LOW and release them for HIGH; the pull-ups create the HIGH state.
2. SDA transitions HIGH-to-LOW while SCL is HIGH.
3. SDA transitions LOW-to-HIGH while SCL is HIGH.
4. The receiver drives ACK/NACK for that byte.
5. The I2C address selects the device; the internal address selects a location/register within that device.
6. It begins another address/direction phase without first releasing the bus with STOP.
7. Rise time becomes longer/slower.
8. LOW-state current increases and may exceed device sink-current requirements.
9. A device holds SCL LOW to delay bus progress until it is ready.
10. Target address, power, ground, pull-ups, pin routing, target readiness, R/W bit, signal levels, rise time, controller configuration.
11. MSSP.
12. Protocol decode may still succeed while electrical levels, rise times, noise, or margins are poor.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- explain open-drain shared-line behavior;
- identify START, STOP, ACK, and NACK;
- trace a simple write and read transaction;
- distinguish target address from internal address;
- explain the pull-up/capacitance rise-time tradeoff;
- identify the PIC16F883 MSSP as the I2C hardware block;
- explain clock stretching and arbitration conceptually;
- troubleshoot with both protocol decode and raw waveform evidence.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Section 13.0 "Master Synchronous Serial Port (MSSP) Module" and I2C Mode material — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 MSSP registers, pins, I2C modes, timing, flags, and bus-collision behavior.

- Microchip Technology Inc., *Using the PICmicro MSSP Module for I2C Communications*, AN735 — https://www.microchip.com/en-us/application-notes/an735
  - Used for: I2C master transaction sequencing and MSSP conceptual operation.

- Microchip Technology Inc., *Using the MSSP Module to Interface I2C Serial EEPROMs with PIC16 Devices*, AN976 — https://www.microchip.com/en-us/application-notes/an976
  - Used for: external EEPROM transaction examples and I2C/PIC integration context.

[Back to top](#top) · [Topics index](README.md)
