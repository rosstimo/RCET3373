<a id="top"></a>

# RCET 3373 — PIC16F883 Data EEPROM and Persistent State

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

RAM is useful while a program is running, but its contents disappear when power is removed.

Data EEPROM is useful when a small amount of application state must survive reset or power loss, for example:

- calibration constants;
- user settings;
- counters that change infrequently;
- configuration values;
- learned/commissioned parameters.

Persistent storage introduces different engineering constraints than RAM:

- writes take time;
- writes have finite endurance;
- an interrupted write can leave uncertain state;
- accidental writes must be prevented;
- data formats must survive software revisions and partial updates.

The PIC16F883 data EEPROM is a good platform for learning those constraints directly.

[Back to top](#top) · [Topics index](README.md)

<a id="learning-outcomes"></a>
## 2. What you should be able to do

After working through this guide, you should be able to:

- distinguish GPR RAM, program Flash, and data EEPROM;
- identify the PIC16F883 EEPROM address, data, and control registers;
- trace the EEPROM read sequence;
- trace the protected EEPROM write sequence;
- explain the roles of `WREN`, `WR`, `WRERR`, and `EEIF`;
- explain why the `0x55/0xAA` unlock sequence exists;
- explain why interrupts are commonly excluded during the critical unlock/start sequence;
- distinguish write-completion signaling from normal program flow;
- explain EEPROM endurance and why continuously writing unchanged data is poor design;
- design simple persistent-state updates that reduce corruption and wear.

[Back to top](#top) · [Topics index](README.md)

<a id="prerequisites"></a>
## 3. Prerequisites and related topics

Before this guide, review:

- [Memory Systems](memory-systems.md#volatile-nonvolatile);
- [Data Memory, SFRs, and Banking](data-memory-sfrs-banking.md#core-model);
- [Interrupts and Context Saving](interrupts-context-saving.md#event-flag-enable).

Related topics:

- [I2C](i2c.md#core-model) for external serial EEPROM communication;
- [Embedded-System Integration and Troubleshooting](embedded-system-integration-troubleshooting.md#persistent-state) for higher-level integrity strategies.

[Back to top](#top) · [Topics index](README.md)

<a id="core-model"></a>
## 4. Core model and vocabulary

<a id="eeprom-registers"></a>
### Data EEPROM interface registers

Important PIC16F883 resources include:

| Register / bit | Role |
| --- | --- |
| `EEADR` | selected data-EEPROM address |
| `EEDAT` | data byte read or written |
| `EECON1.EEPGD` | selects data EEPROM versus program-memory access path |
| `EECON1.RD` | initiates a data EEPROM read |
| `EECON1.WREN` | permits writes |
| `EECON1.WR` | starts / indicates EEPROM write operation |
| `EECON1.WRERR` | records certain interrupted-write conditions |
| `EECON2` | write-only unlock-sequence mechanism, not a physical storage register |
| `PIR2.EEIF` | EEPROM write-complete interrupt flag |
| `PIE2.EEIE` | EEPROM write interrupt enable |

Use the device data sheet for exact bit positions and reset states.

<a id="persistent-state"></a>
### Persistent state should be written deliberately

RAM variables may change thousands or millions of times while a program runs.

EEPROM should generally be updated because persistent information **actually changed**, not simply because a loop executes.

A useful design question is:

> Does this state need to survive power loss, or is RAM sufficient?

[Back to top](#top) · [Topics index](README.md)

<a id="how-it-works"></a>
## 5. How it works

<a id="eeprom-read"></a>
### Reading data EEPROM

The conceptual read sequence is:

```text
select data EEPROM
-> place address in EEADR
-> set RD
-> hardware fetches EEPROM byte
-> read EEDAT
```

The data sheet describes the device-specific timing and register behavior.

A read does not consume EEPROM write endurance.

<a id="eeprom-write"></a>
### Writing data EEPROM is intentionally protected

A write requires more than writing `EEDAT`.

Conceptually:

```text
select data EEPROM
-> choose EEADR
-> load EEDAT
-> enable writes with WREN
-> enter protected unlock sequence
-> set WR
-> hardware performs self-timed write
-> completion indicated by WR clearing / EEIF
-> disable writes when no longer needed
```

The protected unlock sequence includes the documented writes:

```text
0x55
0xAA
```

to `EECON2`, followed immediately by starting the write.

The unusual sequence reduces the chance that random or runaway code accidentally begins a nonvolatile write.

**Official table reference:** Section 10.0, **Data EEPROM and Flash Program Memory Control**, in the [PIC16F882/883/884/886/887 Data Sheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf) contains the documented EEPROM write sequence and register summary.

Use that sequence as the authority rather than copying an old code fragment from memory.

<a id="critical-sequence"></a>
### Protect the critical unlock/start sequence

The unlock/start instructions must occur in the required order without being interrupted by unrelated execution.

A common approach is:

1. remember the prior global-interrupt state as needed;
2. prevent interrupts during the critical unlock/start sequence;
3. write `0x55` to `EECON2`;
4. write `0xAA` to `EECON2`;
5. set `WR`;
6. restore interrupt behavior appropriately after the critical start sequence.

Do not blindly enable interrupts afterward if they were intentionally disabled before the EEPROM routine. Preserve the application's intended interrupt state.

<a id="write-completion"></a>
### Write completion is asynchronous to ordinary instruction flow

Once started, EEPROM programming is performed by the device's internal write mechanism.

Software can determine completion by the documented status/flag behavior instead of assuming one fixed software delay.

If EEPROM completion interrupts are enabled:

```text
write completes
-> EEIF set
-> EEIE + PEIE + GIE allow interrupt service
```

If interrupts are not used, software can poll the documented completion state.

<a id="endurance"></a>
### EEPROM has finite write endurance

EEPROM is designed to be rewritten, but not infinitely.

A design that writes the same address every pass through a fast loop can consume endurance unnecessarily.

Better strategies include:

- only write when the persistent value actually changes;
- accumulate rapidly changing state in RAM and save less frequently;
- distribute writes when the application requires very high update counts;
- use checksums/version fields when corruption detection matters;
- design power-loss-sensitive multi-byte updates carefully.

Endurance, retention, and cycling conditions come from the applicable device/memory documentation.

<a id="interrupted-write"></a>
### Interrupted writes need a recovery strategy

A reset or power problem during a write can leave the intended update incomplete.

`WRERR` provides device-specific information about certain interrupted writes, but application-level data integrity may require more.

For important multi-byte state, common patterns include:

- store a version or sequence number;
- store a checksum/CRC;
- keep old and new copies until an update is verified;
- update a validity marker only after data is complete.

The right strategy depends on how costly corrupted state would be.

[Back to top](#top) · [Topics index](README.md)

<a id="worked-examples"></a>
## 6. Worked examples

<a id="change-only-example"></a>
### Worked example: write only when a setting changes

Suppose a user selects a mode stored in EEPROM.

Poor approach:

```text
main loop executes
-> write current mode to EEPROM every pass
```

Better approach:

```text
read saved mode at startup
-> keep current mode in RAM
-> user changes mode
-> compare new value with saved value
-> write EEPROM once when the persistent value changes
```

**Meaning**

The application gets the same persistent behavior with dramatically fewer erase/write cycles.

<a id="multi-byte-example"></a>
### Worked example: protect a two-byte setting

Suppose two bytes form one logical calibration value.

A power loss after the first byte but before the second could create a mixed old/new value.

One simple integrity scheme could store:

```text
byte 0: calibration low
byte 1: calibration high
byte 2: checksum
byte 3: version/sequence
```

At startup, software checks whether the stored record is self-consistent before trusting it.

The exact format is an application design choice, not a PIC16F883 hardware requirement.

[Back to top](#top) · [Topics index](README.md)

<a id="apply-verify-troubleshoot"></a>
## 7. Apply, verify, and troubleshoot

<a id="eeprom-checklist"></a>
### EEPROM bring-up checklist

If persistent data is not behaving correctly:

1. verify whether you are accessing data EEPROM rather than program memory;
2. verify `EEADR`;
3. verify `EEDAT`;
4. verify write enable;
5. verify the exact unlock/start sequence;
6. verify the critical sequence is not being interrupted;
7. verify write completion before starting another dependent operation;
8. verify `WREN` is cleared when writes are not intended;
9. inspect `WRERR`/completion flags as documented;
10. power-cycle the board to prove persistence;
11. check whether your software rewrites unchanged values unnecessarily.

<a id="power-cycle-proof"></a>
### Prove persistence with a power cycle

A value that survives one function call is not proof of EEPROM operation.

A useful test is:

1. write a known value;
2. wait for confirmed completion;
3. read it back;
4. remove/reapply power or reset in a way appropriate to the test;
5. read the value again;
6. compare with the expected persistent record.

[Back to top](#top) · [Topics index](README.md)

<a id="practice"></a>
## 8. Practice

1. Why would a calibration constant belong in EEPROM instead of ordinary GPR RAM?
2. Which register selects the EEPROM address?
3. Which register carries the EEPROM data byte?
4. What is the purpose of `WREN`?
5. Why is the `0x55/0xAA` sequence used?
6. Why should the critical unlock/start sequence not be interrupted?
7. Why should software avoid writing the same EEPROM value every main-loop pass?
8. What does `EEIF` represent?
9. Why can a two-byte persistent value require an integrity strategy?
10. What test proves that data is truly persistent rather than only stored in RAM?
11. What should software do with `WREN` after writes are no longer intended?

[Back to top](#top) · [Topics index](README.md)

<a id="answer-key"></a>
## 9. Answer key

1. EEPROM retains data without normal operating power; GPR RAM does not.
2. `EEADR`.
3. `EEDAT`.
4. It permits EEPROM write operations; leaving it disabled reduces accidental-write risk.
5. It is a protected unlock sequence designed to make accidental write initiation less likely.
6. An interrupt between required unlock/start operations would break the documented sequence and can prevent the intended write.
7. EEPROM has finite write endurance, so unnecessary writes waste lifetime.
8. EEPROM write completion.
9. Power loss between byte writes can create a mixed/incomplete record; checksums, versioning, or redundant records can detect/recover from that.
10. Read the value after a real reset/power-cycle condition that clears volatile RAM.
11. Clear/disable it when writes are not needed.

[Back to top](#top) · [Topics index](README.md)

<a id="retrieval-check"></a>
## 10. What you should be able to explain without notes

You should be able to:

- distinguish RAM, program Flash, and data EEPROM roles;
- identify EEPROM address/data/control registers;
- trace a read;
- trace the protected write sequence;
- explain `WREN`, `WR`, `EEIF`, and `WRERR`;
- explain the critical unlock sequence;
- explain why EEPROM should not be written continuously;
- describe a simple strategy for detecting incomplete persistent updates;
- prove persistence with a power-cycle test.

[Back to top](#top) · [Topics index](README.md)

<a id="references"></a>
## 11. References

- Microchip Technology Inc., *PIC16F882/883/884/886/887 Data Sheet*, DS40001291H, Section 10.0 "Data EEPROM and Flash Program Memory Control" — https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/40001291H.pdf
  - Used for: EEPROM registers, read/write sequences, unlock mechanism, write completion, flags, and device behavior.

- Microchip Technology Inc., *EEPROM Endurance Tutorial*, AN1019 — https://www.microchip.com/en-us/application-notes/an1019
  - Used for: endurance terminology, cycling, retention, and persistent-memory design context.

[Back to top](#top) · [Topics index](README.md)
