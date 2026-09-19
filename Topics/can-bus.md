<a id="top"></a>

# RCET 3373 — CAN Bus Foundations

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

Robots often contain several embedded controllers distributed around one machine:

- motor controllers;
- sensor nodes;
- battery/power controllers;
- actuator modules;
- supervisory computers.

A point-to-point UART or a board-local SPI/I2C bus is not always the best way to connect all of them.

Controller Area Network (CAN) was designed for reliable multi-node communication in electrically noisy environments. It provides:

- a shared differential bus;
- message identifiers rather than fixed point-to-point addressing;
- hardware arbitration when several nodes want to transmit;
- strong error detection and fault-handling mechanisms;
- widespread use in vehicles, industrial systems, and embedded networks.

For robotics, the useful question is:

> When should several embedded controllers become nodes on a network rather than peripherals hanging from one central controller?

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- distinguish a CAN controller from a CAN transceiver;
- explain the CAN_H/CAN_L differential physical layer;
- explain why a CAN bus is normally terminated at both physical ends;
- distinguish dominant and recessive bus states conceptually;
- explain identifier-based nondestructive arbitration;
- explain why a lower numerical identifier wins arbitration against a higher one when frames begin together;
- identify the major fields of a classical CAN data frame;
- distinguish classical CAN from CAN FD at a high level;
- explain error detection/fault confinement conceptually;
- identify why CAN can fit distributed robotics better than UART, I2C, or SPI in some systems.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Logic Levels, Noise Margin, and Loading](logic-levels-interfaces.md#core-model);
- [Digital Timing](digital-timing.md#core-model);
- [Address, Data, and Control Buses](buses-adc-dac.md#core-model).

Related topics:

- [UART](uart-asynchronous-serial.md#core-model);
- [I2C](i2c.md#core-model);
- [SPI](spi.md#core-model);
- [Communication Protocol Selection](communication-protocol-selection.md#core-model).

The PIC16F883 used for the introductory course work does **not** contain a CAN controller. CAN implementation therefore belongs to later alternate-MCU/controller exploration rather than pretending the current course PIC has hardware it does not provide.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="can-node"></a>
### A CAN node has several layers

A typical CAN node contains:

```text
application software
      ↓
MCU / application processor
      ↓
CAN controller
      ↓
CAN transceiver
      ↓
CAN_H / CAN_L bus
```

The **CAN controller** handles protocol functions such as:

- frame construction/parsing;
- arbitration;
- bit timing;
- CRC/error logic;
- transmit/receive buffering.

The **CAN transceiver** converts controller logic-level TX/RX signals into the differential electrical signaling used on CAN_H/CAN_L.

A microcontroller with an internal CAN peripheral still needs a suitable transceiver to connect to the physical bus.

<a id="bus-topology"></a>
### Physical bus topology

A normal high-speed CAN network is a linear bus with short stubs to nodes.

Conceptually:

```text
120 Ω                                       120 Ω
  |                                           |
  +---- node ---- node ---- node ---- node ---+
       CAN_H / CAN_L twisted differential pair
```

Termination is placed at the two physical ends of the main bus.

Adding arbitrary termination at every node would load the bus incorrectly.

**Official visual reference:** Microchip Developer Help, [CAN Protocol Fundamentals](https://developerhelp.microchip.com/xwiki/bin/view/applications/can/overview/fundamentals/), includes CAN node, bus topology, transceiver, waveform, and frame visuals.

Use those figures to distinguish the MCU/controller side from the differential CAN_H/CAN_L physical layer.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="dominant-recessive"></a>
### Dominant and recessive states

CAN uses bus states commonly described as:

- **dominant**;
- **recessive**.

Conceptually, dominant wins when one transmitter asserts dominant while another releases/sends recessive.

This electrical/logical property makes nondestructive arbitration possible.

<a id="identifier-arbitration"></a>
### Identifiers describe messages and determine arbitration priority

CAN frames carry an identifier.

The identifier is not simply “the address of the receiving node.”

Several nodes can choose to consume the same message identifier, and one node can transmit several message types.

If two nodes begin transmitting together, they monitor the bus while sending the arbitration field.

A transmitter that sends recessive but observes dominant knows that a higher-priority message is present and stops transmitting without corrupting the winning frame.

Because dominant represents the arbitration-winning state at the first differing identifier bit, lower numerical identifiers have higher arbitration priority.

Example:

```text
ID 0x120
ID 0x220
```

If both begin together under the same frame-format conditions, `0x120` wins arbitration.

This is **priority**, not proof that one node is more important in every application.

<a id="frame"></a>
### Classical CAN data-frame concepts

A classical CAN data frame includes fields for concepts such as:

- start of frame;
- identifier/arbitration;
- control information;
- payload data;
- CRC;
- acknowledgement;
- end of frame.

Classical CAN supports payloads up to 8 bytes per data frame.

The protocol also inserts additional bits for synchronization and error-detection purposes; the actual wire time is therefore greater than payload bits alone.

<a id="error-handling"></a>
### Error detection and fault confinement

CAN includes several hardware-level mechanisms for detecting communication problems.

Conceptually, nodes monitor whether the observed bus behavior matches what is allowed/expected.

The protocol maintains error state so a persistently faulty node can eventually limit its ability to disrupt the network.

The key student takeaway is:

> Error handling is part of CAN's hardware protocol model, not an afterthought implemented only by application code.

Exact error counters/states should be studied from the controller/CAN specification when implementing a system.

<a id="can-fd"></a>
### Classical CAN versus CAN FD

At a high level:

| Feature | Classical CAN | CAN FD |
| --- | --- | --- |
| Maximum data payload | 8 bytes | 64 bytes |
| Arbitration concept | CAN arbitration | same core concept |
| Data-phase rate | classical rate | may use faster data phase |
| CRC | classical format | extended for larger/faster frames |

CAN FD keeps the recognizable CAN bus/arbitration model while allowing larger payloads and a higher-rate data phase.

Not every classical CAN controller can receive CAN FD frames.

<a id="robotics-fit"></a>
### Why CAN can fit robotics

CAN is attractive when:

- many embedded nodes share one network;
- wiring weight/complexity matters;
- electrical noise is significant;
- message priority matters;
- nodes must communicate without one central serial host;
- robust hardware error handling is valuable.

It may be unnecessary when two devices sit centimeters apart on one PCB and a simpler SPI/I2C connection is sufficient.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="arbitration-example"></a>
### Worked example: two nodes transmit together

Suppose two nodes begin standard-ID frames simultaneously:

```text
Node A ID = 0x100
Node B ID = 0x180
```

They transmit identical arbitration bits until the first bit where the identifiers differ.

At that point:

- one node transmits dominant;
- the other transmits recessive;
- the recessive transmitter observes dominant and stops;
- the dominant frame continues without being corrupted.

**Result**

The lower numerical identifier `0x100` wins arbitration.

The losing node can retry later according to normal controller behavior.

<a id="controller-transceiver-example"></a>
### Worked example: MCU has CAN peripheral but no transceiver

Suppose a microcontroller data sheet says:

```text
CAN FD controller included
```

That does not mean CAN_H/CAN_L can be wired directly to MCU GPIO pins.

The node still needs a compatible CAN transceiver between:

```text
controller TX/RX <-> transceiver <-> CAN_H/CAN_L
```

The controller implements the protocol logic; the transceiver implements the bus electrical interface.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="can-checklist"></a>
### CAN network checklist

For a new CAN network, identify:

1. controller type and supported CAN version;
2. transceiver type;
3. bus bit rate;
4. physical bus length/topology;
5. termination at both ends;
6. stub lengths;
7. identifier plan;
8. payload format;
9. message rates;
10. priority/arbitration implications;
11. grounding/common-mode constraints;
12. error counters/states;
13. measurement/test points.

<a id="can-measurement"></a>
### Measure both logic-side and bus-side behavior

Useful observations include:

- controller TX;
- controller RX;
- CAN_H;
- CAN_L;
- decoded frames from a CAN-capable analyzer.

Comparing TX with CAN_H/CAN_L helps separate:

- controller/protocol behavior;
- transceiver/physical-bus behavior.

If frames fail only when additional nodes/cable are connected, investigate termination/topology/physical layer rather than immediately rewriting application code.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. What is the difference between a CAN controller and a CAN transceiver?
2. Why are CAN_H and CAN_L used as a differential pair?
3. Where are the main termination resistors normally placed?
4. What happens if one transmitter sends recessive while another sends dominant during arbitration?
5. Which has higher arbitration priority: identifier `0x080` or `0x180`?
6. Is a CAN identifier simply the destination node address?
7. What is the maximum classical CAN data payload?
8. What is the maximum CAN FD data payload?
9. Name two reasons CAN is attractive for distributed robotics.
10. Why might SPI still be a better choice than CAN for one nearby sensor on the same PCB?
11. If an MCU includes a CAN controller, what additional hardware is normally required to connect to the bus?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. The controller implements CAN protocol logic; the transceiver converts logic-level controller signals to/from the differential physical bus.
2. Differential signaling improves rejection of common-mode noise and supports robust communication in noisy environments.
3. At the two physical ends of the main bus.
4. Dominant wins; the recessive transmitter detects loss of arbitration and stops transmitting.
5. `0x080`, because the lower numerical identifier wins at the first differing arbitration bit.
6. No. It identifies/prioritizes the message; multiple nodes can consume a given message.
7. 8 bytes.
8. 64 bytes.
9. Examples: multi-node bus, robust differential physical layer, hardware arbitration, message priority, hardware error handling, reduced point-to-point wiring.
10. SPI can be simpler/faster for a short local controller-peripheral connection where network arbitration/robust multi-node features are unnecessary.
11. A suitable CAN transceiver.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- draw MCU/controller/transceiver/bus layers;
- explain termination and bus topology;
- explain dominant/recessive states;
- explain nondestructive identifier arbitration;
- distinguish message identifier from node address;
- identify the major frame concepts;
- compare classical CAN and CAN FD;
- explain why CAN is useful in distributed robotics;
- separate protocol/controller failures from transceiver/physical-layer failures.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Developer Help, *Controller Area Network (CAN) Protocol Fundamentals* — https://developerhelp.microchip.com/xwiki/bin/view/applications/can/overview/fundamentals/
  - Used for: CAN node layers, topology, differential bus, controller/transceiver distinction, frame concepts, and waveform examples.

- Microchip Technology Inc., *Controller Area Network (CAN) Peripherals* — https://www.microchip.com/en-us/products/microcontrollers/8-bit-mcus/peripherals/communication-connectivity/can
  - Used for: CAN controller/transceiver requirements and high-level classical CAN/CAN FD comparison.

- Microchip Developer Help, *Controller Area Network Flexible Data-Rate (CAN FD)* — https://developerhelp.microchip.com/xwiki/bin/view/applications/can/overview/canfd/
  - Used for: CAN FD payload/data-rate and protocol-extension context.

- Microchip University, *CAN and CAN FD Protocol and Physical Layer Basics* — https://skills.microchip.com/can-and-can-fd-protocol-and-physical-layer-basics
  - Supplemental official training covering arbitration, bit timing, errors, topology, transceivers, and physical-layer design.

[Back to top](#top) · [Topics index](README.md)
