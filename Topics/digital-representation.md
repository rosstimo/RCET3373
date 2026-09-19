# RCET 3373 - Digital Representation Self-Learning Guide

## What you should be able to do

After working through this guide, you should be able to:

- convert unsigned values among binary, hexadecimal, and decimal;
- explain why hexadecimal is convenient for digital systems;
- split a byte into named bit fields;
- interpret the same bit pattern differently when the context changes;
- represent negative values using two's complement at a specified width.

## Binary and hexadecimal describe the same bits

Hexadecimal does not change the value. It is a compact way to write binary.

One hexadecimal digit maps exactly to four bits.

```text
0010 1111 = 0x2F
1111 0000 = 0xF0
0xA7      = 1010 0111
```

When working with registers, group binary values into four-bit nibbles instead of repeatedly converting through decimal.

## A register is often a collection of fields

A byte does not have to represent one ordinary number.

```text
bit:  7 | 6 5 4 | 3 2 1 0
      E | mode  | channel
```

The same eight bits can represent an enable bit, a mode field, and a channel field at the same time.

This is why embedded-system work often uses binary or hexadecimal rather than decimal. Those representations make the bit structure visible.

## Signed and unsigned interpretation depends on context

The bit pattern itself is not inherently signed.

For example, `1111 1011` can be interpreted as an unsigned value or as an 8-bit two's-complement value. The agreed representation tells you which meaning applies.

### Forming a negative two's-complement value

To represent `-5` in eight bits:

```text
+5:      0000 0101
invert:  1111 1010
add 1:   1111 1011
```

Fixed width matters. The same bit pattern can mean something different when the width or representation changes.

For more worked subtraction examples, see [Two's Complement Notes](twos_complement_notes.md).

## Masks isolate fields

A mask uses 1s in the bit positions you want to keep.

To isolate bits 6:3 of an 8-bit value:

```text
bits:  7 6 5 4 3 2 1 0
mask:  0 1 1 1 1 0 0 0
       0x78
```

Applying the mask with a bitwise AND clears the unrelated bits and preserves the selected field.

## Practice

1. Convert `1101 0110` to hexadecimal and unsigned decimal.
2. Write `0x3A` in binary.
3. Interpret `1111 1011` as an 8-bit two's-complement value.
4. Write a mask that isolates bits 6:3 of an 8-bit register.
5. A register contains `1011 0110`. Bit 7 is an enable bit, bits 6:4 are a mode, and bits 3:0 are a channel. State the value of each field.

## Answer key

1. `1101 0110 = 0xD6 = 214` unsigned.
2. `0x3A = 0011 1010`.
3. `1111 1011 = -5` in 8-bit two's complement.
4. `0111 1000 = 0x78`.
5. Enable = 1, mode = `011` = 3, channel = `0110` = 6.

## Common mistakes to avoid

- treating hexadecimal as a different underlying value;
- assuming the most significant bit is always a sign bit;
- assuming leading zeros never matter;
- reading every register as one decimal number instead of as fields;
- forgetting that signed interpretation requires a fixed width.
