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

Tests are run in order and can't be skipped (except for the soak test where ESC skips the current test). On a first failure, then the audit will stop at that point, outputting an error to identify the test failure and some extra info to help narrow down the failure. There is no way to resume testing from after this failing test, mainly because subsequent tests rely on behaviour that must be working from previous tests. NB. A few tests will output a warning and continue.

Motivation: to consolidate the sprawling set of individual tests I've been creating over the years for AppleWin.
- AppleWin 1.30.1 passes all tests for mb-audit v0.1-beta. Configurations: (a) 2x MB-C; (b) 1x Phasor

Current test matrix for real hardware:
- ReactiveMicro's Mockingboard-C (but with WDC-65C22's), enhanced //e, 65C02, SSI263(socket-B), all versions of mb-audit
- Applied Engineering's Phasor, enhanced //e, 65C02, SSI263(socket-A), all versions of mb-audit
- SMS's Sound/Speech I, mb-audit v0.4-beta
- Ian Kim's SD Music Deluxe, mb-audit v0.9
- MEGA Audio, mb-audit v1.4x
- Echo+, mb-audit v1.51

![DSC00534-s](https://user-images.githubusercontent.com/6696896/117582673-188b6300-b0fb-11eb-9baf-4ba27d112542.png)

### Operation

Here's a high-level overview of what mb-audit does:
- Detect CPU type: 6502, 65C02, 65816 for false-read & extra 65C02 instruction tests.
- Detect Apple II model for lowercase support.
- Scan slots 7..1 to detect any cards with a 6522 at $Cn00 and/or $Cn80
  - Detect 6522 using Timer2 as this is more robust than Timer1
- Display overview of what has been detected in slots 1-7 (where ?=6522 detected):
```
mb-audit v0.1-beta, 2021                     
                    1  2  3  4  5  6  7 
               $00: ?        ?          
               $80: ?        ?          
                SP:                     
65C02 detected                          
```

Then for each card found:
- Do basic 6522 hardware checks (data lines, address lines and IRQ) for both 6522s:
   - If 6522-B fails then continue and do tests for 6522-A.
   - On failure: output the failure results for one or both 6522s (then stop).
- Detect connected sub-units: SSI263s, SC-01, AY-3-8913s
- Determine if the card is a Phasor, MEGA Audio, MB4C(*1), Echo+ or SD Music
  - (*1) Untested on real hardware
- Display a more detailed summary of what has been detected, eg:
```
mb-audit v0.1-beta, 2021                     
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
  - S/S = Sound/Speech-I
  - M = MEGA Audio
  - M4C = MB4C
  - E = Echo+
  - C = MB-C (or MB-A or Sound-II)
  - P = Phasor
  - SDM = SD Music (and other clones with a single-6522)
- For the 'SP' row (Speech chips):
  - A = SSI263(socket-A)
  - B = SSI263(socket-B)
  - V = Votrax/SC01

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
mb-audit v0.1-beta, 2021                     
                    1  2  3  4  5  6  7 
               $00: ?        C          
               $80: ?        C          
                SP:          B          
65C02 detected                          
Slot #4 :Mockingboard failed test: 6522 
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

### Test Details

6522 tests
- In general it will do the same test for 6522s at $Cn00 and $Cn80 (if the cards has 2x 6522s).
  - And in most cases it will test both Timer1 and Timer2.
- Some tests of note:
  - test all the instructions that can _write_ to Timer1/2 (excluding read-modify-write instructions)
    - include 65C02 (but not 65816) instructions
  - test all the instructions that can _read_ from Timer1/2 (excluding read-modify-write instructions)
    - include 65C02 (but not 65816) instructions
  - test reading timers just before/after underflow
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
  - NB. Not all cards support reading the AY registers (eg. MEGA Audio).
- Test address lines and data lines.
- Test cards which have a single 6522, but 2x AY-3-8913s (eg. Echo+, MB4C).
- Test Phasor's GAL logic for mixing the PSG AY1/AY2 LATCH & WRITE/READ functions (Phasor "native" mode).
- Manual tone tests:
  - Keys: 1,2,3 for AY-3-8913 (chip-A) at $Cn00
  - Keys: 4,5,6 for AY-3-8913 (chip-B) at $Cn80
  - Keys: Q,W,E for 2nd AY-3-8913 (chip-A2) at $Cn00 (Phasor only)
  - Keys: R,T,Y for 2nd AY-3-8913 (chip-B2) at $Cn80 (Phasor only)
  - ESC to quit
  - TAB to cycle Phasor -> Echo+ -> Mockingboard mode (Phasor card)

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

Running on a IIgs
- enter Control Panel, eg. using Ctrl-Cmd-Esc during boot
  - For MAME on Windows, reboot using Ctrl-L.Alt-F12, then Ctrl-L.Alt-Esc for Control Panel
- then from Control Panel, set:
  - System Speed=Normal
  - Slot-4=Your card
- Ctrl-Reset so that these new settings take effect

### To Do
- confirm mb-audit works with all the advertised real hardware
- add more tests
  - eg. 6522: write to IFR.T1/2 on same cycle as underflow, resulting in IFR.T1/2=0 in the ISR
  - eg. Mockingboard: check INACTIVE state is emulated

### Acknowledgements

Thanks to Andrew Roughan for his help with testing against his MEGA Audio and single-6522 clone cards, and Brian J. Bernstein for testing against his Echo+ card.
