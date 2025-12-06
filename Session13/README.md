# Session 13 Summary

## Table of Contents
- [Overview of SuiteSparse GraphBLAS Structure](#overview-of-suitesparse-graphblas-structure)
  - [Progress Review](#progress-review-000025)
  - [Source Code Organization](#source-code-organization-000500)
- [Element-wise Add vs Element-wise Union](#element-wise-add-vs-element-wise-union-000900)
  - [Set Union Semantics](#set-union-semantics-000915)
  - [Use Cases and Advantages](#use-cases-and-advantages-001220)
- [Implementation Details](#implementation-details)
  - [Format Agnostic Algorithm Design](#format-agnostic-algorithm-design-002600)
  - [Transpose Optimization](#transpose-optimization-003700---004700)
  - [Index Binary Operators and the "Flip IJ"](#index-binary-operators-and-the-flip-ij-003100)
- [Zombie Entries](#zombie-entries-003300)
  - [Concept](#concept-003400)
  - [The Zombie Function](#the-zombie-function-003450)
- [Debugging Infrastructure](#debugging-infrastructure-005100)
  - [Debug Levels](#debug-levels-005140)
  - [Assertion Macros](#assertion-macros-005100)
- [Code Generation and Factory Kernels](#code-generation-and-factory-kernels-005600)
  - [Factory vs JIT](#factory-vs-jit-005700)
  - [Binary Operator Factory](#binary-operator-factory-005740)
  - [Disabling Factory Kernels](#disabling-factory-kernels-010900)
- [Template System](#template-system-010300)
  - [Macro-based Templates](#macro-based-templates-010305)
  - [Template Organization](#template-organization-005735)
- [Sparse Add Algorithm Structure](#sparse-add-algorithm-structure-011800)
  - [Multi-phase Approach](#multi-phase-approach-011900)
  - [Work Division Strategy](#work-division-strategy-000640---000810)
- [Control Flow Through Element-wise Add](#control-flow-through-element-wise-add-011400)
  - [Entry Points](#entry-points-011420)
  - [Accumulator-Mask Phase](#accumulator-mask-phase-011600)
- [Language and API Design Choices](#language-and-api-design-choices-010400)
  - [Why C Instead of C++](#why-c-instead-of-c-010408)
- [Technical Details and Optimizations](#technical-details-and-optimizations)
  - [Operator Value Independence](#operator-value-independence-004130)
  - [Special Case: Full Matrix Add with Accumulator](#special-case-full-matrix-add-with-accumulator-004700)
  - [ISO-valued Matrices](#iso-valued-matrices-005500)
- [Recursive Nature of GraphBLAS](#recursive-nature-of-graphblas-012040)
  - [Interconnected Operations](#interconnected-operations-012050)
- [Naming Conventions and Code Navigation](#naming-conventions-and-code-navigation-005000)
  - [Strict Function Call Format](#strict-function-call-format-005010)
  - [Method Naming](#method-naming-004920)
- [Next Session Preview](#next-session-preview-011900)

---

## Overview of SuiteSparse GraphBLAS Structure
[00:00:00]

### Progress Review
[00:00:25]

Tim began by reviewing what has been covered in previous sessions:
- Apply operations
- Basic matrix data structures
- Transpose
- Builder
- Built-in objects
- JIT (covered in an early session)
- Reduce

The session focused on element-wise add operations, which Tim proposed as the next topic to explore.

**Key Quote:** "I thought we would look over what we've done so far. We have done apply, we've done the basic matrix data structures, we've done transpose."

### Source Code Organization
[00:05:00]

Tim provided an overview of the GraphBLAS folder structure, highlighting major components:

- **Matrix multiply (mxm):** 15,000 lines of code
- **Assign:** A "beast" with 40 different kernels
- **Element-wise add:** 4,000 lines of code
- Select, serialize/deserialize, task parallelism
- Compression wrappers for LZ4 and Zstandard

**Key Quote:** "Matrix multiply, that's 15,000 lines of code right there. Assign is a beast. There's 40 different kernels in assign."

## Element-wise Add vs Element-wise Union
[00:09:00]

### Set Union Semantics
[00:09:15]

Tim explained that element-wise add performs a **set union** of matrix patterns:

**Element-wise Add:**
- If entry exists in both A and B: `C(i,j) = op(A(i,j), B(i,j))`
- If entry exists only in A: `C(i,j) = A(i,j)`
- If entry exists only in B: `C(i,j) = B(i,j)`

**Element-wise Union:**
- If entry exists in both A and B: `C(i,j) = op(A(i,j), B(i,j))`
- If entry exists only in A: `C(i,j) = op(alpha, A(i,j))`
- If entry exists only in B: `C(i,j) = op(B(i,j), beta)`

**Key Quote:** "It should have been called simply Union. You're doing a set union of the pattern of both matrices. And then what you have is an operator that says what to do when you have 2 entries that overlap at the same location."

### Use Cases and Advantages
[00:12:20]

Tim provided an example of why element-wise union is necessary:

For computing `C = A - B` (matrix subtraction):
- With element-wise add: entries in B but not A would just be copied as B, which is incorrect
- With element-wise union: using alpha=0 and beta=0, you get `0 - B = -B`, which is correct

**Key Quote:** "If you want to compute the linear algebraic operation of A minus B, subtract 2 matrices, use union, pass in Alpha and Beta as 0 and use the minus operator. You're done, simple."

Another critical use case is user-defined types with comparison operators where you need placeholder values (like infinity) for missing entries.

## Implementation Details

### Format Agnostic Algorithm Design
[00:26:00]

Tim explained his strategy for handling different matrix storage formats (by-row vs by-column):

**The "Agnostic Trick":**
1. Align all matrices to the same format by manipulating transpose descriptors
2. If both inputs need transpose, toggle the output format instead
3. From that point forward, the code doesn't distinguish between row and column orientation

**Key Quote:** "I don't want to have lots of code to do both store by row and store by column transpose or not. I need to conform, I need to map all these into one or few algorithms."

### Transpose Optimization
[00:37:00 - 00:47:00]

When explicit transposes are necessary, Tim performs several optimizations:

1. **Transpose + Typecast:** If transposing anyway, apply typecasting simultaneously
2. **Pattern-only transpose:** If operator doesn't use values, transpose only the sparsity pattern
3. **Mask transpose:** Transpose and typecast mask to boolean in one operation

**Performance Consideration:** User code should manage transposes when possible to avoid repeated work:

**Key Quote:** "If you see a transpose in the output, know there be costs to that. If you want me to transpose it, I'll transpose it, but I'll throw it away. If you want to keep the transpose, transpose the mask first please, and then call me and then reuse it later yourself."

Tim noted that LAGraph caches transposes in graph objects, and MKL Sparse takes a similar approach, but GraphBLAS itself doesn't cache transposes within matrix objects.

### Index Binary Operators and the "Flip IJ"
[00:31:00]

For positional operators that depend on row/column indices:

- When format changes require index reinterpretation, Tim uses "flip IJ"
- For built-in operators, he can swap operator variants (e.g., first_i becomes first_j)
- For user-defined operators, indices must be passed in reversed order

**Key Quote:** "If the operator's positional, I have to revise the operator essentially. I can do this in 2 ways. I can just wherever I call the operator, later on I'll just reverse the roles of I and J. Or it's a lot simpler - I have a first eye operator and you reverse it, let me just rename it first J."

## Zombie Entries
[00:33:00]

### Concept
[00:34:00]

Tim introduced the concept of "zombies" - entries marked for future deletion that remain in the data structure:

- **Live entry:** row index >= 0
- **Zombie entry:** tagged with negative index via `GB_zombie()` function
- **Special value:** -1 represents "nothing there"

### The Zombie Function
[00:34:50]

```c
// Self-inverting function around -1
GB_zombie(0) = -2
GB_zombie(-2) = 0
GB_zombie(-1) = -1
```

**Key Quote:** "A zombie is an entry marked for future deletion. It's still in the land of the living, but it's dead. The way I mark it is I have a function - it's a self-inverting function."

Tim mentioned zombies appear in many algorithms but didn't go into full detail since set/get element operations haven't been covered yet.

## Debugging Infrastructure
[00:51:00]

### Debug Levels
[00:51:40]

Tim explained his custom debugging system:

**Enabling debug mode:**
- Global: Edit `GB_dev.h` to enable assertions across all GraphBLAS
- Per-file: Define `GB_DEBUG` at the top of a specific file

**Debug print levels:**
- `GB_0`: No output (production code)
- `GB_1`: Print name, dimensions, number of entries only
- `GB_2`: Print first 30 entries
- `GB_3`: Print all entries tersely
- `GB_5`: Print all entries in full detail

**Key Quote:** "They're sanity checks. They may, if I'm debugging, and if I have a bug here in this code, I might want to say, well, okay, print me out the matrix. Set the debug level to 2 for this print. That means print the first 30 entries."

The debug system uses `GB_0`, `GB_1`, etc. instead of plain numbers to make searching and toggling debug statements easier.

### Assertion Macros
[00:51:00]

- Assertions are NOT controlled by `-DNDEBUG`
- Must be explicitly enabled to prevent accidental performance degradation
- Include checks like `assert_matrix_ok()` that validate entire matrix structures

## Code Generation and Factory Kernels
[00:56:00]

### Factory vs JIT
[00:57:00]

**Factory Kernels:**
- Pre-compiled at build time
- Located in `factory/` subdirectories
- Fast but numerous (causing long compile times)
- Cannot handle typecasting (combinatorial explosion)

**JIT Kernels:**
- Compiled at runtime
- Use `include/` and `template/` subdirectories
- Can specialize on matrix sparsity formats
- More flexible but require runtime compilation

**Key Quote:** "The factory kernels cannot typecast. There's too many different variants. So my pre-compiled kernels, called factory kernels, which are tons - this is why GraphBLAS takes so long to compile, and I'm trying to reduce the number of factory kernels I use."

### Binary Operator Factory
[00:57:40]

The `bin_op_factory` is a massive switch case (1,000 lines) that dispatches to pre-compiled kernels for:
- Min/max operators
- Arithmetic operators (plus, times, minus, divide)
- Boolean operators
- Comparison operators
- Floating-point operations (power, hypot, fmod, remainder)
- Complex number construction
- Bitwise operations

### Disabling Factory Kernels
[01:09:00]

Users can control which factory kernels are compiled:

- Edit configuration files to disable specific types (e.g., `int16`)
- Reduces binary size (used by FalconDB)
- Disabled kernels return "no value" and fallback to JIT

**Key Quote:** "Every factory kernel has a disable flag. So this entire file can be disabled. I don't disable these, they're basic, very simple functions. But I disable the bigger things."

## Template System
[01:03:00]

### Macro-based Templates
[01:03:05]

Tim's templates are code fragments, not complete functions:
- Function headers are separate to allow stitching multiple templates together
- Heavy use of macros for flexibility
- Enables both factory and JIT to share the same template code

**Key Quote:** "This is not going to be a full function. It's going to be a code fragment that's going to sit inside of a function, because the function header can differ depending on how I want to use the template."

### Template Organization
[00:57:35]

Three subdirectories in each operation folder:
- **factory/**: Pre-compiled factory kernel wrappers (`.c` files)
- **include/**: Header files for both factory and JIT (`.h` files)
- **template/**: Core algorithm templates for both factory and JIT (`.c` files)

**JIT limitation:** Only uses `include/` and `template/`, never `factory/`

## Sparse Add Algorithm Structure
[01:18:00]

### Multi-phase Approach
[01:19:00]

The sparse add algorithm is divided into phases:

**Phase 0:** Determine output sparsity structure
- Find which vectors contain entries (for hypersparse)
- Perform set union of vector patterns

**Phase 1:** Split work and count entries
- Divide work across threads
- Count the size of the output

**Phase 2:** Numerical computation
- Populate output with actual computed values
- Only this phase uses JIT or factory kernels

**Key Quote:** "Most of my sparse algorithms are multiple phases - the symbolic phase to count and figure out what I'm doing and how big, and then I divvy up the work and compute tasks, and I do the numerical work."

### Work Division Strategy
[00:06:40 - 00:08:10]

Tim explained his approach to parallel task distribution:

**Problem:** Simply dividing by columns can create load imbalance if one column is very dense

**Solution:** Allow threads to split individual vectors
- Thread 1 might take columns 1-4 plus half of column 5
- Thread 2 takes the rest of column 5 plus columns 6-N
- Works for both row and column storage
- Critical for vectors and matrices with dense rows/columns

**Key Quote:** "What if you just have a vector? You've got to chop the vector into pieces. What if you have a matrix with a dense vector column in it or row stored by row? Well, you might have to put one thread on this whole chunk of this matrix, and you might have to put a bunch of threads on that one dense vector all by itself."

## Control Flow Through Element-wise Add
[01:14:00]

### Entry Points
[01:14:20]

The call hierarchy for element-wise add:

1. **User API:** `GrB_eWiseAdd_*` or `GrB_eWiseUnion_*` methods
2. **Top-level dispatcher:** Checks inputs, extracts descriptors
3. **GB_ewise():** Handles both add and multiply, manages transposes and format alignment
4. **Special case handlers:** `GB_ewise_fulla()` for all-dense matrices
5. **GB_add():** Main sparse add implementation with phase 0, 1, 2

### Accumulator-Mask Phase
[01:16:00]

After computing the temporary result matrix T, the final step determines if additional work is needed:

**Direct transplant (no additional work) when:**
- No accumulator present
- No transpose of output needed
- Either no mask OR mask was already applied
- C is empty or C_replace is true

**Otherwise:** Call the mask/accumulator method which may recursively call add again

**Key Quote:** "Just about all of my GraphBLAS methods finish either with a transplant or a transplant conform."

## Language and API Design Choices
[01:04:00]

### Why C Instead of C++
[01:04:08]

Tim explained why GraphBLAS uses C with macros rather than C++:

**Reasons for C:**
- Easier to bind to other languages (Python, PostgreSQL, etc.)
- C++ templates only work at compile time
- GraphBLAS needs runtime templating (via JIT)
- GraphBLAS matrices can change type at runtime (spec requirement)
- Must support user-defined types that can be freed and reallocated

**Key Quote:** "C plus plus can only do compile time templating. I can do runtime templating. The GraphBLAS matrix can change its type. You can free it and reallocate it with a new type - that's in the spec. So I have to be able to handle that."

## Technical Details and Optimizations

### Operator Value Independence
[00:41:30]

Some operators don't actually need the values from input matrices:
- Index-only operators (depend only on i,j positions)
- Positional operators like `second` (only use one input)

**Optimization:** When operator doesn't need values from a matrix:
- Only transpose the pattern, not the values
- Significant performance improvement
- Only works with element-wise union (not add, because add passes raw values through)

**Key Quote:** "If the operator doesn't care what the value of its first input is, then I can just get the pattern of A. I don't care, I'll never read the values of A. Great, that's good to know."

### Special Case: Full Matrix Add with Accumulator
[00:47:00]

The `GB_ewise_fulla` kernel handles a very specific case:
- All three matrices (C, A, B) are completely dense
- No mask or mask is not complemented
- No transpose of output
- No typecasting
- No positional operators
- Matrices are not iso-valued
- Accumulator is the same operator as the element-wise operator

This allows for a highly optimized simple loop without any pattern checking.

### ISO-valued Matrices
[00:55:00]

**ISO-valued optimization:**
- Matrix where all entries have the same value
- Store value once instead of for every entry
- For operations producing ISO output: compute operator once, then only compute pattern
- One kernel for all operators (no factory kernel needed)
- Mentioned in context of matrix data structure session

## Recursive Nature of GraphBLAS
[01:20:40]

### Interconnected Operations
[01:20:50]

Tim and Michel discussed the highly interconnected nature of the codebase:

- Element-wise add can call the mask phase
- The mask phase can call element-wise add again (when accumulator is present)
- Matrix operations recursively use other matrix operations
- Creates a complex call graph that doesn't fit traditional tree structures

**Key Quote (Tim):** "It's all because everything talks to everything. I tried to use doxygen for a while for my code. I had this call return structure of my code, and it was a rat's nest."

**Key Quote (Michel):** "It's algebra, so they're all interconnected. You go up, down, left, right, you're gonna run into mirrors."

## Naming Conventions and Code Navigation
[00:50:00]

### Strict Function Call Format
[00:50:10]

Tim maintains a strict convention for function calls:
- Function name followed by single space and opening parenthesis
- No variation from this rule throughout the codebase
- Enables easy searching: `function_name (` finds all call sites

**Key Quote:** "I'm very, very consistent about how I call a function. Always I have the name of the function, I have a single space followed by a single parenthesis, and I never ever deviate from that rule."

### Method Naming
[00:49:20]

- `GB_ewise_fulla`: Full (dense) add with accumulator
- Terse naming convention
- 'a' suffix indicates accumulator is present
- Format indicators in name (full, sparse, bitmap, hypersparse)

## Next Session Preview
[01:19:00]

The session concluded with agreement to continue the deep dive into sparse element-wise add in the next session, starting at the `GB_add()` entry point that implements the multi-phase sparse algorithm.

**Key Quote:** "We start here next time." [Pointing to GB_add sparse implementation]

Michel noted the importance of tracking the location for the next session given the complexity of the deep dive.
