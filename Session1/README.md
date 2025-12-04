# Session 1 Summary

Interview with Dr. Tim Davis about SuiteSparse GraphBLAS code organization and architecture.

## Table of Contents
- [Code Organization & Restructuring](#code-organization--restructuring)
  - [Source Code Reorganization](#source-code-reorganization-000116)
  - [File Organization Conventions](#file-organization-conventions-000222)
  - [Module Structure](#module-structure-001100)
- [Code Generation & Factory Kernels](#code-generation--factory-kernels)
  - [Code Generation System](#code-generation-system-003400)
  - [Five Types of Kernels](#five-types-of-kernels-000530)
- [Kernel Selection & Execution Flow](#kernel-selection--execution-flow)
  - [Reduce to Scalar Example](#reduce-to-scalar-example-004400)
- [JIT System Architecture](#jit-system-architecture)
  - [JIT Compilation Process](#jit-compilation-process-012400)
  - [JIT Package Location](#jit-package-location-012900)
  - [Pre-JIT Kernels](#pre-jit-kernels-014200)
- [Data Structures](#data-structures)
  - [Matrix Pending States](#matrix-pending-states-005500)
  - [Iso-Valued Matrices](#iso-valued-matrices-010540)
  - [Four Matrix Formats](#four-matrix-formats-015945)
- [Macros & Templates](#macros--templates)
  - [Monoid Macros](#monoid-macros-011300)
  - [Generic Kernel Macros](#generic-kernel-macros-014800)
- [Workspace Management](#workspace-management)
  - [Werk Space](#Werk-space-005120)
- [API & Spec Issues](#api--spec-issues)
  - [C Scalars Problem](#c-scalars-problem-015200)
  - [Reduce to Scalar API](#reduce-to-scalar-api-015005)
- [Testing Strategy](#testing-strategy-020300)
- [Build System](#build-system)
  - [Compact Mode](#compact-mode-000530)
  - [CMake Configuration](#cmake-configuration-004400)
- [Future Directions](#future-directions)
  - [Next Session Topics](#next-session-topics-015900)
  - [GPU Heuristics](#gpu-heuristics-012100)
- [Technical Details](#technical-details)
  - [Variable Length Arrays](#variable-length-arrays-005830)
  - [Query Functions](#query-functions-013700)
  - [Terminal Monoids](#terminal-monoids-013210)
- [Compile Time Statistics](#compile-time-statistics)

---

## Code Organization & Restructuring

### Source Code Reorganization [00:01:16]

Dr. Davis has recently reorganized the GraphBLAS source code from a single flat directory into a modular structure to make it easier for contributors to navigate.

**Key Changes:**
- Previously: All source files in one directory (difficult to navigate)
- Now: Organized into logical modules with subdirectories (apply, assign, reduce, mxm, etc.)
- Each module contains three consistent subdirectories: `factory/`, `include/`, and `template/`

### File Organization Conventions [00:02:22]

**Template Files:**
- Use `#include` to include template source code
- Naming convention: Files in `template/` subdirectories
- Distinction: `.h` files are headers, template files are included for code generation

### Module Structure [00:11:00]

Each module represents a GraphBLAS operation (apply, assign, reduce, mxm, etc.). Some modules are:
- **Primary modules**: Support end-user operations (apply, assign, etc.)
- **Helper modules**: Internal utilities (alias, memory management, etc.)
- **Operator modules**: Define operators and monoids (binary_op, monoid, etc.)

## Code Generation & Factory Kernels

### Code Generation System [00:34:00]

GraphBLAS uses MATLAB scripts to generate factory kernels at development time.

**Statistics:**
- Factory kernels: ~750,000 lines of generated C code
- Generator code: Only ~4,600 lines of MATLAB
- Result: 22,000+ factory kernel files

### Five Types of Kernels [00:05:30]

1. **Non-disableable kernels**: Basic operations that can't be turned off (like iso-valued operations)
2. **Factory kernels**: Large switch-case implementations, ~1,000 variants for different operators/types
3. **CPU JIT kernels**: Compiled at runtime or loaded from cache
4. **CUDA JIT kernels**: GPU kernels with same architecture as CPU JIT
5. **Generic kernels**: Slowest fallback using function pointers and memcpy for everything

**Quote:** "For a generic kernel, every multiply, every add, every typecast, even typecast double to int is through a function call pointer.  They are slow, but allow GraphBLAS to compute any operation without the need to shell out to a compiler for the JIT.  Some platforms can't do that (the Apple iPad) and some end-user applications will not want to allow it.  To get good performance, an application developer can run the app in a development and allow GraphBLAS to compile its JIT kernels.  Then those JIT kernels can be converted into Pre-JIT kernels, which I'll describe later."

## Kernel Selection & Execution Flow

### Reduce to Scalar Example [00:44:00]

The execution flow for `GrB_reduce` demonstrates the kernel selection hierarchy:

1. **Input validation**: Check for null pointers, invalid objects, type compatibility [00:53:00]
2. **Wait/finish operations**: Handle pending work (zombies, pending tuples, jumbled entries) [00:55:00]
3. **CUDA kernel**: First priority if available and appropriate [01:00:00]
4. **Iso kernel**: Special fast path for iso-valued matrices [01:05:00]
5. **Factory kernel**: Pre-compiled type-specific kernels [01:08:00]
6. **CPU JIT kernel**: Compile or load specialized kernel [01:24:00]
7. **Generic kernel**: Final fallback using function pointers [01:47:00]

**Quote:** "CUDA gets the first dibs on the problem. The CUDA heuristic can choose to do it on the CPU because it's iso, or it has too many zombies, or it's a tiny amount of work."

## JIT System Architecture

### JIT Compilation Process [01:24:00]

**Encoding & Hashing:**
- Each kernel is encoded into a compact bit representation
- Hash value computed for quick lookup in hash table
- JIT loader checks hash table before compiling

### JIT Package Location [01:29:00]

- Default location: `~/.Suitesparse/` (or Windows equivalent)
- Version-specific: Each GraphBLAS version gets its own cache
- Two subdirectories: `src/` for generated C files, `lib/` for compiled binaries

### Pre-JIT Kernels [01:42:00]

Users can "pre-compile" JIT kernels into the library:
1. Run application with JIT enabled
2. Copy generated kernels to GraphBLAS source tree
3. Recompile GraphBLAS - kernels are now baked in
4. Can disable runtime JIT for production

**Quote:** "It's very easy to specialize GraphBLAS with the jit. Run it once, find all the kernels you need, slap them in and you're done. You can now turn off the jit for production use if you like."

## Data Structures

### Matrix Pending States [00:55:00]

Matrices can have three types of pending work:
- **Zombies**: Pending deletions
- **Pending tuples**: Pending insertions
- **Jumbled**: Entries not in sorted order in any given row/column

Operations can be tolerant of these states (e.g., reduce to scalar doesn't need sorted entries).

**Quote:** "The reduce to scalar is tolerant of zombies. It'll just skip over them, and it doesn't care if the entries in every row are in ascending order of column index, so the input matrix can stay jumbled."

### Iso-Valued Matrices [01:05:40]

A key optimization for matrices where all present entries have the same value:
- Store only one value instead of array of identical values
- Example: Unweighted graph (all entries = 1)
- Reduces 1 billion values to a single stored value
- Reduction requires only log₂(n) operations

**Quote:** "I don't need to store a billion ones. I just store one of them with a flag that says they're all the same."

### Four Matrix Formats [01:59:45]

GraphBLAS supports four fundamental sparse matrix formats:
- **Sparse** (CSR/CSC)
- **Hypersparse** (CSR/CSC with additional indirection)
- **Bitmap** (dense with bitmap mask)
- **Full** (pure dense)

Each can be stored by row or column, and can be iso-valued, leading to 16 variants.

## Macros & Templates

### Monoid Macros [01:13:00]

Factory and JIT kernels use extensive macro systems to specialize operations. Each monoid defines:
- `GB_ADD`: The addition operation
- `GB_UPDATE`: In-place update operation
- `GB_IDENTITY`: Identity value
- `GB_TERMINAL`: Early-exit value for terminal monoids
- Panel size optimizations for reduce-to-scalar

**Example comparison:**
- Plus monoid: `z += y`
- Min monoid: `z = (z < y) ? z : y` (with NaN handling)

### Generic Kernel Macros [01:48:00]

Generic kernels use the same templates as factory/JIT but with slower implementations:
- Factory/JIT: `z += y` (direct operation)
- Generic: Function pointer call through monoid's add function
- All assignments become memcpy operations

## Workspace Management

### Werk Space [00:51:20]

GraphBLAS uses a statically-allocated workspace on the stack:
- Size: ~32KB
- Purpose: Avoid malloc for small temporary allocations
- Managed with push/pop mechanism
- Used for scalars, small thread-local arrays

**Etymology note:** "Werk" is German for "work" - naming plays on "workspace" → "Werkspace" → "Werk".
It is pronounced as "verk".  I use the word to make it easier to grep the source code to find
where the Werkspace is used.

## API & Spec Issues

### C Scalars Problem [01:52:00]

The GraphBLAS C API has extensive bloat from supporting both `GrB_Scalar` and C scalar types:

**Impact:**
- 89 variants of `GrB_apply` (should be ~4-5)
- Massive macro complexity for polymorphic dispatch
- Complex switch cases for type checking

**Quote:** "C Scalars should be banished forever from all GraphBLAS methods, but that would require a backward-breaking change to the C API."

**Proposal:** Remove C scalar support in GraphBLAS 3.0, use only `GrB_Scalar` objects.

### Reduce to Scalar API [01:50:05]

Unlike other GraphBLAS operations, reduce to scalar has no mask parameter (scalar mask would be pointless). This requires special-case accumulator handling.

## Testing Strategy [02:03:00]

GraphBLAS testing uses a dual-implementation approach:

**Method:**
1. Implement entire GraphBLAS spec in "slow" MATLAB
2. Store matrices as two dense matrices (values + presence boolean)
3. Call both C library and MATLAB implementation
4. Compare results with `isequal`
5. Use for statement coverage testing

**Quote:** "I wrote the whole GraphBLAS spec as raw MATLAB. It's super slow, it's not efficient, but it is simple, easy to understand, and thus easy to verify visually. I don't care that it's slow, ... but it works and is correct."

## Build System

### Compact Mode [00:05:30]

GraphBLAS can be compiled in "compact mode" to reduce binary size:
- Disable factory kernels (per-operator, per-type control)
- Used by FalkorDB for smaller binaries and faster compilation
- Falls back to JIT (fast, but requires a compiler) or generic kernels (slower but functional)

### CMake Configuration [00:44:00]

The main GraphBLAS header is auto-generated by CMake from a template:
- Adds version numbers
- Detects CUDA availability
- Handles platform-specific complex type support
- Sets compilation date

## Future Directions

### Next Session Topics [01:59:00]

Suggested topics for future sessions:
1. **Transpose operation**: Smaller than mxm, introduces data structure details
2. **Data structures**: The GrB_Matrix internal representation
3. **JIT deep dive**: Detailed walk-through of CPU and CUDA JIT systems
4. **CUDA kernels**: GPU-specific implementation details

### GPU Heuristics [01:21:00]

Current GPU selection is simple (if available, use it). Future improvements:
- Track which pieces of matrix were last accessed on GPU
- Use "stickiness" with hysteresis to avoid thrashing
- User hints via get/set options for matrix placement
- Consider problem size for automatic CPU/GPU selection

**Quote:** "My plan is that within a matrix to keep track of, where's the last time I touched this piece of the matrix... and that'll up the chance that I would do this work on the GPU if it's already there."

## Technical Details

### Variable Length Arrays [00:58:30]

GraphBLAS uses C variable-length arrays for scalars of unknown type:
- Most compilers: True VLA based on type size
- MSVC: Fixed 1024-byte array (doesn't support VLA)
- Macro `GB_VLA` handles platform differences

### Query Functions [01:37:00]

Every JIT kernel library exports two functions:
1. **Kernel function**: The actual computation
2. **Query function**: Returns kernel metadata
   - Hash value
   - User-defined type definitions (as strings)
   - User-defined function definitions (as strings)
   - Identity/terminal values

Used to detect when user types/operators change and recompilation is needed.

### Terminal Monoids [01:32:10]

Some monoids can short-circuit (e.g., logical AND):
- Terminal value: false (for AND)
- Algorithm checks for terminal and exits early
- Significant speedup for early-exit cases

**Quote:** "The moment you see a false, we can stop... Some algorithms, particularly reduced to scalar, are going to exploit that and check, and we'll quit the work early."

## Code Size Statistics

- Total GraphBLAS source: ~16,000 lines for mxm alone
- Factory kernels: ~750,000 lines (generated from 4,600 lines)
- Reduction module: ~1,000 lines
- Matrix clear: 91 lines (simplest module)

