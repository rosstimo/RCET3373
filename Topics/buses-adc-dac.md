# RCET 3373 - Buses, ADC, and DAC Self-Learning Guide

## What you should be able to do

After working through this guide, you should be able to:

- explain address, data, and control roles in a transaction;
- determine basic address-space size from address width;
- distinguish temporary and persistent memory roles at a system level;
- calculate ideal ADC/DAC resolution using the course convention;
- distinguish resolution from accuracy;
- explain why finite digital codes cannot preserve arbitrary analog detail.

For deeper memory coverage, see [Memory Systems](memory-systems.md).

## Address, data, and control

A useful mental model is:

```text
address = where
data    = what
control = what operation / when
```

A conceptual memory read looks like this:

1. the processor presents an address;
2. control indicates a read;
3. the selected device presents data;
4. the processor captures the data when it is valid.

A write uses the same three roles, but data flows toward the selected destination.

A memory-mapped peripheral register uses an address too. Reading or writing that address can interact with hardware instead of ordinary RAM.

## Address width determines the number of locations

If an address has `N` independent bits, it can select:

```text
2^N locations
```

Examples:

- 4 address bits -> 16 locations;
- 8 address bits -> 256 locations.

The width of each stored location is a separate property.

## Memory roles in an embedded system

At a high level:

- RAM is used for temporary working variables and state;
- EEPROM is useful for data that should survive power loss and can be changed occasionally;
- Flash is nonvolatile and is commonly used for program or block-oriented storage;
- ROM is the general concept of fixed nonvolatile information.

These categories describe practical roles. Device-specific behavior must still be checked in the applicable documentation.

## ADC and DAC: finite digital representations

For the course's ideal converter model:

```text
levels = 2^N
ideal step ~= full-scale range / levels
```

An N-bit converter has `2^N` levels but a maximum unsigned code of `2^N - 1`.

### Example: 8-bit, 0 to 5 V

```text
levels = 256
step ~= 5 V / 256
     ~= 19.53 mV/count
```

### Example: 10-bit, 0 to 3.3 V

```text
levels = 1024
step ~= 3.3 V / 1024
     ~= 3.22 mV/count
```

## Resolution is not accuracy

**Resolution** describes the ideal code step.

**Accuracy** describes closeness to the true or expected quantity under real system errors.

More bits can improve ideal resolution without guaranteeing an accurate measurement system.

Quantization is the unavoidable mapping of a continuous quantity into a finite set of codes.

## Practice

1. How many locations can a 10-bit address select?
2. How many levels does a 12-bit ideal converter have?
3. For an 8-bit 0-5 V converter, what is the ideal step size?
4. What is the difference between 256 levels and maximum code 255?
5. Why does increasing converter bit depth not automatically guarantee a more accurate measurement?

## Answer key

1. `2^10 = 1024` locations.
2. `2^12 = 4096` levels.
3. About `19.53 mV/count`.
4. There are 256 distinct codes, numbered from 0 through 255.
5. Accuracy also depends on the actual system errors, reference behavior, noise, and other non-ideal effects. More bits change ideal resolution, not every source of error.
