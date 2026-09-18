# RCET 3373 Computer Memory - Self-Learning Guide

*RCET 3373 student guide: concepts, worked examples, practice, and answer key*

**Fall 2026 • Tim Rossiter • Idaho State University**

Use this guide to learn or review the memory lesson without the lecture. The goal is not to memorize a list of memory types. You should be able to reason from the address/data/control interface, explain why different storage cells behave differently, and solve basic memory-organization problems.

# What you should be able to do

- Read notation such as 4K × 8 and determine address lines, data lines, and total storage.

- Explain a memory read and write as bus transactions.

- Explain the purpose of chip select, output enable, write enable, and high-impedance outputs.

- Compare SRAM and DRAM from the physical storage cell.

- Explain why DRAM requires refresh and why it multiplexes addresses.

- Distinguish PROM, EPROM, EEPROM, and Flash by how their contents can be changed.

- Recognize two different ways to expand a memory system.

# 1. Start with the system, not the memory type

A processor does not ask memory for "the variable" or "the instruction." At the hardware interface it supplies a binary address that identifies a location. It then uses control signals to say whether the operation is a read or a write. The data bus carries the word stored at that location.

**Address bus:** Identifies which location is being accessed.

**Data bus:** Carries the contents. It is normally bidirectional.

**Control bus:** Carries read/write, enable, and timing-related signals.

| **Key idea:** During a read, memory drives the data bus. During a write, the CPU or other bus master drives the data bus. The address still comes from the device requesting the memory operation. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 2. Memory capacity and addresses

Memory is commonly described as number of words × bits per word. For example, 4K × 8 means 4096 addressable locations, with eight bits stored at each location. Because 4096 = 2^12, the device needs 12 address bits. Because each word is eight bits, it needs eight data bits for a parallel interface.

<img src="student-guide-media/media/image1.png" style="width:3in;height:2.59144in" />

*Figure: 16 locations, each containing an 8-bit word. Source: Kleitz, Chapter 16, Figure 16-1.*

## Worked example A: 32K × 8

1.  32K = 32 × 1024 = 32768 locations.

2.  32768 = 2^15, so 15 address lines are required.

3.  The word width is eight bits, so there are eight data bits.

4.  32768 × 8 bits = 262144 bits = 32768 bytes = 32 KiB.

## Worked example B: 8K × 16

5.  8K = 8192 = 2^13 locations → 13 address lines.

6.  Each location contains 16 bits → 16 data bits.

7.  16 bits = two bytes per word, so 8192 × 2 = 16384 bytes = 16 KiB.

| **Rule:** N address lines can select 2^N unique locations. The word width determines how many data bits are transferred for each selected address. |
|----------------------------------------------------------------------------------------------------------------------------------------------------|

# 3. Read and write transactions

## Read

8.  The CPU places the desired binary address on the address bus.

9.  The CPU activates the appropriate chip-select/read/output-enable controls.

10. The memory decodes the address and selects one internal word.

11. After the device access time, valid data appear on the data bus.

12. The CPU captures the data while it is valid.

## Write

13. The CPU places the destination address on the address bus.

14. The CPU places the new word on the data bus.

15. The write and chip-select controls are activated.

16. The memory stores the input data in the selected location.

17. Address and data remain valid long enough to satisfy setup and hold requirements.

| **High impedance:** When a memory chip is not selected, its output drivers normally disconnect from the shared data bus. This Hi-Z state prevents multiple devices from fighting over the same wires. |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 4. Nonvolatile memory: keeping information without power

The ROM family is best understood as a progression in how easily stored information can be created, erased, and rewritten.

| **Type** | **How contents are created/changed** | **Practical idea**                                  |
|----------|--------------------------------------|-----------------------------------------------------|
| MROM     | Programmed during manufacturing      | Not normally changed                                |
| PROM     | Programmed by user once              | No erase                                            |
| EPROM    | User programmable                    | Whole chip erased with UV light                     |
| EEPROM   | Electrical in-circuit rewrite        | Fine-grained updates                                |
| Flash    | Electrical in-circuit rewrite        | Erase/program organized in larger blocks or sectors |

In embedded systems, nonvolatile memory is valuable for program code, boot information, calibration data, configuration, and values that must survive a power cycle. Different jobs may use different nonvolatile technologies.

# 5. SRAM: a bit stored as a bistable state

Static RAM uses a latch-like transistor circuit to store each bit. Once a 0 or 1 is written, the cross-coupled circuit reinforces that state as long as power is present. No periodic refresh is required. That is what static means.

<img src="student-guide-media/media/image2.png" style="width:3.4in;height:3.13569in" />

*Simplified SRAM cell. Source: Kleitz, Chapter 16, Figure 16-5(c).*

| **Do not confuse the terms:** SRAM is static but volatile. It does not need refresh while powered, but ordinary SRAM loses its contents when power is removed. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|

# 6. SRAM timing

A valid address does not instantly produce valid output data. Internal decoding and switching take time. Data sheets show this with timing diagrams. During a read, access time describes the delay between a valid request and valid output data. During a write, setup and hold intervals tell you how long address and data must remain stable around the active write event.

<img src="student-guide-media/media/image3.png" style="width:4.25in;height:6.13636in" />

*Typical SRAM read/write timing. Source: Tocci, Chapter 12, Figure 12-22.*

# 7. DRAM: a bit stored as capacitor charge

Dynamic RAM uses a much smaller storage cell: an access transistor and a capacitor. A charged capacitor represents one logic state and a discharged capacitor represents the other. The simplicity of the cell allows high density and low cost per bit, but the charge leaks away.

<img src="student-guide-media/media/image4.png" style="width:4in;height:2.24422in" />

*Simplified DRAM cell. Source: Kleitz, Chapter 16, Figure 16-9.*

## Refresh

Because capacitor voltage decays, DRAM rows must be refreshed periodically. Refresh reads/restores stored state before the voltage becomes too small to interpret reliably. Modern memory controllers schedule this maintenance automatically.

## Address multiplexing

Large DRAMs would require many package pins if every address bit had a dedicated pin. A common solution is to reuse the address pins. First the row portion of the address is presented and latched. Then the column portion is presented on the same physical pins and latched. RAS and CAS are control strobes associated with these two phases. The tradeoff is fewer pins in exchange for more sequencing and latency.

<img src="student-guide-media/media/image5.png" style="width:3in;height:4.92697in" />

*Address multiplexing concept. Source: Tocci, Chapter 12, Figure 12-28.*

# 8. SRAM and DRAM comparison

| **Property**           | **SRAM**                  | **DRAM**                      |
|------------------------|---------------------------|-------------------------------|
| Cell                   | Bistable/latch-like       | Capacitor + transistor        |
| Refresh                | No                        | Yes                           |
| Density                | Lower                     | Higher                        |
| Cost per bit           | Higher                    | Lower                         |
| General speed tendency | Faster                    | More latency/control overhead |
| Typical use            | Cache, small fast buffers | Large main memory             |

# 9. Expanding memory

## Increase word width

Suppose you need 16 × 8 memory but only have 16 × 4 chips. Use two chips in parallel. Both receive the same address and control signals. One supplies four data bits and the other supplies the other four. The number of addresses stays 16; the word becomes wider.

## Increase the number of locations

Suppose you need 32 × 8 memory but only have 16 × 8 chips. Use two banks. The lower address bits select a location inside both chips, while an additional high-order address bit or decoder selects which chip is active. The word width stays eight bits; the number of addresses increases.

| **Ask yourself first:** Am I trying to add bits to each word, or am I trying to add more addressable words? That determines the wiring strategy. |
|--------------------------------------------------------------------------------------------------------------------------------------------------|

# 10. What this means for a microcontroller

When you open the PIC data sheet, the same ideas will appear under device-specific names and memory maps. Look for program memory that stores instructions, data memory used for working values, register and special-function-register regions, and any nonvolatile data storage. Treat every map as an address problem: what addresses exist, what lives at them, and what kind of read/write behavior does each region have?

# Practice problems

1. A memory is organized as 64K × 8. How many address lines and data lines are required? What is the total capacity in bytes?

2. A memory is organized as 2K × 16. How many address lines are required? What is its capacity in bytes?

3. During a memory read, which device drives the data bus?

4. Why do memory outputs use a high-impedance state?

5. Explain why SRAM is called static even though it is volatile.

6. What physical element in a DRAM cell makes refresh necessary?

7. Why are DRAM addresses often multiplexed?

8. What do RAS and CAS accomplish conceptually?

9. What is the practical difference between EEPROM and Flash emphasized in this lesson?

10. You have 1K × 4 RAM chips. How many are required to make 1K × 8? Describe the address/control wiring.

11. You have 1K × 8 RAM chips. How many are required to make 4K × 8? What additional function is needed to select the active chip?

12. A DRAM array has 256 rows and 256 columns of one-bit cells. How many total cells does it contain, and how many address bits identify one cell?

# Answer key

1. 64K = 2^16 → 16 address lines; eight data lines; 65,536 bytes = 64 KiB.

2. 2K = 2048 = 2^11 → 11 address lines. Each word is two bytes → 4096 bytes = 4 KiB.

3. The selected memory device drives the data bus during a read.

4. Hi-Z disconnects inactive outputs so multiple devices can share the same bus without contention.

5. Its cell maintains state without periodic refresh while power is applied. Removing power still destroys the state.

6. The storage capacitor leaks charge with time.

7. To reduce package pin count and board routing by reusing the same pins for row and column portions of the address.

8. They latch/select the row and column address phases in the DRAM access sequence.

9. EEPROM supports finer-grained electrical updates; Flash is organized around larger erase/program units such as blocks or sectors.

10. Two chips. Both share the same address and control lines. One connects to four data bits and the other to the other four.

11. Four chips. Lower address lines are shared; high-order address bits feed chip-select decoding so only one bank is active.

12. 256 × 256 = 65,536 cells = 64K × 1. Since 65,536 = 2^16, a full cell address is 16 bits.

# What to be able to explain without notes

- A complete read transaction

- A complete write transaction

- Why 4K locations require 12 address bits

- Why SRAM is static but volatile

- Why DRAM refresh exists

- Why DRAM multiplexes addresses

- Why tri-state outputs matter

- The difference between word-width expansion and capacity expansion

# References used to build this guide

- Kleitz, Digital Electronics: A Practical Approach, Chapter 16 (supplied course reference).

- Tocci et al., Memory Devices, Chapter 12 (supplied course reference).

- Maini, Digital Electronics: Principles, Devices and Applications, Chapter 15 (supplied course reference).
