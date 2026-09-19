<a id="top"></a>

# RCET 3373 — Subroutines and the PIC16F883 Return Stack

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

A subroutine lets code temporarily leave the current execution path, perform reusable work, and then return to the correct instruction automatically.

The key problem is:

> If execution jumps somewhere else, how does the processor remember where to come back?

On the PIC16F883, `CALL` and the hardware return stack solve that problem.

Understanding the return stack also matters for:

- nested subroutines;
- interrupts;
- exact timing;
- page-crossing control flow;
- diagnosing stack-depth bugs.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- explain what address `CALL` saves;
- explain what `RETURN`, `RETLW`, and `RETFIE` restore;
- distinguish the hardware return stack from GPR data memory;
- calculate simultaneous stack depth;
- include accepted interrupts in stack-depth reasoning;
- explain the role of `PCLATH<4:3>` for `CALL`/`GOTO` page selection;
- calculate a callable delay including `CALL` and `RETURN`;
- use MPLAB X Simulator to verify PC/stack/control-flow reasoning;
- recognize the PIC16F883 stack's depth and overflow limitations.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [PIC16F883 program memory and vectors](pic16f883-architecture.md#program-memory);
- [Instruction Timing and Software Delays](instruction-timing-software-delays.md#core-model).

Related topics:

- [Interrupts and Context Saving](interrupts-context-saving.md#core-model);
- [Lookup Tables and Dynamic Timing](lookup-tables-dynamic-timing.md#pcl-pclath);
- [Measurement Strategy and C Timing](measurement-c-timing.md#measurement-strategy).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="call-return-model"></a>
### `CALL` saves the return path

`GOTO` changes the Program Counter but does not remember where execution came from.

`CALL` does two things:

1. saves the address of the instruction after the `CALL` on the hardware return stack;
2. loads the Program Counter with the subroutine target.

`RETURN` pops the most recent saved return address and resumes execution there.

A useful mental model is a stack of bookmarks.

<a id="hardware-stack"></a>
### PIC16F883 hardware return stack

The PIC16F883 provides an eight-level hardware return-address stack.

Important properties:

- stores return addresses, not general variables;
- separate from ordinary GPR data memory;
- `CALL` pushes;
- accepted interrupt entry pushes;
- `RETURN`, `RETLW`, and `RETFIE` pop;
- software cannot use it like ordinary RAM;
- excessive nesting can overwrite older return information because this classic stack does not provide the kind of software-visible overflow protection found on many larger processors.

**Official visual reference:** In Section 2.0 of the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf), inspect the program-memory/stack figure.

Focus on the 13-bit PC and the eight return-stack levels as structures separate from GPR data memory.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="nested-depth"></a>
### Sequential calls and nested calls are different

Sequential:

```asm
call A
call A
call A
call A
```

If each call returns before the next one begins:

```text
maximum simultaneous stack depth = 1
```

Nested:

```text
Main -> A -> B -> C
```

Before C returns, three return addresses are simultaneously active:

```text
maximum simultaneous depth = 3
```

Count the deepest path, not the total number of `CALL` instructions that execute over time.

<a id="interrupt-depth"></a>
### Interrupt entry also uses the return stack

When an interrupt is accepted, the current return address is pushed so `RETFIE` can resume the interrupted code.

The PIC16F883 has one interrupt vector and no hardware interrupt-priority levels.

Normal interrupt acceptance clears `GIE`, so another ordinary interrupt is not accepted until global interrupts are enabled again.

For stack planning, an accepted interrupt can add one level to whatever call depth already exists.

<a id="pclath-call-goto"></a>
### `PCLATH` and larger program-memory targets

The PC is 13 bits wide, while `CALL` and `GOTO` encode only part of the destination address directly.

For these instructions, `PCLATH<4:3>` supplies upper destination bits needed for page selection.

That means control-flow correctness in larger programs includes both:

- the encoded target field;
- the appropriate page-selection state.

Assembler helpers can manage this, but the hardware model still matters when diagnosing page-crossing problems.

<a id="return-instructions"></a>
### `RETURN`, `RETLW`, and `RETFIE`

All three return using the hardware stack, but they are used for different purposes.

| Instruction | Main use |
| --- | --- |
| `RETURN` | ordinary subroutine return |
| `RETLW k` | return while placing literal `k` in W |
| `RETFIE` | return from interrupt and restore interrupt-enable behavior |

`RETLW` is especially useful for lookup-table patterns covered later.

<a id="callable-delay"></a>
### Callable delay timing includes `CALL` and `RETURN`

For:

```asm
    call    Delay50us

Delay50us:
    movlw   15
    movwf   count
loop50:
    decfsz  count, F
    goto    loop50
    return
```

define the interval:

> Start with `CALL`; end after `RETURN` completes.

At 4 MHz:

```text
CALL                2 cycles
setup               2 cycles
countdown loop    3n - 1 cycles
RETURN              2 cycles
----------------------------
total              3n + 5 cycles
```

Solve:

```text
3n + 5 = 50
n = 15
```

The routine is exactly 50 instruction cycles for that boundary.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="stack-depth-example"></a>
### Worked example: stack depth with an interrupt

Suppose the execution path is:

```text
Main -> A -> B
```

While B is executing, an interrupt is accepted.

Before the interrupt:

```text
depth = 2
```

Interrupt entry pushes one additional return address:

```text
depth = 3
```

If the ISR does not call another subroutine, maximum depth on that path is 3.

<a id="five-ms-example"></a>
### Worked example: build a longer callable delay

Suppose a longer routine repeatedly calls the exact 50 us subroutine and its explicit outer overhead gives:

```text
T = 53N + 5
```

For a 5000 us target:

```text
N = floor((5000 - 5) / 53)
  = 94

T = 53(94) + 5
  = 4987 us
```

Residual:

```text
5000 - 4987 = 13 us
```

A one-byte delay with:

```text
3n + 1 = 13
n = 4
```

can supply the remaining 13 us if inserted within the defined path without double-counting the final return.

The important habit is to account for every call/return/control instruction exactly once.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="simulator-workflow"></a>
### Verify control flow in MPLAB X Simulator

Useful checks:

1. select **Simulator** as the project tool;
2. halt at the reset/startup path;
3. inspect Program Memory;
4. watch the Program Counter;
5. step across a `CALL`;
6. verify entry into the subroutine;
7. step through `RETURN`;
8. confirm execution resumes at the instruction after the original `CALL`;
9. use breakpoints to observe nested calls;
10. inspect generated instructions around assembler directives when timing matters.

Simulation verifies code/control-flow reasoning.

It does not prove oscillator loading, wiring, analog behavior, or physical timing accuracy.

<a id="stack-debug-checklist"></a>
### Return-path debugging checklist

If a program returns to the wrong place or behaves unpredictably:

1. trace the deepest nested call path;
2. include possible interrupt entry;
3. inspect whether an ISR itself calls subroutines;
4. check for unmatched `CALL`/return logic;
5. inspect page selection for far `CALL`/`GOTO` targets;
6. verify computed-GOTO/table code has not corrupted PC-related state;
7. use Simulator to observe PC progression.

For timing claims, also use [Measurement Strategy and C Timing](measurement-c-timing.md#measurement-strategy).

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. What address does `CALL` save?
2. What does `RETURN` restore?
3. How deep is the PIC16F883 hardware return stack?
4. Why can four sequential calls have maximum depth 1?
5. For `Main -> A -> B -> C`, what is the maximum call depth before any returns?
6. If an interrupt is accepted while depth is 3, what stack depth is reached before any ISR calls?
7. Which `PCLATH` bits contribute upper destination information for `CALL`/`GOTO`?
8. Why must `CALL` and `RETURN` be included in an exact callable-delay timing equation?
9. Why is `n = 15` correct for the shown 50-cycle routine?
10. What can Simulator prove, and what still requires hardware verification?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. The address of the instruction after the `CALL`.
2. The most recently saved return address from the hardware stack.
3. Eight levels.
4. Each call returns before the next begins, so only one return address is active at a time.
5. Three.
6. Four; interrupt entry pushes one additional return address.
7. `PCLATH<4:3>`.
8. They execute inside the defined boundary and consume documented instruction cycles.
9. `3(15) + 5 = 50`.
10. Simulator can verify instruction/control/register behavior; physical oscillator, wiring, loading, and analog/hardware timing still need real hardware evidence.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- explain the saved return address for `CALL`;
- distinguish return stack from GPR RAM;
- calculate simultaneous stack depth;
- include interrupt entry in depth reasoning;
- explain the roles of `RETURN`, `RETLW`, and `RETFIE`;
- explain why page selection matters for larger `CALL`/`GOTO` targets;
- derive the `3n + 5` callable-delay expression;
- verify a call/return path in Simulator.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Memory Organization and Instruction Set Summary — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PC width, eight-level return stack, `CALL`, `RETURN`, `RETLW`, `RETFIE`, and cycle counts.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: program-counter, stack, paging, and control-flow architecture.

- [PIC16F883 Subroutines and Stack](https://github.com/rosstimo/pic_projects/blob/main/References/PIC16F883-Subroutines-Stack.md)
  - RCET shared implementation/reference material for call/return, stack depth, and timing examples.

- Microchip Technology Inc., *MPLAB X IDE* — https://www.microchip.com/en-us/development-tool/MPLAB-X-IDE
  - Used for: simulator/debug environment context.

[Back to top](#top) · [Topics index](README.md)
