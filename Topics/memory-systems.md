<a id="top"></a>

# RCET 3373 — Memory Systems

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

Memory is not one technology.

Computer and embedded systems use different kinds of storage because the design goals conflict:

- fast access;
- low cost per bit;
- high density;
- nonvolatile retention;
- frequent rewriting;
- simple interfaces;
- low power.

The useful skill is not memorizing a list of memory names. It is being able to reason from:

1. how locations are addressed;
2. how data moves during reads and writes;
3. how the storage cell preserves a bit;
4. what timing/control requirements follow from that mechanism;
5. what tradeoffs make one memory type useful for a particular job.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- interpret organization notation such as `4K × 8`;
- determine address bits, data width, and total capacity;
- explain conceptual memory read and write transactions;
- explain why shared data buses use high-impedance outputs;
- compare SRAM and DRAM from their storage mechanisms;
- explain why DRAM requires refresh and commonly uses row/column addressing;
- distinguish MROM, PROM, EPROM, EEPROM, and Flash by how contents are changed;
- distinguish expansion for greater word width from expansion for more locations;
- connect general memory concepts to the PIC16F883 memory map.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Address, Data, and Control Buses](buses-adc-dac.md#core-model);
- [Digital Timing](digital-timing.md#core-model).

Related topics:

- [PIC16F883 Architecture](pic16f883-architecture.md#data-memory);
- [Data Memory, SFRs, and Banking](data-memory-sfrs-banking.md#core-model);
- later persistent-storage work with EEPROM.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="organization"></a>
### Memory organization

Memory is often described as:

```text
number of locations × bits per location
```

Example:

```text
4K × 8
```

means:

- 4096 addressable locations;
- 8 bits stored at each location;
- 12 address bits because `4096 = 2^12`;
- 8 data bits for a parallel word transfer.

<a id="transactions"></a>
### Read and write transactions

A conceptual read:

```text
address -> select/read controls -> memory drives data -> receiver captures data
```

A conceptual write:

```text
address + new data -> select/write controls -> memory stores data
```

<a id="volatile-nonvolatile"></a>
### Volatile and nonvolatile

**Volatile** memory loses its stored state when operating power is removed.

**Nonvolatile** memory retains information without normal operating power.

These terms describe retention, not speed.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="parallel-read-write"></a>
### Parallel memory interface

A basic parallel memory device uses some combination of:

- address pins;
- data pins;
- chip select/enable;
- output enable/read controls;
- write enable;
- timing requirements.

When the device is not selected, its data outputs commonly enter **high impedance (Hi-Z)** so several devices can share a bus.

<a id="nonvolatile-family"></a>
### Nonvolatile memory family

| Type | How contents are created/changed | Practical idea |
| --- | --- | --- |
| Mask ROM | Programmed during manufacturing | Fixed production content |
| PROM | User-programmed once | One-time programmable |
| EPROM | Electrically programmed, UV erased | Whole-device erase |
| EEPROM | Electrically rewritten in circuit | Fine-grained persistent updates |
| Flash | Electrically rewritten | Erase/program organized in larger units |

The exact erase/program granularity and endurance are device-specific.

<a id="sram"></a>
### SRAM: bistable storage

Static RAM stores a bit using a bistable/latch-like transistor structure.

As long as power remains valid, the circuit reinforces its state without periodic refresh.

That is why it is **static**.

SRAM is still normally **volatile**.

<a id="sram-timing"></a>
### SRAM timing still matters

A valid address does not instantly create valid output data.

Real SRAM specifications include quantities such as:

- address access time;
- output-enable delay;
- read/write cycle time;
- write-pulse width;
- data setup/hold;
- high-impedance transition timing.

**Official visual reference:** Infineon's [CY7C1338G SRAM data sheet](https://www.infineon.com/assets/row/public/documents/10/49/infineon-cy7c1338g-4-mbit-128-k-32-flow-through-sync-sram-datasheet-en.pdf), Figure 5, **Read/Write Timing**, is a useful example of how a manufacturer relates address, control, data, and Hi-Z behavior.

Focus on the timing arrows between address/control events and valid data, not on memorizing that particular SRAM's numeric values.

<a id="dram"></a>
### DRAM: capacitor-based storage

Dynamic RAM commonly stores each bit using a small capacitor plus an access transistor.

The capacitor charge leaks with time, so stored information must be periodically restored.

That maintenance operation is **refresh**.

DRAM achieves very high density because the storage cell is compact, but the system pays for that density with refresh and more complicated access/control behavior.

<a id="dram-row-column"></a>
### Rows, columns, and address multiplexing

DRAM arrays are organized by rows and columns.

Historically and in modern forms, row/column organization helps large memories select storage efficiently. Many DRAM interfaces reuse address/command resources across phases rather than dedicating a separate package pin to every internal address dimension.

Conceptually:

```text
select/open row -> select column -> transfer data
```

Modern DRAM protocols are much more elaborate than early RAS/CAS examples, but the row/column mental model remains useful.

<a id="sram-dram-comparison"></a>
### SRAM and DRAM comparison

| Property | SRAM | DRAM |
| --- | --- | --- |
| Basic storage idea | Bistable/latch-like state | Capacitor charge |
| Refresh | No periodic refresh | Required |
| Density | Lower | Higher |
| Cost per bit | Higher | Lower |
| Control complexity | Lower | Higher |
| Common system role | Cache, small fast buffers | Large main memory |

<a id="memory-expansion"></a>
### Memory expansion

There are two different expansion problems.

**Increase word width**

If you need `16 × 8` but only have `16 × 4` devices, use two devices in parallel:

- share address/control;
- one device supplies part of the data word;
- the other supplies the remaining bits.

**Increase number of locations**

If you need `32 × 8` but only have `16 × 8` devices:

- share the low-order address lines;
- use an additional high-order address bit or decoder to select the active bank.

Ask first:

> Do I need more bits per word, or more addressable words?

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="32k8-example"></a>
### Worked example: 32K × 8

**Known**

```text
32K × 8
```

**Find**

Address bits, data bits, and capacity.

**Reasoning**

```text
32K = 32768 = 2^15
```

Therefore 15 address bits are required.

Each location contains 8 bits.

```text
32768 × 8 bits = 32768 bytes = 32 KiB
```

**Result**

- 15 address bits;
- 8 data bits;
- 32 KiB capacity.

<a id="8k16-example"></a>
### Worked example: 8K × 16

```text
8K = 8192 = 2^13
```

Therefore 13 address bits are required.

Each word is 16 bits = 2 bytes.

```text
8192 × 2 bytes = 16384 bytes = 16 KiB
```

<a id="expansion-example"></a>
### Worked example: build 1K × 8 from 1K × 4 parts

Two devices are required.

Both receive the same address and control signals.

One device connects to four data bits; the other connects to the other four.

The number of locations remains 1K. The word width doubles from four to eight bits.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="memory-datasheet-checklist"></a>
### Memory data-sheet checklist

When approaching an unfamiliar memory device, identify:

1. organization: locations × word width;
2. address pins or command/address mechanism;
3. data width;
4. read controls;
5. write controls;
6. chip-select/enable behavior;
7. Hi-Z behavior;
8. read/write timing;
9. power and retention requirements;
10. nonvolatile erase/program limits when applicable.

<a id="pic-connection"></a>
### What this means for the PIC16F883

The PIC16F883 contains several logically different memory regions.

Use the [PIC16F883 Architecture guide](pic16f883-architecture.md#memory-spaces) to distinguish:

- program memory;
- data memory;
- special-function registers;
- general-purpose RAM;
- nonvolatile data EEPROM.

The general memory questions still apply:

- What addresses exist?
- What is stored there?
- What mechanism reads/writes it?
- Is it volatile?
- Does access have special timing or sequencing requirements?

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. A memory is organized as `64K × 8`. How many address bits and data bits are required? What is the total capacity in bytes?
2. A memory is organized as `2K × 16`. How many address bits are required? What is the capacity in bytes?
3. During a basic memory read, which device drives the data bus?
4. Why do memory outputs use a high-impedance state?
5. Why is SRAM called static even though it is volatile?
6. What physical storage idea makes DRAM refresh necessary?
7. Why do DRAM systems use row/column organization?
8. What practical distinction separates EEPROM from Flash in the introductory model?
9. How many `1K × 4` devices are required to build `1K × 8`?
10. How many `1K × 8` devices are required to build `4K × 8`?
11. In question 10, what additional function is needed so only one bank responds?
12. A memory has 256 rows and 256 columns of one-bit cells. How many cells exist?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. `64K = 2^16`: 16 address bits, 8 data bits, 65,536 bytes = 64 KiB.
2. `2K = 2^11`: 11 address bits. With 16-bit words, total capacity is 4096 bytes = 4 KiB.
3. The selected memory device drives the data during a read.
4. Hi-Z disconnects inactive outputs so devices can share the bus without contention.
5. Its cell maintains state without periodic refresh while power is present; loss of power still destroys the stored state.
6. Charge stored on a small capacitor leaks with time.
7. It efficiently organizes very large arrays and supports staged selection of storage.
8. EEPROM is commonly used for finer-grained electrical updates; Flash commonly erases/programs larger blocks or sectors. Exact behavior is device-specific.
9. Two devices in parallel.
10. Four devices/banks.
11. High-order address decoding/chip selection.
12. `256 × 256 = 65,536` one-bit cells.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- interpret `locations × word width` notation;
- calculate address bits and total capacity;
- describe a complete read and write transaction;
- explain why Hi-Z matters;
- explain why SRAM is static but volatile;
- explain why DRAM refresh exists;
- compare SRAM and DRAM from their storage mechanisms;
- distinguish common nonvolatile-memory families;
- distinguish word-width expansion from address-space expansion;
- connect the general model to PIC16F883 memory spaces.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Memory Organization and Data EEPROM chapters — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 program/data/nonvolatile-memory application context.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A, Memory Organization material — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: embedded memory organization and address/data/control context.

- Microchip Technology Inc., *Using SRAM with a PIC16CXX*, TB011 — https://www.microchip.com/en-us/application-notes/tb011
  - Used for: supplemental external-SRAM/address-data-control example.

- Infineon Technologies, *CY7C1338G 4-Mbit Flow-Through Sync SRAM Data Sheet*, Figure 5 "Read/Write Timing" — https://www.infineon.com/assets/row/public/documents/10/49/infineon-cy7c1338g-4-mbit-128-k-32-flow-through-sync-sram-datasheet-en.pdf
  - Used for: official example of address/control/data timing and Hi-Z transitions in a real SRAM.

- Micron Technology, *DDR5 SDRAM* — https://www.micron.com/products/memory/dram-components/ddr5-sdram
  - Used for: modern DRAM context including banks, command/address behavior, and refresh operations.

### Further reading

- Wikipedia, *Static random-access memory* — https://en.wikipedia.org/wiki/Static_random-access_memory
  - Supplemental SRAM cell diagrams and overview.

- Wikipedia, *Dynamic random-access memory* — https://en.wikipedia.org/wiki/Dynamic_random-access_memory
  - Supplemental DRAM cell, row/column, and refresh diagrams.

- Kleitz, *Digital Electronics: A Practical Approach*, Chapter 16; Tocci et al., memory chapter; Maini, *Digital Electronics*, memory chapter.
  - Course-supplied secondary references used in the inherited lesson; useful for additional diagrams and examples.

[Back to top](#top) · [Topics index](README.md)
