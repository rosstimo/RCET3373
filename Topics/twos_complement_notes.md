<a id="top"></a>

# RCET 3373 — Two's Complement

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

A fixed-width register contains bits, not a plus sign or minus sign.

When those bits are interpreted as a signed integer, embedded systems commonly use **two's complement**. The representation is useful because the same binary addition hardware can handle both positive and negative values.

The important skill is not memorizing that “MSB = 1 means negative.” It is being able to reason about:

- the selected bit width;
- the signed range;
- how a negative value is encoded;
- how to interpret a stored pattern;
- how addition/subtraction behave when the result wraps at the fixed width.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- state the two's-complement range for an N-bit signed integer;
- encode a negative integer at a specified bit width;
- decode a two's-complement bit pattern;
- perform subtraction by adding the two's complement of the subtrahend;
- explain why the carry out of the fixed width is discarded;
- distinguish arithmetic wraparound from the signed interpretation of the result;
- recognize when a calculation exceeds the representable signed range.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [binary, hexadecimal, and fixed-width representation](digital-representation.md#core-model).

Related topics:

- [PIC16F883 STATUS flags](pic16f883-architecture.md#status-flags);
- [instruction timing and arithmetic flow](instruction-timing-software-delays.md#instruction-cycle).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

For an N-bit two's-complement integer:

```text
minimum = -2^(N-1)
maximum =  2^(N-1) - 1
```

For 8 bits:

```text
-128 through +127
```

The most-significant bit contributes a negative weight in the signed interpretation:

```text
bit weights for 8-bit two's complement:

-128  64  32  16  8  4  2  1
```

That gives another way to decode a pattern without first taking its magnitude.

Example:

```text
1111 1000
= -128 + 64 + 32 + 16 + 8
= -8
```

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="negating-a-value"></a>
### Form the negative of a value

To negate a fixed-width binary value:

1. invert every bit;
2. add 1;
3. keep only the selected width.

Example: encode -5 in eight bits.

```text
+5:      0000 0101
invert:  1111 1010
add 1:   1111 1011
```

Therefore:

```text
1111 1011 = -5
```

<a id="why-addition-works"></a>
### Why addition still works

Add +5 and -5:

```text
  0000 0101
+ 1111 1011
-----------
1 0000 0000
```

The result is stored in eight bits, so the ninth carry bit is discarded:

```text
0000 0000
```

The fixed-width arithmetic naturally wraps modulo `2^N`.

<a id="decode-negative"></a>
### Decode a negative pattern

Method 1: invert and add 1 to find the magnitude.

```text
1111 1000
invert -> 0000 0111
add 1 -> 0000 1000
magnitude = 8
result = -8
```

Method 2: use the unsigned value.

For an N-bit pattern whose MSB is 1:

```text
signed value = unsigned value - 2^N
```

For `1111 1000`:

```text
248 - 256 = -8
```

<a id="signed-range"></a>
### Signed range is asymmetric

For eight bits:

| Pattern | Signed value |
| --- | ---: |
| `0111 1111` | +127 |
| `0000 0000` | 0 |
| `1111 1111` | -1 |
| `1000 0000` | -128 |

There is one more negative value than positive because zero occupies one of the nonnegative patterns.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="nine-minus-eight"></a>
### Worked example: 9 - 8

**Known**

```text
9 = 0000 1001
8 = 0000 1000
```

**Find**

`9 - 8`.

**Reasoning**

Convert the subtraction to addition:

```text
9 - 8 = 9 + (-8)
```

Find -8:

```text
8:       0000 1000
invert:  1111 0111
add 1:   1111 1000
```

Add:

```text
  0000 1001
+ 1111 1000
-----------
1 0000 0001
```

Discard the carry outside the eight-bit width.

**Result**

```text
0000 0001 = 1
```

<a id="eight-minus-nine"></a>
### Worked example: 8 - 9

**Known**

```text
8 = 0000 1000
9 = 0000 1001
```

**Reasoning**

```text
8 - 9 = 8 + (-9)
```

Find -9:

```text
9:       0000 1001
invert:  1111 0110
add 1:   1111 0111
```

Add:

```text
  0000 1000
+ 1111 0111
-----------
  1111 1111
```

Decode `1111 1111`.

**Result**

```text
1111 1111 = -1
```

<a id="range-example"></a>
### Worked example: detect an out-of-range result

In signed 8-bit arithmetic:

```text
100 + 40 = 140
```

But +140 is outside the representable range of -128 to +127.

If the arithmetic is forced into eight bits, the stored bit pattern wraps. That stored pattern should not be mistaken for the mathematically correct signed result.

The general lesson is:

> A valid bit pattern can still represent the wrong mathematical answer when the true result is outside the allowed range.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

When signed results look surprising, check these in order:

1. What bit width is being used?
2. Is the value supposed to be signed or unsigned?
3. Is the true mathematical result inside the signed range?
4. Did a carry leave the fixed-width result?
5. Did the sign change unexpectedly because of overflow?
6. Are you interpreting a register or field that is actually unsigned?

Do not assume that a high MSB always means “negative.” That interpretation applies only when the field is defined as a signed two's-complement integer.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. State the signed range of an 8-bit two's-complement value.
2. State the signed range of a 16-bit two's-complement value.
3. Encode -12 in eight bits.
4. Decode `1110 1101` as an 8-bit two's-complement value.
5. Compute `14 - 9` using eight-bit two's-complement addition.
6. Compute `9 - 14` using eight-bit two's-complement addition.
7. Why is `1000 0000` equal to -128 rather than “negative zero”?
8. Why can the same pattern `1111 1011` mean 251 in one context and -5 in another?
9. Is +150 representable in signed 8-bit two's complement? Explain.

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. -128 through +127.
2. -32768 through +32767.
3. +12 = `0000 1100`; invert and add 1 -> `1111 0100`.
4. Unsigned value 237; `237 - 256 = -19`.
5. -9 = `1111 0111`; adding to 14 gives `0000 0101` after discarding the carry, so the result is +5.
6. -14 = `1111 0010`; adding to 9 gives `1111 1011`, which is -5.
7. Two's complement has only one zero, `0000 0000`. `1000 0000` has the signed weight -128.
8. The bits do not define signedness by themselves. The surrounding data type or hardware definition supplies the interpretation.
9. No. Signed 8-bit two's complement stops at +127.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- derive the signed range from the bit width;
- encode and decode a negative value;
- explain why invert-plus-one produces the additive inverse;
- perform subtraction through addition;
- explain why carry out is discarded at fixed width;
- recognize when a true result is outside the representable signed range;
- distinguish signed interpretation from the underlying stored bit pattern.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: course application context involving the PIC mid-range ALU, arithmetic instructions, and status behavior.

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883 instruction-set and STATUS-register context.

### Further reading

- Wikipedia, *Two's complement* — https://en.wikipedia.org/wiki/Two%27s_complement
  - Supplemental visual and mathematical overview of fixed-width two's-complement representation.

[Back to top](#top) · [Topics index](README.md)
