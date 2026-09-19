<a id="top"></a>

# RCET 3373 — UART and Asynchronous Serial Communication

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

UART-style asynchronous serial communication is one of the most common ways embedded systems exchange simple streams of bytes.

Unlike a synchronous bus, the transmitter does not send a separate clock wire. Both endpoints agree on timing in advance, then use framing bits to mark each character.

A useful mental model is:

```text
byte
-> framed serial bits
-> wire
-> receiver samples bits
-> recovered byte
```

The PIC16F883 implements this through its EUSART peripheral.

Understanding the framing and error model transfers directly to serial consoles, USB-to-UART adapters, sensors, radio modules, development boards, and industrial equipment.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- explain asynchronous serial framing;
- identify idle, start, data, optional parity, and stop intervals conceptually;
- relate baud rate to bit time;
- identify PIC16F883 TX and RX pins and major EUSART registers;
- configure the EUSART for asynchronous transmit/receive at a selected baud rate;
- calculate a baud-rate-generator value and percent error;
- explain the roles of `TXREG`, `RCREG`, `TXIF`, and `RCIF`;
- distinguish framing error from receive overrun;
- explain why UART logic-level serial is not automatically RS-232 electrical signaling;
- verify serial traffic with a terminal, logic analyzer, or oscilloscope.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Digital Timing](digital-timing.md#period-frequency);
- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md#core-model);
- [Interrupts and Context Saving](interrupts-context-saving.md#event-flag-enable).

Related topics:

- [I2C](i2c.md#core-model) and later [SPI](spi.md#core-model) for synchronous alternatives;
- [Communication Protocol Selection](communication-protocol-selection.md#core-model) for system-level comparison.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="frame"></a>
### Asynchronous frame

A common 8-N-1 frame is:

```text
idle HIGH
   ↓
start bit = LOW
data bit 0
data bit 1
...
data bit 7
stop bit = HIGH
idle HIGH
```

"8-N-1" means:

- 8 data bits;
- no parity;
- 1 stop bit.

The exact framing must match at both ends.

<a id="baud-bit-time"></a>
### Baud rate and bit time

For ordinary binary UART signaling, one symbol carries one bit, so baud rate is numerically equal to bits per second.

```text
bit time = 1 / baud rate
```

At 9600 baud:

```text
Tbit ≈ 104.17 us
```

A complete 8-N-1 character uses 10 bit times:

```text
1 start + 8 data + 1 stop = 10 bits
```

so the maximum raw character rate is approximately:

```text
9600 / 10 = 960 characters/s
```

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="eusart-hardware"></a>
### PIC16F883 EUSART hardware

In asynchronous mode, important resources include:

- `RC6/TX/CK`: transmit pin;
- `RC7/RX/DT`: receive pin;
- `TXSTA`: transmit-mode controls;
- `RCSTA`: receive/serial-port controls and receive error bits;
- `BAUDCTL`: baud-rate-generator controls;
- `SPBRG` and optionally `SPBRGH`: baud-rate divisor;
- `TXREG`: transmit data register;
- `RCREG`: receive data register;
- `PIR1.TXIF`: transmit-buffer status flag;
- `PIR1.RCIF`: receive-data flag;
- `PIE1.TXIE` and `RCIE`: interrupt enables.

**Official visual reference:** In Section 12.0, **Enhanced Universal Synchronous Asynchronous Receiver Transmitter (EUSART)**, of the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf), inspect the asynchronous transmitter and receiver block diagrams.

Focus on the separation between the software-visible data registers and the shift registers that actually serialize/deserialize bits.

<a id="baud-generator"></a>
### Baud-rate generator

The EUSART baud-rate generator divides `FOSC`.

The exact equation depends on:

- asynchronous/synchronous mode;
- `BRGH`;
- `BRG16`;
- the `SPBRG/SPBRGH` value.

For one common asynchronous configuration:

```text
BRGH = 1
BRG16 = 0

baud = FOSC / [16(SPBRG + 1)]
```

Use the data-sheet baud-rate table for the exact mode.

<a id="transmit"></a>
### Transmit path

Conceptually:

```text
software writes TXREG
-> hardware moves byte into transmit shift register when ready
-> start/data/stop bits shift onto TX pin
```

`TXIF` indicates the state of the transmit buffer path.

Do not confuse "TXREG can accept another byte" with "the final stop bit of the previous byte has already left the pin."

The `TRMT` status bit is useful when the state of the transmit shift register itself matters.

<a id="receive"></a>
### Receive path

Conceptually:

```text
RX pin
-> receiver detects start bit
-> samples data bits
-> receive shift register
-> completed character transferred to RCREG
-> RCIF indicates unread receive data
```

Read `RCREG` to consume received data according to the documented buffering behavior.

<a id="errors"></a>
### Framing and overrun errors

**Framing error (`FERR`)**

The receiver did not observe the expected stop-bit condition for the character.

Possible causes include:

- baud mismatch;
- noise;
- wrong electrical signaling;
- wrong framing;
- corrupted line.

**Overrun error (`OERR`)**

Receive data arrived faster than software drained the receive buffering.

On this device, receive operation must be recovered according to the EUSART procedure documented for `CREN`.

The two errors mean different things:

```text
FERR -> this character's framing was bad
OERR -> software failed to keep up with received characters
```

<a id="logic-level-vs-rs232"></a>
### UART protocol is not the same thing as RS-232 voltage levels

The PIC TX/RX pins use MCU logic-level voltages.

Traditional RS-232 uses a different physical-layer voltage/signaling convention.

If communicating with a true RS-232 port, a suitable transceiver is required.

A USB-to-UART adapter normally provides USB on the computer side and logic-level UART on the target side, but its I/O voltage must still be compatible with the MCU.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="9600-baud-example"></a>
### Worked example: 9600 baud at 4 MHz

Use the common asynchronous mode:

```text
BRGH = 1
BRG16 = 0
baud = FOSC / [16(SPBRG + 1)]
```

Solve for `SPBRG`:

```text
SPBRG ≈ FOSC/(16×baud) - 1
       ≈ 4,000,000/(16×9600) - 1
       ≈ 25.04
```

Choose integer:

```text
SPBRG = 25
```

Actual baud:

```text
4,000,000 / [16(25+1)]
= 9615.38 baud
```

Percent error relative to 9600:

```text
(9615.38 - 9600) / 9600 × 100
≈ +0.16%
```

This is an example calculation. Always verify the selected mode and acceptable error against the actual data-sheet table and the other endpoint.

<a id="throughput-example"></a>
### Worked example: 8-N-1 character timing

At 9600 baud:

```text
Tbit ≈ 104.17 us
```

Ten bits per 8-N-1 character:

```text
Tchar ≈ 10 × 104.17 us
      ≈ 1.0417 ms
```

Maximum continuous raw character rate:

```text
≈ 960 characters/s
```

Protocol delimiters, parsing, pauses, or application overhead can reduce useful payload throughput further.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="uart-checklist"></a>
### Bring-up checklist

For a new UART/EUSART connection:

1. identify TX and RX pins;
2. confirm logic-voltage compatibility;
3. cross TX from one device to RX on the other;
4. share a reference/ground when required;
5. confirm both sides use the same baud/framing;
6. verify asynchronous mode;
7. calculate/configure the baud generator;
8. enable transmitter/receiver as required;
9. send a known repeating byte/string;
10. verify on a terminal or logic analyzer;
11. add receive handling;
12. deliberately check error/status conditions.

<a id="measurement"></a>
### Verify the wire, not only the terminal window

For a known repeated character:

- measure bit time;
- identify start/stop polarity;
- decode the data bits;
- compare measured baud with the configuration;
- verify idle state.

A logic analyzer can decode UART framing, but you should still be able to recognize the frame structure manually.

<a id="uart-troubleshooting"></a>
### Troubleshooting order

If no useful text appears:

1. verify electrical levels and ground;
2. verify TX/RX are crossed correctly;
3. verify the pins are configured for the serial peripheral;
4. verify `SPEN`, transmit/receive enables, and asynchronous mode;
5. verify baud-rate-generator settings;
6. measure actual bit time;
7. check `FERR` and `OERR`;
8. verify the receive buffer is drained;
9. verify software is not assuming ASCII when the peer sends binary data;
10. only then investigate higher-level parsing/protocol issues.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. What does 8-N-1 mean?
2. What is the bit time at 19200 baud?
3. How many bit times are used by one 8-N-1 character?
4. Why can UART operate without a separate clock wire?
5. What does `TXREG` do?
6. What does `RCREG` do?
7. Distinguish `FERR` from `OERR`.
8. At 4 MHz, `BRGH=1`, `BRG16=0`, what `SPBRG` value is a good starting choice for 9600 baud?
9. Why is a terminal displaying garbage often a reason to measure bit time?
10. Why can you not connect a PIC logic-level UART directly to an arbitrary traditional RS-232 voltage interface?
11. Which interrupt gates apply to EUSART RX interrupt service?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. Eight data bits, no parity, one stop bit.
2. `1/19200 ≈ 52.08 us`.
3. Ten: one start, eight data, one stop.
4. Both endpoints agree on baud rate and use the start bit plus local timing to sample the character.
5. It is the software-visible transmit data register feeding the transmit path.
6. It provides received character data to software.
7. `FERR` indicates bad stop-bit framing for a received character; `OERR` indicates receive buffering overran because software did not keep up.
8. `SPBRG = 25`, giving about 9615 baud in that mode.
9. Garbage commonly results from baud/framing mismatch; actual bit time distinguishes configuration assumptions from wire behavior.
10. RS-232 uses a different electrical physical layer; a compatible transceiver is needed.
11. `RCIE`, `PEIE`, and `GIE`, plus correct `RCIF`/receive handling.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- sketch an 8-N-1 frame;
- convert baud rate to bit time;
- calculate a PIC16F883 baud-generator setting for a chosen mode;
- trace transmit and receive data paths;
- distinguish buffer-ready flags from shift-register completion;
- distinguish framing and overrun errors;
- explain logic-level UART versus RS-232;
- outline a measurement-driven UART bring-up process.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Section 12.0 "Enhanced Universal Synchronous Asynchronous Receiver Transmitter (EUSART)" — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 EUSART registers, transmit/receive paths, baud-rate formulas, errors, pins, flags, and interrupts.

- Microchip Developer Help, *EUSART/AUSART: Enhanced/Addressable Universal Asynchronous Receiver Transmitter* — https://developerhelp.microchip.com/xwiki/bin/view/products/mcu-mpu/8bit-pic/peripherals/eusart-ausart/
  - Used for: baud-rate-generator operation and current explanatory material.

- Microchip Technology Inc., *Asynchronous Communications with the PICmicro USART*, AN774 — https://www.microchip.com/en-us/application-notes/an774
  - Used for: asynchronous serial framing, baud-rate, transmit/receive, and error-handling context.

[Back to top](#top) · [Topics index](README.md)
