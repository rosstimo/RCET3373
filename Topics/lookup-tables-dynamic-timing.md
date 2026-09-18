# RCET3373 - Lookup Tables and Dynamic Software Timing - Self-Learning Guide

# What you should be able to do

After working through this guide, you should be able to:

- trace a nested `DECFSZ` delay and identify where each loop branches;
- calculate the demonstrated delay from instruction-cycle counts;
- explain why the selected inner count gives a 20 us outer-count resolution;
- distinguish a lookup-table index from the value returned by the table;
- explain how PCL and PCLATH participate in a computed GOTO;
- trace `ADDWF PCL` + `RETLW` lookup-table execution;
- recognize when a short table can still cross a PCL low-byte boundary;
- convert a half-cycle delay to square-wave frequency;
- explain why equal time steps do not create equal frequency steps;
- identify why polling and other caller code change the measured waveform;
- explain why hardware timers and interrupts become useful after software delays.

# 1. Start with the control flow

A two-level software delay uses one countdown inside another.

A representative structure is:

```asm
    movlw   N
    movwf   out_count

outer_delay:
    movlw   I
    movwf   in_count

inner_delay:
    decfsz  in_count, F
    goto    inner_delay

    nop
    decfsz  out_count, F
    goto    outer_delay
```

The important branch targets are:

- the inner `GOTO` returns to `DECFSZ in_count`;
- the outer `GOTO` returns to **reload the inner count**;
- the outer `GOTO` must **not** return to the code that reloads `out_count`.

If `out_count` is reloaded every time, it never progresses toward zero.

A useful flowchart is:

```text
load outer count
      |
load inner count <-------------------+
      |                               |
decrement inner                      |
      |                               |
inner zero? -- no -------------------+
      |
     yes
      |
   NOP / outer work
      |
decrement outer
      |
outer zero?
  | no
  +----------> load inner count
  |
 yes
  |
 done
```

# 2. Count the executed instructions

For PIC16F883 at the current 4 MHz oscillator:

```text
TCY = 4/FOSC = 1 us
```

Useful instruction timing:

```text
MOVLW   1 cycle
MOVWF   1 cycle
MOVF    1 cycle
NOP     1 cycle
GOTO    2 cycles
CALL    2 cycles
RETURN  2 cycles
RETLW   2 cycles
DECFSZ  1 cycle normally
DECFSZ  2 cycles when the zero-result skip is taken
```

A countdown loop using:

```asm
    decfsz  count, F
    goto    loop
```

uses:

```text
3 cycles for each nonfinal iteration
2 cycles for the final successful DECFSZ
```

So the countdown part alone is:

```text
3(n-1) + 2 = 3n - 1 cycles
```

# 3. Derive the W03D03 nested-delay equation

Use the explicit timing boundary:

> Begin at the first `MOVLW N` and end after the final successful outer `DECFSZ` completes.

Break the code into regions.

## Outer setup

```asm
movlw N
movwf out_count
```

Cost:

```text
2 cycles
```

## Inner setup + inner countdown

Each outer iteration executes:

```asm
movlw I
movwf in_count
```

plus the inner countdown.

So:

```text
2 + (3I - 1) = 3I + 1 cycles
```

## NOP

The NOP is inside the outer loop, so it occurs once per outer iteration:

```text
N cycles total
```

## Outer countdown control

Across N outer iterations:

```text
3N - 1 cycles
```

## Combine the pieces

```text
T = 2 + N(3I + 1) + N + (3N - 1)
```

Simplify:

```text
T = N(3I + 5) + 1
```

# 4. Design a 20 us outer-count resolution

The coefficient multiplied by N tells you how much the delay changes when N changes by one:

```text
Delta_N = 3I + 5
```

If you want:

```text
Delta_N = 20 cycles
```

solve:

```text
3I + 5 = 20
3I = 15
I = 5
```

Therefore:

```text
Tcore = 20N + 1 cycles
```

At 4 MHz:

```text
Tcore = 20N + 1 us
```

Examples:

```text
N=1 -> 21 us
N=2 -> 41 us
N=3 -> 61 us
N=4 -> 81 us
```

The total is not 20, 40, 60, 80. The **change** is 20. The `+1` is one-time/final-path overhead in this explicitly defined interval.

# 5. Effective 8-bit countdown range

For this `DECFSZ` countdown:

```text
loaded 0x01 -> 1 decrement
loaded 0x02 -> 2 decrements
...
loaded 0xFF -> 255 decrements
loaded 0x00 -> 256 decrements
```

Why does zero mean 256 here?

The first decrement of `0x00` wraps to `0xFF`, which is not zero. The counter then continues down until a later decrement changes `0x01` to `0x00`, causing the successful skip.

So effective N is 1..256.

With `Tcore = 20N + 1 us`:

```text
minimum = 21 us
maximum = 5121 us
```

# 6. A selected delay needs persistent storage

Suppose a keypad selects a delay value. You cannot store that selection only in `out_count`, because the delay routine destroys `out_count` while counting down.

Use separate roles:

```text
current_count = persistent selected value
out_count     = working outer countdown
in_count      = working inner countdown
```

Representative code:

```asm
half_cycle_delay:
    movf    current_count, W
    movwf   out_count
    ; run nested delay
    return
```

`MOVF current_count,W` reads the file register into W.

`MOVLW` would be wrong here because `MOVLW` loads a literal constant, not the contents of another register.

# 7. What a lookup table is doing

The goal is to convert one small integer into another value.

Example:

```text
button/key index 0 -> delay count 250
button/key index 1 -> delay count 125
button/key index 2 -> delay count 12
```

The input index and returned delay value are different things.

The table is a compact way to store that mapping in program memory.

# 8. Program Counter, PCL, and PCLATH

PIC16F883 has a 13-bit Program Counter.

Think of it as:

```text
PC = [ upper 5 bits ][ lower 8 bits ]
          ^                 ^
       PCLATH              PCL
   during a PCL write
```

More precisely:

- `PCL` is the readable/writable low 8-bit portion;
- `PC<12:8>` is not directly readable/writable;
- `PCLATH` is a separate readable/writable holding register;
- when software writes PCL, `PCLATH<4:0>` supplies the upper PC bits.

For `CALL` and `GOTO`, a different loading path is used:

- the instruction contains 11 destination bits;
- `PCLATH<4:3>` supplies the upper two bits.

This creates a 2K-word page issue for normal `CALL`/`GOTO` targets.

# 9. Computed GOTO with `ADDWF PCL`

A lookup table can use the input index as an offset from the table's current location.

Representative code:

```asm
    movf    button_value, W
    call    lookup_delay
    movwf   current_count

lookup_delay:
    addwf   PCL, F
    retlw   0xFA
    retlw   0x7D
    retlw   0x0C
```

If W contains zero, execution continues at the first `RETLW`.

If W contains one, execution continues at the second `RETLW`.

If W contains two, execution continues at the third `RETLW`.

`RETLW 0x0C` does two things:

1. loads `0x0C` into W;
2. returns to the caller using the hardware return stack.

The caller then stores W into `current_count`.

# 10. The low-PCL rollover hazard

A common oversimplification is:

> My table only has 16 entries, so it cannot have a paging problem.

That is not always true.

A computed GOTO writes the low 8-bit PCL. If the table starts near the end of a 256-word low-byte block, later entries can cross:

```text
... 0x00FE
... 0x00FF
... 0x0100
... 0x0101
```

The low byte rolls from:

```text
0xFF -> 0x00
```

If the upper bits are not handled correctly, the computed jump can land in the wrong place.

The PIC16F883 data sheet explicitly warns about this.

For the beginner course implementation, the clean solution is:

- keep the complete table within one 256-word low-byte block;
- verify its linked address in MPLAB X Program Memory or the listing.

Do not confuse this 256-word low-PCL boundary with the 2K-word `CALL/GOTO` page boundary. They are different architecture limits.

# 11. `$` is an assembler location counter

The assembler permits:

```asm
goto $-1
```

On a mid-range PIC, `$-1` means one instruction before the current assembler location.

So:

```asm
    decfsz count, F
    goto   $-1
```

can form a tiny countdown loop without a label.

Important:

- `$` is not a runtime read of the Program Counter;
- the assembler resolves the target;
- a normal `GOTO` is emitted;
- labels are often clearer for larger structures.

# 12. Convert half-cycle time to frequency

If the delay occurs once between each output toggle, the delay approximates a half-cycle.

For the simplified core-only model:

```text
t_half = 20N + 1 us
```

The full period is:

```text
Tperiod = 2 t_half
```

Frequency is:

```text
f = 1 / Tperiod
```

## Worked example: N = 12

Known:

```text
N = 12
```

Half-cycle:

```text
t_half = 20(12) + 1
       = 241 us
```

Period:

```text
Tperiod = 482 us
```

Frequency:

```text
f = 1/(482 us)
  = 2074.69 Hz
```

So the core-only nominal output is about:

```text
2.075 kHz
```

# 13. Why high-frequency presets are hard to hit

The half-cycle time changes in equal 20 us steps:

```text
21, 41, 61, 81, ... us
```

But the corresponding frequencies are:

```text
N=1 -> 23.810 kHz
N=2 -> 12.195 kHz
N=3 ->  8.197 kHz
N=4 ->  6.173 kHz
```

The first one-count change reduces frequency by more than 11 kHz.

At the low-frequency end:

```text
N=250 -> 99.98 Hz
N=251 -> about 99.58 Hz
```

The same 20 us time step changes frequency by less than 1 Hz.

The reason is the reciprocal relationship:

```text
f = 1/(2t)
```

A constant time resolution is not a constant frequency resolution.

If accurate high-frequency presets matter, the timing resolution must be smaller or the timing method must change.

# 14. The named delay is not the whole waveform

Suppose you calculate:

```text
Tcore = 20N + 1 us
```

Then your program also executes:

```text
toggle output
scan buttons/keypad
make branch decisions
call lookup table
ADDWF PCL
RETLW
store current_count
CALL delay
RETURN from delay
branch back to main
```

The oscilloscope sees the time between output edges, not the time inside the code section named `delay`.

Therefore:

> Count every executed instruction between the two measurement edges.

If the input-scanning code sometimes takes different branch paths, the edge-to-edge time can vary. That is jitter.

# 15. Busy-wait delay also reduces responsiveness

While the CPU is inside a software delay, it is spending instruction cycles doing the delay.

If input is checked only after the delay finishes, a newly pressed button cannot be noticed until execution returns to the input check.

For the longest W03D03 core delay:

```text
5121 us
```

the program is tied up for about 5.1 ms before even counting the surrounding caller/input path.

This may or may not be acceptable. The system requirement decides that.

# 16. Why timers and interrupts come next

Software delay loops are useful because they make instruction timing visible. They also expose their own limitations:

- the CPU is busy waiting;
- every added instruction changes the timing;
- variable code paths create variable timing;
- input/service response is limited by how often the loop gets back to the work.

A hardware timer can count time independently while the CPU executes other instructions.

An interrupt can notify the CPU when a timer or peripheral event requires service.

That is why timer and interrupt architecture is the natural next step.

# Practice problems

## 1. Branch target

A nested delay has this structure:

```text
load outer
load inner
run inner countdown
decrement outer
if outer not zero -> ?
```

Where should the nonzero outer branch go?

A. load outer  
B. load inner  
C. end

## 2. Timing resolution

For:

```text
T = N(3I + 5) + 1
```

what `I` gives an outer-count resolution of 14 cycles?

## 3. Core delay

Using `Tcore = 20N + 1 us`, calculate the core delay for:

```text
N = 37
```

## 4. Half-cycle frequency

A half-cycle is 401 us. Ignoring all other overhead, calculate the square-wave frequency.

## 5. Effective count

How many decrements occur when an 8-bit `DECFSZ` countdown starts from `0x00`?

## 6. Persistent selection

Why should `current_count` and `out_count` be separate registers?

## 7. Lookup trace

Given:

```asm
lookup:
    addwf   PCL, F
    retlw   0x20
    retlw   0x40
    retlw   0x60
```

What value is returned in W when the zero-based table index is 2?

## 8. PCL/PCLATH

True or false: Because `PC<12:8>` cannot be directly written, PCLATH is also not writable.

Explain.

## 9. Table boundary

A 16-entry table begins close enough to address `0x01FF` that some entries are linked at `0x0200` and above. Why can that matter for an `ADDWF PCL` table?

## 10. `$`

What does `$` mean in pic-as, and what does `goto $-1` do on PIC16F883?

## 11. Whole waveform timing

A student says:

> My delay is exactly 500 us, so my square wave must be exactly 1 kHz.

List at least four pieces of code that might make that conclusion wrong.

## 12. Resolution

Why does a 20 us time step cause a much larger frequency jump near 20 kHz than near 100 Hz?

# Answer key

## 1

**B. load inner.** The inner counter must be reloaded for every outer iteration. Reloading the outer counter would destroy its progress toward zero.

## 2

The N coefficient is the resolution:

```text
3I + 5 = 14
3I = 9
I = 3
```

## 3

```text
T = 20(37) + 1
  = 741 us
```

## 4

Full period:

```text
T = 2(401 us) = 802 us
```

Frequency:

```text
f = 1/(802 us)
  ≈ 1246.88 Hz
  ≈ 1.247 kHz
```

## 5

**256 decrements.** `0x00` first decrements to `0xFF`, then continues until a later decrement reaches `0x00` and triggers the skip.

## 6

`out_count` is destroyed as the delay counts down. `current_count` preserves the selected value so it can reload `out_count` for the next half-cycle.

## 7

Index 2 selects the third entry:

```text
W = 0x60
```

`RETLW` loads the literal into W and returns.

## 8

**False.** The upper Program Counter bits are not directly writable, but PCLATH is a separate readable/writable holding register. PCLATH supplies upper PC bits during particular Program Counter loads.

## 9

The low byte of the address rolls from `0xFF` to `0x00`. A computed GOTO writes PCL, so the upper address bits must be correct through PCLATH. A short table can cross this boundary depending on placement.

## 10

`$` is the assembler's current location within the active program section. On a mid-range PIC, `$-1` means one instruction word before the current location. The assembler emits a normal `GOTO` to that address.

## 11

Possible omitted timing:

- output toggle instruction(s);
- keypad/button scan;
- conditional branches;
- LUT call;
- `ADDWF PCL`;
- `RETLW`;
- `MOVWF current_count`;
- delay `CALL` and `RETURN`;
- branch back to main.

The correct answer depends on the exact executed path between output edges.

## 12

Frequency is reciprocal in period:

```text
f = 1/(2t)
```

At short times, a 20 us change is a large fraction of the period. At long times, it is a small fraction. Equal time steps therefore produce unequal frequency steps.

# What to be able to explain without notes

You should be able to explain:

- why the outer loop branches to inner-count reload rather than outer-count reload;
- where `T = N(3I + 5) + 1` comes from;
- why `I=5` produces a 20 us outer-count step at 4 MHz;
- what `current_count`, `out_count`, and `in_count` each do;
- how an index reaches an `RETLW` entry through `ADDWF PCL`;
- what PCL and PCLATH each contribute to the Program Counter;
- why a short table can still cross a low-PCL boundary;
- why 21 us half-cycle corresponds to about 23.81 kHz rather than about 47.6 kHz;
- why polling and other caller code must be included in measured waveform timing;
- why timers and interrupts are the next logical architecture topic.

# References used to build this guide

Microchip PIC16F882/883/884/886/887 Data Sheet:  
https://ww1.microchip.com/downloads/en/devicedoc/41291f.pdf

MPLAB XC8 PIC Assembler User's Guide:  
https://ww1.microchip.com/downloads/aemDocuments/documents/DEV/ProductDocuments/UserGuides/MPLAB-XC8-PIC-Assembler-Users-Guide-DS50002974.pdf

Current RCET3375 application context:  
`LabAssignments/Lab05-Muzak.md`
