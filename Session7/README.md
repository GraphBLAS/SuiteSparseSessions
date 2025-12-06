# Session 7 Summary

## Table of Contents
- [Overview](#overview)
- [JIT Control States and Error Handling](#jit-control-states-and-error-handling)
  - [JIT Control Levels](#jit-control-levels-000300)
  - [Error Handling Strategy](#error-handling-strategy-000133)
- [Kernel Loading and Validation](#kernel-loading-and-validation)
  - [Loading Process](#loading-process-002251)
  - [Kernel Query Function](#kernel-query-function-002603)
- [Critical Sections and Locking](#critical-sections-and-locking)
  - [Thread Safety](#thread-safety-001632)
- [Macro System and Code Generation](#macro-system-and-code-generation)
  - [Macrify Family Functions](#macrify-family-functions-003000)
  - [Type System](#type-system-004758)
  - [Operator Macrofication](#operator-macrofication-005900)
- [Sparsity Format Handling](#sparsity-format-handling)
  - [Format Macros](#format-macros-012900)
- [Typecasting Implementation](#typecasting-implementation)
  - [Why Custom Typecasting](#why-custom-typecasting-014840)
  - [Cast Function Generation](#cast-function-generation-015200)
- [Preface and Include System](#preface-and-include-system)
  - [User-Defined Preface](#user-defined-preface-003210)
- [Compilation Process](#compilation-process)
  - [Three Compilation Methods](#three-compilation-methods-021015)
  - [Compiler Configuration](#compiler-configuration-021700)
- [JIT Kernel Structure](#jit-kernel-structure)
  - [Runtime vs Pre-JIT Kernels](#runtime-vs-pre-jit-kernels-015745)
  - [Template System](#template-system-020300)
- [Algorithm Example: Apply Bind-First](#algorithm-example-apply-bind-first)
  - [Simple Algorithm](#simple-algorithm-020320)
- [Source Code Organization](#source-code-organization)
  - [JIT Package](#jit-package-022223)
  - [File Organization](#file-organization)
- [User-Defined Types and Masks](#user-defined-types-and-masks)
  - [Mask Type Restrictions](#mask-type-restrictions-013200)
  - [Arbitrary Typecasting Debate](#arbitrary-typecasting-debate-013420)
- [Operator Dependency Analysis](#operator-dependency-analysis)
  - [Future Enhancement Idea](#future-enhancement-idea-005020)
- [Hash Table Implementation](#hash-table-implementation)
  - [Hash Function](#hash-function-022400)
  - [Pre-hashing Strategy](#pre-hashing-strategy-010802)
- [Encoding System](#encoding-system)
  - [Bit-Packed Encoding](#bit-packed-encoding-004040)
  - [Type Codes](#type-codes-004847)
- [Hash Code and Kernel Names](#hash-code-and-kernel-names)
  - [Unique Identification](#unique-identification-010614)
- [Lessons and Design Philosophy](#lessons-and-design-philosophy)
  - [Template Reuse](#template-reuse-020312)
  - [Macro Layering Philosophy](#macro-layering-philosophy-014300)
  - [Windows Challenges](#windows-challenges-021900)
- [Performance Considerations](#performance-considerations)
  - [Function Pointer Overhead](#function-pointer-overhead-012000)
  - [JIT vs Generic Trade-offs](#jit-vs-generic-trade-offs-000400)
  - [Compilation Speed](#compilation-speed-021400)
- [Future Work and Open Questions](#future-work-and-open-questions)
  - [Items Not Fully Covered](#items-not-fully-covered)
  - [Known Issues](#known-issues)
  - [Enhancement Requests](#enhancement-requests)
- [Key Insights](#key-insights)

---

## Overview
This is the third session on the JIT (Just-In-Time) compiler in SuiteSparse GraphBLAS. The session provides a deep dive into the JIT compilation infrastructure, focusing on how GraphBLAS generates, compiles, and manages JIT kernels at runtime.

---

## JIT Control States and Error Handling

### JIT Control Levels [00:03:00]
[![00:03:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=180s)
There are 5 distinct states for JIT control:
1. **On** - Can load and compile
2. **Load** - Can load from disk but cannot compile
3. **Run** - Can use kernels already in memory (dlopen) but cannot load from disk
4. **Pause** - Keep kernels in memory but don't call them
5. **Off** - Flush everything, dlclose all kernels

**Key Quote**: "If I fail to compile, I ratchet down to we can only load. If I fail to load, I ratchet down to run." [00:03:46]

### Error Handling Strategy [00:01:33]
[![00:01:33](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=93s)
When compilation fails (e.g., user-defined operator with typo), GraphBLAS automatically downgrades the JIT state to prevent repeated compilation attempts. This prevents the system from trying to compile invalid code thousands of times.

**Issue**: Need better user control over JIT failure behavior. Currently returns GRB_NO_VALUE internally, but may need new error codes like GRB_JIT_FAIL for development scenarios where users want to know compilation failed. [00:04:16]

**Panic vs Failures**: GRB_PANIC is too extreme - it means "shut off GraphBLAS entirely, something catastrophic happened." JIT failures need a less severe error code. [00:06:48]

---

## Kernel Loading and Validation

### Loading Process [00:22:51]
[![00:22:51](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=1371s)
1. Construct .so filename from hash code and kernel name
2. Attempt `dlopen()` on the library
3. If successful, query the kernel to verify it's valid
4. If stale or invalid, remove the file and recompile

**Cache Organization**: [00:24:05]
- Files organized into 256 buckets (8-bit number) to avoid huge single directories
- For test coverage: 22,000 JIT kernels generated
- Path format: `~/.suitesparse/grb-9.3.0/<bucket>/<lib><kernel_name>.so`

### Kernel Query Function [00:26:03]
[![00:26:03](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=1563s)
Each JIT kernel has two symbols:
1. `jit_query` - Validates the kernel is correct version
2. `jit_kernel` - The actual kernel function

If query fails or kernel is stale, the file is removed and recompiled. [00:26:51]

---

## Critical Sections and Locking

### Thread Safety [00:16:32]
[![00:16:32](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=992s)
- **Critical sections** protect against multiple threads within same process
- **File locking** protects against multiple processes accessing same kernel

**Locking Strategy**: [00:16:50]
- Only one thread per process can JIT at a time (critical section)
- Different processes can JIT independently
- File locks prevent two processes from compiling same kernel simultaneously

**Current Behavior**: If file lock fails, JIT is turned off rather than waiting. This is a "fix me" area that could be improved with configurable wait/no-wait options similar to PostgreSQL. [00:17:00]

---

## Macro System and Code Generation

### Macrify Family Functions [00:30:00]
[![00:30:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=1800s)
The code generation uses extensive macro preprocessing to generate specialized kernels. Main families:
- Apply
- Assign
- Build
- E-wise (element-wise)
- Matrix multiply (MxM)
- Reduce to scalar
- Select
- User-defined operators/types

### Type System [00:47:58]
[![00:47:58](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=2878s)
**Six potential types in operations**:
- C, A, B matrix types (type1, type2, type3)
- X, Y, Z semiring types (multiply inputs and output)

For MxM: Can have 6 different types due to typecasting between matrix types and semiring types.

**Type Macrofication**: [00:55:00]
Only unique types are output to avoid duplication. Uses string comparison to detect when types are identical and only generates typedef once with guards (`#ifndef`).

### Operator Macrofication [00:59:00]
[![00:59:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=3540s)

**Binary Operators**: Two paths:
1. **Built-in operators** - Simple inline definitions (e.g., `z = x ^ ~y` for bitwise XNOR)
2. **User-defined operators** - Function calls with full definitions guarded by `#ifndef`

**Flipping operators**: Handled in macros by swapping X,Y parameters rather than creating new operators. [01:11:55]

---

## Sparsity Format Handling

### Format Macros [01:29:00]
[![01:29:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=5340s)
Each matrix has accessor macros that adapt to its sparsity format:
- **Sparse**: Look up indices in arrays
- **Hyper-sparse**: Additional indirection through hyperlist
- **Bitmap**: Check bitmap for presence
- **Full**: Computed positions (multiplication and modulo)

**Example**: Getting row index
- Sparse: `I = Ai[p]` (array lookup)
- Full: `I = p % nrows` (modulo computation)

**Bitmap Optimization**: [02:06:02]
Single line `if (!GB_B_IS_PRESENT(p)) continue;` handles bitmap case. For non-bitmap matrices, this macro evaluates to `if (!1) continue;` which compiler optimizes away entirely.

---

## Typecasting Implementation

### Why Custom Typecasting [01:48:40]
[![01:48:40](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=6520s)

**Problem 1 - Complex to Bool**: [01:48:28]
Windows doesn't properly handle casting from complex types to bool. GraphBLAS manually checks real and imaginary parts against zero.

**Problem 2 - Float to Integer Overflow**: [01:49:11]
ANSI C leaves casting from out-of-range float to integer as undefined behavior. Different compilers/platforms return different garbage values.

**Solution**: Custom cast functions that return sensible values:
- NaN → 0 (zero being "not a number" historically)
- Negative value to unsigned → 0
- Overflow → INT_MAX or equivalent

**Quote**: "Matlab doesn't do that. They define it. So if you cast a floating point infinity to int32, you get int32 max. That's a pretty decent answer. If you do that in ANSI C, you get 42 or something ridiculous." [01:50:13]

### Cast Function Generation [01:52:00]
[![01:52:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=6720s)
Special functions like `GB_jit_cast_double_to_uint16_t` handle all edge cases with proper range checking before calling standard C cast.

---

## Preface and Include System

### User-Defined Preface [00:32:10]
[![00:32:10](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=1930s)
Users can set custom `#include` directives and comments for JIT kernels via `GrB_set()`:
- C preface for CPU/OpenMP kernels
- CUDA preface for GPU kernels (separate)

**Example**:
```c
// Kernel by gauss_demo.c
#include <math.h>
```

This allows user-defined operators to include necessary headers without GraphBLAS knowing about them in advance.

---

## Compilation Process

### Three Compilation Methods [02:10:15]
[![02:10:15](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=7815s)

1. **Direct compiler call** (Linux/Mac default)
   - Single `system()` call with full compile+link command
   - Faster than CMake
   - Uses shell-based command construction

2. **CMake** (Windows default, optional elsewhere)
   - Creates temporary CMakeLists.txt
   - Configures, builds, installs
   - Removes build directory when done
   - More portable but slower

3. **NVCC** (CUDA kernels)
   - Direct shell out to nvcc
   - Single command for compile and link

### Compiler Configuration [02:17:00]
[![02:17:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=8220s)
- Default: Uses same compiler that built GraphBLAS (from `GB_config.h`)
- Customizable via `GrB_set()` to specify different compiler
- C flags can be augmented (e.g., adding `-I` for custom include paths)

**Platform Differences**: [02:19:00]
- Linux/Mac: Can use direct compiler calls with standard flags
- Windows: Must use CMake (with MSVC) or CMake with MinGW
- Windows doesn't support direct shell-out compilation method

---

## JIT Kernel Structure

### Runtime vs Pre-JIT Kernels [01:57:45]
[![01:57:45](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=7065s)

**Flag**: `GB_JIT_RUNTIME`
- Defined: Kernel compiled at runtime by JITifyer
- Not defined: Pre-compiled kernel baked into `libgraphblas.so`

**Pre-JIT kernels**: [01:59:30]
- Copied into `source/prejit` folder
- Compiled by CMake during GraphBLAS build
- Each has unique function name (not just `GB_jit_kernel`)
- Stored in tables: kernel names, query pointers, kernel pointers
- Loaded into hash table at `GrB_init()` time

### Template System [02:03:00]
[![02:03:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=7380s)
Kernels are just templates - they have no function headers, just code bodies:
```c
// No function signature here
{
    // Algorithm starts directly
    for (int64_t p = 0; p < bnz; p++)
    {
        GB_DECLARE_B(bij);
        GB_GET_B(bij, Bx, p);
        GB_EWISEOP(Cx, p, bij, i, j);
    }
}
```

The function signature is defined in separate wrapper file that `#include`s the template.

---

## Algorithm Example: Apply Bind-First

### Simple Algorithm [02:03:20]
[![02:03:20](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=7400s)
```c
for (int64_t p = 0; p < bnz; p++)
{
    if (!GB_B_IS_PRESENT(p)) continue;  // Bitmap handling
    GB_DECLARE_B(bij);                   // Type declaration
    GB_GET_B(bij, Bx, p);                // Load value (with typecast)
    GB_EWISEOP(Cx, p, bij, i, j);        // Apply operator
}
```

**All complexity hidden in macros**:
- `GB_B_IS_PRESENT(p)` - Returns true for non-bitmap, checks bitmap for bitmap format
- `GB_DECLARE_B(bij)` - Declares variable of correct type (possibly after typecast)
- `GB_GET_B(bij, Bx, p)` - Loads value with typecasting if needed
- `GB_EWISEOP` - Applies the binary operator (expands to user function or built-in op)

**Quote**: "The algorithm is trivially simple. But everything in this algorithm goes through these macros." [02:05:15]

---

## Source Code Organization

### JIT Package [02:22:23]
[![02:22:23](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=8543s)
All source code bundled inside `libgraphblas.so` as compressed binary using zstandard compression. On first use, JITifyer decompresses and writes to cache folder.

**Cache Contents** (~27,000 lines):
- 244 files total
- `graphblas.h` header
- All include files
- All templates (CPU and CUDA mixed together)
- Template directory: ~14,000 lines of algorithms and kernels

**Size Perspective**: [02:31:00]
- GraphBLAS total: ~120,000 lines
- JIT kernels: ~1/8 of total codebase
- Much work in GraphBLAS doesn't need to be JITted

### File Organization

**CPU Side** (well organized):
- `source/jitifyer/` - Main JIT infrastructure
- `source/jitifyer/GB_jit_kernel_*.c` - 44 kernel wrappers
- `source/Template/` - 44 kernel templates

**CUDA Side** (needs reorganization): [02:28:00]
- 8 JIT wrappers in cuda folder
- 8 kernel templates mixed in template folder
- Needs reorganization to match CPU structure

---

## User-Defined Types and Masks

### Mask Type Restrictions [01:32:00]
[![01:32:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=5520s)

**Current limitation**: Cannot use valued mask with user-defined types. User-defined types cannot be typecast to bool (no typecast functions defined).

**Structural masks work fine** - only pattern matters, not values.

**Potential enhancement**: [01:32:40]
Allow users to register a "cast to bool" function for user-defined types. This would enable valued masks with user types.

**Quote**: "It's a hole in the spec. It's not a bug. It's just by design. It's not a super nasty limitation." [01:33:03]

**Ray's request**: Add support for typecast from user-defined type to bool. [01:32:53]

### Arbitrary Typecasting Debate [01:34:20]
[![01:34:20](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=5660s)

**Slippery slope concern**: If we add bool typecast, why not all typecasts?
- User type → 12 built-in types = 12 functions
- User type → other user types = unlimited functions
- Management nightmare: "My brain explodes" [01:35:20]

**Decision**: Only add bool typecast if there's a clear use case. Don't venture into use cases we don't have. [01:35:56]

---

## Operator Dependency Analysis

### Future Enhancement Idea [00:50:20]
[![00:50:20](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=3020s)

**Problem**: Index unary operators receive 4 parameters (value, row, column, theta) but may only use subset. GraphBLAS still does work to compute all 4.

**Potential solution**: Let users declare which parameters operator actually needs:
```
GrB_set(op, "I don't use the X value")
GrB_set(op, "I don't use row index")
```

**Benefit**: Skip unnecessary work (finding row/column indices, loading values).

**Challenge**: How to validate this? Can't trust user to be correct (would cause null dereferences). Compiler dependency analysis is complex.

**Quote**: "It'd be nice to know. I don't have a mechanism for this yet." [00:52:28]

---

## Hash Table Implementation

### Hash Function [02:24:00]
[![02:24:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=8640s)
Uses XX hash on the encoding struct to generate 64-bit hash.

**Special hash values**:
- `0` - Built-in operator, no need to hash
- `UINT64_MAX` - Not jittable (e.g., missing user-defined strings)

**Hash collision handling**: [02:25:26]
If XX hash returns 0 or UINT64_MAX by chance (very rare), return magic number instead to preserve special meanings.

**Magic number**: Arbitrary 64-bit value used to detect if matrix has been initialized/freed. [02:26:10]

### Pre-hashing Strategy [01:08:02]
[![01:08:02](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=4082s)
Every operator, type, semiring, monoid has its own hash code computed in advance. Final hash is built by XOR-ing these together rather than rehashing everything each time.

---

## Encoding System

### Bit-Packed Encoding [00:40:40]
[![00:40:40](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=2440s)

**Example - Element-wise (48 bits)**:
- Format of C, M, A, B: 8 bits (2 bits each for 4 matrices)
- Binary operator: 20 bits (5 hex digits)
  - Operator code: 8 bits
  - X, Y, Z types: 4 bits each
- Additional flags and parameters

**Hexadecimal alignment**: [00:41:07]
Carefully designed so bit boundaries align with hex digit boundaries for easier debugging and parsing.

### Type Codes [00:48:47]
[![00:48:47](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=2927s)
- Built-in types: 1-12 (int, float, double, etc.)
- User-defined: 0xE (14) = "user-defined, don't know which"
- 0 = void (type not used by operator)

**Example**: `0xEEE` means all three types (X, Y, Z) are user-defined. Which specific user type? That's in the operator's string definition. [01:05:23]

---

## Hash Code and Kernel Names

### Unique Identification [01:06:14]
[![01:06:14](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=3974s)

**Hash determines everything**: If operator uses user-defined types, cannot typecast them, so operator name alone determines X, Y, Z types.

**Kernel naming**: [00:24:40]
```
lib + GB_jit_<operation>_<encoding>.so
```

Example: `libGB_jit_ewise_bind1st_4000eee0efec1.so`
- Operation: ewise_bind1st
- Encoding: 4000eee0efec1 (hex representation of bit encoding)

**Pre-JIT naming**: [01:59:40]
When compiled into GraphBLAS library, kernel functions must have full unique names (not just `GB_jit_kernel`) to avoid symbol conflicts.

---

## Lessons and Design Philosophy

### Template Reuse [02:03:12]
[![02:03:12](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=7392s)
Same template used for:
- Factory kernels (built-in specializations)
- Generic kernels (fallback)
- JIT kernels (runtime generated)

All differ only in macro definitions and compilation flags.

### Macro Layering Philosophy [01:43:00]
[![01:43:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=6180s)
Multiple layers of macros calling macros:
- `GB_EWISEOP` → `GB_BINOP` → `GB_ADD` → `add_gauss(x,y)`
- Each layer handles different concerns (operation, typecasting, flipping)
- Allows one template to handle all variations

**Quote**: "Layers upon layers, macros upon macros. What is the bin op? Oh, the bin op is just add. This bin op is one step removed. This ewise op is like 2 steps removed from the actual operator." [01:43:28]

### Windows Challenges [02:19:00]
[![02:19:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=8340s)
- Can't use standard shell commands
- Forward slash vs backslash issues
- CMake required (slower but more portable)
- MinGW build system has special quirks

**Quote about Windows**: "Of course, because why not put an underscore before a function is called by the user? Really." [00:19:46] (referring to `_lockfile` vs `lockfile`)

---

## Performance Considerations

### Function Pointer Overhead [01:20:00]
[![01:20:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=4800s)
For iso-valued matrices, reduce-to-scalar only needs to call operator log(N) times maximum (via recursive doubling). Performance of function pointer call doesn't matter.

**Example**: Computing X^N via repeated squaring calls operator only log2(N) times. [01:22:00]

### JIT vs Generic Trade-offs [00:04:00]
[![00:04:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=240s)
If JIT fails and falls back to generic kernel, it may be slow but correct. However, user may want to know JIT failed during development. Need configurability.

### Compilation Speed [02:14:00]
[![02:14:00](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ&t=8040s)
- Direct compiler call: Fast (one `system()` call)
- CMake: Slower (5 `system()` calls minimum)
- Why 22,000 CMake projects in test coverage? Each JIT kernel gets own CMakeLists.txt in build folder (deleted after)

---

## Future Work and Open Questions

### Items Not Fully Covered
- Hash table insertion/lookup details [02:23:28]
- Pre-JIT initialization process [02:22:01]
- Source code extraction from zstandard compressed package [02:22:20]
- CUDA atomic operations enumification [02:26:54]
- Complete sanitization of user-provided paths [02:21:26]

### Known Issues
- File locking should retry with timeout instead of immediately disabling JIT [00:17:14]
- Need error code for JIT failures (not panic, not no-value) [00:09:00]
- CUDA folder needs reorganization to match CPU structure [02:28:41]
- Windows could potentially use direct compiler calls with proper flag translation [02:20:00]

### Enhancement Requests
- User-defined type to bool typecasting [01:32:40]
- Operator dependency declarations (which parameters actually used) [00:52:28]
- Better control over JIT failure behavior [00:05:58]

---

## Key Insights

1. **Defensive JIT**: System automatically downgrades capabilities when compilation fails to avoid repeated failures
2. **Macro complexity hides algorithm simplicity**: Most kernels are simple loops, but macro preprocessing makes them work for all type/format combinations
3. **Pre-hashing optimization**: Avoid rehashing by caching hash values for all components
4. **Template versatility**: Same code serves factory, generic, and JIT kernels through different macro definitions
5. **Platform portability is hard**: Significant differences between Linux/Mac/Windows require different compilation strategies
6. **Type safety through macros**: Void type used to make accessing unused values a compile error rather than runtime error

**Session Conclusion**: This session covered the complete JIT infrastructure from hash computation through code generation, compilation, and loading. The next session will focus more on the actual algorithms themselves. [02:09:00]
