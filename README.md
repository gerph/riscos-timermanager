# Timer Module

A RISC OS module providing access to hardware timers, implementing the RISC OS 6 Timer interface.

## Overview

The Timer module provides a software interface to hardware timers on RISC OS systems. This implementation provides stub functions that simulate timer behaviour without accessing actual hardware.

## SWI Interface

The module uses SWI base `&C09C0` and provides the following SWIs:

| SWI | Offset | Number | Description |
|-----|--------|--------|-------------|
| `Timer_ReturnNumber` | 0 | `&C09C0` | Returns the number of timers available |
| `Timer_Claim` | 1 | `&C09C1` | Claims a timer for use |
| `Timer_Release` | 2 | `&C09C2` | Releases a claimed timer |
| `Timer_SetRate` | 3 | `&C09C3` | Sets the timer rate |
| `Timer_Convert` | 4 | `&C09C4` | Converts timer values between formats |

### Timer_ReturnNumber

Returns the number of hardware timers available.

**Input:**
- `R0` = flags (must be 0)

**Output:**
- `R0` = 0 (success flags)
- `R2` = number of timers available

### Timer_Claim

Claims a timer for exclusive use.

**Input:**
- `R0` = flags (bits 0-7: measurement format for timer rate; bits 8-15: measurement format to return timer rate; bits 16-31: reserved, must be 0)
- `R1` = timer number
- `R2` = timer value
- `R3` = pointer to callback function (stub: ignored)

**Output:**
- `R0` = 0 (success flags)
- `R2` = timer rate in format specified by R0 bits 8-15

**Errors:**
- 1: Timer number out of range
- 2: Timer already claimed

### Timer_Release

Releases a previously claimed timer.

**Input:**
- `R0` = flags (must be 0)
- `R1` = timer number

**Output:**
- `R0` = 0 (success flags)

**Errors:**
- 1: Timer number out of range
- 3: Timer not claimed

### Timer_SetRate

Sets the rate of a claimed timer.

**Input:**
- `R0` = flags (bits 0-7: measurement format for timer rate; bits 8-15: measurement format to return timer rate; bits 16-31: reserved, must be 0)
- `R1` = timer number
- `R2` = timer rate value

**Output:**
- `R0` = 0 (success flags)
- `R2` = actual timer rate set

**Errors:**
- 1: Timer number out of range
- 3: Timer not claimed

### Timer_Convert

Converts a timer value between different measurement formats.

**Input:**
- `R0` = flags (bits 0-7: measurement format for timer rate; bits 8-15: measurement format to return timer rate; bits 16-31: reserved, must be 0)
- `R1` = timer number
- `R2` = timer value to convert

**Output:**
- `R0` = 0 (success flags)
- `R2` = converted timer value

**Errors:**
- 1: Timer number out of range

## Building

### Prerequisites

- RISC OS build environment with `riscos-amu`
- CMHG module header generator

### Build Commands

```bash
# Build the module (32-bit)
riscos-amu

# Build the module (64-bit)
riscos-amu BUILD64=1

# Clean build artifacts
riscos-amu clean
```

### Output

The built module is output to `rm32/timer,ffa` (32-bit) or `rm64/timer,ffa` (64-bit).

## Testing

A BBC BASIC test program is provided to verify all SWI functions:

```bash
riscos-build-run rm32 test_timer,fd1 --command "*RMLoad rm32.timer" --command "run test_timer,fd1"
```

The test program verifies:
1. Timer count retrieval
2. Timer claiming
3. Error handling for already-claimed timers
4. Error handling for out-of-range timers
5. Rate setting
6. Value conversion
7. Timer release
8. Error handling for unclaimed timers
9. Invalid flags handling
10. Multiple timer claiming

## Project Structure

```
.
├── LICENSE              # MIT License
├── README.md            # This file
├── Makefile,fe1         # Build configuration
├── VersionNum           # Version information
├── cmhg/
│   └── modhead          # Module definition
├── c/
│   ├── module           # Main module interface
│   └── hardware         # Hardware abstraction (stubs)
├── h/
│   └── modhead          # Generated header (auto-generated)
├── rm32/
│   └── timer,ffa        # Built module (32-bit)
└── test_timer,fd1       # BBC BASIC test program
```

## Implementation Notes

- This is a **stub implementation** - no actual hardware is accessed
- The module provides 4 simulated timers
- Default timer rate is 1MHz (1,000,000 Hz)
- No floating point operations are used
- All variables are declared at the top of their blocks (RISC OS C convention)
- 4-space indentation is used throughout

## Error Codes

| Code | Description |
|------|-------------|
| 1 | Timer number out of range |
| 2 | Timer already claimed |
| 3 | Timer not claimed |
| 100 | Invalid flags |
| 101 | Reserved flags must be zero |
| 102 | Unknown SWI |

## Version History

### Version 0.01 (24 Mar 2026)
- Initial release by Charles Ferguson
- Implements all 5 Timer SWIs
- Stub hardware implementation
- BBC BASIC test suite

## License

This software is released under the MIT License. See [LICENSE](LICENSE) for details.

## References

- [RISC OS 6 Timer Interface Documentation](https://www.riscos.com/support/developers/riscos6/hardware/timer.html)
- [RISC OS PRM - Software Interrupts](https://www.riscos.info/modules/)
