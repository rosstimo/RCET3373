
# Teaching 2's Complement – Conversation Notes

## Explaining 2's Complement

2’s complement is the digital equivalent of humans deciding negative numbers should exist, but with fewer tears and way more logic gates.

### The Problem
Computers only know **0s and 1s**. If you want them to handle both positive and negative integers, you need a system. The simplest idea—just slap a sign bit on the front—creates ugly problems when you try to add numbers. Engineers don’t like ugly.

### The Trick
2’s complement solves this by making **addition and subtraction use the exact same circuitry**. No separate “minus” logic needed.

Steps to find the 2’s complement (the negative of a binary number):
1. Write the number in binary.
2. Flip all the bits (0 → 1, 1 → 0).
3. Add 1.

That’s it. The result is the negative of the original number.

### Example
+5 in binary:  
```
00000101
```
Flip the bits:  
```
11111010
```
Add 1:  
```
11111011
```
That’s **-5** in 2’s complement.

### Why It Works
When you add +5 and -5 in binary:
```
  00000101   (+5)
+ 11111011   (-5)
------------
  00000000   (carry bit falls off)
```
No special subtraction logic, no extra rules. Just clean addition.

### Range
For **N bits**, 2’s complement represents:
- Minimum: `-2^(N-1)`
- Maximum: `2^(N-1) - 1`

So for 8 bits:  
Range is -128 to +127.

---

## Example: 9 - 8

### Step 1. Write 9 in binary  
```
00001001   (this is 9)
```

### Step 2. Write 8 in binary  
```
00001000   (this is 8)
```

### Step 3. Subtraction is really addition  
Instead of doing `9 - 8`, we do `9 + (-8)`.

### Step 4. Find -8 (2’s complement of 8)  
Start with 8:  
```
00001000
```
Flip:  
```
11110111
```
Add 1:  
```
11111000
```
This is -8.

### Step 5. Add 9 and -8  
```
  00001001
+ 11111000
----------
  111111001
```
Drop the carry (8 bits):  
```
00000001
```
**Answer: 1**

---

## Example: 8 - 9

### Step 1. Write 8 in binary  
```
00001000
```

### Step 2. Write 9 in binary  
```
00001001
```

### Step 3. Subtraction = addition  
So `8 - 9` → `8 + (-9)`.

### Step 4. Find -9  
```
00001001   (9)
11110110   (flip)
11110111   (add 1 = -9)
```

### Step 5. Add  
```
  00001000
+ 11110111
----------
  111111111
```
Drop the carry:  
```
11111111
```

### Step 6. Interpret  
MSB = 1 → negative number.  
Flip + add 1:  
```
00000000
+1 = 00000001
```
So it’s -1.  

**Answer: -1**

---

## MSB as the Sign Bit

In 8-bit two’s complement:  
- MSB = `0` → number is positive (0 to +127).  
- MSB = `1` → number is negative (-128 to -1).

Examples:  
- `00001001` = +9  
- `11111000` = -8  
- `11111111` = -1  

---

## Two’s Complement Table (+15 to -15)

```
Decimal    Binary
-------    --------
+15        00001111
+14        00001110
+13        00001101
+12        00001100
+11        00001011
+10        00001010
 +9        00001001
 +8        00001000
 +7        00000111
 +6        00000110
 +5        00000101
 +4        00000100
 +3        00000011
 +2        00000010
 +1        00000001
  0        00000000
 -1        11111111
 -2        11111110
 -3        11111101
 -4        11111100
 -5        11111011
 -6        11111010
 -7        11111001
 -8        11111000
 -9        11110111
-10        11110110
-11        11110101
-12        11110100
-13        11110011
-14        11110010
-15        11110001
```

---

## Converting Negative Numbers to Base 10

### Example 1: `11111111`
1. MSB = 1 → negative.  
2. Flip: `00000000`  
3. Add 1: `00000001`  
**Answer: -1**

### Example 2: `11111000`
1. MSB = 1 → negative.  
2. Flip: `00000111`  
3. Add 1: `00001000`  
**Answer: -8**

### Example 3: `11110001`
1. MSB = 1 → negative.  
2. Flip: `00001110`  
3. Add 1: `00001111`  
**Answer: -15**

---

### Shortcut Method
Unsigned value – 256 = signed value (when MSB = 1).  

Example: `11111000` = 248 unsigned.  
248 – 256 = –8.  

---
