<a id="top"></a>

# RCET 3373 — Processor Datapaths, Cycles, and Pipelining

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

An instruction set tells software what operations a processor can perform.

A **microarchitecture** determines how a particular processor implementation performs those operations internally.

Two processors can implement the same instruction-set architecture (ISA) using different:

- datapaths;
- clock rates;
- numbers of cycles per instruction;
- pipeline depths;
- memory systems;
- branch strategies.

This distinction becomes important when moving from the simple PIC16F883 core to modern Arm, RISC-V, ESP, or other industrial microcontrollers.

The goal is not to design a production CPU. The goal is to be able to look at an unfamiliar processor block diagram and reason about where data moves, what work occurs in each stage, and why branches, memory, and hazards affect performance.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- distinguish ISA from microarchitecture;
- identify common datapath elements such as PC, register file, ALU, memory interfaces, multiplexers, and control;
- trace a simple arithmetic, load/store, and branch instruction through a conceptual datapath;
- compare single-cycle and multi-cycle processor implementations;
- explain why a single-cycle clock is limited by the longest instruction path;
- explain how a multi-cycle design reuses hardware across shorter cycles;
- explain the throughput goal of pipelining;
- distinguish latency from throughput;
- identify structural, data, and control hazards conceptually;
- explain branch flush/stall behavior;
- connect the PIC16F883's simple fetch/execute pipeline to deeper pipelines used by other processors.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [PIC16F883 Architecture](pic16f883-architecture.md#architecture-overview);
- [Address, Data, and Control Buses](buses-adc-dac.md#core-model);
- [Instruction Timing and Software Delays](instruction-timing-software-delays.md#pipeline).

Related topics:

- [Embedded C on the PIC16F883](embedded-c-pic16f883.md#build-path);
- future MCU/toolchain exploration, where the same ISA can appear in many different vendor implementations.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="isa-microarchitecture"></a>
### ISA versus microarchitecture

**Instruction Set Architecture (ISA)**

The software-visible contract, including concepts such as:

- instructions;
- registers;
- data types;
- addressing modes;
- exceptions/privilege where applicable.

**Microarchitecture**

The internal organization used to implement the ISA.

A useful rule is:

> ISA says what software sees; microarchitecture says how a particular processor makes it happen.

RISC-V is a good example because the ISA specification intentionally avoids requiring one pipeline or implementation style.

<a id="datapath"></a>
### Conceptual datapath

A simple processor datapath may contain:

```text
PC
 ↓
instruction memory / fetch
 ↓
instruction decode + control
 ↓
register file
 ↓
ALU
 ↓
data memory or result selection
 ↓
register write-back
```

Branches feed a new value back to the PC.

Multiplexers choose among possible inputs, and control logic determines which path is active for each instruction.

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="single-cycle"></a>
### Single-cycle implementation

In a conceptual single-cycle processor, one instruction completes all required datapath work in one clock cycle.

A load might require:

```text
fetch instruction
-> decode/read register
-> calculate address
-> read data memory
-> write register result
```

Because that entire path must fit in one cycle, the clock period must be long enough for the slowest supported instruction path.

Shorter instructions finish their useful work earlier but still occupy the full cycle.

**Visual reference:** MIT 6.375, [Multicycle Processors and Pipeline Processors](https://csg.csail.mit.edu/6.375/6_375_2019_www/handouts/lectures/L11-MulticycleProcessorsAndPipelinedProcessors.pdf), begins with a single-cycle datapath and the long critical path that limits clock speed.

<a id="multi-cycle"></a>
### Multi-cycle implementation

A multi-cycle processor divides an instruction into several clocked phases.

Conceptually:

```text
cycle 1: fetch
cycle 2: decode / register read
cycle 3: execute / address calculation
cycle 4: memory access when needed
cycle 5: write-back when needed
```

Different instructions may require different numbers of cycles.

Advantages can include:

- shorter clock period;
- hardware reused across phases;
- less pressure to complete every function in one long combinational path.

The tradeoff is that one instruction can take several clock cycles.

<a id="pipelining"></a>
### Pipelining overlaps different instructions

A pipeline inserts state between stages so several instructions can be in progress at once.

A conceptual five-stage example:

```text
IF -> ID -> EX -> MEM -> WB
```

After filling, an ideal pipeline might complete approximately one instruction per cycle even though each individual instruction spends several cycles moving through the stages.

This separates:

- **latency:** how long one instruction takes from entry to completion;
- **throughput:** how often completed instructions can emerge.

Pipelining mainly targets throughput.

<a id="pipeline-example"></a>
### PIC16F883 is a small pipeline example

The PIC16F883 uses a simple fetch/execute overlap.

That is why straight-line instructions often complete in one instruction cycle once the pipeline is active, while a control-flow change such as `GOTO` requires an extra cycle to discard/refill the sequentially fetched path.

The same basic idea scales to deeper pipelines:

> Work from different instructions overlaps, so a changed control path can invalidate work already in progress.

<a id="hazards"></a>
### Hazards prevent ideal overlap

Three useful hazard categories are:

**Structural hazard**

Two operations need the same hardware resource at the same time.

**Data hazard**

A later instruction needs a result that an earlier instruction has not produced yet.

**Control hazard**

The next correct instruction address is uncertain because of a branch/jump/exception.

Processors can respond using techniques such as:

- stalls;
- forwarding/bypassing;
- duplicated resources;
- branch prediction;
- pipeline flush/restart.

The exact mechanisms depend on the processor.

<a id="branch-flush"></a>
### Branches can waste work already in the pipeline

Suppose several sequential instructions have already been fetched/decoded when a branch is determined to be taken.

Those instructions belong to the wrong path.

A simple processor can:

```text
discard wrong-path work
-> redirect PC
-> refill pipeline
```

That is a deeper version of the same control-flow penalty already observed on the PIC16F883.

<a id="memory-architecture"></a>
### Memory organization affects the datapath

A Harvard-style system has separate instruction and data paths/memories at some architectural level.

A von Neumann-style system uses a shared memory/path model.

Real modern MCUs can mix these ideas through separate buses, caches, unified address spaces, or multiple memory regions.

Do not classify a modern chip only from one marketing label. Inspect the actual architecture/bus diagram.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="single-cycle-critical-path"></a>
### Worked example: why one slow instruction limits a single-cycle clock

Suppose a conceptual processor has:

```text
fast ALU instruction path = 8 ns
slow load instruction path = 14 ns
```

In a one-cycle-per-instruction implementation, the clock period must accommodate the slowest supported path:

```text
Tclock >= 14 ns
```

Even the 8 ns ALU instruction occupies that 14 ns cycle.

<a id="pipeline-throughput"></a>
### Worked example: latency versus throughput

Suppose a five-stage pipeline uses a 2 ns clock.

Ignoring hazards:

```text
one instruction latency ≈ 5 × 2 ns = 10 ns
```

But after pipeline fill, completions can occur approximately every 2 ns.

That does **not** mean each instruction's latency is 2 ns.

<a id="data-hazard-example"></a>
### Worked example: data dependency

```text
ADD r1, r2, r3
SUB r4, r1, r5
```

The second instruction needs the new value of `r1`.

If `SUB` reaches the point where it needs `r1` before `ADD` has made that value available, the processor has a data hazard.

Possible microarchitectural responses include forwarding or stalling.

The ISA program remains the same; the hardware implementation determines how the dependency is resolved.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="architecture-reading"></a>
### Reading an unfamiliar CPU block diagram

Ask:

1. Where is the PC?
2. Where are instructions fetched?
3. Where are registers stored?
4. Where is the ALU?
5. How are loads/stores connected to memory/peripherals?
6. What state separates pipeline stages?
7. Where is branch target/decision logic?
8. What happens to wrong-path instructions?
9. Are instruction and data paths separate?
10. What caches/buffers/buses exist?
11. Which details belong to the ISA versus this implementation?

<a id="performance-caution"></a>
### Do not infer performance from clock frequency alone

Two MCUs running at the same MHz can execute a workload differently because of:

- pipeline depth;
- instruction set;
- memory wait states;
- caches;
- branch behavior;
- peripheral/DMA support;
- compiler output;
- bus contention.

Clock rate is one input to performance, not a complete performance metric.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Distinguish ISA from microarchitecture.
2. Name five common datapath elements.
3. Why must a single-cycle processor's clock accommodate the slowest instruction path?
4. What is gained by dividing an instruction into several shorter cycles?
5. What does pipelining try to improve primarily: instruction latency or throughput?
6. Distinguish a data hazard from a control hazard.
7. Why can a taken branch require a pipeline flush?
8. How is the PIC16F883 two-cycle `GOTO` behavior related to pipelining?
9. Can two processors implementing the same ISA have different pipeline depths?
10. Why is MHz alone an incomplete performance comparison?
11. Give an example of a structural hazard.
12. In a five-stage 2 ns pipeline, what is the approximate no-hazard latency of one instruction and the ideal completion interval after fill?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. ISA is the software-visible instruction/register contract; microarchitecture is the internal implementation.
2. Examples: PC, instruction memory/fetch, decoder/control, register file, ALU, data memory interface, multiplexers, pipeline registers.
3. Every supported instruction must finish all required one-cycle work before the next state update.
4. The clock period can be shorter and hardware can be reused across phases; individual instructions take more cycles.
5. Throughput.
6. Data hazard: later instruction needs unfinished result; control hazard: correct next instruction address/path is not yet known.
7. Already fetched/decoded sequential instructions may belong to the wrong path.
8. The sequentially prefetched instruction is invalid after the PC changes, so pipeline work is discarded/refilled.
9. Yes. ISA generally does not require one pipeline implementation.
10. Other architecture, memory, branch, compiler, and peripheral factors change executed work per unit time.
11. Example: instruction fetch and data load both need one single-ported memory in the same cycle.
12. Latency ≈ 10 ns; ideal completion interval ≈ 2 ns after fill.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- distinguish ISA and microarchitecture;
- trace a conceptual datapath;
- compare single-cycle, multi-cycle, and pipelined designs;
- distinguish latency and throughput;
- classify structural/data/control hazards;
- explain stalls/forwarding/flushes conceptually;
- connect PIC16F883 control-flow timing to the broader pipeline model;
- approach an unfamiliar processor block diagram with a consistent checklist.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PICmicro Mid-Range MCU Family Reference Manual*, DS33023A, Architecture/CPU/Instruction Flow material — https://ww1.microchip.com/downloads/en/DeviceDoc/33023A.pdf
  - Used for: PIC classic fetch/execute pipeline and processor-architecture context.

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: PIC16F883-specific CPU/program-memory/instruction timing context.

- MIT CSAIL 6.375, *Multicycle Processors and Pipeline Processors* — https://csg.csail.mit.edu/6.375/6_375_2019_www/handouts/lectures/L11-MulticycleProcessorsAndPipelinedProcessors.pdf
  - Used for: educational single-cycle, multi-cycle, and pipelined processor models and timing tradeoffs.

- MIT 6.175, *RISC-V Introduction — Multi-Cycle and Two-Stage Pipeline* — https://csg.csail.mit.edu/6.175/labs/lab5-riscv-intro.html
  - Used for: concrete educational comparison of one-cycle, multi-cycle, and pipelined RISC-V implementations.

- Arm, *Cortex-M0 Processor Datasheet* — https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Processor%20Datasheets/Arm_Cortex-M0_Processor_Datasheet.pdf
  - Used for: industry example of a modern embedded processor pipeline and system integration.

- RISC-V International, *The RISC-V Instruction Set Manual, Volume I: Unprivileged ISA* — https://docs.riscv.org/reference/isa/
  - Used for: example of an ISA specification deliberately separated from a required microarchitecture.

[Back to top](#top) · [Topics index](README.md)
