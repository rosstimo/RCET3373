<a id="top"></a>

# RCET 3373 — Interrupts and Context Saving

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

Polling asks:

> Has something happened yet?

An interrupt lets hardware record an event and request processor attention when the required enable conditions are satisfied.

Interrupts are useful because the CPU can perform other work instead of continuously checking one condition. They also introduce new responsibilities:

- identify what event occurred;
- understand the flag and enable path;
- preserve processor state;
- clear the event correctly;
- return without damaging the interrupted code;
- account for latency and service time.

The transferable model is more important than memorizing one interrupt source.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- trace an interrupt from hardware event to flag, enable logic, vector, ISR, and return;
- distinguish an interrupt flag from its source-enable bit;
- determine whether `PEIE` applies to a source;
- explain the role of `GIE`;
- configure and trace the external `RB0/AN12/INT` source;
- explain what hardware does automatically on interrupt entry;
- explain why one vector can serve many interrupt sources;
- identify and clear the correct source flag;
- explain why W and STATUS may need context preservation;
- explain the Microchip `SWAPF` context-save pattern;
- include interrupt latency and ISR work in timing analysis;
- apply a reusable checklist to an unfamiliar interrupt source.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [PIC16F883 reset and interrupt vectors](pic16f883-architecture.md#vectors);
- [Subroutines and the hardware return stack](subroutines-return-stack.md#hardware-stack);
- [PORTB configuration](digital-io-portb-configuration.md#core-model);
- [Data Memory, SFRs, and Banking](data-memory-sfrs-banking.md#core-model).

Related topics:

- [MPLAB X, PSECTs, and vector placement](mplab-pic-as-psects-vectors.md#hardware-vectors);
- [PIC16F883 Timers](timers.md#core-model);
- [Timing Measurement](measurement-c-timing.md#measurement-strategy).

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="event-flag-enable"></a>
### Event, flag, enable, and global gate

A useful interrupt model is:

```text
hardware event
    ↓
interrupt flag becomes set
    ↓
source enable permits a request
    ↓
optional peripheral gate (PEIE) when applicable
    ↓
global enable (GIE)
    ↓
CPU accepts interrupt
    ↓
interrupt vector
    ↓
software identifies and services source
    ↓
flag cleared correctly
    ↓
context restored
    ↓
RETFIE
```

Keep these ideas separate.

**Flag**

Records that the event happened.

**Source enable**

Controls whether that source is allowed to request interrupt service.

**PEIE**

Additional gate used by peripheral interrupt sources routed through the peripheral interrupt structure.

**GIE**

Global gate for maskable interrupt service.

A flag may become set even while its interrupt is disabled. That matters during initialization because stale pending flags should usually be handled before enabling service.

<a id="direct-peripheral-paths"></a>
### Direct sources and peripheral sources do not use identical gates

Some sources, such as Timer0 and external INT, have direct enable/flag paths in `INTCON`.

Other peripheral sources pass through the peripheral interrupt structure and require `PEIE` in addition to their individual enable and `GIE`.

**Official visual reference:** See Figure 14-7, **Interrupt Logic**, in the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf).

Follow several different source paths from flag and enable bits toward `GIE` and the common interrupt request. Notice which paths pass through `PEIE` and which do not.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="external-int"></a>
### External interrupt on `RB0/AN12/INT`

The shared pin is:

```text
RB0 / AN12 / INT
```

Using it as an external interrupt input requires both ordinary pin configuration and interrupt configuration.

Important controls include:

```text
ANSELH.ANS12       analog/digital selection
TRISB.TRISB0       input direction
OPTION_REG.INTEDG  rising/falling edge selection
INTCON.INTF        external INT flag
INTCON.INTE        external INT enable
INTCON.GIE         global interrupt enable
```

For a pull-up button that is HIGH when released and LOW when pressed:

```text
press -> falling edge
```

so `INTEDG = 0` is an appropriate choice for that circuit.

Use the [PORTB guide](digital-io-portb-configuration.md#configuration-layers) for the pin-configuration reasoning.

<a id="safe-setup"></a>
### Enable interrupts only after setup is ready

A clean setup pattern is:

```text
1. configure shared pin/function
2. choose active edge
3. clear stale flag
4. enable source
5. enable GIE last
```

Conceptually:

```asm
; configure RB0/AN12 as the intended digital input

BANKSEL OPTION_REG
bcf     OPTION_REG, INTEDG

BANKSEL INTCON
bcf     INTCON, INTF
bsf     INTCON, INTE
bsf     INTCON, GIE
```

The exact source code should use the current symbols/helpers provided by the toolchain and be checked against the generated listing when uncertainty matters.

<a id="interrupt-entry"></a>
### What hardware does on interrupt entry

When an interrupt is accepted, the PIC16F883 performs key automatic actions:

```text
GIE is cleared
return PC is pushed onto the hardware return stack
PC is loaded with 0x0004
```

The processor does not automatically jump to a named handler function.

It goes to the one interrupt vector at:

```text
0x0004
```

The vector may then branch to the real handler:

```asm
PSECT isrVect,class=CODE,delta=2
InterruptVector:
    goto IsrHandler
```

For linker placement of `isrVect`, see [MPLAB X, PSECTs, and Vectors](mplab-pic-as-psects-vectors.md#hardware-vectors).

<a id="source-identification"></a>
### One vector serves many sources

PIC16F883 has multiple interrupt sources but one normal interrupt vector.

Software therefore identifies the source by examining the relevant flags.

A conceptual ISR can look like:

```asm
IsrHandler:
    ; save context

    btfsc   INTCON, INTF
    goto    HandleExternalInt

    btfsc   INTCON, T0IF
    goto    HandleTimer0

    goto    LeaveIsr
```

The order of source checks is a software/application decision.

An interrupt flag for another source can become set while `GIE = 0` inside the current ISR. If it remains pending, another interrupt can be accepted after `RETFIE` re-enables interrupts.

<a id="flag-clearing"></a>
### Each source has its own flag-clearing rule

For external INT, `INTF` must be cleared in software.

If the ISR returns while:

```text
INTF = 1
INTE = 1
```

then `RETFIE` restores `GIE` and the still-pending source can immediately request service again.

For every interrupt source, determine from the official documentation:

- what sets the flag;
- what clears it;
- whether software clears the flag bit directly;
- whether reading/writing another register participates in clearing;
- whether a required sequence exists.

Do not assume every peripheral uses the same clearing rule.

<a id="retfie"></a>
### `RETURN` and `RETFIE` are not interchangeable

`RETURN` restores the saved execution address for an ordinary subroutine.

`RETFIE` restores the interrupt return address **and** re-enables global interrupt service according to the device behavior.

Use the instruction intended for the execution context.

<a id="context-saving"></a>
### Interrupt entry does not preserve all CPU context

Hardware automatically preserves the return PC.

It does **not** automatically preserve every processor register or status bit the main program may depend on.

An ISR may change:

- W;
- STATUS flags;
- bank-selection state;
- PCLATH;
- application registers.

If the ISR changes state that interrupted code expected to remain unchanged, the program can fail depending on the exact instant the interrupt occurred.

That is why context save/restore matters.

<a id="common-ram"></a>
### Common RAM is useful for context temporaries

The PIC16F883 data-memory map includes a common GPR region accessible across banks.

The `0x70–0x7F` region is useful for temporary ISR context storage such as:

```asm
w_temp      EQU 0x70
status_temp EQU 0x71
pclath_temp EQU 0x72
```

Because these locations are accessible regardless of the active bank, the ISR can save context before deliberately changing bank-selection state.

**Official table reference:** Use the PIC16F883 data-memory maps in Section 2.0 of the data sheet to verify the common GPR region and mirrored access behavior.

<a id="swapf-pattern"></a>
### Why the Microchip context-save pattern uses `SWAPF`

A classic save/restore pattern includes:

```asm
movwf   w_temp
swapf   STATUS,W
movwf   status_temp

; ISR work

swapf   status_temp,W
movwf   STATUS
swapf   w_temp,F
swapf   w_temp,W
```

The unusual-looking sequence exists because instruction side effects matter.

For PIC16F883:

```text
MOVF   affects Z
MOVWF  does not affect STATUS flags
SWAPF  does not affect STATUS flags
```

Using:

```asm
movf STATUS,W
```

would risk changing `Z` before the original STATUS value has been preserved.

`SWAPF STATUS,W` moves the contents without changing the flags, although the nibbles are temporarily swapped.

The restore sequence swaps them back.

Restoring W with two `SWAPF` operations similarly avoids disturbing the STATUS flags after STATUS has been restored.

<a id="pclath-context"></a>
### PCLATH is conditional context

PCLATH does not need to be saved in every possible ISR.

It matters when main or ISR control flow depends on PCLATH state, such as computed-GOTO or page-sensitive operations.

A conservative reusable template may preserve it even when a smaller specific ISR could omit it.

The distinction is:

```text
required for every ISR? no
useful conservative reusable template choice? often yes
```

<a id="interrupt-latency"></a>
### Interrupt latency is part of response time

Interrupt service does not begin at the exact physical event time.

Figure 14-8 in the PIC16F883 data sheet documents asynchronous external INT timing and the 3-to-4 instruction-cycle latency relationship.

At 4 MHz:

```text
TCY = 1 us
3 cycles = 3 us
4 cycles = 4 us
```

That is latency to the documented interrupt-vector execution point, not the total time until a diagnostic pin toggles inside the handler.

A complete response-time path may include:

```text
event
-> hardware interrupt latency
-> vector instruction
-> context save
-> source checks
-> service action
-> diagnostic/output instruction
```

**Official visual reference:** See Figure 14-8, **INT Pin Interrupt Timing**, in the PIC16F883 data sheet.

Focus on the asynchronous input edge relative to internal clock phases and why service latency can differ by one instruction cycle.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="gate-example"></a>
### Worked example: pending external interrupt with global gate closed

Given:

```text
INTF = 1
INTE = 1
GIE  = 0
```

The event is pending and the source is individually enabled.

The CPU does not accept the interrupt while the global gate is closed.

If software later sets `GIE = 1` while `INTF` remains set, the pending request can be serviced.

<a id="peripheral-gate-example"></a>
### Worked example: peripheral source blocked by `PEIE`

Suppose:

```text
peripheral flag = 1
peripheral enable = 1
PEIE = 0
GIE  = 1
```

A peripheral source routed through `PEIE` is blocked.

Compare Timer0:

```text
T0IF = 1
T0IE = 1
GIE  = 1
```

Timer0 uses a direct path and does not require `PEIE`.

Use Figure 14-7 to verify which path applies.

<a id="latency-example"></a>
### Worked example: external INT latency at 4 MHz

**Known**

```text
FOSC = 4 MHz
TCY = 1 us
documented asynchronous INT latency = 3–4 instruction cycles
```

**Result**

```text
3–4 us to interrupt-vector execution
```

If the vector executes:

```asm
goto IsrHandler
```

that branch adds its own execution time before the first instruction in `IsrHandler`.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="interrupt-checklist"></a>
### Reusable interrupt checklist

For every new interrupt source, answer:

1. **Event:** What hardware condition causes the request?
2. **Flag:** Which bit records the event?
3. **Source enable:** Which bit permits this source to request service?
4. **Peripheral gate:** Does `PEIE` apply?
5. **Global gate:** Does `GIE` apply?
6. **Flag clearing:** How exactly is the source cleared?
7. **Pin/register routing:** What shared-pin or peripheral configuration matters?
8. **Context:** What processor/application state can the ISR alter?
9. **Latency:** What response-time limits matter?
10. **Evidence:** How will correct behavior be demonstrated?

<a id="simulator-hardware"></a>
### Verify in Simulator and on hardware

**Simulator**

- break at `0x0004` and the handler;
- establish expected flag/enable state;
- stimulate/set an interrupt condition;
- verify entry at the vector;
- step through source identification;
- verify context save/restore;
- verify the flag is cleared before `RETFIE`.

**Hardware**

- create a clean observable input event;
- use a separate output marker if practical;
- predict expected response timing;
- measure event-to-marker delay;
- separate hardware latency from ISR work and instrument resolution.

<a id="common-failures"></a>
### Common interrupt failures

- stale flag left set before enabling;
- source enabled but `GIE` disabled;
- peripheral source enabled but `PEIE` disabled;
- wrong edge selected;
- shared pin left in analog mode;
- flag not cleared correctly;
- `RETURN` used instead of `RETFIE`;
- W/STATUS corrupted by ISR;
- context temp stored in bank-dependent RAM before STATUS is saved;
- assuming output marker occurs exactly at the hardware event;
- assuming every interrupt source uses the same flag-clearing method.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Explain the difference between `INTF` and `INTE`.
2. Can `INTF` become set while `GIE = 0`? Why does that matter during setup?
3. What three important automatic actions occur when an interrupt is accepted?
4. Why must software examine flags after entering the one interrupt vector?
5. What does `RETFIE` do that `RETURN` does not?
6. Why can leaving `INTF = 1` before `RETFIE` cause immediate re-entry?
7. Does Timer0 require `PEIE`?
8. Why are common GPR addresses useful for `w_temp` and `status_temp`?
9. Why is `SWAPF STATUS,W` safer than `MOVF STATUS,W` for the classic context-save sequence?
10. At 4 MHz, what time range corresponds to 3–4 instruction cycles?
11. If another source flag becomes set while an ISR runs, can it remain pending until after `RETFIE`?
12. For an unfamiliar UART receive interrupt, list the questions you should answer before writing the ISR.

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. `INTF` records the external INT event; `INTE` controls whether that source can request service.
2. Yes. Flags can record events while masked. Clear stale requests before enabling service when appropriate.
3. `GIE` is cleared, the return PC is pushed onto the hardware stack, and PC is loaded with `0x0004`.
4. Many sources share one vector, so software identifies the active source(s) by their documented flags.
5. `RETFIE` restores the interrupt return path and re-enables global interrupt service.
6. The source remains pending; once `GIE` is restored, the same request can be accepted again.
7. No. Timer0 uses the direct `T0IF/T0IE/GIE` path.
8. They can be accessed without first changing bank selection, allowing original STATUS/bank state to be preserved safely.
9. `MOVF` can change the Z flag; `SWAPF` does not change STATUS flags.
10. 3–4 us.
11. Yes. Its flag can set while `GIE = 0`, then request service after interrupts are re-enabled.
12. Event, flag, source enable, `PEIE` applicability, `GIE`, clearing rule, pin/peripheral setup, context effects, latency, and verification method.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to talk through:

```text
event
-> flag
-> source enable
-> PEIE if applicable
-> GIE
-> automatic interrupt entry
-> vector at 0x0004
-> source identification
-> service
-> correct flag clearing
-> context restore
-> RETFIE
-> interrupted code resumes
```

You should also be able to:

- explain why flags and enables are separate;
- distinguish direct and peripheral interrupt paths;
- explain the `SWAPF` context pattern;
- include interrupt latency in response-time analysis;
- apply the same checklist to a source you have never used before.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Section 14.3 "Interrupts", Section 14.3.1 "RB0/INT Interrupt", Section 14.4 "Context Saving During Interrupts", Figure 14-7 "Interrupt Logic", and Figure 14-8 "INT Pin Interrupt Timing" — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: source routing, vector behavior, flag/enable paths, context-save guidance, and interrupt latency.

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A, Interrupts material — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: broader classic mid-range interrupt architecture.

- [PIC16F883 Interrupt Reference](https://github.com/rosstimo/pic_projects/blob/main/References/PIC16F883-Interrupts.md)
  - RCET shared implementation/reference checklist for external INT, context save/restore, timing, and verification.

- [RCET PIC16F883 starter template](https://github.com/rosstimo/pic_projects/blob/main/Template_Main.S)
  - Related RCET implementation reference for the current reusable vector/context skeleton. Device behavior remains governed by the Microchip documentation above.

[Back to top](#top) · [Topics index](README.md)
