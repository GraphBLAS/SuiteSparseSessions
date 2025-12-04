# Session 6 Summary

Interview with Dr. Tim Davis about SuiteSparse GraphBLAS JIT compilation system internals.
Session Date: August 1, 2024

## Table of Contents
- [Overview](#overview)
- [JIT Architecture and Global State Management](#jit-architecture-and-global-state-management)
  - [Global State and Critical Sections](#global-state-and-critical-sections-000600---001200)
  - [Multi-Threading and Multi-Process Synchronization](#multi-threading-and-multi-process-synchronization-000800---000930)
  - [Version Management and Cache Folders](#version-management-and-cache-folders-001100---001300)
- [JIT Control Modes](#jit-control-modes)
  - [Five JIT Control States](#five-jit-control-states-002330---002700)
  - [Cross-Compilation Pattern for Embedded Systems](#cross-compilation-pattern-for-embedded-systems-002600---002830)
- [JIT Load Process](#jit-load-process)
  - [Primary Entry Point: jitifyer_load](#primary-entry-point-jitifyer_load-002100---002300)
  - [Fast Path Lookup (RUN Mode)](#fast-path-lookup-run-mode-002900---003200)
  - [Hash Table Entry Structure](#hash-table-entry-structure-003000---003100)
  - [Prejit Kernel Handling](#prejit-kernel-handling-003600---003830)
  - [User Operator Special Cases](#user-operator-special-cases-003900---004400)
- [Kernel Loading and Compilation](#kernel-loading-and-compilation)
  - [File Locking for Multi-Process Safety](#file-locking-for-multi-process-safety-005100---005300)
  - [Loading Compiled Kernels](#loading-compiled-kernels-005500---005800)
  - [Compilation Trigger and Control Flow](#compilation-trigger-and-control-flow-005700---010000)
- [The Macrofy System](#the-macrofy-system)
  - [Source Code Generation Overview](#source-code-generation-overview-010000---010200)
  - [Macrofy Preface](#macrofy-preface-010100)
  - [Macrofy Family Structure](#macrofy-family-structure-010200---010300)
  - [Macrofy Reduce Deep Dive](#macrofy-reduce-deep-dive-010300---010800)
  - [Macrofy Type Definitions](#macrofy-type-definitions-010600---010800)
  - [Macrofy Monoid](#macrofy-monoid-010800---011200)
  - [Macrofy Binary Operator](#macrofy-binary-operator-011100---011200)
  - [Macrofy Identity and Terminal Values](#macrofy-identity-and-terminal-values-011200---011400)
  - [Backend Selection (CPU vs CUDA)](#backend-selection-cpu-vs-cuda-005900---010000)
- [Error Handling and Robustness](#error-handling-and-robustness)
  - [JIT Failure Philosophy](#jit-failure-philosophy-005300---005400)
- [Implementation Statistics and Insights](#implementation-statistics-and-insights)
  - [Code Organization](#code-organization-000500---000630)
  - [CUDA Minimalism](#cuda-minimalism-011700)
- [Session Conclusion](#session-conclusion-011700)

---

## Overview

This session continues from Session 5, diving deep into how the GraphBLAS JIT (Just-In-Time) compilation system works internally. The discussion covers kernel compilation, loading, caching, synchronization, and the macrofy process that generates source code.

---

## JIT Architecture and Global State Management

### Global State and Critical Sections [00:06:00 - 00:12:00]

**Discussion:** The JIT system requires global state to maintain persistent information across GraphBLAS sessions, including hash tables, cache folder paths, compiler settings, and C flags.

**Key Quote:** "I don't like globals. Globals are evil. But I have to have them here because you call Grb.Mxm, and I jit a kernel, and I hash stick in a hash table, and you call it again. And I say, 'Oh, I've seen this one before.' I've got to go find the function pointer."

**Technical Details:**
- Core JIT implementation in `jitifyer.c` and `jitifyer.h` (2,700 lines of code)
- Hash table stores compiled kernel function pointers
- OpenMP critical sections protect hash table access from multiple user threads
- File locking protects against multiple GraphBLAS processes accessing same cache

**Timestamp:** [00:06:00]

---

### Multi-Threading and Multi-Process Synchronization [00:08:00 - 00:09:30]

**Discussion:** How the JIT handles concurrent access from multiple threads within a process and multiple separate GraphBLAS processes.

**Key Quote:** "If one user thread compiles a jit kernel, goes into jit cache, the other thread wants to use it. I have to share it. So the hash table is protected with a critical section."

**Technical Details:**
- User threads within same process share hash table via OpenMP critical sections
- Separate processes share cache folder via file locking
- Multiple layers of synchronization to prevent conflicts
- Hash function depends on matrix properties (sparse/hypersparse, zombies, iso-valued, etc.)

**Timestamp:** [00:08:00]

---

### Version Management and Cache Folders [00:11:00 - 00:13:00]

**Discussion:** How different GraphBLAS versions maintain separate cache folders and how version conflicts are handled.

**Key Quote:** "By default, the cache folder is like on Linux, it's .suitesparse/grb.9.3.0, so if you have a different version like 9.3.1 or 9.2.1, it's a different folder."

**Technical Details:**
- Default cache: `.suitesparse/grb.9.3.0` on Linux
- Query function checks compiled kernel version on load
- Mismatched versions cause kernel deletion and recompilation
- Kernel query validates: version number, hash function, operator strings, monoid identity values

**Timestamp:** [00:11:00]

---

## JIT Control Modes

### Five JIT Control States [00:23:30 - 00:27:00]

**Discussion:** The JIT system has five operational modes that control compilation and kernel usage behavior.

**States:**

1. **OFF** - Hash table is empty, no JIT kernels used (not even prejit)
2. **PAUSE** - Don't use JIT kernels, but keep them in hash table
3. **RUN** - Use kernels already in hash table, but don't load from disk or compile new ones
4. **LOAD** - Can use hash table and load from disk, but cannot compile
5. **ON** - Full JIT enabled: compile, load, and use everything

**Key Quote:** "Off means the hash table is empty. Pause means turn off the jit, don't use any jit kernels, but keep them in the hash table. Run says if it's in the hash table you can call it and use it, but you can't load it from disk, you can't compile anything."

**Timestamp:** [00:23:30]

---

### Cross-Compilation Pattern for Embedded Systems [00:26:00 - 00:28:30]

**Discussion:** Using RUN mode to deploy GraphBLAS on platforms without compilers (iPad, iPhone).

**Key Quote:** "You exercise GraphBLAS, create it in development, let it compile all its jit kernels at once, run through all its paces. You get a thousand jit kernels. You're done. Now you want to distribute this to the world. So you take these jit kernels, you put them back into GraphBLAS as a prejit kernel, you recompile GraphBLAS."

**Workflow:**
1. Development: Run with JIT fully ON, exercise all code paths
2. Collect compiled kernels
3. Bake kernels into GraphBLAS as prejit kernels
4. Deploy to target platform with JIT set to RUN mode
5. No compiler needed on target platform (iOS, etc.)

**Timestamp:** [00:26:00]

---

## JIT Load Process

### Primary Entry Point: jitifyer_load [00:21:00 - 00:23:00]

**Discussion:** The main function that handles JIT kernel lookup, loading, and compilation.

**Key Quote:** "jitifyer_load is the primary entry point from the rest of GraphBLAS into the jitifyer. The goal of this method is to produce the function pointer, and it may need to compile a kernel, or maybe to load it."

**Process Flow:**
1. Check if JIT is disabled (OFF/PAUSE) → return no value
2. Try fast lookup in hash table (RUN mode, no critical section)
3. If not found, enter critical section for full lookup/load/compile
4. Check prejit kernels
5. Load from disk if available
6. Compile if necessary

**Timestamp:** [00:21:00]

---

### Fast Path Lookup (RUN Mode) [00:29:00 - 00:32:00]

**Discussion:** Optimized lookup path when JIT is in RUN mode - no critical section needed.

**Key Quote:** "This is greased lightning path for the run case. I don't even have to enter a critical section, because I know nobody within this GraphBLAS context is going to be touching the hash table."

**Technical Details:**
- No locks/critical sections in RUN mode
- Hash table is read-only
- 99.99% of calls use this fast path in tight loops
- Returns function pointer immediately if kernel found

**Timestamp:** [00:29:00]

---

### Hash Table Entry Structure [00:30:00 - 00:31:00]

**Discussion:** Structure stored in hash table for each compiled kernel.

**Components:**
- 16-byte encoding
- 64-bit hash function
- Kernel name/suffix string (for user-defined operators/types)
- `dlopen` handle (library pointer)
- Function pointer to actual kernel
- Prejit index (for prejit kernels)
- Checked flag (for validation status)

**Timestamp:** [00:30:00]

---

### Prejit Kernel Handling [00:36:00 - 00:38:30]

**Discussion:** Special handling for kernels baked into GraphBLAS at compile time.

**Key Quote:** "A prejit kernel has been baked into GraphBLAS already. I don't have to compile or load, it's just sitting right there. But I check it with the query function to make sure it's valid."

**Process:**
1. Look up in prejit kernel table (configured by CMake)
2. Call query function to validate
3. Check version, hash, operator strings, identity values
4. If valid: use kernel
5. If invalid (stale): remove from hash table, recompile if needed

**Timestamp:** [00:36:00]

---

### User Operator Special Cases [00:39:00 - 00:44:00]

**Discussion:** How user-defined operators and types are handled differently than built-ins.

**Key Quote:** "If you give me a user-defined operator and you just give me the string, you don't give me the C function pointer, I can still work with that. I'll compile it myself, and I only need to use the function pointer in cases where I don't care about performance anyway."

**Use Cases:**
- Query function kernels (return function pointer to user operator)
- Size query kernels (return sizeof user-defined type)
- ISO-valued matrix reductions (logarithmic time with function pointer calls)

**Example:** Reducing billion ISO-valued entries: only 30 function pointer calls (log₂ of billion), performance doesn't matter

**Timestamp:** [00:39:00]

---

## Kernel Loading and Compilation

### File Locking for Multi-Process Safety [00:51:00 - 00:53:00]

**Discussion:** Using file locks to prevent race conditions when multiple GraphBLAS processes compile the same kernel.

**Technical Details:**
- Separate lock file in `lock/` subfolder
- Lock file named with hash code
- Protects compilation/loading of same kernel by different processes
- Lock failures cause JIT to be disabled (punt to generic kernel)

**Error Handling Philosophy:** On any JIT failure (full filesystem, permissions, compiler errors), disable JIT entirely to avoid repeated failures in tight loops.

**Timestamp:** [00:51:00]

---

### Loading Compiled Kernels [00:55:00 - 00:58:00]

**Discussion:** Process of loading previously compiled shared libraries.

**Process:**
1. Construct filename: kernel_name + hash + `.so`/`.dylib`/`.dll`
2. Call `dlopen` (POSIX) or equivalent (Windows)
3. Get pointer to query function via `dlsym`
4. Call query function to validate kernel
5. If validation fails: close library, delete file, recompile
6. If validation succeeds: store handle and function pointer in hash

**Key Quote:** "I try to dlopen it. If I deal opened it, I've loaded it into memory. But this is the first time I've seen this kernel this run. So now I'm going to say, 'Okay, library I just loaded, give me a function to your query.'"

**Timestamp:** [00:55:00]

---

### Compilation Trigger and Control Flow [00:57:00 - 01:00:00]

**Discussion:** When compilation is triggered and how LOAD vs ON modes differ.

**Logic:**
- If JIT mode is LOAD and kernel not found on disk → fail (no compilation)
- If JIT mode is ON and kernel not found → compile
- Compilation happens inside critical section (one kernel at a time)
- Only one compilation at a time to avoid complexity of concurrent builds

**Timestamp:** [00:57:00]

---

## The Macrofy System

### Source Code Generation Overview [01:00:00 - 01:02:00]

**Discussion:** Introduction to the macrofy process - converting encoded kernel specifications into C/CUDA source code.

**Key Quote:** "This is every CUDA kernel, every OpenMP kernel source code is built right here. One place. All of them both CUDA and OpenMP one place."

**Components:**
- Preface: includes and headers
- Family-specific code generation
- Query function generation
- All handled in `jitifyer.c`

**Timestamp:** [01:00:00]

---

### Macrofy Preface [01:01:00]

**Discussion:** Initial setup of generated source file.

**Contents:**
- Standard includes
- CUDA flag (`#define` for CUDA kernels)
- User-provided custom includes (e.g., `#include "my_wonky_math.h"`)
- Comments identifying what's being compiled

**Timestamp:** [01:01:00]

---

### Macrofy Family Structure [01:02:00 - 01:03:00]

**Discussion:** Top-level dispatcher for different kernel families.

**Families:**
- Reduce
- Apply
- Assign
- MxM (matrix multiply)
- Build
- Select
- Sort
- User operations
- User types

**Single switch statement routes to appropriate family-specific macrofy function**

**Timestamp:** [01:02:00]

---

### Macrofy Reduce Deep Dive [01:03:00 - 01:08:00]

**Discussion:** Detailed look at how reduce kernel source code is generated.

**Process:**
1. Unpack encoding bits (operation, sparse format, zombies, etc.)
2. Macrofy type definitions
3. Macrofy monoid (identity value, terminal value, operator)
4. Macrofy input matrix properties
5. Set algorithm parameters (panel size for chunked reduction)
6. Emit definitions and kernel code

**Key Quote:** "The job of macrofy reduce is to produce this whole file. I have to produce lots of things. I first have to get the code to know what I'm doing. I've encoded this already - which reduction operator, which id - as bits. I'm unpacking the bits that I packed for the code."

**Timestamp:** [01:03:00]

---

### Macrofy Type Definitions [01:06:00 - 01:08:00]

**Discussion:** How type definitions are generated for kernels.

**Approach:**
- Use `#define` instead of `typedef` for simplicity
- Example: `#define GBZ_TYPE FC32_t`
- Up to 6 different types possible in complex kernels
- Skip duplicate types to avoid redefinition

**Timestamp:** [01:06:00]

---

### Macrofy Monoid [01:08:00 - 01:12:00]

**Discussion:** Comprehensive look at generating monoid code.

**Components Generated:**
1. Binary operator function (static inline)
2. Identity value definition
3. Terminal value definition (if applicable)
4. Atomic operation macros
5. CUDA-specific atomic operations
6. Special case flags

**Process Flow:**
1. Macrofy binary operator
2. Macrofy identity value
3. Macrofy terminal value
4. Generate atomic macros
5. Emit CUDA variants (even for CPU kernels - unused but simpler)

**Timestamp:** [01:08:00]

---

### Macrofy Binary Operator [01:11:00 - 01:12:00]

**Discussion:** Converting binary operators from codes back to source strings.

**Cases:**
1. **ISO-valued matrix:** No operator needed
2. **User-defined:** Emit function call
3. **Built-in:** Emit inline operation (e.g., `z = x + y`)

**Key Quote:** "If it's built-in, then this macro like for plus real becomes this: z = x + y. And the update becomes z += y. Some of them don't have update form, so they just have a call to the static inline function."

**Every built-in operator defined as string template - "writing my own compiler here"**

**Timestamp:** [01:11:00]

---

### Macrofy Identity and Terminal Values [01:12:00 - 01:14:00]

**Discussion:** Generating code for monoid identity and terminal values.

**Built-in Types:**
- Simple assignment: `GB_IDENTITY = 0`, `GB_IDENTITY = 1.0`, etc.
- Handles: 0, 1, true, false, INT32_MAX, infinity, -infinity, etc.

**User-Defined Types:**
- Identity value stored as raw bytes (e.g., 16 zeros)
- Use `memset` for all-zero patterns
- Otherwise emit byte array initialization

**Key Quote:** "I detected that when you defined the monoid, you gave me an identity value of all 16 zero bytes. So I can use memset to set them all to 0. I can't just use a simple assignment of 0, it's a struct."

**Timestamp:** [01:12:00]

---

### Backend Selection (CPU vs CUDA) [00:59:00 - 01:00:00]

**Discussion:** How the system handles different compilation backends.

**Technical Details:**
- File extension: `.c` for CPU, `.cu` for CUDA
- Specialized compilation paths for each backend
- Different compilers invoked (gcc/clang vs nvcc)
- Same macrofy code generates both types

**Key Quote:** "There's only a few places where I have to specialize. The name of the file: is it a .c file or .cu file for CUDA."

**Timestamp:** [00:59:00]

---

## Error Handling and Robustness

### JIT Failure Philosophy [00:53:00 - 00:54:00]

**Discussion:** How the system handles JIT compilation failures.

**Key Quote:** "Whenever I have trouble in the JIT, I turn the JIT off, because I know I don't have to use the JIT. And to avoid repeated error, like if you do MxM in a kernel in a loop 1,000 times, and every time I'm going to try to compile 1,000 times - let's not do that. Let's only fail compiled once, and then punt to generic."

**Philosophy:**
- On any error: disable JIT, fall back to generic kernel
- Prevents repeated compilation attempts in tight loops
- User sees verbose message about what happened
- Potential improvement: add flag for "strict mode" to make JIT failures fatal

**Timestamp:** [00:53:00]

---

## Implementation Statistics and Insights

### Code Organization [00:05:00 - 00:06:30]

**Key Files:**
- `jitifyer.c` - 2,700 lines, core JIT logic
- `jitifyer.h` - header file
- `gb_file.c` - low-level file operations (portable `dlopen`, file locking)
- Macrofy functions - separate files for encodify, enumify, macrofy

**Design Principle:** Keep static variables in single file to avoid globals

**Timestamp:** [00:05:00]

---

### CUDA Minimalism [01:17:00]

**Discussion:** How little CUDA-specific code exists in the JIT.

**Key Quote:** "If you look at jitifyer.c to see where it talks about CUDA, that's it. Every word as they mention the word CUDA in the JIT. It's so very terse."

**CUDA references are minimal:**
- Backend selection for `.cu` files
- CUDA-specific `#define` in preface
- Device function qualifiers in macros
- Otherwise same code for CPU and GPU

**Timestamp:** [01:17:00]

---

## Session Conclusion [01:17:00]

**Coverage Summary:**
- JIT architecture and global state management
- Thread and process synchronization
- Five JIT control modes (OFF, PAUSE, RUN, LOAD, ON)
- Hash table lookups and fast paths
- Prejit kernel handling
- File locking and multi-process safety
- Kernel loading from disk
- Macrofy system for source code generation
- Type, operator, and monoid code generation
- Backend selection and compilation triggers

**Not Covered (for Session 7):**
- How compilation is invoked (cmake, direct compiler calls)
- Query function generation
- Actual compilation commands and flags
- Testing and coverage

**Key Achievement:** Complete understanding of the "meta-compiler" that generates GraphBLAS JIT kernels from encoded specifications.

**Timestamp:** [01:17:00]
