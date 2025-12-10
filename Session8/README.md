# Session 8 Summary

## Table of Contents
- [Overview](#overview)
- [Code Organization and Structure](#code-organization-and-structure-000018)
  - [Source Directory Layout](#source-directory-layout)
- [Debugging and Error Handling](#debugging-and-error-handling-001930)
  - [Assertion System](#assertion-system)
  - [Error Logging Limitations](#error-logging-limitations)
  - [Memory Debugging](#memory-debugging-002700)
- [Matrix Representation and Formats](#matrix-representation-and-formats-000500)
  - [Internal Representation](#internal-representation)
- [Mask Handling](#mask-handling-001200)
  - [GB_get_mask Function](#gb_get_mask-function)
- [User-Callable Method Structure](#user-callable-method-structure-000800)
  - [Standard Workflow for All Methods](#standard-workflow-for-all-methods)
  - [Descriptor Handling](#descriptor-handling)
- [Type Compatibility and Validation](#type-compatibility-and-validation-003300)
  - [Type Checking](#type-checking)
- [Operator Management](#operator-management-003100)
  - [Operator Simplification](#operator-simplification)
  - [Operator Flipping](#operator-flipping-005000)
- [Wait and Pending Operations](#wait-and-pending-operations-004200)
  - [Matrix State Management](#matrix-state-management)
- [Apply Operation Deep Dive](#apply-operation-deep-dive-004500)
  - [Three Main Execution Paths](#three-main-execution-paths)
- [Operator Type Handling](#operator-type-handling-011900)
  - [Positional Operators](#positional-operators)
  - [ISO Value Handling](#iso-value-handling-011300)
- [Parallel Execution](#parallel-execution-012000)
  - [Task Slicing Strategy](#task-slicing-strategy)
  - [Column Index Lookup](#column-index-lookup-013500)
- [Factory Kernel System](#factory-kernel-system-014500)
  - [Factory Organization](#factory-organization)
  - [Three-tier Execution Strategy](#three-tier-execution-strategy-015000)
- [Bind First/Second Operations](#bind-firstsecond-operations-015500)
  - [Binary Operator with Scalar](#binary-operator-with-scalar)
- [Index Unary Operators](#index-unary-operators-015900)
  - [User-Defined Index Unary Operators](#user-defined-index-unary-operators)
- [JIT Kernel Selection](#jit-kernel-selection-020200)
  - [Algorithm Variants](#algorithm-variants)
- [Accum-Mask Phase](#accum-mask-phase-020400)
  - [Final Phase of All Operations](#final-phase-of-all-operations)
- [Key Insights and Design Principles](#key-insights-and-design-principles)
- [Technical Details](#technical-details)
- [Session Notes](#session-notes)

---

## Overview
This session provides a detailed walkthrough of algorithm implementation in SuiteSparse GraphBLAS, focusing primarily on the `GrB_apply` operation. Dr. Davis demonstrates the internal structure of the library, explaining how algorithms are organized, optimized, and executed across different execution paths (CPU, JIT, CUDA).

---

## Code Organization and Structure [00:00:18]
[![00:00:18](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=18s)

### Source Directory Layout
"I just released GraphBLAS 9.3.1. Now all the source files have been restructured so it's easier to follow."

- **Factory folders**: Template files used only to construct factory kernels, not needed for JIT
- **Include folders**: Various definitions for algorithm sets
- **Template folders**: Templatized kernels used in JIT and factory kernels, containing data-type-dependent algorithms
- **Source files** (first level): Compiled directly into libgraphblas.so, not templatized, only work on integers/structure

Key principle: Algorithms that only work on structure (not data-dependent) are not JIT-compiled and reside in primary source files.

---

## Debugging and Error Handling [00:19:30]
[![00:19:30](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=1170s)

### Assertion System
"If I want to turn on the assertions, I can go to GB_debug.h and recompile the code, then all these assertions become active."

- **GB_assert macros**: Development-time checks that validate matrix integrity
- **GB_Matrix_check**: Exhaustive validation that walks across entire matrix (very slow but comprehensive)
- Used via `assert(GB_Matrix_check(A, GB_0))` where GB_0 means don't print anything
- Can change GB_0 to GB_5 to enable printing during debugging

### Error Logging Limitations
"The spec committee decided to put the error string into the output object itself. But what if the error is so bad the output object is null or broken? There's no other place to put the error."

- Error strings stored in output matrices (C matrix)
- Some methods have no place to log errors, can only return error codes
- Error strings are helpful for Python wrappers that raise exceptions with error messages

### Memory Debugging [00:27:00]
[![00:27:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=1620s)
"I have my own valgrind. I have a table of all the mallocs I've ever done, and I delete the ones I free. So it's the list of all live malloc spaces with their sizes."

Features:
- Custom memory tracking table (order N search time, just for debugging)
- Validates malloc sizes on free (catches memory corruption)
- Tracks memory usage across the library
- Easier to use than valgrind for quick memory leak detection

---

## Matrix Representation and Formats [00:05:00]
[![00:05:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=300s)

### Internal Representation
"All the kernels are agnostic to storage format. When you're looking at code internally, the code is written as if all matrices are stored by column. If I have a CSR matrix and I call this code, the i's and j's are reversed."

Key concepts:
- **Internal view**: All matrices treated as column-stored (CSC)
- **CSR matrices**: Treated as CSC with i and j reversed
- **Matrix structure**: Collection of vectors with a flag indicating row or column orientation
- **Transposition handling**: CSR transposed is identical to operating on CSC

---

## Mask Handling [00:12:00]
[![00:12:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=720s)

### GB_get_mask Function
"If the mask is present and iso-valued but not structural, I can make it structural. If the iso value is nonzero, it's always a structural mask. If the value is 0, you have no mask at all."

Optimizations:
- Iso-valued masks with non-zero value → converted to structural masks
- Iso-valued masks with zero value → mask discarded entirely, complement flag negated
- Quick mask check: Empty complemented mask means nothing happens (early return)

---

## User-Callable Method Structure [00:08:00]
[![00:08:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=480s)

### Standard Workflow for All Methods
"Every GrB method has this workflow: I log where I am, I start the burble, I check my input matrices, I grab the descriptor, I grab the mask, and I call the workhorse internal method."

Typical sequence:
1. `GB_WHERE` - Establishes error logging location
2. Input validation (check for null/faulty matrices)
3. `GB_GET_DESCRIPTOR` - Extract descriptor components
4. `GB_get_mask` - Process mask matrix
5. Call internal workhorse method (e.g., `GB_apply`)

### Descriptor Handling
"Once I extract its contents, I no longer use the descriptor. I don't pass it to my internal methods. I just pass in the pieces like is A transpose or not, is the mask complement, and so forth."

---

## Type Compatibility and Validation [00:33:00]
[![00:33:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=1980s)

### Type Checking
"I have to check type compatibility, dimension compatibility, and the whole mask-accum process with the types."

Key validation points:
- Operator input types vs matrix types (typecasting compatibility)
- Accumulator compatibility (can C pass through accumulator?)
- Dimension matching (considering transpose descriptors)
- Temporary matrix T type (Z type of operator output)

---

## Operator Management [00:31:00]
[![00:31:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=1860s)

### Operator Simplification
"I simplify some of the operators down. There's a lot of really strange Boolean operators in the spec, like Boolean divide. I prune them down to a smaller set of actual useful operators."

**Binary operator renaming** (`GB_binop_rename`):
- Boolean operators simplified to minimal useful set
- `SECOND` with bind_first becomes `IDENTITY`
- Index unary operators mapped to equivalent binary operators where possible
- Reduces code branches by eliminating duplicate functionality

### Operator Flipping [00:50:00]
[![00:50:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=3000s)

"Unfortunately I use the word 'flip' for 2 different senses."

**Two meanings of "flip"**:
1. **Flip_ij**: Reversal of i and j indices (for CSR/CSC agnosticism)
   - Used when matrix storage format differs from internal representation
   - Applies to user-defined positional operators
2. **Flip_xy**: Reversal of operator arguments (X and Y)
   - Used when rewriting operations (e.g., `A*B → B^T * A^T`)
   - Non-commutative operators need explicit reversal (divide → reverse_divide, less_than → greater_than)

**Third meaning - Zombie flip** [00:56:30]:
```
GB_flip(i) = -(i+2)  // Maps 0→-2, 1→-3, 3→-5, etc.
```
- Self-inverse function for marking entries as dead (zombies)
- Should be renamed to "zombify" to avoid confusion

**UPDATE:** I have renamed `GB_FLIP` to `GB_ZOMBIE` and changed its definition.
It now computes `GB_ZOMBIE(i)=(~(i))`, the one's complement of i.  Like the
original `GB_FLIP`, the function is its own inverse.

---

## Wait and Pending Operations [00:42:00]
[![00:42:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=2520s)

### Matrix State Management
"This matrix wait will occur if the input matrix has any pending entries or if it has zombies. If it's just jumbled, I'll let it go because the algorithm's agnostic."

**Wait conditions**:
- Pending tuples need assembly
- Zombies need removal
- Jumbled matrices (out-of-order indices) may be tolerated by some algorithms

**Algorithm-specific tolerance**:
- Apply operations don't care if matrix is jumbled
- Some algorithms require sorted indices, trigger full wait

---

## Apply Operation Deep Dive [00:45:00]
[![00:45:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=2700s)

### Three Main Execution Paths

**1. Transpose + Apply** [01:03:00]
"You may be doing C = F(A^T). That's a transpose and the apply of an operator at the same time. That's one kernel, one pass."
- Fused operation handled by `GB_transpose`
- Avoids creating temporary matrix
- Single-pass algorithm

**2. In-place Apply** [01:06:00]
"C = F(C): C and A are aliased to each other, no mask, no accumulator, no typecasting. I call the apply method directly and work on the data in place."
- Special case: `GB_apply_op` called directly on matrix values
- No memory allocation needed
- Fastest execution path

**3. Out-of-place Apply with Shallow Matrix** [01:10:00]
"I construct a new matrix C, but it's going to be a shallow matrix. The P array is shallow, the H array is shallow, the I array is shallow. Only the numerical values are different."
- Creates shallow copy of structure via `GB_shallow_op`
- Shares integer arrays (P, H, I) with input matrix
- Only allocates new space for numerical values (X array)

**Pure shallow case** [01:11:00]:
"If you're applying identity, this whole code takes constant time because the values are shallow too."

---

## Operator Type Handling [01:19:00]
[![01:19:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=4740s)

### Positional Operators
"The positional operators are either 32-bit, 64-bit, or Boolean. You can apply TRIL as a Boolean operator."

Built-in positional operators hardcoded (not in factory):
- `POSITION_I` / `ROW_INDEX`: Returns row index
- `POSITION_J` / `COL_INDEX`: Returns column index
- Boolean variants (TRIL, TRIU)
- Direct implementation in switch-case (always available, cannot be disabled)

### ISO Value Handling [01:13:00]
[![01:13:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=4380s)
"What if you have a matrix that's all ones and you apply a unary operator that replaces all entries with their row index? It's not ISO now. I have to expand from convert an ISO matrix to a non-ISO matrix."

ISO value complications:
- Input ISO → output non-ISO requires expansion
- Must allocate full value array before applying operator
- Detected by examining operator semantics

---

## Parallel Execution [01:20:00]
[![01:20:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=4800s)

### Task Slicing Strategy
"I want to create 32 tasks per thread. If I have 10 threads, I'll create 320 tasks. I cut the matrix up into 320 pieces."

**GB_SLICE_MATRIX design** [01:23:00]:
```
Each task described by 3 integers:
- k_first: First vector to work on
- k_last: Last vector to work on
- p_start: Starting position in first vector
```

**Advantages of this approach**:
- Handles single large vector: All threads work on pieces of one vector
- Handles power-law matrices: Multiple threads on celebrity column, single threads on small columns
- Uniform interface for all sparsity patterns

**OpenMP pragmas** [01:36:00]:
```c
#pragma omp parallel for num_threads(nthreads) schedule(dynamic,1)
```
- Always specify num_threads (for GraphBLAS control)
- Dynamic scheduling with chunk size 1
- Tasks are large, so dynamic scheduling important for load balancing

### Column Index Lookup [01:35:00]
[![01:35:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=5700s)
"If it's hypersparse, you have to look it up: `j = Ah[k]`. If it's not hypersparse, `j = k`."

Handles different sparsity formats:
- Sparse/bitmap/full: j = k (direct mapping)
- Hypersparse: j = Ah[k] (lookup required)
- Macro abstracts this detail for algorithms

---

## Factory Kernel System [01:45:00]
[![01:45:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=6300s)

### Factory Organization
"The factory kernels can be disabled entirely. They're a little bit limiting but rather than creating other methods, I just put them in a big switch case."

**Factory structure**:
1. **Switch factory**: Giant switch-case on operator opcode
2. **Worker function**: Actual implementation for specific (op, type) combination
3. **Macro-based naming**: `GB_WORKER(z_name, a_name, op_name)`
4. **Disable macros**: Can exclude operators to reduce binary size

Example factory kernel: `GB_uop_apply__sqrt_fp32_fp32.c`
- Implements sqrt for float → float
- Same source file provides both apply and transpose variants
- Includes template algorithm code

### Three-tier Execution Strategy [01:50:00]
[![01:50:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=6600s)

**Tier 1: Factory kernels** (fastest)
- Pre-compiled for common (operator, type) combinations
- Direct C code, fully optimized
- Can be disabled at compile-time with `GB_COMPACT`

**Tier 2: JIT compilation** (medium speed)
- Compiles kernel at runtime for specific case
- Requires operator definition string
- Returns `GrB_NO_VALUE` if can't compile
- Fast as Factory kernels once the JIT is compiled

**Tier 3: Generic execution** (slowest) [01:54:00]
"I have a function pointer. I typecast a billion times on each scalar and apply the operator a billion times. It's crazy slow."
- Function pointer calls for each element
- Separate typecast function per element
- Only fallback when factory and JIT both unavailable

---

## Bind First/Second Operations [01:55:00]
[![01:55:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=6900s)

### Binary Operator with Scalar

**Bind_first**: `C = op(scalar, A[i,j])`
- Scalar is first argument to binary operator
- Requires scalar typecast to operator's X type
- Uses `GB_bind1st` template

**Bind_second**: `C = op(A[i,j], scalar)`
- Scalar is second argument
- Typecast scalar to operator's Y type
- Uses `GB_bind2nd` template

Both have separate factory kernels and JIT paths.

---

## Index Unary Operators [01:59:00]
[![01:59:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=7140s)

### User-Defined Index Unary Operators
"This is calling a function pointer that has to be given the row and column indices, and I don't know if you're going to use them. Maybe you just want the theta. I'll give them to you anyway, sorry."

**Challenges**:
- Must provide i, j, value, and theta to all user-defined index unary operators
- Getting column index j is expensive (requires slicing and iteration)
- No way for user to indicate which parameters are actually needed

**Future optimization opportunity** [02:01:00]:
"If you could somehow tell me this operator only depends on the row index and theta, I could use faster algorithms. Version 3 of GraphBLAS - you could do a GrB_set to tell the operator, 'here's a hint: don't give me the input values to the matrix.'"

---

## JIT Kernel Selection [02:02:00]
[![02:02:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=7320s)

### Algorithm Variants
"If I know at compile time it depends on J, I use the more complex algorithm. Otherwise I can use the simpler algorithm."

**For built-in operators**:
- Compile-time knowledge of dependencies
- Can choose optimal algorithm template
- `GB_apply_unop_ip` vs `GB_apply_unop_ijp`

**For user-defined operators**:
- Must assume worst case (depends on all parameters)
- Always uses most general (slowest) algorithm
- Could be optimized with user hints

---

## Accum-Mask Phase [02:04:00]
[![02:04:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=7440s)

### Final Phase of All Operations
"At the end of all these primary methods there's an accum-mask phase."

Used by: apply, reduce, mxm, kronecker, transpose, eWiseAdd, eWiseMult, assign, extract, select

**Special case** [02:04:00]:
"This may take O(1) time if all the stars align: no accumulator, no mask, T is T, C is empty. Just slap the data in and I'm done."

Optimal path:
- No mask present
- No accumulator present
- C is empty
- T can be moved into C directly (shallow copy)

Otherwise, calls `GB_accum_mask` which may invoke `GB_eWiseAdd` and other algorithms.

---

## Key Insights and Design Principles

### Algorithm Coupling
"These algorithms are coupled to each other. If you look at who calls what, it's a rat's nest, because they're all gluing together like Legos. That's what linear algebra looks like."

### Optimization Philosophy
"I want to save time. If you want to take A = absolute value of A, why make a memory copy? I want to do this in place."

Multiple execution paths prioritized by speed:
1. Special cases (in-place, shallow copy, etc.)
2. CUDA acceleration when available
3. Factory kernels (pre-compiled)
4. JIT compilation (runtime compilation)
5. Generic fallback (function pointers)

### Code Complexity Management
"I probably should have broken this up into 5 different functions. They're totally non-overlapping branches."

- `GB_apply_op.c` has 5 completely independent algorithm branches
- Could be separate functions but kept together
- Each branch handles different operator type combinations

### Memory Efficiency
"The output matrix is going to be shallow. Most of it's going to be shallow. The P array is shallow, the H array is shallow, the I array is shallow."

Aggressive use of shallow copies:
- Reuse structure when only values change
- Avoid memory allocation where possible
- Constant-time operations when structure unchanged

---

## Technical Details

### Work Space Management [00:10:00]
[![00:10:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=600s)
"I have this little strangely misspelled thing called `werkspace`. It's a statically declared array of bytes on the stack used for small things that would otherwise require malloc."

Purpose:
- Avoid `alloca()` (not portable, especially on Windows)
- Small arrays (e.g., 400 integers for task descriptors)
- Falls back to malloc if space insufficient

### Typecasting in Factory Kernels [01:52:00]
[![01:52:00](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y&t=6720s)
"My operators don't do typecasting unless it's identity operator. Then I typecast - those are typecasting operators."

Only identity operator factories include typecast logic:
- User explicitly wants typecast (e.g., float → double)
- Built-in typecast kernels for common conversions
- Other operators require exact type match or use JIT/generic path

---

## Session Notes

**GraphBLAS Version**: 9.3.1 (recently released)
**Recording Time**: Approximately 2 hours
**Next Session Topics**: GB_accum_mask, transpose algorithm, eWiseAdd
