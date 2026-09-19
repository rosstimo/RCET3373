<a id="top"></a>

# RCET 3373 — Address, Data, and Control Buses

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

Processors, memories, and peripherals exchange information through electrical and logical interfaces.

A useful first model separates three jobs:

```text
address = where
data    = what
control = what operation / when
```

That model helps you reason about external memory, memory-mapped peripherals, register maps, and many serial/parallel interfaces without memorizing one device at a time.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- explain the roles of address, data, and control information in a transaction;
- calculate the number of unique locations represented by an N-bit address;
- distinguish address width from data width;
- explain a conceptual read and write transaction;
- explain why memory-mapped peripheral registers behave like addresses even when they are not ordinary RAM;
- identify why chip-select/enable signals and high-impedance outputs are needed on shared buses.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Digital Representation](digital-representation.md#core-model).

Related topics:

- [Memory Systems](memory-systems.md#core-model);
- [PIC16F883 data memory](pic16f883-architecture.md#data-memory);
- [Data Memory, SFRs, and Banking](data-memory-sfrs-banking.md#core-model);
- [Digital Timing](digital-timing.md#timing-diagrams);
- [ADC and DAC Foundations](adc-dac-foundations.md#core-model).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="bus-roles"></a>
### Address, data, and control

| Role | Question answered | Typical examples |
| --- | --- | --- |
| Address | Where? | memory location, register address, device select |
| Data | What value? | instruction, variable, register contents |
| Control | What operation and when? | read, write, enable, clock/strobe |

The physical implementation varies. A parallel bus may dedicate separate wires to each role. A serial protocol may encode address, data, and control at different times on the same wires.

<a id="address-space"></a>
### Address width defines address-space size

If an address contains N independent bits:

```text
number of unique addresses = 2^N
```

The size of each location is a separate property.

For example:

```text
8 address bits -> 256 locations
16-bit data width -> 16 bits transferred per selected location
```

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="read-transaction"></a>
### Conceptual read transaction

A simplified parallel memory read is:

1. the bus master presents an address;
2. control signals request a read and select the intended device;
3. the selected device decodes the address;
4. after its access delay, the device drives valid data;
5. the bus master captures the data while it is valid;
6. the device releases the shared data bus when the transaction ends.

<a id="write-transaction"></a>
### Conceptual write transaction

A simplified write is:

1. the bus master presents the destination address;
2. the bus master drives the new data value;
3. control signals identify a write and select the destination;
4. the receiving device stores the data when timing requirements are satisfied.

<a id="shared-bus"></a>
### Shared buses require ownership

If several devices share data wires, only the intended source should drive them at one time.

Inactive devices commonly place outputs into a **high-impedance (Hi-Z)** state so another device can use the same conductors without electrical contention.

<a id="memory-mapped-io"></a>
### Memory-mapped I/O uses addresses for hardware

An address does not have to refer to ordinary storage.

In a memory-mapped system, reading or writing a particular address can interact with a hardware peripheral register.

That is the key connection to the PIC16F883 SFR map: addresses select control/status registers that affect real hardware.

**Visual reference:** See the data-memory maps in the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf).

Focus on how ordinary GPR storage and hardware-control SFRs coexist in the same data-address space.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="address-width-example"></a>
### Worked example: address-space size

**Known**

A system uses 10 address bits.

**Find**

The number of unique locations.

**Reasoning**

```text
locations = 2^10
          = 1024
```

**Result**

The address can identify 1024 unique locations.

<a id="memory-organization-example"></a>
### Worked example: 8K × 16 organization

**Known**

```text
8K locations
16 bits per location
```

**Find**

Address bits, data width, and total byte capacity.

**Reasoning**

```text
8K = 8192 = 2^13
```

Therefore 13 address bits are needed.

Each selected location transfers 16 bits, so the parallel data width is 16 bits.

```text
8192 locations × 2 bytes/location = 16384 bytes
```

**Result**

- 13 address bits;
- 16 data bits;
- 16 KiB total storage.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

When a bus transaction is not working, separate the problem into roles:

1. **Address:** Is the intended device/location actually selected?
2. **Data:** Is the correct side driving the data and are the values valid?
3. **Control:** Is the operation identified correctly?
4. **Timing:** Are address/data/control relationships valid long enough?
5. **Ownership:** Are two outputs fighting for the same shared wires?
6. **Electrical interface:** Do voltage levels and loading satisfy the connected devices?

The same checklist scales from a simple external-memory exercise to more complicated peripheral buses.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. How many locations can a 4-bit address select?
2. How many locations can a 12-bit address select?
3. A memory is organized as 2K × 16. How many address bits are needed?
4. For the same 2K × 16 memory, how many data bits are transferred per addressed word?
5. During a read, which side normally drives the data bus?
6. During a write, which side normally drives the data toward memory?
7. Why do inactive devices use high-impedance outputs on a shared bus?
8. Why can a peripheral register have an address even though it is not ordinary RAM?
9. Give one example of a control signal or control concept in a transaction.

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. `2^4 = 16` locations.
2. `2^12 = 4096` locations.
3. `2K = 2048 = 2^11`, so 11 address bits.
4. 16 data bits.
5. The selected memory/peripheral source drives the data toward the bus master.
6. The bus master drives the new data toward the selected destination.
7. Hi-Z disconnects inactive outputs so another device can drive the shared wires without contention.
8. The address is a selection mechanism. The selected destination can be hardware state/control rather than ordinary storage.
9. Examples include read/write direction, chip select, output enable, write enable, clock, or strobe.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- explain “address = where, data = what, control = what operation/when”;
- calculate address-space size from address width;
- distinguish address width from word/data width;
- walk through a read transaction;
- walk through a write transaction;
- explain why Hi-Z matters on a shared data bus;
- explain memory-mapped I/O in terms of address selection.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A, memory-organization and I/O material — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: address/data/control and memory/peripheral organization context.

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Memory Organization chapter — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: device-specific examples of address maps, SFRs, GPRs, and memory-mapped hardware registers.

- Microchip Technology Inc., *Using SRAM with a PIC16CXX*, TB011 — https://www.microchip.com/en-us/application-notes/tb011
  - Used for: supplemental example of external address/data/control behavior using PIC I/O.

[Back to top](#top) · [Topics index](README.md)
