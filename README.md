# mb-audit

The purpose of "mb-audit" is two fold:
1. A set of tests that exercise real Mockingboard hardware to identify any faults.
   - Detect and test all slots with a Mockingboard/Phasor card in the Apple II machine (so not the Mockingboard-D for the //c).
   - Support all variants of Mockingboard, eg. all those from Sweet Micro Systems and newer versions: ReactiveMicro, A2Heaven, IanKim's etc. (NB. currently only a few types have actually been tested outside of emulation).
   - Attempt to test all components, ie. 6522s, AY-3-8913s, SC-01 & SSI263s.
2. A set of deep, probing tests that exercise emulators' Mockingboard implementation to test coverage and correctness.
   - Tests for 6502-specific and 6502/65C02-common false-read addressing modes.
   - Tests for 65C02-specific addressing modes / instructions.
   - No 65816 specialisation yet, other than taking the 6502 (not 65C02) path for false-read addressing mode support.

As it is intended as a full-correctness suite for emulators, then it goes deeper than what emulators currently need to emulate. The way the tests have been ordered is that (more or less) the more important tests are done first, with the more esoteric tests towards the end.

Tests are run in order and can't be skipped. On a first failure, then the audit will stop at that point, outputting an error to identify the test failure and some extra info to help narrow down the failure. There is no way to resume testing from after this failing test, mainly because subsequent tests rely on behaviour that must be working from previous tests.

Motivation: to consolidate the sprawling set of individual tests I've been creating over the years for AppleWin.
- AppleWin 1.30.1 passes all tests for mb-audit v0.1. Configurations: (a) 2x MB-C; (b) 1x Phasor

Current test matrix for real hardware:
- ReactiveMicro's Mockingboard-C, non-enhanced //e, 65C02, SSI263(socket-B) mb-audit v0.1
- Applied Engineering's Phasor, non-enhanced //e, 65C02, SSI263(socket-A), mb-audit v0.1

# Operation

- Detect CPU type: 6502, 65C02, 65816 for floating-bus & extra 65C02 instruction tests.
- Detect Apple II model for lowercase support.
- Scan slots 7..1 to detect any cards with a 6522 at $Cn00 and/or $Cn80
  - Detect 6522 using Timer2 as this is more robust than Timer1
- Display overview of what has been detected (where ?=6522 detected):
```
mb-audit v0.1, 2021                     
                    1  2  3  4  5  6  7 
               $00: ?        ?          
               $80: ?        ?          
                SP:                     
65C02 detected                          
```

Then for each card found:
- Do basic 6522 hardward checks (address line, data lines, and IRQ)
- Detect connected sub-units: SSI263s, SC-01, AY-3-8913s
- Determine if the card is a Phasor or MEGA Audio(*1) or MB4C(*1) or Echo+(*1)
  - (*1) Untested on real hardware
- Display a more detailed summary of what has been detected, eg:
```
mb-audit v0.1, 2021                     
                    1  2  3  4  5  6  7 
               $00: P        C          
               $80: P        C          
                SP:A         B          
65C02 detected                          
```
Key:
- For the '$00' and '$80' rows:
  - ? = 6522(VIA)
  - 1 = Sound-I or Speech-I
  - S = Sound/Speech-I
  - M = MEGA Audio
  - M4C = MB4C
  - E=Echo+
  - C=MB-C (or MB-A or Sound-II)
  - P=Phasor
- For the 'SP' row (Speech chips):
  - A=SSI263(socket-A)
  - B=SSI263(socket-B)
  - V=Votrax/SC01

Continuing for each card found:
- Do 6522 tests
- Do AY-3-8913 tests
- Do SSI263 tests
- Do SC-01 tests

After all cards have been tested then:
- Do multi-card tests
- For each card test 6522 after a RESET
- For each card test all AY-3-8913's (this is a manual tone test)

When a test fails, the display will indicate the failing test and reason:
```
mb-audit v0.1, 2021                     
                    1  2  3  4  5  6  7 
               $00: ?           C       
               $80: ?           C       
                SP:             B       
65C02 detected                          
Slot #5 :Mockingboard failed test: 6522 
Test: 11:04:0A                          
Expected:09 Actual:F1                   
```

Where:
- `Test: CC:TT:nn`
  - CC = Component (1x=6522, 2x=AY8913, 3x=SC01, 4x=SSI263)
    - So CC=11 (in the example above) means 6522
  - TT = Test number (see source code!)
  - nn = Sub-test number (see source code!)
- `Expected:EE Actual:AA`
  - EE = expected value
  - AA = actual value

# Details on tests

6522 tests
- In general it will do the same test for 6522s at $Cn00 and $Cn80 (if the cards has 2x 6522s).
  - And in most cases it will test both Timer1 and Timer2.
- Some tests of note:
  - test all the instructions that can _write_ to Timer1/2 (excluding read-modify-write instructions)
  - test all the instructions that can _read_ to Timer1/2 (excluding read-modify-write instructions)
  - test timers before/after underflow
  - test Timer1 with very small latch values (both one-shot & free-running modes)
  - test Timer1's period is N+2 cycles
  - test cycle-accurate (sub-instruction) reading of IFR at Timer1/2 underflow
  - test that the 6522's 16-byte I/O space is mirrored throughout the $Cnxx address space
  - test Phasor's Echo+ mode
  - test up to 4 simultaneous, active interrupts (1 card)
  - test up to 8 simultaneous, active interrupts (2 cards)
  - test that certain 6522 state (eg. pending IRQ, enabled timers) are disabled on RESET

AY-3-8913 tests
- Try to write then read back the AY registers.
  - NB. Not all cards support reading the AY registers (eg. Phasor).
- Test address line and data lines.
- Test cards which have a single 6522, but 2 AY-3-8913s (eg. Echo+, MB4C).
- Manual tone tests:
  - Keys: 1,2,3 for AY-3-8913 (chip-A) at $Cn00
  - Keys: 4,5,6 for AY-3-8913 (chip-B) at $Cn80
  - Keys: Q,W,E for 2nd AY-3-8913 (chip-A2) at $Cn00 (Phasor only)
  - Keys: R,T,Y for 2nd AY-3-8913 (chip-B2) at $Cn80 (Phasor only)
  - ESC to quit

SSI263 tests
- Support a single or both socketed SSI263's.
- Play back and time the phrase "Classic Adventure" (from Sweet Micro Systems's game of the same name).
- The infamous Willy Byte bug that writes to the SSI263 to inadvertently (via a false-read) clear the 6522 Timer1 interrupt!
  - Code will test different false-reads addressing-modes depending on the CPU type.
- Test Phasor's native mode which has a different SSI263 interface.

SC-01 tests
- Support a single SC-01, and also AppleWin's hybrid Mockingboard-C/Phasor cards with both SC-01 and 2x SSI263.
- Play back and time the phrase "The Spy Strikes Back" (from Penguin Software's game of the same name).
  - A real Speech/Sound I card has a hardware potentiometer to control the playback rate. Potentially the test will fail if the rate is too low.
