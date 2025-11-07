# SMX File Format and Virtual Machine Specification

## Overview

This document describes the SMX (SourceMod eXecutable) binary file format and how to implement a virtual machine to execute SMX files. SMX is the compiled bytecode format for SourcePawn scripts.

**Target Audience**: VM implementors who want to execute SMX files, not compiler developers.

**Scope**: This document covers:
- The SMX container file format
- All required and optional sections
- The bytecode instruction set
- Memory model and execution semantics
- Runtime type information (RTTI)
- Debug information

### Quick Facts

- **Magic**: `0x53504646` ("SPFF")
- **Endianness**: Little-endian
- **Cell Size**: 32-bit (4 bytes)
- **Current Version**: 0x0107 (SourcePawn 1.7)
- **Compression**: Optional gzip on section data
- **Instruction Format**: Variable-length (1-6 cells per instruction)
- **Registers**: PRI, ALT, FRM, STK, HEP, CIP (all 32-bit)
- **Memory**: CODE (read-only), DAT (data+heap), STK (stack)

## Table of Contents

1. [File Format Overview](#file-format-overview)
2. [Container Structure](#container-structure)
3. [Sections](#sections)
4. [Memory Model](#memory-model)
5. [Instruction Set](#instruction-set)
6. [Execution Model](#execution-model)
7. [Type System and RTTI](#type-system-and-rtti)
8. [Debug Information](#debug-information)
9. [Implementation Guide](#implementation-guide)

---

## File Format Overview

### Basic Structure

An SMX file is a container format with the following structure:

```
[File Header]
[Section Table]
[String Table]
[Section Data (optionally compressed)]
```

### Key Properties

- **Magic Number**: `0x53504646` (ASCII: "SPFF" - SourcePawn File Format)
- **Endianness**: Little-endian (same as x86)
- **Cell Size**: 32-bit (4 bytes) - the fundamental data unit
- **Alignment**: All code offsets must be 4-byte aligned
- **Compression**: Optional gzip compression of section data
- **Platform**: Cross-platform within same endianness architectures
- **String Encoding**: Null-terminated, typically UTF-8 or ASCII

---

## Container Structure

### File Header (`sp_file_hdr_t`)

Located at offset 0 in the file:

```c
typedef struct sp_file_hdr_s {
    uint32_t magic;        // 0x53504646 (SPFF)
    uint16_t version;      // File format version
    uint8_t  compression;  // Compression type
    uint32_t disksize;     // Size on disk (with compression)
    uint32_t imagesize;    // Size in memory (decompressed)
    uint8_t  sections;     // Number of sections
    uint32_t stringtab;    // Offset to string table
    uint32_t dataoffs;     // Offset where compression starts
} sp_file_hdr_t;
```

#### Version Numbers

- **0x0101**: SourcePawn 1.0 (SourceMod 1.0)
- **0x0102**: SourcePawn 1.1 (SourceMod 1.1+)
- **0x0107**: SourcePawn 1.7 (current standard)
- **0x0200**: SourcePawn 2.0 (experimental)

Major version (bits 8-15) indicates product compatibility.
Minor version (bits 0-7) indicates format compatibility within a product.

#### Compression

- **0**: No compression
- **1**: Gzip compression

When compression is enabled:
- `dataoffs` indicates where compressed region starts
- `disksize` = compressed data size + `dataoffs`
- `imagesize` = total size after decompression
- Decompression should happen "in-place" replacing compressed bytes

### Section Table

Immediately follows the header. Contains `sections` entries of type:

```c
typedef struct sp_file_section_s {
    uint32_t nameoffs;  // Offset into string table
    uint32_t dataoffs;  // Offset to section data
    uint32_t size;      // Section size in bytes
} sp_file_section_t;
```

### String Table

Located at `stringtab` offset. Contains null-terminated strings used for section names. This is separate from the `.names` section used for runtime symbol names.

---

## Sections

### Required Sections

#### `.code` Section

Contains the bytecode and code metadata.

```c
typedef struct sp_file_code_s {
    uint32_t codesize;     // Size of bytecode
    uint8_t  cellsize;     // Cell size (always 4)
    uint8_t  codeversion;  // Bytecode version
    uint16_t flags;        // Flags (legacy, usually 0)
    uint32_t main;         // Entry point offset
    uint32_t code;         // Offset to bytecode (relative to section start)
    uint32_t features;     // Feature flags (codeversion >= 13)
} sp_file_code_t;
```

**Bytecode Versions**:
- **9**: Minimum supported version
- **10**: DEBUG flag removed
- **13**: Feature flags added (current)

**Feature Flags**:
- **Bit 1 (0x02)**: Direct array addressing (INIT_ARRAY opcode)
- **Bit 2 (0x04)**: Heap save/restore opcodes
- **Bit 3 (0x08)**: Null functions (INVALID_FUNCTION = 0 instead of -1)

The bytecode starts at offset `code` from the section start. Each instruction is a sequence of cells (4-byte values).

#### `.data` Section

Contains the initialized data segment.

```c
typedef struct sp_file_data_s {
    uint32_t datasize;  // Size of data section
    uint32_t memsize;   // Total memory required (data + heap)
    uint32_t data;      // Offset to data (helper field)
} sp_file_data_t;
```

The actual data follows the header. This section is loaded into the DAT segment at runtime.

#### `.natives` Section

Array of native function declarations that the VM must provide:

```c
typedef struct sp_file_natives_s {
    uint32_t name;  // Index into .names section
} sp_file_natives_t;
```

Each entry is a name reference. The index in this array is the native index used by `SYSREQ_C` and `SYSREQ_N` opcodes.

#### `.publics` Section

Public functions that can be called from the host:

```c
typedef struct sp_file_publics_s {
    uint32_t address;  // Code offset (relative to bytecode start)
    uint32_t name;     // Index into .names section
} sp_file_publics_t;
```

#### `.names` Section

Null-terminated string table for runtime symbol names. Names are referenced by byte offset from the section start.

### Optional Sections

#### `.pubvars` Section

Public global variables:

```c
typedef struct sp_file_pubvars_s {
    uint32_t address;  // Offset in data section
    uint32_t name;     // Index into .names section
} sp_file_pubvars_t;
```

#### `.tags` Section (Legacy)

Tag names from the compiler (deprecated, replaced by RTTI):

```c
typedef struct sp_file_tag_s {
    uint32_t tag_id;  // Compiler tag ID
    uint32_t name;    // Index into .names section
} sp_file_tag_t;
```

---

## Memory Model

### Memory Layout

A SourcePawn VM has three memory segments:

```
[CODE] - Read-only bytecode
[DAT]  - Data segment (globals + heap)
[STK]  - Stack (grows downward)
```

### Registers

The VM maintains these virtual registers (32-bit cells):

- **PRI** (Primary): General purpose register, often holds return values
- **ALT** (Alternate): Secondary register for operations
- **FRM** (Frame): Current stack frame base pointer
- **STK** (Stack): Stack pointer (grows down)
- **HEP** (Heap): Heap pointer (grows up in DAT)
- **CIP** (Code Instruction Pointer): Current instruction offset

### Addressing Modes

#### Absolute (DAT relative)
```
[offset] = DAT + offset
```

#### Stack-relative
```
[FRM + offset] = DAT + FRM + offset
```

#### Indirect
```
[[address]] = Value at computed address
```

### Memory Regions

#### Data Segment (DAT)

Layout:
```
[Global Variables] [Heap →]
^                  ^
DAT                HEP (grows right)
```

- Starts at offset 0 in memory
- Contains initialized globals from `.data` section
- Heap grows upward from initial heap position

#### Stack (STK)

```
← [Stack] 
          ^
          STK (grows left/down)
```

- Grows downward (decreasing addresses)
- Stack pointer (STK) points to next free slot
- Frame pointer (FRM) points to current call frame base

### Call Frame Layout

When a function is called, the stack frame is:

```
[Old FRM] [Return Address] [Parameters] [Local Variables]
^
FRM
```

---

## Instruction Set

### Instruction Format

Instructions are variable-length sequences of cells:
```
[Opcode] [Operand1] [Operand2] ...
```

Each opcode and operand is exactly one cell (4 bytes).

### Opcode Categories

#### Data Movement

**LOAD.pri**, **LOAD.alt** `offset`
- Load from DAT[offset] into PRI/ALT
- `PRI/ALT = *(DAT + offset)`

**LOAD.S.pri**, **LOAD.S.alt** `offset`
- Load from stack-relative address
- `PRI/ALT = *(DAT + FRM + offset)`

**LOAD.I**
- Indirect load: `PRI = *ALT`

**LODB.I** `size`
- Load byte/short: Read `size` bytes from ALT into PRI
- Sizes: 1, 2, or 4 bytes

**STOR.pri**, **STOR.alt** `offset`
- Store to DAT[offset]: `*(DAT + offset) = PRI/ALT`

**STOR.S.pri**, **STOR.S.alt** `offset`
- Store to stack-relative: `*(DAT + FRM + offset) = PRI/ALT`

**STOR.I**
- Indirect store: `*ALT = PRI`

**STRB.I** `size`
- Store byte/short: Write `size` bytes from PRI to address in ALT

**CONST.pri**, **CONST.alt** `value`
- Load immediate: `PRI/ALT = value`

**ADDR.pri**, **ADDR.alt** `offset`
- Load address: `PRI/ALT = FRM + offset`

**MOVE.pri**, **MOVE.alt**
- Copy between registers
- MOVE.pri: `PRI = ALT`
- MOVE.alt: `ALT = PRI`

**XCHG**
- Swap PRI and ALT: `PRI ↔ ALT`

#### Array Operations

**LIDX**
- Array index: `PRI = *(ALT + PRI * cellsize)`

**IDXADDR**
- Index address: `PRI = ALT + PRI * cellsize`

**INITARRAY.pri**, **INITARRAY.alt** `addr` `iv_size` `data_copy_size` `data_fill_size` `fill_value`
- Initialize multi-dimensional array
- Complex array initialization with copying and filling

**GENARRAY** `dims`, **GENARRAY.Z** `dims`
- Generate array (legacy)
- GENARRAY.Z initializes to zero

#### Stack Operations

**PUSH.pri**, **PUSH.alt**
- Push register to stack
- `STK -= 4; *(DAT + STK) = PRI/ALT`

**PUSH.C** `value`
- Push constant: `STK -= 4; *(DAT + STK) = value`

**PUSH** `offset`
- Push from DAT: `STK -= 4; *(DAT + STK) = *(DAT + offset)`

**PUSH.S** `offset`
- Push from stack: `STK -= 4; *(DAT + STK) = *(DAT + FRM + offset)`

**PUSH.ADR** `offset`
- Push address: `STK -= 4; *(DAT + STK) = FRM + offset`

**PUSH2-5.C/./S/ADR** `val1` `val2` ...
- Push multiple values in one instruction (optimized)

**POP.pri**, **POP.alt**
- Pop from stack: `PRI/ALT = *(DAT + STK); STK += 4`

**STACK** `amount`
- Adjust stack pointer: `STK += amount`

**SWAP.pri**, **SWAP.alt**
- Swap register with top of stack
- SWAP.pri: `PRI ↔ *(DAT + STK)`

#### Arithmetic

**ADD**
- Add: `PRI = PRI + ALT`

**SUB**
- Subtract: `PRI = PRI - ALT`

**SUB.ALT**
- Reverse subtract: `PRI = ALT - PRI`

**SMUL**
- Signed multiply: `PRI = PRI * ALT`

**SDIV**
- Signed divide: `PRI = PRI / ALT` (quotient)

**SDIV.ALT**
- Reverse divide: `PRI = ALT / PRI`

**ADD.C** `value`
- Add constant: `PRI += value`

**SMUL.C** `value`
- Multiply constant: `PRI *= value`

**NEG**
- Negate: `PRI = -PRI`

**INC.pri**, **INC.alt**
- Increment register: `PRI/ALT++`

**INC** `offset`, **INC.S** `offset`
- Increment memory location

**INC.I**
- Increment indirect: `(*ALT)++`

**DEC.pri**, **DEC.alt**
- Decrement register: `PRI/ALT--`

**DEC** `offset`, **DEC.S** `offset`
- Decrement memory location

**DEC.I**
- Decrement indirect: `(*ALT)--`

#### Bitwise

**AND**
- Bitwise AND: `PRI = PRI & ALT`

**OR**
- Bitwise OR: `PRI = PRI | ALT`

**XOR**
- Bitwise XOR: `PRI = PRI ^ ALT`

**NOT**
- Logical NOT: `PRI = (PRI == 0) ? 1 : 0`

**INVERT**
- Bitwise NOT: `PRI = ~PRI`

**SHL**
- Shift left: `PRI = PRI << ALT`

**SHR**
- Logical shift right: `PRI = PRI >>> ALT` (unsigned)

**SSHR**
- Arithmetic shift right: `PRI = PRI >> ALT` (signed)

**SHL.C.pri**, **SHL.C.alt** `amount`
- Shift left by constant

#### Comparison

**EQ**
- Equal: `PRI = (PRI == ALT) ? 1 : 0`

**NEQ**
- Not equal: `PRI = (PRI != ALT) ? 1 : 0`

**SLESS**
- Signed less than: `PRI = (PRI < ALT) ? 1 : 0`

**SLEQ**
- Signed less or equal: `PRI = (PRI <= ALT) ? 1 : 0`

**SGRTR**
- Signed greater than: `PRI = (PRI > ALT) ? 1 : 0`

**SGEQ**
- Signed greater or equal: `PRI = (PRI >= ALT) ? 1 : 0`

**EQ.C.pri**, **EQ.C.alt** `value`
- Compare register with constant

#### Control Flow

**PROC**
- Function prologue (marks function start, usually no-op)

**ENDPROC**
- Function epilogue (marks function end, cleanup)

**CALL** `offset`
- Call function at offset:
```
PUSH return_address
CIP = offset
```

**RETN**
- Return from function:
```
CIP = POP()
FRM = POP()
```

**JUMP** `offset`
- Unconditional jump: `CIP = offset`

**JZER** `offset`, **JNZ** `offset`
- Jump if PRI is zero/non-zero

**JEQ** `offset`, **JNEQ** `offset`
- Jump if PRI == ALT / PRI != ALT

**JSLESS** `offset`, **JSLEQ** `offset`, **JSGRTR** `offset`, **JSGEQ** `offset`
- Signed comparison jumps

**SWITCH** `table_offset`
- Switch statement (uses case table)

**CASETBL** `ncases` `default` `[case_value, case_offset]*`
- Case table data (follows SWITCH, handled specially)

#### System Calls

**SYSREQ.C** `native_index`
- Call native by index (from `.natives`)
```
result = natives[native_index](params_on_stack)
PRI = result
```

**SYSREQ.N** `native_index` `nparams`
- Call native with explicit parameter count

#### Heap Management

**HEAP** `amount`
- Allocate heap: `ALT = HEP; HEP += amount`

**HEAP.SAVE**
- Save heap pointer for scope cleanup

**HEAP.RESTORE**
- Restore heap pointer (free scope allocations)

**TRACKER.PUSH.C** `amount`
- Track heap allocation for cleanup

**TRACKER.POP.SETHEAP**
- Pop and restore heap from tracker

#### Floating Point

Floating point operations reinterpret cell bits as IEEE-754 floats:

**FLOAT**
- Convert integer to float: `PRI = (float)PRI`

**FLOAT.ADD**, **FLOAT.SUB**, **FLOAT.MUL**, **FLOAT.DIV**
- Floating point arithmetic

**FLOAT.CMP**
- Compare floats (legacy, returns ordered result)

**FLOAT.GT**, **FLOAT.GE**, **FLOAT.LT**, **FLOAT.LE**, **FLOAT.EQ**, **FLOAT.NE**
- Floating point comparisons: `PRI = (PRI op ALT) ? 1 : 0`

**FLOAT.NOT**
- Test if zero: `PRI = (PRI == 0.0 || isnan(PRI)) ? 1 : 0`

**FABS**
- Absolute value: `PRI = fabs(PRI)`

**ROUND**, **FLOOR**, **CEIL**
- Rounding operations

#### Utility

**ZERO.pri**, **ZERO.alt**
- Set register to zero

**ZERO** `offset`, **ZERO.S** `offset`
- Zero memory location

**MOVS** `size`
- Memory move: Copy `size` bytes from PRI to ALT

**FILL** `size`
- Memory fill: Fill `size` cells starting at ALT with PRI

**BOUNDS** `limit`
- Array bounds check: If PRI >= limit, trigger error

**HALT** `code`
- Halt execution with error code

**BREAK**
- Breakpoint/line marker (debugging)

**NOP**
- No operation

**LREF.S.pri**, **LREF.S.alt** `offset`
- Load reference: `PRI/ALT = *(DAT + *(DAT + FRM + offset))`

**SREF.S.pri**, **SREF.S.alt** `offset`
- Store reference: `*(DAT + *(DAT + FRM + offset)) = PRI/ALT`

**STRADJUST.PRI**
- String adjustment (legacy)

**LOAD.BOTH** `offset1` `offset2`
- Load both registers: `PRI = DAT[offset1]; ALT = DAT[offset2]`

**LOAD.S.BOTH** `offset1` `offset2`
- Load both from stack

**CONST** `offset` `value`, **CONST.S** `offset` `value`
- Set memory to constant

---

## Execution Model

### Program Startup

1. **Load File**: Parse SMX container, decompress if needed
2. **Validate**: Check magic, version, sections
3. **Allocate Memory**:
   - CODE: Load bytecode (read-only)
   - DAT: Size = `.data.memsize`, initialize from `.data` section
   - STK: Allocate separate stack (typically 256KB-1MB)
4. **Initialize Registers**:
   - CIP = `.code.main` (entry point)
   - FRM = stack_base
   - STK = stack_base
   - HEP = initial_heap (after globals)
5. **Bind Natives**: Link native indices to host functions
6. **Execute**: Start interpreter loop at main entry point

### Function Calls

#### Calling Convention

Parameters are pushed right-to-left onto the stack:

```
PUSH.C param3
PUSH.C param2
PUSH.C param1
PUSH.C num_params  // Parameter count
CALL function_offset
STACK 16  // Clean up 4 params × 4 bytes
```

Inside function:
- Parameters accessed via positive offsets from FRM
- Local variables accessed via negative offsets from FRM

#### Frame Layout

```
[param3]         FRM + 16
[param2]         FRM + 12
[param1]         FRM + 8
[num_params]     FRM + 4  (first parameter is always count)
[return_addr]    FRM + 0
[old_FRM]        FRM - 4
[local1]         FRM - 8
[local2]         FRM - 12
...
```

### Native Function Calls

When SYSREQ.C/SYSREQ.N is executed:

1. Pause VM execution
2. Call host-provided native function
3. Native reads parameters from stack using context API
4. Native returns cell value
5. Result placed in PRI
6. VM resumes

#### Native Callback Interface

Native signature (C/C++):
```c
cell_t NativeCallback(IPluginContext* context, const cell_t* params);
// params[0] = number of parameters
// params[1..N] = actual parameters
```

The context provides APIs for:
- **Memory Access**: Read/write DAT memory
- **String Operations**: Read/write strings from memory
- **Array Access**: Handle array parameters
- **Native References**: Access by-reference parameters
- **Error Reporting**: Report runtime errors

**Important**: Natives must:
- Validate all parameters and memory addresses
- Check array bounds before access
- Return error codes via context if operation fails
- Not leak stack items (save/restore STK if modified)
- Not leak heap items (save/restore HEP if modified)

#### Parameter Passing

Parameters are on the stack at the time of native call:
```
STK → [param_count] [param1] [param2] ...
```

For by-reference parameters, the value is an address in DAT that can be dereferenced.

For array parameters, the value is typically a DAT address pointing to the array data.

### Cross-Module Function Calls

SMX modules can call public functions in other loaded modules through the host environment. This is not handled at the bytecode level but through the host's runtime API.

#### How It Works

1. **Public Functions**: Each SMX module exports public functions via the `.publics` section
2. **Function Lookup**: Host provides API to find functions by name in any loaded plugin
3. **Cross-Call Mechanism**: Host marshals the call between different plugin contexts

#### API Flow (SourceMod/Host Implementation)

**From SourcePawn code:**
```javascript
// Get handle to another plugin (provided by host)
Handle plugin = LibraryExists("other_plugin");

// Look up function by name
Function func = GetFunctionByName(plugin, "PublicFunction");

// Call the function
Call_StartFunction(plugin, func);
Call_PushCell(42);
Call_PushString("hello");
int result;
Call_Finish(result);
```

**Host implementation:**
1. `GetFunctionByName(plugin, name)`:
   - Host looks up `name` in target plugin's `.publics` section
   - Returns function ID if found
   
2. `Call_StartFunction(plugin, func)`:
   - Host prepares cross-context call
   - Sets up marshalling between source and target contexts
   
3. `Call_Push*()`:
   - Host buffers parameters
   - May need to copy data between plugin memory spaces
   
4. `Call_Finish()`:
   - Host switches to target plugin's context
   - Pushes parameters onto target's stack
   - Executes CALL to function offset from `.publics`
   - Captures return value
   - Switches back to caller's context
   - Handles any copyback for by-reference parameters

#### Memory Isolation

Each SMX module has its own:
- Separate DAT segment (globals + heap)
- Separate STK segment (stack)
- Separate CODE segment (bytecode)

**Important**: Direct memory addresses cannot be passed between modules. The host must:
- Copy data when passing strings/arrays between contexts
- Handle by-reference parameters by copying data back after call
- Manage memory addresses separately per context

#### Implementation Requirements

For a VM to support cross-module calls, the host must:

1. **Plugin Management**:
   - Load multiple SMX files simultaneously
   - Maintain separate contexts per plugin
   - Track plugin handles/identifiers

2. **Function Registry**:
   - Index all public functions from all loaded plugins
   - Provide lookup API: `FindFunction(plugin, name) -> funcid`

3. **Call Marshalling**:
   - Buffer parameters during Call_Push* operations
   - Switch execution context for cross-calls
   - Copy parameter data between contexts
   - Handle copyback for by-reference parameters

4. **Context Switching**:
```c
// Pseudo-code for cross-context call
int CrossContextCall(Context* caller, Context* target, funcid_t func) {
    // Save caller state
    SaveContext(caller);
    
    // Switch to target
    SetActiveContext(target);
    
    // Copy parameters from caller stack to target stack
    for (int i = 0; i < buffered_params.count; i++) {
        PushToStack(target, buffered_params[i]);
    }
    
    // Execute function in target context
    cell_t result;
    target->Execute(func, &result);
    
    // Handle copyback for by-reference params
    CopybackParameters(caller, target);
    
    // Switch back to caller
    SetActiveContext(caller);
    RestoreContext(caller);
    
    return result;
}
```

#### Security Considerations

- **Memory safety**: Each plugin's memory is isolated
- **Access control**: Host can restrict which plugins can call each other
- **Resource limits**: Each plugin has separate heap/stack limits
- **Error isolation**: Errors in one plugin shouldn't crash others

#### Bytecode Perspective

From the bytecode's perspective, cross-module calls look like native calls:
- The calling code uses `Call_StartFunction` (implemented as a native)
- Parameters are pushed using natives (`Call_PushCell`, etc.)
- `Call_Finish` (a native) performs the actual cross-context call
- The VM doesn't know it's calling another plugin; the host handles it

**Key Point**: Cross-module functionality is a **host feature**, not an SMX bytecode feature. The SMX format only provides the `.publics` section for exporting functions; the host environment implements the cross-call mechanism.

### Error Handling

Runtime errors can occur from:
- **Bounds violations**: BOUNDS check fails
- **Invalid memory access**: Out of bounds memory access
- **Stack overflow/underflow**: Stack limit exceeded
- **Divide by zero**: SDIV with zero divisor
- **Integer overflow**: Special case: `-INT_MIN / -1` causes overflow
- **Unbound native**: Calling native that wasn't provided
- **Invalid opcode**: Unrecognized instruction
- **HALT instruction**: Explicit error with error code
- **Array too big**: Heap allocation request too large
- **Timeout**: Execution time limit exceeded (if watchdog enabled)

Error codes are defined in `sp_vm_types.h` (SP_ERROR_* constants).

VM should provide stack traces using debug info when errors occur.

---

## Type System and RTTI

### RTTI Overview

Runtime Type Information is stored in several optional sections prefixed with `rtti.`:
- `rtti.data`: Blob containing complex type encodings
- `rtti.methods`: Function signatures
- `rtti.natives`: Native function signatures
- `rtti.enums`: Enumeration definitions
- `rtti.enumstructs`: Enum struct definitions
- `rtti.classdefs`: Class/struct definitions
- `rtti.fields`: Struct fields
- And more...

### RTTI Table Structure

Tables have a common header:

```c
struct smx_rtti_table_header {
    uint32_t header_size;  // Size of this header
    uint32_t row_size;     // Size of each row
    uint32_t row_count;    // Number of rows
};
// Followed by row_count rows of row_size bytes each
```

### Type Identifiers

A type ID is a 32-bit value:
- Bits 0-3: Type kind
- Bits 4-31: Type payload

**Type Kinds**:
- **0x0 (Inline)**: Type encoded in payload
- **0x1 (Complex)**: Payload is offset into `rtti.data`

### Control Bytes in rtti.data

Type encodings use these control bytes:

**Simple Types**:
- `0x01`: bool
- `0x06`: int32
- `0x0c`: float32
- `0x0e`: char8
- `0x10`: any (unchecked)
- `0x11`: Function (top function type)

**Complex Types**:
- `0x30`: Fixed array - followed by: size (uint32), element type
- `0x31`: Dynamic array - followed by: element type
- `0x32`: Function - followed by: arg count, variadic flag, return type, param types

**Named Types** (followed by table index):
- `0x42`: Enum
- `0x43`: Typedef
- `0x44`: Typeset
- `0x45`: Classdef
- `0x46`: Enum struct

**Modifiers**:
- `0x70`: void (return type only)
- `0x71`: Legacy variadic
- `0x72`: By-reference parameter
- `0x73`: const

### Variable-Length Integer Encoding

uint32 values in `rtti.data` are encoded with a variable-length encoding to save space:

```
Byte 0: [Continue bit (1)] [7 bits of value]
Byte 1: [Continue bit (1)] [7 bits of value]
...
Last:   [0] [7 bits of value]
```

**Encoding Algorithm**:
```c
uint32_t decode_varuint32(const uint8_t* data) {
    uint32_t value = 0;
    uint32_t shift = 0;
    while (true) {
        uint8_t byte = *data++;
        value |= (byte & 0x7f) << shift;
        if ((byte & 0x80) == 0)  // No continue bit
            break;
        shift += 7;
    }
    return value;
}
```

**Examples**:
- `0x05` = 5 (1 byte)
- `0x80 0x01` = 128 (2 bytes: 0x00|0x01 = 0x80)
- `0xFF 0x7F` = 16383 (2 bytes: 0x7F|0x7F = 0x3FFF)

This encoding is used for:
- Array sizes in type definitions
- Table indices in named types
- Parameter counts in function types

### Method Signatures

The `rtti.methods` table describes functions:

```c
struct smx_rtti_method {
    uint32_t name;         // Name offset in .names
    uint32_t pcode_start;  // Start of function code
    uint32_t pcode_end;    // End of function code
    uint32_t signature;    // Offset in rtti.data
};
```

Signature encoding at rtti.data offset:
```
[num_args: uint8]
[variadic_flag: uint8]  // 0 or kLegacyVariadic
[return_type: type]
[param_1: kByRef? type]
[param_2: kByRef? type]
...
```

### Example Type Encodings

**int32**: `0x06`

**float32[]**: `0x31 0x0c` (array of float)

**int[10]**: `0x30 0x0a 0x06` (fixed array size 10 of int32)

**Function int(float, bool&)**:
```
0x32  // Function
0x02  // 2 parameters
0x00  // Not variadic
0x06  // Return: int32
0x0c  // Param 1: float32
0x72 0x01  // Param 2: by-ref bool
```

---

## Debug Information

### Legacy Debug Sections

Older SMX files may contain:
- `.dbg.info`: Debug info header
- `.dbg.files`: Source files
- `.dbg.lines`: Line number mapping
- `.dbg.symbols`: Variable symbols (legacy format)
- `.dbg.natives`: Native signatures (legacy)
- `.dbg.strings`: Debug string table

### Modern Debug Sections

Newer files use RTTI-based debug:
- `.dbg.methods`: Method to locals mapping
- `.dbg.locals`: Local variable information
- `.dbg.globals`: Global variable information

### Debug Structures

**Debug Info Header** (.dbg.info):
```c
struct sp_fdbg_info_t {
    uint32_t num_files;
    uint32_t num_lines;
    uint32_t num_syms;
    uint32_t num_arrays;
};
```

**File Entry** (.dbg.files):
```c
struct sp_fdbg_file_t {
    uint32_t addr;  // Code address
    uint32_t name;  // String offset
};
```

**Line Entry** (.dbg.lines):
```c
struct sp_fdbg_line_t {
    uint32_t addr;  // Code address
    uint32_t line;  // Line number
};
```

**Debug Variable** (.dbg.locals, .dbg.globals):
```c
struct smx_rtti_debug_var {
    int32_t address;      // Variable address
    uint8_t vclass;       // Variable class (0=global, 1=local, 2=static, 3=arg)
    uint32_t name;        // Name offset
    uint32_t code_start;  // Scope start
    uint32_t code_end;    // Scope end
    uint32_t type_id;     // Type identifier
};
```

### Using Debug Info

To get a stack trace:
1. Look up current CIP in `.dbg.lines` → get line number
2. Look up current CIP in `.dbg.files` → get source file
3. Look up CIP in `rtti.methods` → get function name
4. Walk call stack using FRM chain
5. Repeat for each frame

To inspect variables:
1. Check scope: `code_start <= CIP < code_end`
2. Read variable: Use address and vclass to locate in memory
3. Decode type: Use type_id from RTTI

---

## Implementation Guide

### Minimal VM Requirements

A minimal VM must:

1. **Parse SMX format**
   - Read header, sections
   - Handle optional compression
   - Validate structure

2. **Implement core sections**
   - Load `.code` bytecode
   - Load `.data` segment
   - Read `.natives`, `.publics`, `.names`

3. **Implement instruction set**
   - All generated opcodes (marked `_G` in opcode list)
   - Skip ungenerated opcodes (`_U`)

4. **Provide execution environment**
   - Memory management (CODE, DAT, STK)
   - Register simulation
   - Call/return mechanics
   - Native function binding

5. **Handle errors**
   - Bounds checking
   - Stack overflow detection
   - Runtime error reporting

### Recommended Implementation Steps

#### Phase 1: Parser
1. Read and validate file header
2. Parse section table
3. Decompress if needed
4. Extract required sections

#### Phase 2: Memory Setup
1. Allocate CODE memory (read-only)
2. Allocate DAT memory (data + heap)
3. Allocate STK memory (stack)
4. Initialize DAT from `.data` section

#### Phase 3: Basic Interpreter
1. Implement instruction decoder
2. Implement register state
3. Implement basic opcodes (LOAD, STOR, CONST, etc.)
4. Implement arithmetic operations

#### Phase 4: Control Flow
1. Implement CALL/RETN
2. Implement jumps (JUMP, JZER, etc.)
3. Implement function calls

#### Phase 5: Natives
1. Create native binding API
2. Implement SYSREQ opcodes
3. Provide context for natives to access memory/stack

#### Phase 6: Advanced Features
1. Implement array operations
2. Implement floating point
3. Implement heap management
4. Add error handling

#### Phase 7: Debug Support (Optional)
1. Parse debug sections
2. Implement stack traces
3. Implement source mapping
4. Support variable inspection

#### Phase 8: Multi-Plugin Support (Optional)

For environments that need to run multiple SMX modules:

1. **Plugin Context Management**:
   - Load multiple SMX files
   - Maintain separate execution contexts (CODE/DAT/STK per plugin)
   - Track plugin handles/identifiers

2. **Cross-Module Calls**:
   - Implement function registry for all loaded plugins
   - Create API: `GetFunctionByName(plugin, name)`
   - Implement call marshalling between contexts
   - Handle parameter copying between plugin memory spaces
   - Support copyback for by-reference parameters

3. **Isolation**:
   - Ensure memory isolation between plugins
   - Separate error handling per plugin
   - Independent resource limits

See [Cross-Module Function Calls](#cross-module-function-calls) for implementation details.

### Testing

Test your VM with:
1. **Simple arithmetic**: Test basic operations
2. **Function calls**: Test call/return mechanics
3. **Arrays**: Test array operations
4. **Natives**: Test system calls
5. **Real plugins**: Use actual SourceMod plugins
6. **Error cases**: Test bounds checks, invalid ops

### Performance Considerations

- **Interpreter**: Simple loop over opcodes
  - Direct threading: Jump table per opcode
  - Switch dispatch: Single switch statement
  
- **JIT (Advanced)**: Compile bytecode to native code
  - SourcePawn's reference VM includes JIT for x86/x64
  - Fallback to interpreter on other architectures

- **Memory**: Consider memory protection (CODE read-only)

- **Stack**: Check limits on PUSH/CALL operations

### Reference Implementation

The SourcePawn VM source code is the definitive reference:
- `vm/interpreter.cpp`: Interpreter implementation
- `vm/pcode-reader.h`: Instruction decoder
- `vm/smx-v1-image.cpp`: SMX parser
- `vm/plugin-context.cpp`: Execution context

### Common Implementation Pitfalls

1. **Forgetting alignment**: All code offsets must be 4-byte aligned
2. **Wrong endianness**: SMX is always little-endian
3. **Cell size confusion**: Always 4 bytes, never check cellsize != 4
4. **Stack direction**: Stack grows DOWN (decreasing addresses)
5. **Frame layout**: Return address is at FRM+0, not FRM-4
6. **Parameter access**: First param is count at params[0], not params[1]
7. **Compression**: Must decompress in-place if compression is enabled
8. **Native leaks**: Natives must restore STK/HEP if they modify them
9. **Bounds checks**: BOUNDS checks unsigned comparison (PRI >= limit)
10. **Integer overflow**: Check for `-INT_MIN / -1` special case in SDIV

### Validation Checklist

Before running bytecode, verify:
- [ ] Magic number is `0x53504646`
- [ ] Version is supported (0x0101-0x0107 for SP1)
- [ ] Code version is >= 9 and <= 13
- [ ] All section offsets are within file bounds
- [ ] All name table offsets are valid
- [ ] Entry point (main) is valid code offset
- [ ] Data section size is reasonable
- [ ] All required sections present (.code, .data, .natives, .publics, .names)

### Debugging Tips

1. **Enable opcode tracing**: Print each instruction before execution
2. **Watch register state**: Log PRI/ALT/FRM/STK/HEP after each instruction
3. **Validate memory accesses**: Add bounds checking even if not required
4. **Compare with reference**: Run same plugin on SourcePawn VM and compare
5. **Start simple**: Test with hand-written bytecode before real plugins
6. **Use existing plugins**: SourceMod plugins are great test cases

---

## Appendix A: Complete Opcode Reference

### Opcode Enumeration

Opcodes are numbered sequentially. See `include/smx/smx-v1-opcodes.h` for the complete list. Key opcodes by number:

```
0:   NONE (reserved)
1:   LOAD.pri
2:   LOAD.alt
3:   LOAD.S.pri
4:   LOAD.S.alt
...
```

Each opcode has a fixed size (number of cells including opcode itself).

### Opcode Size Table

Most opcodes are 1-2 cells:
- 1 cell: Nullary operations (LOAD.I, ADD, XOR, etc.)
- 2 cells: Unary operations with one operand (CONST, JUMP, etc.)
- 3 cells: Binary operations with two operands (LOAD.BOTH, CONST, etc.)
- Variable: CASETBL, PUSH2-5 family

---

## Appendix B: Memory Layout Examples

### Example 1: Simple Global and Call

```
.data section (initialized):
  [0] = 100  // global variable

Code:
  PROC              // Function start
  LOAD.pri 0        // PRI = global (100)
  ADD.C 5           // PRI = 105
  STOR.pri 0        // global = 105
  RETN              // Return
```

### Example 2: Function Call

```
Caller:
  PUSH.C 20         // Push argument
  PUSH.C 1          // Push param count
  CALL 100          // Call function at offset 100
  STACK 8           // Clean up 2 cells

Callee (offset 100):
  PROC              // Function prologue
  LOAD.S.pri 12     // Load param 1 (FRM+12)
  ADD.C 10          // Add 10
  RETN              // Return (result in PRI)
```

### Example 3: Array Access

```
Array at DAT+100, access element 3:
  CONST.pri 3       // Index
  CONST.alt 100     // Base address
  IDXADDR           // ALT = 100 + 3*4 = 112
  LOAD.I            // PRI = *(DAT + 112)
```

---

## Appendix C: Version History

### Code Versions
- **Version 9**: Original SourcePawn 1.0
- **Version 10**: DEBUG flag removed
- **Version 13**: Feature flags, INITARRAY, HEAP_SAVE/RESTORE

### File Versions
- **0x0101**: SourceMod 1.0
- **0x0102**: SourceMod 1.1 - JIT improvements
- **0x0107**: SourceMod 1.7 - Transitional syntax, improved RTTI

---

## Appendix D: Common Patterns

### Pattern: String Copy
```
MOVS size        // Copy 'size' bytes from PRI to ALT
```

### Pattern: Array Fill
```
FILL count       // Fill 'count' cells at ALT with PRI
```

### Pattern: Bounds Check
```
CONST.pri index
BOUNDS array_size  // Error if index >= array_size
```

### Pattern: Reference Parameter
```
// Caller:
ADDR.pri offset   // Get address of variable
PUSH.pri          // Push address
...

// Callee:
LOAD.S.pri 8      // Load address
// Now can use STOR.I to modify through reference
```

### Pattern: Multi-dimensional Array Access

For `array[i][j]` where each dimension is size `dim1` and `dim2`:
```
// Given: array base at DAT+base, i in PRI, j in ALT
CONST.pri i         // i
SMUL.C dim2         // i * dim2
ADD                 // (i * dim2) + j
SHL.C.pri 2         // ((i * dim2) + j) * 4 (cellsize)
ADD.C base          // base + offset
MOVE.alt            // Address in ALT
LOAD.I              // Load value
```

---

## Glossary

- **Cell**: Basic 32-bit data unit in SourcePawn
- **DAT**: Data segment containing globals and heap
- **STK**: Stack segment for call frames and local variables
- **PRI/ALT**: Primary and alternate general-purpose registers
- **FRM**: Frame pointer for current call frame
- **HEP**: Heap pointer (allocation point in DAT)
- **CIP**: Code instruction pointer (program counter)
- **Native**: Host-provided function callable from SMX
- **Public**: SMX function callable from host
- **RTTI**: Runtime Type Information
- **Pcode**: P-code, the bytecode format used by SourcePawn

---

## References

1. SourcePawn GitHub Repository: https://github.com/alliedmodders/sourcepawn
2. SourceMod Documentation: https://wiki.alliedmods.net/
3. SMX Header Files:
   - `include/smx/smx-headers.h`: Container format
   - `include/smx/smx-v1.h`: SMX v1 structures
   - `include/smx/smx-v1-opcodes.h`: Opcode definitions
   - `include/smx/smx-typeinfo.h`: RTTI structures
4. VM Implementation:
   - `vm/interpreter.cpp`: Reference interpreter
   - `vm/smx-v1-image.cpp`: SMX file parser

---

*This documentation is based on SourcePawn 1.12 (November 2024). For the most current information, refer to the source code in the repository.*
