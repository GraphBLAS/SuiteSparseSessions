# Session 5 Summary

## Table of Contents
- [Overview](#overview)
- [JIT System Architecture](#jit-system-architecture)
  - [JIT Kernel Management](#jit-kernel-management-000002)
  - [JIT Control Settings](#jit-control-settings-001940)
- [Kernel Loading and Compilation](#kernel-loading-and-compilation)
  - [The Jitifier Load Process](#the-jitifier-load-process-001417)
  - [Global State and Concurrency](#global-state-and-concurrency-001216)
  - [File Locking for Multi-Process Safety](#file-locking-for-multi-process-safety-004030)
- [Kernel Creation and Macrification](#kernel-creation-and-macrification)
  - [Kernel Families and Encoding](#kernel-families-and-encoding-003438)
  - [The Macrification Process](#the-macrification-process-003640)
  - [File Naming Convention](#file-naming-convention-003719)
  - [Type Definitions and Macrification](#type-definitions-and-macrification-005100)
- [Binary Operators and Monoids](#binary-operators-and-monoids)
  - [Operator Encoding](#operator-encoding-010800)
  - [Operator String Generation](#operator-string-generation-011430)
  - [Operator Flipping for Non-Commutative Operations](#operator-flipping-for-non-commutative-operations-011020)
  - [Integer Division Edge Cases](#integer-division-edge-cases-012100)
  - [Monoid Macrification](#monoid-macrification-010100)
  - [Shared Definitions and Defaults](#shared-definitions-and-defaults-013600)
- [Matrix Format Accessors](#matrix-format-accessors)
  - [Format-Specific Macros](#format-specific-macros-014150)
- [Template Code and Compilation](#template-code-and-compilation)
  - [Reduction Template Structure](#reduction-template-structure-014800)
  - [Panel-Based Reduction Optimization](#panel-based-reduction-optimization-014400)
  - [JIT vs Pre-JIT Kernels](#jit-vs-pre-jit-kernels-015100)
  - [Query Function Generation](#query-function-generation-015400)
- [Next Steps](#next-steps)
- [Technical Details Summary](#technical-details-summary)

---

## Overview

This session provides a deep dive into the JIT (Just-In-Time) compilation system in SuiteSparse GraphBLAS, focusing on how kernels are created, hashed, compiled, verified, and loaded. Dr. Davis walks through the complete macrification process, showing how template code is specialized for specific operators and types.

---

## JIT System Architecture

### JIT Kernel Management [00:00:02]

**Discussion:**
The session starts with an overview of the core JIT system components: kernel creation, hashing, compilation, loading, and verification.

**Key Points:**
- The JIT system handles kernel lifecycle from creation to execution
- Verification prevents using stale kernels when operator definitions change
- Example: If you define `add_gauss` operator and later change its definition without changing the name, the JIT detects this mismatch

**Key Quote:**
> "What if you create an operator called add Gauss, and you did this a week ago, and you change your code and you change the definition but you don't change the name? I think oh, I've got add Gauss here. Here it is, and it's the wrong string. Well, that's not good. I check for that." [00:00:23]

---

### JIT Control Settings [00:19:40]

**Discussion:**
GraphBLAS provides five different JIT control settings that give users fine-grained control over compilation and execution.

**Settings:**
1. **On** - Full JIT: compile, load, and run
2. **Load** - Load and run pre-compiled kernels, but don't compile new ones
3. **Run** - Only run already-loaded kernels, no file access
4. **Pause** - Don't use JIT at all, but keep loaded kernels in memory
5. **Off** - Completely disable JIT and deallocate everything

**Key Quote:**
> "If you have a set of operations which are not performance critical but they're doing some crazy type casting, they may compile a bunch of jets you don't need. Just turn, pause the jet, or just say, Jit run. Don't load anything. Don't try to compile the thing. Just go with what you got." [00:20:40]

---

## Kernel Loading and Compilation

### The Jitifier Load Process [00:14:17]

**Discussion:**
The `jitifier_load` function is the main entry point for obtaining a JIT kernel function pointer. Despite its name, it does more than just loading.

**Process Flow:**
1. Check if kernel already exists in hash table (fast path)
2. If not found, attempt to load from disk
3. If not on disk or stale, compile from scratch
4. Query the kernel to verify it matches expectations
5. Add to hash table and return function pointer

**Key Quote:**
> "This is the goal of this is to give me that function pointer to the jit kernel I want to call. I may have to construct that, may have to compile it. It might be able to quickly look up in the hash table." [00:14:25]

**Timestamp:** [00:24:00]

---

### Global State and Concurrency [00:12:16]

**Discussion:**
The JIT system uses carefully managed global state to track compiled kernels, cache locations, and compiler settings.

**Global Variables:**
- Hash table of compiled kernels
- Cache directory path for compiled kernels and source files
- Compiler name and flags (configurable via `GrB_get` and `GrB_set`)
- JIT control state

**Concurrency Handling:**
- Critical sections protect hash table modifications between user threads
- File locking prevents conflicts between different applications compiling the same kernel
- Run mode allows lock-free kernel lookups for maximum performance

**Key Quote:**
> "I have to have them because I have to keep track of permanent track of all this stuff, the hash table, the settings like where do you keep all your cache? Where is the cache of all the compiled kernels and their source files." [00:12:50]

**Timestamp:** [00:23:00]

---

### File Locking for Multi-Process Safety [00:40:30]

**Discussion:**
To handle the case where two different applications might try to compile the same kernel simultaneously, the JIT uses file-based locking.

**Implementation:**
- Creates a lock file based on the kernel hash
- Hash collisions are acceptable - different kernels will just wait on each other
- If locking fails, the JIT is disabled (perhaps too pessimistic, as Tim notes)

**Key Quote:**
> "I'm paranoid. What if you have 2 different applications that are both calling graph laws, and they both want to compile the same jit kernel at the same time. That's just going to be a mess. I have a file locking mechanism." [00:40:08]

---

## Kernel Creation and Macrification

### Kernel Families and Encoding [00:34:38]

**Discussion:**
Kernels are organized into families (reduce, mxm, apply, etc.), each with specific encoding requirements.

**Kernel Families:**
- Reduce family: reduction to scalar (7 hex digits in encoding)
- Mxm family: matrix-matrix multiply (16 hex digits, 62 bits)
- Apply family: element-wise operations (12 digits)
- Each family has different numbers of operators, types, and format requirements

**Encoding Components:**
- Sparsity format (2 bits)
- Zombies flag (1 bit)
- Data type (4 bits for 13 built-in types)
- Operator codes (5-8 bits depending on operator type)
- Monoid properties (identity, terminal values)

**Timestamp:** [00:35:15]

---

### The Macrification Process [00:36:40]

**Discussion:**
Macrification is the process of transforming template code into specialized C code by generating preprocessor macros that define types, operators, and operations.

**Macrify Methods:**
- `macrify_name` - generates the kernel file name
- `macrify_family` - dispatches to family-specific macrification
- `macrify_reduce` - specializes reduction kernels
- `macrify_typedefs` - generates user-defined type definitions
- `macrify_monoid` - generates monoid operator macros
- `macrify_binop` - generates binary operator definitions
- `macrify_input` - generates matrix accessor macros

**Key Quote:**
> "The macrifying is the process of all these macrify methods here. It's the process of creating the strings that go along or define a particular jit kernel, or the name of the file, or the name of the kernel, which is the same thing." [00:37:00]

---

### File Naming Convention [00:37:19]

**Discussion:**
JIT kernel files follow a strict naming convention that encodes the kernel type and its specialization.

**Naming Pattern:**
```
GB_jit__<kernel_name>__<encoding>__<suffix>
```

**Example:**
```
GB_jit__reduce__03f31e1
```

**Components:**
- `GB_jit` - namespace prefix (always)
- Double underscores separate regions
- `reduce` - kernel name
- `03f31e1` - 7 hex digits encoding (28 bits for reduce family)
- Optional suffix for user-defined operators/types

**Timestamp:** [00:38:00]

---

### Type Definitions and Macrification [00:51:00]

**Discussion:**
User-defined types must be defined twice in the generated code: once as actual C code and once as a string for verification.

**Dual Definition Pattern:**
```c
// Actual typedef for compilation
typedef struct { double x, y; } MyCx;

// String version for verification
#define GB_MyCx_DEFN "typedef struct { double x, y; } MyCx"
```

**Purpose:**
- The actual typedef is used during compilation
- The string version is embedded in the kernel for query/verification
- This allows checking if the type definition has changed

**Key Quote:**
> "I have the C code, and I re-inject it as a string, and then I can call this query function to say, Cough up your strings, please. And so I can look at them to see if they're different than the ones I'm getting now as a mismatch. I delete this kernel, recompile it." [00:27:00]

---

## Binary Operators and Monoids

### Operator Encoding [01:08:00]

**Discussion:**
GraphBLAS has 139 different built-in binary operators, each with a unique encoding.

**Operator Categories:**
- First 31 codes are permissible for monoids (require only 5 bits)
- Full operator set requires 8 bits
- User-defined operators all encode to code 0, distinguished by suffix

**Special Operators:**
- Integer division (handles divide-by-zero safely)
- Complex operations (Microsoft compiler workarounds)
- Reverse operators (rdiv, rsub, etc.) for non-commutative operations
- Positional operators (uses row/column indices)

**Timestamp:** [01:16:00]

---

### Operator String Generation [01:14:30]

**Discussion:**
Each operator can have up to three different string representations for different contexts and platforms.

**Three String Types:**
1. **Basic expression** - How to compute `z = f(x, y)`
2. **Update string (U)** - CPU-optimized update like `z += y`
3. **GPU string (G)** - CUDA-optimized version

**Example (min operator):**
```c
// CPU: uses if statement
if (x < y) z = x;

// CUDA: uses ternary (avoids branches)
z = (x < y) ? x : y;
```

**Key Quote:**
> "I decided this is faster on Cuda because cuda doesn't like branches. Cuda doesn't like ifs, it doesn't mind ternaries as much as ifs. You'd think the compilers wouldn't care between those 2 right? But this is just a matter of tweaking the code and optimizing." [01:18:30]

**Timestamp:** [01:15:00]

---

### Operator Flipping for Non-Commutative Operations [01:10:20]

**Discussion:**
When matrices are transposed or reordered, non-commutative operators must have their operands flipped.

**Implementation:**
- Flipping is handled at the macro level, not by changing strings
- If `flip_xy` is true, the macro becomes `OP(y, x)` instead of `OP(x, y)`
- All built-in non-commutative operators have reverse variants (div/rdiv, sub/rsub, etc.)

**Key Quote:**
> "I may decide to say, Hey, wait, I don't have that. I'm going to do this instead. I may choose to multiply instead of a times b, I might choose to do b transpose times a transpose. What if the multiplicative operator is not commutative? Oh, okay, then I have to eventually reverse those operands." [01:12:00]

**Timestamp:** [01:12:25]

---

### Integer Division Edge Cases [01:21:00]

**Discussion:**
Integer division requires special handling because C's undefined behavior for division by zero would abort the program.

**GraphBLAS Integer Division Rules:**
- `0 / 0 = 0` (integer NaN equivalent)
- `x / 0 = INT_MAX` (positive infinity)
- `x / 0 = INT_MIN` (negative infinity)
- Never aborts the program

**Rationale:**
- Matrix operations may encounter many zeros
- Aborting on division by zero is unacceptable in a library
- Follows MATLAB's approach to integer division

**Key Quote:**
> "If you happen to divide by 0 integer division by 0, what do you get? Well, you don't get a Nan. You don't get infinity like in 32 Max and 64. Your program dies. You get an abort. Oh, my gosh, you gotta be kidding! That's the undefined behavior that gcc says it's undefined. So let's abort. I don't do that. I do the Matlab thing." [01:22:00]

**Timestamp:** [01:21:30]

---

### Monoid Macrification [01:01:00]

**Discussion:**
Monoids require extensive macro definitions for identity values, terminal values, atomics, and SIMD operations.

**Generated Macros:**
- `GB_Z_TYPE` - the monoid's data type
- `GB_ADD(z,x,y)` - the binary operation
- `GB_UPDATE(z,y)` - update operation (z op= y)
- `GB_IDENTITY` - identity value constant
- `GB_DECLARE_IDENTITY` - declares and initializes identity
- `GB_Z_ATOMIC` - whether atomic updates are available
- `GB_Z_HAS_OMP_ATOMIC` - OpenMP atomic support
- `GB_Z_HAS_CUDA_ATOMIC` - CUDA atomic support
- `GB_Z_NBITS` - size in bits
- `GB_PRAGMA_SIMD_REDUCTION` - SIMD reduction pragma

**Timestamp:** [01:03:00]

---

### Shared Definitions and Defaults [01:36:00]

**Discussion:**
A shared header file provides default definitions for any macros not defined during macrification.

**Default Handling:**
- If atomic support isn't explicitly defined, it defaults to false
- If no identity byte exists, memset optimization is disabled
- Terminal values default to "no terminal"
- This allows family-specific macrification to omit irrelevant features

**Key Quote:**
> "What happens if I use them? Well, either they're not defined, or later on, they'll be the empty string if I don't define them here. At the very end, just before I go in and grab my actual templatized kernel code, I have this all the monoids shared definitions, and this is a bunch of if not defined, define it as the default." [01:36:00]

**Timestamp:** [01:37:00]

---

## Matrix Format Accessors

### Format-Specific Macros [01:41:50]

**Discussion:**
Each matrix gets its own set of accessor macros that abstract away the storage format.

**Accessor Categories:**
- Position accessors (`GB_A_P(k)` - start of column k)
- Index accessors (`GB_A_H(p)`, `GB_A_I(p)` - get row index)
- Value accessors (`GB_A_X(p)` - get value at position p)
- Format flags (`GB_A_IS_BITMAP`, `GB_A_IS_HYPER`, etc.)
- Property flags (`GB_A_HAS_ZOMBIES`, `GB_A_ISO`)

**Type Casting:**
- `GB_A_TYPE` - actual storage type of matrix A
- `GB_A2_TYPE` - type after casting for operator input
- Handles both flipped and non-flipped operator cases

**Key Quote:**
> "Every matrix here the a matrix has its own P macro. It's bitmap. So how do you get where's the start of the column? Oh, it's just a multiply. These are all format, your data format, matrix format accessors." [01:42:05]

---

## Template Code and Compilation

### Reduction Template Structure [01:48:00]

**Discussion:**
Template files use the generated macros to create specialized kernel implementations.

**Template Features:**
- Compile-time selection between panel-based and panel-free algorithms
- Separate handling for zombies, bitmap format, and standard sparse
- Uses macros for all type-dependent and operator-dependent code
- Multiple algorithm variants selected by `#if` directives

**Example Selection:**
```c
#if (GB_A_HAS_ZOMBIES || GB_A_IS_BITMAP || GB_PANEL_SIZE == 1)
    // Panel-free reduction
#else
    // Panel-based reduction (faster for most cases)
#endif
```

**Timestamp:** [01:48:30]

---

### Panel-Based Reduction Optimization [01:44:00]

**Discussion:**
Panel-based reduction is significantly faster than scalar reduction due to better pipelining.

**Panel Size Selection:**
- Boolean operators: 8 elements
- Min/max: 16 elements
- Floating-point plus/times: depends on type (8-64 elements)
- Integer operations: type-dependent (8-256 elements)

**Why It's Faster:**
- Reduces to temporary array instead of single accumulator
- Eliminates data dependencies between operations
- Allows better CPU pipelining
- Final reduction across panel is amortized

**Key Quote:**
> "It's faster to load in a bunch of entries and reduce all of them to a temporary array of 32 words. The reason that's better is just the compiler, the hardware likes that better. There's no dependency between operations. They're all independent. So the pipeline works better, and it's way faster." [01:44:35]

**Platform Dependency:**
> "These are really wonky numbers. I've just tried different cases. This is probably specific to an intel processor because that's where I've optimized it for." [01:45:50]

**Timestamp:** [01:45:00]

---

### JIT vs Pre-JIT Kernels [01:51:00]

**Discussion:**
The same template code can generate either runtime JIT kernels or compile-time pre-JIT kernels through conditional compilation.

**JIT Kernel (runtime):**
```c
#define GB_JIT_RUNTIME
// Function named: GB_jit_kernel
// Compiled at runtime, stored in cache
```

**Pre-JIT Kernel (compile-time):**
```c
// No GB_JIT_RUNTIME defined
// Function named: GB_jit__reduce__03f31e1
// Compiled into GraphBLAS library
```

**Differences:**
- JIT kernels always named `GB_jit_kernel` in their .so files
- Pre-JIT kernels have unique names matching their specialization
- Both use the same template code with different macro definitions

**Key Quote:**
> "When I'm compiling at runtime, but graph blast build time for the library, this is not defined. And so I'm going to rename this kernel with its name, which is GB jit reduce 0cf 1f 0 8 2. Its name will match its file name, which makes sense." [01:52:00]

**Timestamp:** [01:51:27]

---

### Query Function Generation [01:54:00]

**Discussion:**
Every kernel includes a query function that returns its defining characteristics for verification.

**Query Function Returns:**
- Hash code for sanity checking
- GraphBLAS version number (e.g., 9.3.0)
- Type definition strings
- Operator definition strings
- Identity and terminal values

**Verification Process:**
1. Load kernel from disk or hash table
2. Call query function
3. Compare returned values with expected values
4. If mismatch, delete kernel and recompile
5. If match, use the kernel

**Timestamp:** [01:55:00]

---

## Next Steps

The session ends with plans for the next session to cover:
- How to compile a kernel (invoking the C compiler)
- Loading compiled kernels with dlopen
- Hash table management
- Kernel verification and validation

**Key Quote:**
> "Next session: How do I compile a kernel? Session 6. Compiling a kernel and hashing it." [01:56:00]

---

## Technical Details Summary

### Key File Locations
- JIT source code: `Source/JITPackage/GB_jitifyer.c` (2,700 lines)
- Template kernels: `Source/JITPackage/GB_jit_kernel_*.c`
- Macro definitions: Various `GB_*_shared_definitions.h` files
- Pre-JIT kernels: Built into GraphBLAS library

### Encoding Bit Budgets
- Reduce family: 28 bits (7 hex digits)
- MxM family: 62 bits (16 hex digits)
- Apply family: 48 bits (12 hex digits)
- Monoid operators: 5 bits (31 possible)
- All operators: 8 bits (139 built-in)
- Data types: 4 bits (13 built-in + user-defined)

### Performance Considerations
- Hash table lookups are very fast in JIT_RUN mode (no locks)
- Compilation is slow but happens once per kernel specialization
- Panel sizes are platform-dependent (tuned for Intel)
- Atomic operations enable parallel reductions where supported
- SIMD pragmas provide additional vectorization hints

