<a id="top"></a>

# RCET 3373 — Digital Representation

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

Microcontrollers store and manipulate bit patterns. Humans choose how to interpret those patterns.

The same eight bits might represent:

- an unsigned number;
- a signed number;
- eight independent flags;
- several packed fields;
- part of an address;
- a hardware configuration register.

Binary makes individual bits visible. Hexadecimal makes groups of bits compact enough to read comfortably. Decimal is often convenient for human quantities, but it can hide the structure that matters in embedded systems.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- convert unsigned values among binary, hexadecimal, and decimal;
- explain why one hexadecimal digit maps exactly to four bits;
- split a byte or register into named bit fields;
- distinguish a bit pattern from the meaning assigned to it;
- create and apply masks to isolate selected bits;
- identify when fixed width changes the interpretation of a value;
- recognize when a signed interpretation requires [two's complement](twos_complement_notes.md#core-model).

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

No PIC-specific prerequisite is required.

Related topics:

- [Two's Complement](twos_complement_notes.md#core-model) for signed fixed-width integers;
- [Data Memory, SFRs, and Banking](data-memory-sfrs-banking.md#sfrs-and-gprs) for the PIC16F883 register context;
- [PIC16F883 Architecture](pic16f883-architecture.md#instruction-format) for how register addresses and instruction fields use binary encodings.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="bits-bytes-nibbles"></a>
### Bits, nibbles, and bytes

A **bit** has two possible states: 0 or 1.

A **nibble** is four bits.

A **byte** is eight bits.

Hexadecimal fits digital work especially well because one hexadecimal digit represents exactly one nibble.

| Binary | Hex | Decimal |
| --- | ---: | ---: |
| `0000` | `0` | 0 |
| `0001` | `1` | 1 |
| `0010` | `2` | 2 |
| `0011` | `3` | 3 |
| `0100` | `4` | 4 |
| `0101` | `5` | 5 |
| `0110` | `6` | 6 |
| `0111` | `7` | 7 |
| `1000` | `8` | 8 |
| `1001` | `9` | 9 |
| `1010` | `A` | 10 |
| `1011` | `B` | 11 |
| `1100` | `C` | 12 |
| `1101` | `D` | 13 |
| `1110` | `E` | 14 |
| `1111` | `F` | 15 |

<a id="pattern-versus-meaning"></a>
### Pattern versus meaning

The bit pattern does not carry its interpretation by itself.

For example:

```text
1111 1011
```

can mean:

- 251 as an unsigned 8-bit integer;
- -5 as an 8-bit two's-complement integer;
- eight individual flags;
- a collection of smaller fields.

The surrounding hardware, software, data type, or protocol tells you which interpretation applies.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="binary-hex"></a>
### Binary and hexadecimal describe the same bits

Convert between binary and hexadecimal one nibble at a time:

```text
0010 1111 = 0x2F
1111 0000 = 0xF0
0xA7      = 1010 0111
```

You do not need to convert through decimal when the goal is simply to see or rewrite the bit pattern.

<a id="bit-fields"></a>
### Registers can contain several fields

A byte does not have to represent one number.

```text
bit:  7 | 6 5 4 | 3 2 1 0
      E | mode  | channel
```

One byte can therefore hold:

- one enable flag;
- a three-bit mode value;
- a four-bit channel value.

This is typical of microcontroller special-function registers.

<a id="masks"></a>
### Masks isolate selected bits

A mask uses 1s where values should be preserved and 0s where they should be cleared.

To isolate bits 6:3:

```text
bits:  7 6 5 4 3 2 1 0
mask:  0 1 1 1 1 0 0 0
       0x78
```

A bitwise AND with `0x78` preserves bits 6:3 and clears the others.

If the field must later be interpreted as a small ordinary number, it may also need to be shifted toward bit 0.

<a id="fixed-width"></a>
### Width matters

Leading zeros can be significant because the chosen width defines the representation.

```text
0x0F = 0000 1111   in 8 bits
0x0F = 0000 0000 0000 1111   in 16 bits
```

The numerical value may be the same, but field locations, masks, sign interpretation, overflow behavior, and storage requirements can change with width.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="field-example"></a>
### Worked example: decode a packed register

**Known**

```text
register = 1011 0110
bit 7    = enable
bits 6:4 = mode
bits 3:0 = channel
```

**Find**

The value of each field.

**Reasoning**

Split the bit pattern using the field boundaries:

```text
1 | 011 | 0110
```

Interpret each field independently.

**Result**

- enable = 1;
- mode = `011` = 3;
- channel = `0110` = 6.

**Meaning**

The byte is not best understood as the decimal number 182. Its useful meaning is the combination of three fields.

<a id="mask-example"></a>
### Worked example: isolate bits 6:3

**Known**

```text
value = 0b11010110
mask  = 0b01111000
```

**Reasoning**

Apply bitwise AND:

```text
11010110
01111000
--------
01010000
```

**Result**

The selected field remains in bits 6:3. Shifting right three places would produce `00001010`, decimal 10.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

When a register value seems wrong, separate three questions:

1. **What bits are physically stored?**
2. **How are those bits divided into fields?**
3. **How should each field be interpreted?**

Common errors include:

- converting the whole register to decimal before identifying fields;
- assuming the most-significant bit is always a sign bit;
- applying a mask but forgetting that the field is still shifted away from bit 0;
- losing meaningful leading zeros;
- mixing signed and unsigned interpretations.

When reading a PIC16F883 register diagram, use the field layout and bit labels in the data sheet before deciding what the byte “means.”

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Convert `1101 0110` to hexadecimal and unsigned decimal.
2. Write `0x3A` in 8-bit binary.
3. How many bits are represented by the hexadecimal value `0xB7`?
4. Write a mask that isolates bits 6:3 of an 8-bit register.
5. A register contains `1011 0110`. Bit 7 is enable, bits 6:4 are mode, and bits 3:0 are channel. Decode the fields.
6. The pattern `1111 1011` is stored in a byte. Why can you not determine its signed/unsigned meaning from the bits alone?
7. After masking bits 6:3 with `0x78`, why might a right shift still be useful?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. `1101 0110 = 0xD6 = 214` unsigned.
2. `0x3A = 0011 1010`.
3. Two hexadecimal digits represent eight bits.
4. `0111 1000 = 0x78`.
5. Enable = 1, mode = 3, channel = 6.
6. Signedness is an interpretation supplied by the data type, hardware definition, or protocol. The same stored pattern can have several meanings.
7. The mask preserves the field in its original bit positions. Shifting it toward bit 0 makes it easier to interpret as an ordinary small integer.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- convert binary and hexadecimal directly by nibble;
- distinguish a stored bit pattern from its interpretation;
- break a register into fields;
- create a mask for a selected field;
- explain why a shift may follow a mask;
- explain why width and leading zeros can matter;
- identify when the [two's-complement](twos_complement_notes.md#core-model) model is required.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: examples of register bit fields, binary register organization, addresses, and device-specific embedded context.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: mid-range PIC register, data-memory, and instruction-field context.

### Further reading

- Wikipedia, *Hexadecimal* — https://en.wikipedia.org/wiki/Hexadecimal
  - Supplemental overview of hexadecimal notation and its relationship to binary.

[Back to top](#top) · [Topics index](README.md)
