# RCET3373 - PIC16F883 Interrupts and Context Saving - Self-Learning Guide

# What you should be able to do

After working through this guide, you should be able to:

- explain why an interrupt is different from continuously polling an input;
- distinguish an interrupt **flag** from an interrupt **enable**;
- trace a source through `GIE` and, when applicable, `PEIE`;
- configure and reason through the external `RB0/INT` interrupt;
- explain what the PIC16F883 hardware does automatically when an interrupt is serviced;
- identify the interrupt vector at `0004h` and explain why one vector can serve many sources;
- explain the difference between `RETURN` and `RETFIE`;
- explain why an interrupt flag must be cleared correctly;
- explain why W and STATUS may need to be saved and restored in software;
- explain why the common GPR region `0x70-0x7F` is useful for context saving;
- explain why the Microchip context-save sequence uses `SWAPF`;
- calculate the documented external-interrupt latency at the course's 4 MHz oscillator;
- apply the same event/flag/enable/clear model to a new interrupt-capable peripheral.

# 1. Start with the problem: polling consumes attention

Suppose a program needs to react when a button changes state.

One approach is polling:

```text
read button
button pressed?
  no -> continue main loop
  yes -> handle button
repeat
```

Polling is useful and sometimes completely appropriate. The limitation is that the processor only notices the event when execution reaches the polling code.

If the processor is busy doing something else, such as a long software delay, the response can be delayed.

An interrupt gives the hardware another way to request attention.

The main program can continue its normal work. When an enabled interrupt condition occurs, the processor temporarily redirects execution to the interrupt vector, software handles the event, and then execution returns to the interrupted code.

That does **not** mean interrupts are instantaneous. Interrupt entry has latency, and the interrupt service routine itself takes execution time.

# 2. Separate two questions: did the event happen, and may it interrupt?

A good interrupt mental model begins with two different bits.

## Flag

The **flag** records that the interrupt condition occurred.

Examples:

```text
INTF  external INT flag
T0IF  Timer0 overflow flag
```

## Enable

The **enable** controls whether that source is allowed to request interrupt service.

Examples:

```text
INTE  external INT enable
T0IE  Timer0 interrupt enable
```

Think of it this way:

```text
flag = the event happened
interrupt enable = software gives that source permission to interrupt
```

These bits are not the same thing.

The PIC16F883 data sheet explicitly states that individual interrupt flags can become set even when their corresponding interrupt is disabled or `GIE` is clear.

That means this state is possible:

```text
INTF = 1
INTE = 0
GIE  = 0
```

The event has been recorded, but no interrupt service occurs.

This is one reason stale flags matter during setup.

# 3. `GIE` is the final global gate

`GIE`, the Global Interrupt Enable bit, is `INTCON<7>`.

If `GIE = 0`, normal interrupt requests are globally blocked.

If `GIE = 1`, an otherwise enabled interrupt request can reach the CPU.

A simplified direct-source path looks like:

```text
source event
    |
source flag = 1
    |
source enable = 1
    |
GIE = 1
    |
CPU interrupt service
```

For example, the external INT path requires:

```text
INTF = 1
INTE = 1
GIE  = 1
```

# 4. `PEIE` adds another gate for peripheral interrupts

Many peripheral interrupt sources are stored in the `PIR1`/`PIR2` flag registers and controlled through `PIE1`/`PIE2` enable registers.

Those peripheral requests also pass through `PEIE`, the Peripheral Interrupt Enable bit.

A simplified peripheral path is:

```text
peripheral event
    |
peripheral flag = 1
    |
peripheral source enable = 1
    |
PEIE = 1
    |
GIE = 1
    |
CPU interrupt service
```

Not every interrupt uses `PEIE`.

Figure 14-7 shows several `INTCON` sources that feed the global path without the peripheral gate, including:

- external `INT`;
- PORTB interrupt-on-change;
- Timer0 overflow.

So do not memorize this incorrect rule:

> Every interrupt requires PEIE.

Instead, trace the source through Figure 14-7 or the appropriate peripheral documentation.

# 5. External interrupt on `RB0/AN12/INT`

The first concrete interrupt source is the external INT function on the shared pin:

```text
RB0 / AN12 / INT
```

The important interrupt-control bits are:

```text
OPTION_REG<6>  INTEDG  selects rising/falling edge
INTCON<4>      INTE    enables external INT
INTCON<1>      INTF    external INT flag
INTCON<7>      GIE     global interrupt enable
```

## Edge selection

The data sheet defines:

```text
INTEDG = 1 -> rising edge
INTEDG = 0 -> falling edge
```

For a button held high with a pull-up resistor and connected to ground when pressed:

```text
not pressed -> high
pressed     -> low
```

The press therefore creates a falling edge.

That makes:

```text
INTEDG = 0
```

a sensible choice for that particular circuit.

# 6. A safe setup sequence

The data sheet defines the bits and their behavior. A practical course setup pattern is:

```text
1. configure the shared pin for the intended input/function
2. select the interrupt edge
3. clear any stale interrupt flag
4. enable the individual source
5. enable global interrupts last
```

For external INT, conceptually:

```asm
; shared-pin/input setup as required by the application

; choose edge
bcf     OPTION_REG, INTEDG     ; falling edge example

; remove stale pending request
bcf     INTCON, INTF

; enable source
bsf     INTCON, INTE

; enable interrupts after setup is ready
bsf     INTCON, GIE
```

The exact bank-selection instructions depend on where execution is currently operating. `BANKSEL register` can ask pic-as to emit the necessary bank-selection code.

Example:

```asm
BANKSEL OPTION_REG
bcf     OPTION_REG, INTEDG
```

`BANKSEL` is convenient, but you should still understand the STATUS bank bits and inspect generated code when timing matters.

# 7. Shared pins still require the usual peripheral setup reasoning

The physical pin is not only called `INT`. It is shared:

```text
RB0 / AN12 / INT
```

That should trigger the same questions used for every new peripheral:

```text
What other functions share this pin?
Which SFRs control those functions?
What is the reset state?
Which settings matter for the intended electrical path?
```

For the course's button example, the associated PORTB configuration includes `TRISB` and `ANSELH` considerations in addition to the interrupt bits.

The clean engineering approach is to configure the intended digital-input behavior explicitly and verify it on the actual device.

Do not rely on remembering one isolated bit from a previous project.

# 8. What happens when the interrupt is actually serviced?

The PIC16F883 data sheet gives three important automatic actions when an interrupt is serviced:

```text
GIE is cleared
return address is pushed onto the hardware stack
PC is loaded with 0004h
```

That means the processor does **not** jump directly to a C-style function name or automatically identify the source for you.

The hardware takes execution to one address:

```text
0004h
```

This is the interrupt vector.

Compare it to Reset:

```text
0000h  Reset vector
0004h  Interrupt vector
```

In the current course template, main program code is deliberately placed beginning at `0008h` so those vector locations remain clear and easy to inspect.

# 9. A vector can contain a branch to the real handler

A typical course structure is:

```asm
PSECT resetVect,class=CODE,delta=2
ResetVector:
    goto Setup

PSECT isrVect,class=CODE,delta=2
InterruptVector:
    goto IsrHandler

PSECT code,class=CODE,delta=2
```

with linker placement such as:

```text
-presetVect=0000h
-pisrVect=0004h
-pcode=0008h
```

The vector itself only needs enough code to send execution to the real ISR handler.

This makes Program Memory easy to inspect:

```text
0000h -> goto Setup
0004h -> goto IsrHandler
0008h -> ordinary code area
```

# 10. Minimal external-interrupt trace

Suppose setup is complete and main code is running.

A button press creates the selected edge on INT.

The conceptual sequence is:

```text
MainLoop is executing
    |
valid INT edge occurs
    |
INTF becomes 1
    |
INTE and GIE allow service
    |
hardware clears GIE
    |
hardware pushes return PC
    |
PC = 0004h
    |
InterruptVector: goto IsrHandler
    |
ISR handles source
    |
INTF is cleared
    |
RETFIE
    |
return to interrupted code
```

Main code does not need to poll `RB0/INT` for this to happen.

# 11. Why clearing the flag matters

For external INT, `INTF` must be cleared in software.

Imagine this sequence:

```text
INTF = 1
ISR runs
INTF is never cleared
RETFIE executes
```

`RETFIE` sets `GIE` again.

The source is still pending because `INTF` is still 1 and `INTE` is still enabled.

The processor can immediately interrupt again.

A badly handled flag can therefore trap the program in repeated interrupt service.

The exact clearing mechanism is source-specific. For every new interrupt source, find the documentation that tells you:

```text
What sets the flag?
How is it cleared?
Does software clear the bit directly?
Does reading/writing another register clear it?
Is there a required sequence?
```

# 12. `RETURN` and `RETFIE` are not interchangeable

A normal subroutine return is:

```asm
return
```

A return from interrupt is:

```asm
retfie
```

Both use the return address from the hardware stack.

`RETFIE` also sets `GIE`, re-enabling unmasked interrupts.

For the PIC16F883, `RETFIE` takes two instruction cycles.

A useful model is:

```text
RETURN  -> restore execution address
RETFIE  -> restore execution address + enable interrupts
```

# 13. One vector can serve many sources

The PIC16F883 has many interrupt sources but only one normal interrupt vector.

After execution reaches the ISR, software must determine what caused the interrupt.

The data sheet describes polling the interrupt flags.

A conceptual structure is:

```text
IsrHandler
    |
check highest-priority source flag
    |
if not set -> check next flag
    |
if set -> handle that source
    |
LeaveIsr
```

The priority order is a software/application decision. The processor is not secretly choosing the order for you on this device.

Example idea:

```asm
IsrHandler:
    ; save context

    btfsc   INTCON, INTF
    goto    HandleExternalInt

    btfsc   INTCON, T0IF
    goto    HandleTimer0

    goto    LeaveIsr
```

The actual tests should also be consistent with which sources are enabled and relevant to the application.

# 14. What if another flag becomes set while the ISR is running?

During normal interrupt service, `GIE` is clear.

Another interrupt condition can still set its flag.

Suppose:

```text
external INT caused the current ISR
Timer0 overflows while the ISR is running
T0IF becomes 1
```

The CPU does not normally interrupt the ISR again because `GIE = 0`.

If the ISR handles external INT and returns with `T0IF` still pending, `RETFIE` sets `GIE`.

The Timer0 request can then cause another interrupt entry.

This means an ISR does not necessarily need to handle every possible pending source in one pass.

How much work to do per ISR entry is a design decision.

# 15. Use one cleanup/exit path

A useful course structure is to funnel source-specific handlers to one exit point:

```text
IsrHandler
    |
source checks
    |
source-specific handler
    |
LeaveIsr
    |
restore context
    |
RETFIE
```

This gives one place to verify that:

- context is restored;
- no cleanup step is skipped;
- there is only one normal `RETFIE` path;
- source-specific code does not accidentally return as though it were a normal subroutine.

This is not the only possible ISR structure, but it is easy to audit and appropriate for the course.

# 16. The hardware stack does not save W and STATUS

Interrupt entry automatically saves the return program address.

It does **not** automatically preserve the entire processor state.

Imagine main code is in the middle of this kind of operation:

```text
W contains a value needed by the next instruction
STATUS contains the current bank selection
Z or C contains the result of a previous operation
```

Then an interrupt occurs.

If the ISR changes W, changes banks, or performs arithmetic, main code may resume with different state than it had before the interrupt.

That can create a bug that depends on exactly when the interrupt occurred.

That is why context saving matters.

# 17. Common RAM `0x70-0x7F` is useful for ISR temporaries

The PIC16F883 data-memory organization provides a common GPR region:

```text
0x70 through 0x7F
```

These locations are accessible across the register banks without first changing the bank selection.

That makes them good places for temporary context-storage registers such as:

```asm
w_temp          EQU 0x70
status_temp     EQU 0x71
pclath_temp     EQU 0x72
```

Why is this helpful?

The interrupt can happen while main code is in any bank.

If the first ISR instruction had to switch banks before it could save STATUS, that bank switch could destroy part of the state you were trying to preserve.

Common RAM lets you store the temporary data without solving a bank-selection problem first.

# 18. Study the Microchip context-save pattern

The PIC16F883 data sheet shows a context-save sequence built around this idea:

```asm
movwf   W_TEMP
swapf   STATUS,W
movwf   STATUS_TEMP

; ISR work

swapf   STATUS_TEMP,W
movwf   STATUS
swapf   W_TEMP,F
swapf   W_TEMP,W
```

At first glance the `SWAPF` instructions may look unnecessarily strange.

They are there for a reason.

# 19. Instruction status effects explain the `SWAPF` sequence

For the PIC16F883:

```text
MOVF   affects Z
MOVWF  affects no STATUS flags
MOVLW  affects no STATUS flags
SWAPF  affects no STATUS flags
```

This means:

```asm
movf STATUS,W
```

would be a bad way to save STATUS before it is preserved, because `MOVF` can change `Z`.

Instead:

```asm
swapf STATUS,W
```

copies the STATUS byte into W while swapping its upper and lower nibbles, but without modifying the STATUS flags.

The swapped byte can then be stored safely.

The nibble swap is temporary bookkeeping. The later restore sequence swaps it back.

# 20. Why does restoring W require two `SWAPF` instructions?

Near the end of the ISR, STATUS has already been restored.

You do not want the instruction used to restore W to change the newly restored flags.

The documented sequence is:

```asm
swapf   W_TEMP,F
swapf   W_TEMP,W
```

Suppose the original W value was:

```text
W = 0x3A
```

After saving, `W_TEMP` contains:

```text
0x3A
```

First:

```asm
swapf W_TEMP,F
```

changes the stored temporary value to:

```text
0xA3
```

without affecting STATUS flags.

Then:

```asm
swapf W_TEMP,W
```

swaps the nibbles again while writing the result to W:

```text
W = 0x3A
```

Again, no STATUS flags are disturbed.

# 21. What about PCLATH?

The PIC16F883 data sheet says PCLATH does not have to be saved for every interrupt situation.

However, if main code and ISR code use computed GOTOs or otherwise depend on PCLATH state, it may need preservation.

The current `rosstimo/pic_projects/Template_Main.S` uses the conservative pattern:

```asm
movf    PCLATH,W
movwf   pclath_temp
```

and later:

```asm
movf    pclath_temp,W
movwf   PCLATH
```

This adds some ISR overhead, but it creates a reusable template that is safer when later projects use computed branches.

The important distinction is:

```text
device requirement for every ISR? no
conservative reusable template choice? yes
```

# 22. Full course-style ISR skeleton

A reusable structure can look like:

```asm
PSECT isrVect,class=CODE,delta=2
InterruptVector:
    goto IsrHandler

PSECT code,class=CODE,delta=2

IsrHandler:
    movwf   w_temp
    swapf   STATUS,W
    movwf   status_temp
    movf    PCLATH,W
    movwf   pclath_temp

    ; identify source
    btfsc   INTCON, INTF
    goto    ExternalIntIsr

    btfsc   INTCON, T0IF
    goto    Timer0Isr

    goto    LeaveIsr

ExternalIntIsr:
    ; handle external event
    bcf     INTCON, INTF
    goto    LeaveIsr

Timer0Isr:
    ; handle Timer0 event
    bcf     INTCON, T0IF
    goto    LeaveIsr

LeaveIsr:
    movf    pclath_temp,W
    movwf   PCLATH
    swapf   status_temp,W
    movwf   STATUS
    swapf   w_temp,F
    swapf   w_temp,W
    retfie
```

This is a structure to understand and adapt, not a block to copy blindly into every project.

For each source, verify its actual flag behavior and required clearing sequence from the relevant documentation.

# 23. Interrupt latency is part of timing

Figure 14-8 documents asynchronous external-event interrupt latency of:

```text
3 or 4 instruction cycles
```

The exact value depends on when the asynchronous event arrives relative to internal instruction timing.

For the course oscillator:

```text
FOSC = 4 MHz
TCY = FOSC / 4 = 1 MHz instruction-cycle rate
TCY = 1 us
```

Therefore the documented external INT latency is approximately:

```text
3 cycles -> 3 us
4 cycles -> 4 us
```

This is the time to reach interrupt-vector execution under the documented timing case. It does not include all of the ISR code that follows.

If an application has strict timing requirements, include:

```text
interrupt latency
+ vector branch
+ context save
+ source identification
+ actual service work
```

in the response-time analysis.

# 24. Timer0 fits the same model

Timer0 provides another interrupt source.

When the 8-bit timer rolls over:

```text
0xFF -> 0x00
```

`T0IF` is set.

The important control path is:

```text
Timer0 rollover
-> T0IF
-> T0IE
-> GIE
-> interrupt service
```

`PEIE` is not in this path.

That is visible in Figure 14-7.

The same reasoning process will work later for ADC, serial communication, and other peripherals, although their flags, enable bits, and clearing mechanisms differ.

# 25. A reusable interrupt checklist

For every new interrupt source, answer these questions from the official documentation.

## Event

What hardware condition causes the request?

## Flag

Which bit records the event?

## Source enable

Which bit allows this source to request service?

## Peripheral gate

Does `PEIE` apply?

## Global gate

Does `GIE` apply?

## Flag clearing

How is the flag cleared?

## Pin/register routing

If hardware pins are involved, what shared functions and configuration registers matter?

## Context

What CPU/SFR state can the ISR alter that main code may depend on?

## Timing

What interrupt latency and service time matter to the application?

## Evidence

How will you prove that the interrupt occurs and is serviced correctly?

# Worked example 1: should this external event interrupt the CPU?

Given:

```text
INTF = 1
INTE = 1
GIE  = 0
```

Does the CPU service the interrupt?

No.

The event is pending because `INTF = 1`, and the source is enabled because `INTE = 1`, but the global gate is closed.

If software later sets `GIE = 1` while the flag remains set, the pending request can be serviced.

# Worked example 2: peripheral versus direct path

Suppose:

```text
ADC interrupt flag = 1
ADC interrupt enable = 1
PEIE = 0
GIE  = 1
```

The peripheral request is blocked by `PEIE`.

Now compare Timer0:

```text
T0IF = 1
T0IE = 1
GIE  = 1
```

Timer0 is a direct `INTCON` path, so `PEIE` is not required.

# Worked example 3: latency at 4 MHz

Known:

```text
FOSC = 4 MHz
TCY = 4 / FOSC = 1 us
external asynchronous interrupt latency = 3-4 instruction cycles
```

Therefore:

```text
minimum documented latency = 3 us
maximum documented latency = 4 us
```

If the vector contains:

```asm
goto IsrHandler
```

that branch itself requires additional execution time before the handler's first instruction executes.

# Practice problems

1. Explain the difference between `INTF` and `INTE`.

2. Can `INTF` become set while `GIE = 0`? What does that imply for initialization?

3. What three actions does the PIC16F883 perform automatically when an interrupt is serviced?

4. Why does an ISR on this device need to examine flags?

5. What does `RETFIE` do that `RETURN` does not?

6. Why can leaving `INTF = 1` before `RETFIE` cause trouble?

7. Does Timer0 require `PEIE`? Explain using Figure 14-7.

8. Why are `0x70-0x7F` useful for `w_temp` and `status_temp`?

9. Why is `SWAPF STATUS,W` safer than `MOVF STATUS,W` while saving context?

10. At 4 MHz, what time range corresponds to a 3-4 instruction-cycle external interrupt latency?

11. Suppose an ADC flag becomes set while the ISR is servicing external INT. What can happen after `RETFIE` if the ADC interrupt remains enabled and pending?

12. For an unfamiliar UART receive interrupt, what questions should you answer before writing the ISR?

# Answer key

1. `INTF` records the external INT event. `INTE` controls whether that source is enabled to request interrupt service.

2. Yes. Individual flags can set while masked or while `GIE` is clear. Clear stale flags before enabling interrupts when appropriate.

3. `GIE` is cleared, the return address is pushed to the hardware stack, and `PC` is loaded with `0004h`.

4. There is one normal interrupt vector for many sources, so software identifies the source by inspecting flags.

5. `RETFIE` restores the return address and sets `GIE`.

6. The request can remain pending. When `RETFIE` sets `GIE`, the processor may immediately service the same interrupt again.

7. No. Timer0's `T0IF/T0IE` path feeds the global logic without `PEIE`.

8. They are common GPR addresses accessible across banks, so context can be saved without first changing the bank selection.

9. `MOVF` affects Z. `SWAPF` does not affect STATUS flags, so it can move the STATUS contents into W without changing the flags being saved.

10. `3-4 us`.

11. Its flag can remain pending while `GIE` is clear. After `RETFIE` re-enables interrupts, another interrupt entry can occur to service the pending ADC source.

12. At minimum: event, flag, source enable, `PEIE`, `GIE`, flag-clearing method, associated pin/SFR configuration, context effects, latency/timing, and verification method.

# What to be able to explain without notes

You should be able to talk through this complete path:

```text
event
-> flag
-> enable logic
-> hardware clears GIE
-> return PC goes to stack
-> PC becomes 0004h
-> ISR identifies source
-> service source
-> clear flag correctly
-> restore context
-> RETFIE
-> resume interrupted code
```

If you can apply that path to a source you have never used before, you understand the interrupt architecture rather than merely remembering one example.

# References used to build this guide

- Microchip, **PIC16F882/883/884/886/887 Data Sheet**, DS40001291H.
- Section 14.3, Interrupts.
- Figure 14-7, Interrupt Logic.
- Figure 14-8, INT Pin Interrupt Timing.
- Section 14.3.1, RB0/INT Interrupt.
- Section 14.4, Context Saving During Interrupts.
- PIC16F883 instruction descriptions for `MOVF`, `MOVWF`, `SWAPF`, and `RETFIE`.
- RCET3373 W03D04 teaching record and corrected accuracy review.
- `rosstimo/pic_projects/Template_Main.S` for the current course-adjacent ISR skeleton.
