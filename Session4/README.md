# Session 4 Summary

## Table of Contents
- [Overview](#overview)
- [Dynamic Graphs and Matrix Operations](#dynamic-graphs-and-matrix-operations)
  - [Cartesian Masks and Extended Select Operations](#cartesian-masks-and-extended-select-operations-000000)
  - [Dynamic Graph Support Through Pending Updates](#dynamic-graph-support-through-pending-updates-000420)
- [JIT System Architecture](#jit-system-architecture)
  - [High-Level Overview](#high-level-overview-000820)
  - [Starting Point: Matrix Reduce to Scalar](#starting-point-matrix-reduce-to-scalar-000930)
- [Encoding System](#encoding-system)
  - [The Encodify Process](#the-encodify-process-001700)
  - [Enumification of Operators](#enumification-of-operators-004025)
  - [Hash Function Design](#hash-function-design-005000)
- [Kernel Management](#kernel-management)
  - [JIT Kernel Codes](#jit-kernel-codes-005200)
  - [Hash Table and Caching](#hash-table-and-caching-005600)
- [JIT Compilation Process](#jit-compilation-process)
  - [Cache Directory Structure](#cache-directory-structure-010900)
  - [Kernel Naming Convention](#kernel-naming-convention-011700)
  - [Macroification Process](#macroification-process-012100)
  - [JIT Query Function](#jit-query-function-013000)
- [Pre-JIT Kernels](#pre-jit-kernels)
  - [Pre-compilation for Restricted Platforms](#pre-compilation-for-restricted-platforms-013500)
- [JIT Package System](#jit-package-system)
  - [Embedded Source Code](#embedded-source-code-014700)
  - [Source Code Packaging](#source-code-packaging-014800)
- [CUDA Integration](#cuda-integration)
  - [CUDA JIT Kernels](#cuda-jit-kernels-014200)
  - [GPU/CPU Decision Heuristics](#gpucpu-decision-heuristics-015600)
- [Callback Mechanism](#callback-mechanism)
  - [Avoiding Library Linking Issues](#avoiding-library-linking-issues-015200)
- [Future Work and Extensions](#future-work-and-extensions)
  - [Planned JIT Kernels](#planned-jit-kernels-015900)
  - [Index Binary Operators](#index-binary-operators-015930)
  - [Unified Shared Memory Benefits](#unified-shared-memory-benefits-020444)
- [Technical Details](#technical-details)
  - [Performance Considerations](#performance-considerations-015601)
  - [Critical Section Management](#critical-section-management-010600)
- [Key Quotes](#key-quotes)
- [Next Session Preview](#next-session-preview)

---

## Overview
This session focuses on the Just-In-Time (JIT) compilation system in SuiteSparse GraphBLAS, providing an in-depth look at how GraphBLAS generates, compiles, and manages runtime kernels for optimal performance.

---

## Dynamic Graphs and Matrix Operations

### Cartesian Masks and Extended Select Operations [00:00:00]

Dr. Davis discusses the need for more powerful selection operations in GraphBLAS:

> "We need a select that you can have or a Cartesian mask. You have a matrix and I want to put a vector, say W and a vector V here or something. And I want to have a rule that says, pick that if those vectors, you know, pick AIJ if V sub I and WJ."

**Key concepts:**
- The current select operation is limited - relies on a non-opaque data structure (raw C array)
- Proposed Cartesian mask would allow selection based on properties of row and column vectors
- Extract operation reshapes output, which is often not desirable
- "Select should be what extract should have been" [00:04:14]

**Timestamp:** [00:01:00] - [00:04:20]

### Dynamic Graph Support Through Pending Updates [00:04:20]

Discussion of how GraphBLAS handles dynamic operations:

> "I thought you stick an entry in, I'll say thank you very much. I'll do it later. I'll just pocket it in the list, and it's super simple to implement."

**Features:**
- Pending tuples and zombies allow quasi-dynamic behavior
- Blocking mode enables deferred operations
- Most assign methods use pending tuples internally
- Set element operations fast due to deferred work

**Quote:** "GRB set element was a method. You just want to go blink, stick it in. Well, there's no way that's gonna be fast if it's totally static CSR." [00:06:46]

**Timestamp:** [00:04:20] - [00:08:10]

---

## JIT System Architecture

### High-Level Overview [00:08:20]

The JIT system is approximately 11,000 lines of code and represents a major component of GraphBLAS.

**Core components:**
1. Encodify - Convert operation to bit-encoded representation
2. Jitify - Load or compile kernel
3. Execute - Call compiled kernel

> "It's a 3 step process, and every function that ends with underscore jit is going to look just like this."

**Timestamp:** [00:08:20] - [00:10:30]

### Starting Point: Matrix Reduce to Scalar [00:09:30]

The simplest interesting GraphBLAS function used to explain JIT:

> "The way to start with the jit is to start with the simplest interesting graph plus function, which is GRB matrix reduced to scalar."

**Workflow:**
1. User calls GRB_reduce
2. Method checks for CUDA kernels
3. Checks for ISO full matrix optimization (log n operations)
4. Attempts factory kernel
5. Falls back to JIT
6. Generic kernel as last resort

**Quote:** "There's no place in the spec it actually needs the value of identity of the monoid ever. But in one place, which is reduce a scalar to a matrix when it's a non opaque C scalar and there's no entries in the matrix." [00:12:56]

**Timestamp:** [00:09:30] - [00:17:00]

---

## Encoding System

### The Encodify Process [00:17:00]

Every JIT call must quickly encode the operation:

> "This has to be done every time the methods called, whether it's pre-compiled or preloaded at all."

**Encoding structure (28 bits for reduce):**
- Sparsity format (2 bits)
- Zombie presence (1 bit)
- Matrix data type (4 bits for 14 types)
- Monoid data type (4 bits)
- Reduction operator (5 bits for 23 operators)
- Identity value (5 bits for 31 cases)
- Terminal value (5 bits for 30 cases)
- Has cheeseburger (1 bit - CUDA atomic support)

**Quote:** "28 bits encodes all possible reductions I could see." [00:39:00]

**Timestamp:** [00:17:00] - [00:40:00]

### Enumification of Operators [00:40:25]

How binary operators are encoded:

> "I go and you know, user cases are always code 0. I just have to encode all possible sets of operations."

**Process:**
- Min, max, plus map to specific codes
- Boolean operators unified (Min Boolean = logical AND = times Boolean)
- FP32 vs FP64 handled differently (fmin vs fminf)
- 255 possible operator codes, 32 valid for monoids

**Quote:** "If 2 monoids map to the same string, it doesn't I can just you reuse that same kernel." [00:44:24]

**Timestamp:** [00:40:25] - [00:50:00]

### Hash Function Design [00:50:00]

Hash computation for kernel lookup:

> "I don't need to go and read the string every time and hash the string the hash encoding then just has to hash a hundred 20 bit number down to a 64 bit hash."

**Special hash values:**
- 0 = built-in operator (no hashing needed)
- UINT64_MAX = cannot JIT this operation
- Combines encoding hash with monoid/operator hash via XOR

**Timestamp:** [00:50:00] - [00:59:00]

---

## Kernel Management

### JIT Kernel Codes [00:52:00]

Enumeration of all JIT kernel families:

**Current kernels (44 total):**
- 1: Reduce to scalar
- 2-8: Matrix multiply variants (7 kernels)
- 9-22: Element-wise operations (~14 kernels)
- 23-30: Apply operations (8 kernels)
- 31: Build operation
- 32-34: Select operations (3 kernels)
- 35-36: User-defined type support (2 kernels)
- 37-41: Assign operations (5 kernels)
- 42+: Extract, mask, Kronecker (not yet JITted)
- 1000+: CUDA kernels

**Quote:** "I've got 44 jit kernels. I need about 43 more just to cover these cases. Then, of course, there's all the cuda kernels." [02:03:19]

**Timestamp:** [00:52:00] - [00:55:00]

### Hash Table and Caching [00:56:00]

Global hash table for all compiled kernels:

> "Every kernel, all of them, every hash, every jit kernel or prejit is in this big hash table."

**Hash table entry structure (56 bytes):**
- Encoding (16 bytes)
- Hash value (8 bytes)
- Kernel code (4 bytes)
- Suffix length (4 bytes)
- DL handle pointer (8 bytes)
- Function pointer (8 bytes)
- Suffix string (variable, up to 256 bytes)

**Quote:** "This has to happen super quick. I gotta get to it quick cause I've got it compiled and loaded in memory. I want to find it and run it right away." [00:14:00]

**Timestamp:** [00:56:00] - [01:07:00]

---

## JIT Compilation Process

### Cache Directory Structure [01:09:00]

Organization of JIT cache at ~/.suite_sparse/GRB_9_3/:

```
.suite_sparse/GRB_9_3/
├── c/           # Compiled libraries (255 subdirectories by hash)
├── lock/        # File locking for concurrent access
├── src/         # Generated C source files
└── tmp/         # Temporary files
```

**Versioning:** "If I change graph plus at all, like point 1 3.9 point 3 1 I abandoned all previously compiled jits automatically." [01:10:56]

**Timestamp:** [01:09:00] - [01:12:00]

### Kernel Naming Convention [01:17:00]

Generated kernel files follow pattern:

```
GB_jit__<method>__<code>__<suffix>
```

**Example:** `GB_jit__reduce__83f3ee2__add_gauss`
- Method: reduce
- Code: 83f3ee2 (hex encoding of operation)
- Suffix: add_gauss (user-defined operator)

**Quote:** "The name of that kernel is called reduce, and that goes into this part of the name of the file, and also the name of the function." [01:17:45]

**Timestamp:** [01:17:00] - [01:21:00]

### Macroification Process [01:21:00]

How templates are specialized:

> "I have to construct an encoding of this problem based on the kernel it is and based upon the particular instances."

**Generated macro header:**
- Type definitions (GB_Z_TYPE)
- Operator definitions (GB_ADD, GB_UPDATE)
- Identity value initialization
- Terminal value (if any)
- Matrix format macros
- Template inclusion

**Example for user-defined Gaussian integers:**
```c
#define GB_Z_TYPE gauss_type
#define GB_ADD(z,x,y) add_gauss(&z, &x, &y)
#define GB_DECLARE_IDENTITY(z) memset(&z, 0, sizeof(gauss_type))
```

**Timestamp:** [01:21:00] - [01:30:00]

### JIT Query Function [01:30:00]

Validation mechanism for loaded kernels:

> "What if you compile a user-defined data type. You call it Gauss type or something. You create a function called add Gauss, and you change the name of the you keep the name the same, but you change the string. Oh, my goodness!"

**GB_jit_query checks:**
- Hash code matches
- GraphBLAS version matches
- String definitions match (for user-defined types)
- Identity value matches
- Terminal value matches
- Type size hasn't changed

**Quote:** "That's expensive. I don't do that in every call to this kernel. I do it only when I load it into memory." [01:32:00]

**Timestamp:** [01:30:00] - [01:35:00]

---

## Pre-JIT Kernels

### Pre-compilation for Restricted Platforms [01:35:00]

Support for platforms without runtime compilation:

> "You can still have on any system you can have what I call prejit kernels. You run graph laws, you run some applications. It generates all these jit kernels, and you look at your cache file, your dot suites bars GRB, you copy all those C files I generate back into graph laws in a special folder called Prejit."

**Process:**
1. Run application, collect JIT kernels
2. Copy generated C files to GraphBLAS source/prejit/
3. Rebuild GraphBLAS
4. Kernels compiled into library with unique names
5. Hash table populated at initialization

**Runtime vs Pre-JIT difference:**
- Runtime: All kernels named `GB_jit_kernel`
- Pre-JIT: Unique names like `GB_jit__reduce__83f3ee2__add_gauss`
- Pre-JIT uses demacrofy to parse kernel names back to encodings

**Quote:** "The prejit and the jit look very similar all the way through the code, just the prejit are already compiled and built into lib graph laws, and the jit ones are not." [01:40:20]

**Timestamp:** [01:35:00] - [01:42:00]

---

## JIT Package System

### Embedded Source Code [01:47:00]

Solution to the source code distribution problem:

> "I bundle it into the binary. It's crazy, but it works."

**Requirements solved:**
- Library needs its own source code for JIT compilation
- Cannot rely on finding correct version of source files
- Must work across all platforms (Linux, Windows, macOS, iOS)

**JIT Package contents (300KB compressed):**
- All template source files
- Include headers
- CUDA kernels
- Compressed using zstandard

**Quote:** "If you're a Linux just for a manager, you don't wanna bundle so you can bundle source code. But I didn't wanna require they do that. Well, I bundle all my source code anyway." [01:50:58]

**Timestamp:** [01:47:00] - [01:51:00]

### Source Code Packaging [01:48:00]

CMake build process:

1. Reads all template source files
2. Compresses each with zstandard
3. Emits as C arrays of bytes
4. Links into `GB_jit_package.c`
5. On `GRB_init()`, decompresses to cache if needed

**Example structure:**
```c
static uint8_t GB_jit_kernel_reduce[] = {
    0x28, 0xb5, 0x2f, 0xfd, 0x20, ...  // 131 lines
};
```

**Quote:** "When GRB init is called, I go look at your cash folder and I say, do you need source? Oh, your source code is already there. You're good or not. Oh, I need to put source code here." [01:48:54]

**Timestamp:** [01:48:00] - [01:51:00]

---

## CUDA Integration

### CUDA JIT Kernels [01:42:00]

Parallel structure for GPU kernels:

> "The cuda case is actually interesting to look at, and again, this is still a high level view."

**Differences from CPU JIT:**
- Kernel codes 1000+ (vs 1-100 for CPU)
- Different template files (.cu extension)
- Same encoding and hash system
- `#define GB_CUDA_KERNEL` in macro header

**Shared infrastructure:**
- Same encodify/jitify workflow
- Same hash table
- Same cache directory structure
- Template differs, everything else identical

**Quote:** "They differ in their templates, right? They have different templates. If I go to my templates, they're up here." [01:46:34]

**Timestamp:** [01:42:00] - [01:47:00]

### GPU/CPU Decision Heuristics [01:56:00]

Runtime decision-making for heterogeneous systems:

> "I'll hopefully make the right decisions based upon where the data already is."

**Factors considered:**
- Unified shared memory availability
- Last location data was touched
- Problem size threshold
- User hints via GRB_get/GRB_set

**Quote:** "Last time I touched it was on the GPU, so let me do it on the GPU. Last time I touched on the CPU, but oh, this is a big problem, big by some threshold, and therefore let's just do it on the GPU. It'll be faster." [01:57:37]

**Timestamp:** [01:56:00] - [01:59:00]

---

## Callback Mechanism

### Avoiding Library Linking Issues [01:52:00]

JIT kernels need to call back into GraphBLAS:

> "I have. It's very simple. I just made myself a little struct of function pointers. And I put in there like I need about 12 or 13 or 14 different functions that my jit kernels need to call back into."

**Problem:** Cannot safely link JIT kernels against `-lgraphblas` (version mismatch risk)

**Solution:** Pass function pointers down
- Struct of ~14 function pointers
- Passed to every JIT kernel
- No library dependencies in compiled kernels
- Guaranteed correct version

**Quote:** "If you just do like C make with dash L graph laws, and like, there's my library, I'm linking it might get a different graph plus library. Different version. That would be a nightmare." [01:52:12]

**Timestamp:** [01:52:00] - [01:54:00]

---

## Future Work and Extensions

### Planned JIT Kernels [01:59:00]

Significant expansion planned:

**Currently missing:**
- ~43 more CPU kernels for generic-only methods
- Extract/subref operations
- Mask phase kernels
- Kronecker product
- Additional assign variants (40+ more)

**Quote:** "There's 186 parallel loops in all of GraphBLAS. I have to have a 186 kernels for those potentially, maybe less, probably less." [02:01:59]

**Timestamp:** [01:59:00] - [02:02:00]

### Index Binary Operators [01:59:30]

Major upcoming feature:

> "I already almost support it, because I've got some built-in operators that depend on the index. So it's not going to be total break the code kind of event."

**Impact on JIT:**
- New operator enumeration codes
- Updated macrification process
- Additional parameters passed to kernels
- "Very unstable development branch for a while" during implementation

**Timestamp:** [01:59:30] - [02:00:02]

### Unified Shared Memory Benefits [02:04:44]

Philosophy for heterogeneous computing:

> "The beauty of unified shared memory - any piece of hardware, any accelerator says I don't care. Here's a pointer. Where is it? I don't care. I'll handle it for you."

**Advantages:**
- No manual memory management
- Automatic data migration
- Like virtual memory page faults
- Eliminates overlay-style programming

**Quote:** "It's like a page fault right? I don't have to write a virtual memory system. You used to do that. Do you remember the days of overlays?" [02:05:11]

**Timestamp:** [02:04:44] - [02:06:00]

---

## Technical Details

### Performance Considerations [01:56:01]

Threshold decisions for JIT vs generic:

> "It may take longer to look it up in the hash than just to call my generic method. I could say, well, at some point, it's not even worth calling the jit. Just call the generic method."

**Trade-offs:**
- Hash lookup: ~5 nanoseconds
- Function pointer call: ~3 nanoseconds
- For tiny problems, overhead may exceed benefit
- No threshold implemented yet

**Timestamp:** [01:56:01] - [01:57:00]

### Critical Section Management [01:06:00]

Thread safety in JIT compilation:

> "I protect it with critical sections. So if you have 2 user threads talking to the jit like only one can compile at a time. It just got too crazy otherwise."

**Concurrency model:**
- Global hash table protected
- Single compilation at a time
- Multiple kernel calls can proceed in parallel
- Potential performance issue for user-level parallelism

**Timestamp:** [01:06:00] - [01:07:30]

---

## Key Quotes

**On JIT necessity:**
> "I've got 44 jit kernels. I need about 80 of them more right? I've got way more I need to write." [02:00:06]

**On complexity:**
> "The jit is a big beast. You've seen this whole arc of oh, I wanna call the CUDA. I have called the jit wrapper, and it calls the encodifier." [01:55:18]

**On unified memory:**
> "You have a non-uniform memory space. Just give me a uniform pointer and if you need to make it move fine, it's just that pointer. This is the data I want, and just do it, and it just works." [02:04:59]

**On reliability:**
> "It was a failure point. I was just it was just a nightmare getting it to work and reliably link against the right thing. And it wasn't. So I've created this just this callback mechanism." [01:53:54]

---

## Next Session Preview

The next session will cover:
1. Deep dive into the jitifier.c core implementation
2. Macrification process details
3. Template system mechanics
4. Transition to matrix-matrix multiply (MXM) as foundation

> "So next time let's dive into the this, the core of the jitifier." [01:58:36]
