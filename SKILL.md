# RISC OS Module Development - Lessons Learned

This document captures key insights about developing RISC OS modules, gathered from implementing the Timer module.

## Module Structure

### Core Files

A RISC OS module requires three essential components:

1. **CMHG file** (`cmhg/modhead`) - Module definition describing:
   - Module title and help string
   - SWI chunk base number
   - Entry points (initialisation, finalisation, SWI handler)
   - SWI decoding table (names of exported SWIs)

2. **Module interface** (`c/module`) - C code implementing:
   - `Mod_Init()` - Called when module is loaded
   - `Mod_Final()` - Called when module is unloaded
   - `Mod_Handler()` - Dispatches SWI calls to appropriate functions

3. **Makefile** (`Makefile,fe1`) - Build configuration using AMU (RISC OS make)

### File Naming Conventions

- C source files: `c/<name>` (no extension in filename, but `#include` uses `.h`)
- C header files: `h/<name>` (no extension in filename)
- Makefiles: `Makefile,fe1` (comma suffix is RISC OS file type)
- BBC BASIC files: `<name>,fd1`
- Built modules: `rm32/<name>,ffa` (32-bit RAM module)

## SWI Interface

### SWI Numbering

SWIs are numbered relative to a base chunk:
```
SWI_Base = &C09C0
TIMER_RETURNNUMBER = &C09C0 + 0 = &C09C0
TIMER_CLAIM        = &C09C0 + 1 = &C09C1
```

### SWI Handler Signature

The SWI handler receives registers through a structure:
```c
_kernel_oserror *Mod_Handler(int swi, _kernel_swi_regs *r, void *pw)
{
    switch (swi) {
    case 0:
        // Access registers via r->r[0], r->r[1], etc.
        // Return values by setting r->r[0], r->r[1], etc.
    }
}
```

### XSWI Convention

For error handling, RISC OS uses the XSWI convention:
- Standard SWI: Sets V flag on error, error block pointer in R0
- XSWI prefix: Returns error flags in R0 (bit 0 = V flag), preserves registers

```basic
REM BBC BASIC - use XSWI for error handling
SYS "XTimer_ReturnNumber", 0 TO r0%,r1%,r2%;flags%
IF flags% AND 1 THEN
  error_num% = r0%!0  REM Error number at R0+0
ENDIF
```

### SWI Names

SWI names follow the pattern `<Module>_<Function>` with PascalCase:
- `Timer_ReturnNumber`
- `Timer_Claim`
- `Timer_Release`

CMHG automatically generates both standard and XSWI variants:
- `Timer_ReturnNumber` (standard)
- `XTimer_ReturnNumber` (XSWI with error flags)

## C Coding Standards

### Variable Declarations

All variables must be declared at the **top of their block**, before any statements:
```c
_kernel_oserror *some_function(int arg)
{
    int result;      /* Declaration first */
    int error_code;  /* All declarations together */
    
    result = do_something();  /* Statements after */
    return NULL;
}
```

### NULL Pointer Handling

The RISC OS C compiler is strict about NULL:
```c
/* Use explicit casts for function pointers */
timers[i].callback = (void (*)(int))0;

/* Use explicit cast for NULL returns */
return (_kernel_oserror *)0;

/* Define error blocks statically */
static _kernel_oserror err_out_of_range = {1, "Timer number out of range"};
return &err_out_of_range;
```

### No Floating Point

Modules should not use floating point operations. Use integer arithmetic only.

### No Threading

RISC OS is single-threaded. Do not use threading or assume re-entrancy.

## Memory Management

### Error Blocks

Error blocks have a fixed structure:
```c
typedef struct {
    int errnum;      /* Error number */
    char errmess[252]; /* Error message */
} _kernel_oserror;
```

Define errors statically and return pointers to them.

### Private Word

The `pw` parameter in init/final/handler is a module-private workspace pointer.

## Build System

### AMU Makefiles

The build uses AMU (RISC OS make) with `,fe1` filetype:
```makefile
OBJS = o.modhead \
       o.module \
       o.hardware

include CModule
```

### Build Targets

- `amu` or `amu ram` - Build RAM module
- `amu rom` - Build ROM module
- `amu BUILD26=1` - Build 26-bit compatible version
- `amu BUILD64=1` - Build 64-bit version
- `amu export` - Export headers/libraries
- `amu clean` - Remove build artifacts

### Output Directories

- `oz32/` - 32-bit object files
- `rm32/` - 32-bit modules (RAM)
- `aof32/` - 32-bit AOF files (ROM)

## Testing

### Loading Modules

```basic
*RMLoad rm32.timer    REM Load module
*RMKill timer         REM Unload module
```

### BASIC Testing

Use SYS calls from BBC BASIC:
```basic
REM Standard SWI (error raises BASIC error)
SYS "Timer_ReturnNumber", 0 TO ,,,num_timers%

REM XSWI (check flags for errors)
SYS "XTimer_Claim", 0, 0, 1000, 0 TO r0%,r1%,rate%;flags%
IF flags% AND 1 THEN
  error_num% = r0%!0
ENDIF
```

### Build Testing

```bash
riscos-build-run rm32 test_timer,fd1 \
  --command "*RMLoad rm32.timer" \
  --command "run test_timer,fd1"
```

## Common Pitfalls

### 1. Function Pointer Casts

The compiler warns about function pointer casts:
```c
/* Warning expected - cast between function pointer and non-function object */
void (*callback)(int) = (void (*)(int))r->r[3];
```

### 2. Error Block Return

When returning errors, the V flag must be set. CMHG veneers handle this automatically when you return a non-NULL error pointer.

### 3. Register Preservation

For XSWI calls, R0 should contain flags (0 on success), not the result. Results go in R2 onwards.

### 4. Module Title vs SWI Prefix

The module title ("Timer") becomes the SWI prefix. All SWIs are prefixed with this name.

## Debugging

### Debug Output

Use conditional compilation for debug output:
```c
#define DEBUG

#ifdef DEBUG
#define dprintf if (1) printf
#else
#define dprintf if (0) printf
#endif

dprintf("Debug: value=%d\n", value);
```

Note: `printf` output goes to the debug stream, visible in build logs.

### Build Verification

Always verify builds:
```bash
riscos-amu clean
riscos-amu
riscos-build-run rm32 --command "*RMLoad rm32.timer"
```

## CI/CD

### GitHub Actions

Uses `riscos-build-online` client to submit builds to build.riscos.online:
```yaml
- name: Build through build.riscos.online
  run: |
    curl -L -o riscos-build-online <url>
    zip -r source.zip * .robuild.yaml
    ./riscos-build-online -i source.zip -o /tmp/built
```

### .robuild.yaml

Describes the build process on RISC OS:
```yaml
jobs:
  build:
    env:
      BUILD32: 1
    script:
      - amu -f Makefile ram
      - RMLoad rm32.timer
      - RMKill timer
    artifacts:
      - path: Artifacts
```

## Resources

- [RISC OS PRM - Software Interrupts](https://www.riscos.info/modules/)
- [RISC OS 6 Timer Documentation](https://www.riscos.com/support/developers/riscos6/hardware/timer.html)
- [CMHG Documentation](https://www.riscos.info/modules/cmhg.htm)
- [PRM 1-26 - SWI Chunk Allocation](https://www.riscos.info/modules/)
