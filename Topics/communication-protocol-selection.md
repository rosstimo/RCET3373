<a id="top"></a>

# RCET 3373 — Embedded Communication Protocol Selection

*Self-learning guide*

[Topics index](README.md)

<a id="contents"></a>
## Contents

- [1. Why this matters](#why-this-matters)
- [2. What you should be able to do](#learning-outcomes)
- [3. Prerequisites and related topics](#prerequisites)
- [4. Core model and vocabulary](#core-model)
- [5. How the protocols differ](#how-it-works)
- [6. Worked examples](#worked-examples)
- [7. Apply, verify, and troubleshoot](#apply-verify-troubleshoot)
- [8. Practice](#practice)
- [9. Answer key](#answer-key)
- [10. What you should be able to explain without notes](#retrieval-check)
- [11. References](#references)

[Back to top](#top) · [Topics index](README.md)

<a id="why-this-matters"></a>
## 1. Why this matters

UART, I2C, SPI, and CAN can all move digital information, but they solve different system problems.

Choosing a protocol by familiarity alone can create unnecessary wiring, poor fault tolerance, awkward addressing, insufficient throughput, or needless complexity.

A better process begins with the system requirements:

- How many devices?
- How far apart?
- On one PCB or throughout a machine?
- Point-to-point or shared bus?
- Is a clock wire acceptable?
- How much data and how often?
- Is deterministic message priority useful?
- How electrically noisy is the environment?
- How much hardware/software complexity is justified?

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- compare UART, I2C, SPI, and CAN by topology and physical interface;
- distinguish synchronous and asynchronous communication;
- distinguish device selection/addressing approaches;
- compare likely wiring cost and node count;
- identify where arbitration and error handling occur;
- explain why maximum bit rate alone is not enough to choose a protocol;
- select a reasonable protocol for a stated embedded-system scenario and defend the choice;
- identify when a bridge/gateway between protocols is appropriate.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Study the detailed guides first when needed:

- [UART and Asynchronous Serial](uart-asynchronous-serial.md#core-model)
- [I2C](i2c.md#core-model)
- [SPI](spi.md#core-model)
- [CAN Bus](can-bus.md#core-model)

Also review:

- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md#core-model)
- [Digital Timing](digital-timing.md#core-model)

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

Protocol selection is a system-design tradeoff.

Use categories rather than one-dimensional rankings.

| Question | UART | I2C | SPI | CAN |
| --- | --- | --- | --- | --- |
| Clock wire | No | Yes | Yes | No shared clock wire |
| Typical topology | Point-to-point | Shared local bus | Controller + selected peripherals | Multi-node shared network |
| Typical signal count | TX, RX (+ ground) | SDA, SCL (+ pull-ups) | SCK + two data + one/more CS | Differential CAN_H/CAN_L + transceivers |
| Device selection | Connection / higher layer | Bus address | Chip-select | Message identifiers / filtering |
| Simultaneous TX/RX | possible full duplex | normally half-duplex shared data | full duplex shift | shared half-duplex bus |
| Arbitration | not inherent | multi-controller arbitration exists | normally controller determines access | hardware identifier arbitration |
| Typical physical scope | local/point-to-point | board/local wiring | board/local high-speed peripheral | distributed embedded network |
| Built-in protocol error/fault handling | limited | ACK/NACK, bus rules | minimal | strong hardware error handling/fault confinement |

These are useful defaults, not universal electrical limits. Actual distance/rate depend on devices, wiring, signaling standards, transceivers, and environment.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How the protocols differ

<a id="uart-fit"></a>
### UART: simple point-to-point stream

Choose UART when:

- two endpoints need a simple byte stream;
- low pin count matters;
- a separate clock wire is undesirable;
- serial console/debug access is useful;
- higher-level framing can be handled in software if needed.

UART does not inherently solve multi-node arbitration or robust distributed-network fault handling.

<a id="i2c-fit"></a>
### I2C: many local peripherals on two shared wires

Choose I2C when:

- several nearby peripherals can share a local bus;
- moderate speed is sufficient;
- addresses are useful;
- two signal wires are attractive;
- open-drain/pull-up behavior is acceptable.

Watch bus capacitance, pull-up design, address conflicts, and device-specific transaction rules.

<a id="spi-fit"></a>
### SPI: simple synchronous peripheral throughput

Choose SPI when:

- devices are physically close;
- higher throughput/low protocol overhead is useful;
- extra signal wires are acceptable;
- the controller can dedicate chip-selects or use external decoding.

SPI is usually straightforward electrically and logically, but chip-select wiring scales poorly as the number of peripherals increases.

<a id="can-fit"></a>
### CAN: distributed multi-node embedded network

Choose CAN when:

- nodes are distributed around a machine;
- robust differential signaling is valuable;
- shared-bus arbitration matters;
- message priority matters;
- hardware error detection/fault confinement is useful;
- the cost of CAN controllers/transceivers is justified.

CAN is more infrastructure than needed for many one-board peripheral connections.

<a id="layers"></a>
### Protocol name is not the whole physical interface

Always separate:

```text
application message
-> protocol/controller
-> physical interface/transceiver
-> wire/bus
```

Examples:

- UART protocol can run through MCU logic-level pins, RS-232 transceivers, RS-485 transceivers, radio links, or USB-UART bridges;
- CAN protocol requires a CAN physical interface/transceiver for the normal differential bus;
- SPI/I2C electrical behavior depends on device I/O specifications and board wiring.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="sensor-board-example"></a>
### Worked example: six low-speed sensors on one PCB

**Requirement**

- six addressable sensors;
- all within a few centimeters;
- moderate data rate;
- minimize signal count.

**Reasoning**

I2C is a natural first candidate because many devices can share SDA/SCL and use addresses.

Before choosing it, verify:

- unique/compatible addresses;
- bus capacitance;
- pull-ups;
- required data rate.

<a id="display-example"></a>
### Worked example: fast local display

**Requirement**

- one MCU and one nearby display;
- significant repetitive data;
- board-local connection;
- several GPIO pins available.

SPI is a strong candidate because synchronous clocking and low protocol overhead can provide efficient local transfer.

<a id="robot-example"></a>
### Worked example: distributed robot controllers

**Requirement**

- several controllers around a mobile robot;
- motors create electrical noise;
- nodes need peer-to-peer message exchange;
- some messages are more time-critical than others.

CAN is a strong candidate because the system benefits from:

- differential shared wiring;
- multi-node arbitration;
- identifier-based priority;
- hardware error handling.

<a id="debug-example"></a>
### Worked example: development console

A PC terminal needs to display diagnostic text from one MCU.

UART through a USB-to-UART bridge is usually simpler than building a CAN or I2C debug console for this one task.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="selection-checklist"></a>
### Selection checklist

For any new interface, write down:

1. number of nodes;
2. physical distance/topology;
3. data rate and payload pattern;
4. latency requirements;
5. message priority requirements;
6. electrical-noise environment;
7. available pins;
8. need for addressing;
9. need for arbitration;
10. need for error/fault handling;
11. existing hardware support;
12. transceiver/pull-up/chip-select hardware;
13. software/driver complexity;
14. test equipment and debug visibility.

Then choose the simplest protocol that satisfies the real requirements with acceptable margin.

<a id="gateway"></a>
### Sometimes the right answer is more than one protocol

A system may use:

```text
SPI sensor -> local MCU -> CAN network
```

or:

```text
I2C sensors -> controller -> UART debug console
```

A gateway translates between interface domains.

There is no requirement that one protocol serve every internal and external connection in a robot.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

For each scenario, select a reasonable first-choice protocol and explain the tradeoff.

1. One MCU sends debug text to a PC through a USB adapter.
2. Four addressable environmental sensors share one small board.
3. One MCU drives a nearby high-update-rate display with several spare GPIO pins.
4. Six motor/sensor controllers are distributed around a noisy robot.
5. Two devices need full-duplex local synchronous byte exchange at high rate.
6. A design needs many local peripherals but has very few free pins.
7. Why is “choose the protocol with the highest maximum data rate” a poor general rule?
8. Why might a robot legitimately contain UART, I2C, SPI, and CAN at the same time?
9. Which of these protocols inherently provides identifier-based nondestructive arbitration?
10. Which commonly needs a separate chip-select signal per peripheral?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. UART is a natural simple point-to-point console choice.
2. I2C is a natural candidate if addresses and electrical/timing limits fit.
3. SPI is a strong candidate for fast local peripheral traffic.
4. CAN is a strong candidate for distributed robust multi-node networking.
5. SPI.
6. I2C is a strong candidate because many addressed devices can share two signals.
7. Topology, wiring, robustness, addressing, hardware support, latency, software complexity, and environment can matter more than headline rate.
8. Each protocol can serve the connection type it fits best.
9. CAN.
10. SPI in the common controller/peripheral arrangement.

Other answers can be valid if the requirement analysis is technically sound.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- compare UART, I2C, SPI, and CAN without declaring one universally best;
- select based on topology, wiring, rate, noise, and robustness;
- distinguish addressing/chip select/message identifiers;
- explain why physical interface and protocol are separate layers;
- defend a protocol choice from system requirements;
- recognize where protocol gateways make sense.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

Detailed technical authority is maintained in the protocol-specific guides and their official references:

- [UART References](uart-asynchronous-serial.md#references)
- [I2C References](i2c.md#references)
- [SPI References](spi.md#references)
- [CAN References](can-bus.md#references)

For the PIC16F883-specific UART/I2C/SPI implementations, use the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf).

For CAN fundamentals, use Microchip Developer Help, [CAN Protocol Fundamentals](https://developerhelp.microchip.com/xwiki/bin/view/applications/can/overview/fundamentals/).

[Back to top](#top) · [Topics index](README.md)
