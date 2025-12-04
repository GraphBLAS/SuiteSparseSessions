# Session 14 Summary

## Table of Contents
- [Element-wise Operations Overview](#element-wise-operations-overview)
  - [Element-wise Add vs. Element-wise Union](#element-wise-add-vs-element-wise-union-000046)
  - [Typecast Complexity](#typecast-complexity-000309)
- [Code Structure and Multi-Phase Algorithms](#code-structure-and-multi-phase-algorithms)
  - [Top-Level Method Organization](#top-level-method-organization-000403)
  - [Three-Phase Algorithm Structure](#three-phase-algorithm-structure-002328)
- [32-bit vs 64-bit Integer Infrastructure](#32-bit-vs-64-bit-integer-infrastructure)
  - [New Matrix Integer Types](#new-matrix-integer-types-001620)
  - [Macro-based Type Handling](#macro-based-type-handling-004247)
  - [GB_GET/GB_SET/GB_INC Macros](#gb_getgb_setgb_inc-macros-004333)
- [Workspace Management ("Verk")](#workspace-management-verk)
  - [The Verk Data Structure](#the-verk-data-structure-004703)
  - [Stack-based Workspace with Push/Pop](#stack-based-workspace-with-pushpop-005536)
  - [Microsoft Compiler Limitations](#microsoft-compiler-limitations-005908)
- [Debug Infrastructure](#debug-infrastructure)
  - [Assertion and Checking Philosophy](#assertion-and-checking-philosophy-001324)
  - [Debug Code as Documentation](#debug-code-as-documentation-012710)
- [Parallelism and Task Management](#parallelism-and-task-management)
  - [Hypersparse Vector Mappings](#hypersparse-vector-mappings-002431)
  - [Parallel Set Union of Hyperlists](#parallel-set-union-of-hyperlists-012158)
  - [Task Structure and Types](#task-structure-and-types-013527)
  - [EK Slice vs EY Slice](#ek-slice-vs-ey-slice-013950)
  - [P_Slice: Recursive Work Partitioning](#p_slice-recursive-work-partitioning-015600)
  - [Slice_Vector for Dense Columns](#slice_vector-for-dense-columns-020849)
- [Code Style and Naming Conventions](#code-style-and-naming-conventions)
  - [Greppable Function Calls](#greppable-function-calls-002753)
  - [Memory Management Philosophy](#memory-management-philosophy-013616)
  - [Type Code Enumerations](#type-code-enumerations-011243)
- [Performance Considerations](#performance-considerations)
  - [Mask Application Decisions](#mask-application-decisions-002111)
  - [Thread and Chunk Sizing](#thread-and-chunk-sizing-010635)
  - [Task Count Strategy](#task-count-strategy-014948)
- [Broader Usage Patterns](#broader-usage-patterns)
  - [GB_Add Usage Beyond Element-wise](#gb_add-usage-beyond-element-wise-001000)
  - [Disjoint Matrices Optimization](#disjoint-matrices-optimization-000938)
  - [EY_Slice Usage Across Methods](#ey_slice-usage-across-methods-021315)
- [Key Takeaways](#key-takeaways)

---

## Element-wise Operations Overview

This session focused on element-wise operations in SuiteSparse GraphBLAS, specifically element-wise add and related operations, with deep dives into parallelism, task management, and the new 32/64-bit integer infrastructure.

### Element-wise Add vs. Element-wise Union [00:00:46]

**Discussion:**
- **Element-wise Add**: Performs set union on two matrices of the same size with a binary operator applied at intersections
- **Element-wise Union**: Extension that allows scalar placeholders (alpha/beta) when entries are in one matrix but not the other
- Key difference: E-wise Add just typecasts values when in A but not B (or vice versa), while E-wise Union always applies the operator with scalar placeholders

**Key Quote:**
> "E-wise Add, when you're in A but not B or B but not A, you just fall through to the output. That's kind of awkward actually. Some cases, like if the operator is less than, what does it mean to typecast A to C? That doesn't have much meaning. So I have a variant called E-wise Union." [00:01:12]

**Timestamps:**
- [00:00:46] - Element-wise operations introduction
- [00:01:30] - Difference between Add and Union explained

### Typecast Complexity [00:03:09]

**Discussion:**
Dr. Davis explains the typecast requirements differ between E-wise Add and E-wise Union, making Union more flexible for user-defined types.

**Key Quote:**
> "For E-wise Union, the output of the operator must be castable to matrix C. That's it. For E-wise Add, not only must that happen, but A must be castable to C and B must be castable to C. That's really awkward if you're trying to create an element-wise operator that compares two user-defined types and produces a Boolean." [00:03:14]

**Timestamps:**
- [00:03:09] - Typecast differences explained
- [00:03:33] - User-defined type challenges

## Code Structure and Multi-Phase Algorithms

### Top-Level Method Organization [00:04:03]

**Discussion:**
The E-wise operations have multiple entry points that eventually funnel into common core methods. The code uses a Boolean flag to distinguish between E-wise Add and E-wise Union behavior.

**Key Quote:**
> "Both of these methods eventually come back and call E-wise, which does both E-wise multiply, E-wise add, and also E-wise union. That method calls the method in the add folder, which is gb_add.c." [00:04:42]

**Timestamps:**
- [00:04:03] - Method hierarchy overview
- [00:05:24] - Core working method identified

### Three-Phase Algorithm Structure [00:23:28]

**Discussion:**
E-wise Add uses a multi-phase approach:
- **Phase 0**: Finalize sparsity decision and find which vectors will appear in C (set union of vectors)
- **Phase 1**: Analysis phase to split output and count work
- **Phase 2**: Actual computation

**Key Quote:**
> "Phase 0 is going to finalize the sparsity decision and find what vectors will appear in C. It's a set union of the vectors of A and B basically. And I want to find that set union of the vectors to compute for the output, and also a mapping of which vector in C corresponds to which in A and B." [00:23:39]

**Timestamps:**
- [00:23:28] - Multi-phase algorithm introduction
- [00:24:04] - Phase 0 details
- [00:25:42] - Mapping explanation

## 32-bit vs 64-bit Integer Infrastructure

### New Matrix Integer Types [00:16:20]

**Discussion:**
Matrices now have up to three different integer types:
- **P array** (pointers/offsets): Can be 32-bit or 64-bit
- **I array** (row indices for CSC or column indices for CSR): Can be 32-bit or 64-bit
- **J array** (hyperlist and hyperhash): Can be 32-bit or 64-bit

**Key Quote:**
> "Every matrix now has up to 3 different integers in it. There's the offset array, and there's a Boolean flag that says true or false, are you 32 bit. And the row indices of a CSC or the column indices of a CSR, they can also be 32 or 64." [00:16:41]

**Timestamps:**
- [00:16:20] - Three integer types introduction
- [00:37:01] - Matrix data structure details

### Macro-based Type Handling [00:42:47]

**Discussion:**
Dr. Davis developed a macro system to handle 32/64-bit decisions at runtime without creating thousands of kernel variants. The system uses three pointers (void*, uint32_t*, uint64_t*) where only one of the typed pointers is non-null.

**Key Quote:**
> "I decided what I could do is do it on a very late basis. Most of my algorithms push the decision 'are you 32 bit or 64 bit' down to the very, very fine grain step. I declare 3 pointers: one is a void pointer, and the other 2 are either unsigned 32 or 64. One is null." [00:42:47]

**Timestamps:**
- [00:42:47] - Macro templating approach explained
- [00:43:33] - Example usage shown
- [01:09:40] - JIT kernel specialization

### GB_GET/GB_SET/GB_INC Macros [00:43:33]

**Discussion:**
Three fundamental macros handle runtime type selection:
- **GB_GET**: Reads from 32-bit or 64-bit array, returns 64-bit value
- **GB_SET**: Writes to appropriate 32-bit or 64-bit array
- **GB_INC**: Increments value in appropriate array

In JIT kernels, these macros compile to direct array access with no branching.

**Timestamps:**
- [00:43:33] - Macro examples
- [00:45:03] - JIT kernel optimization

## Workspace Management ("Verk")

### The Verk Data Structure [00:47:03]

**Discussion:**
The "verk" (German-inspired name for "work") is a workspace structure that serves three purposes:
1. Small stack-based workspace (16KB)
2. Error handling setup (logger, error messages)
3. Integer control settings (P/J/I control for 32/64-bit decisions)

**Key Quote:**
> "I named it this way, not because I'm making fun of the German work, but I wanted to have a workspace that I could look for easily and grep for it. I use the word work all over the place, so let's make it 'verk' and then I can grab for it." [00:47:43]

**Timestamps:**
- [00:47:03] - Verk introduction
- [00:51:05] - Data structure details
- [00:53:23] - Integer control explanation

### Stack-based Workspace with Push/Pop [00:55:36]

**Discussion:**
The verk provides a 16KB static workspace with push/pop operations to avoid malloc overhead for small temporary arrays. This approach reduces malloc calls, which is important for languages like Julia that track allocations.

**Key Quote:**
> "I could just malloc this thing. But then there's more mallocs and frees going along, and codes like Julia don't like to see lots of mallocs. They like to see as few mallocs as possible. So I reduced this by putting this little tiny space." [00:55:15]

**Timestamps:**
- [00:55:36] - Push/pop mechanism
- [00:56:23] - Push implementation details

### Microsoft Compiler Limitations [00:59:08]

**Discussion:**
The entire verk workspace system was necessitated by Microsoft's C compiler not supporting variable-length arrays (VLAs), a standard C99 feature. This forced creation of a manual stack implementation.

**Key Quote:**
> "All I would do is just say 'work[3 times n_tasks plus one]' - there, full stop. Just use a variable length array in C. It's fine. Little compiler will just say 'here, sure, go for it.' Microsoft says, 'what is that?' It doesn't support it." [00:58:43]

Additionally, user-defined types are limited to 1024 bytes on Windows due to the same VLA limitation.

**Timestamps:**
- [00:59:08] - Microsoft VLA problem
- [01:00:14] - Stack-based workaround
- [01:03:28] - User-defined type size limit

## Debug Infrastructure

### Assertion and Checking Philosophy [00:13:24]

**Discussion:**
Dr. Davis uses extensive debug assertions throughout the code. Matrices can be checked with different levels:
- Level 0: Check only (no printing)
- Level 2: Print first 30 entries

The `GB_ASSERT_MATRIX_OK` macro expands to full checking when `GB_DEBUG` is defined at the top of a file.

**Key Quote:**
> "I consider this kind of code really an integral part of the algorithm - to assert all my inputs, do the work, assert my outputs are okay. If you have a bug, what I'll do is say I want to print Level 2, which means print the first 30 entries, and it'll print all kinds of things about the matrix." [00:14:24]

**Timestamps:**
- [00:13:24] - Debug scaffolding introduction
- [00:14:00] - Print levels explained
- [00:35:06] - Matrix state assertions

### Debug Code as Documentation [01:27:10]

**Discussion:**
Dr. Davis includes debug code that re-implements algorithms in simple single-threaded form. This serves dual purposes: runtime verification and code documentation.

**Key Quote:**
> "This is Debug Code, but it's handy. I consider this central part of my software. This code is serving 2 purposes: one, it's nice for debug, because if I compute something wrong this code will tell me. And two, it's actually useful when I look back at the code - 'what is this slice vector thing doing?' Oh, it's just doing a union of 2 sorted sets!" [01:27:52]

**Timestamps:**
- [01:27:10] - Debug verification example
- [01:28:30] - Single-threaded reference implementation

## Parallelism and Task Management

### Hypersparse Vector Mappings [00:24:31]

**Discussion:**
When matrices are hypersparse, creating mappings between vector IDs is crucial:
- C to A mapping: Which vector in A corresponds to vector K in C?
- C to B mapping: Same for B matrix
- C to M mapping: Same for mask matrix

These mappings avoid expensive hypersparse lookups during computation.

**Key Quote:**
> "The 10th vector of C is maybe column 100. Column 100 is the 10th vector of C, it's the 5th vector of M, the 17th vector of A and the 42nd vector of B. I have all these mappings that I have to keep track of, and that's what phase 0 is computing." [00:25:01]

**Timestamps:**
- [00:24:31] - Mapping purpose explained
- [01:20:59] - Hyperhash lookup usage
- [01:29:22] - Workspace array scattering

### Parallel Set Union of Hyperlists [01:21:58]

**Discussion:**
When both A and B are hypersparse, computing the set union of their hyperlists is done in parallel. The algorithm:
1. Slice both sorted lists into tasks
2. Each task computes partial union (counting phase)
3. Cumulative sum determines offsets
4. Second pass fills result array

**Key Quote:**
> "I'm doing a set union of these pieces in parallel, and it has to be done in two passes. The first pass just does a count. Then I accumulate the sum of the tasks so every task knows where to start, and then I have to do this work again but actually fill the ch array." [01:23:07]

**Timestamps:**
- [01:21:58] - Parallel set union algorithm
- [01:23:18] - Two-pass approach
- [01:24:46] - Task coordination

### Task Structure and Types [01:35:27]

**Discussion:**
The task list uses a struct array to manage parallel work. Tasks come in two varieties:
- **Coarse tasks**: Handle multiple vectors together
- **Fine tasks**: Slice individual dense vectors into sub-pieces

The task struct contains start/finish positions for up to 4 different matrices (C, A, B, M), allowing consistent slicing across inputs.

**Key Quote:**
> "I might decide, 'oh, here's a pile of vectors in the matrix, I'll call this a coarse task doing multiple vectors.' But this vector has got a ton of entries in it - split it individually into subtasks which I call fine tasks." [01:42:48]

**Timestamps:**
- [01:35:27] - Task struct introduction
- [01:42:34] - Coarse vs fine tasks explained
- [01:47:24] - Multi-matrix coordination

### EK Slice vs EY Slice [01:39:50]

**Discussion:**
Two different slicing strategies:
- **EK Slice**: For single matrix - divides entries (E) uniformly and finds vectors (K) they fall into, may split vectors
- **EY Slice**: For multiple matrices - must maintain coherent slicing across all input/output matrices

**Key Quote:**
> "EK slice is slicing the entries and finding the K's that they're in. This is easy to do but only really works nicely if I have one matrix to deal with. If I have lots of matrices, I have to make slicing that's coherent across all the different matrices." [01:43:43]

**Timestamps:**
- [01:39:50] - EK slice explained with diagram
- [01:43:43] - Multi-matrix coherence requirement

### P_Slice: Recursive Work Partitioning [01:56:00]

**Discussion:**
The P_Slice method recursively partitions irregular work into balanced tasks:
1. Work is represented as a cumulative sum array
2. Recursively split region in half
3. Examine work on each side
4. Assign tasks proportionally to work amount
5. Recurse on each half

Comes in two variants:
- **Approximate balance**: Fast O(T log T) recursive splitting
- **Perfect balance**: Slower O(T log N) binary search per task

**Key Quote:**
> "I don't do a binary search, just cut it cheaply in half. Then I look at how much work there is on either side of that halfway point, and if the bulk of the work is on to the left, I'll put most of the tasks over there, and just maybe fewer tasks over to the right, and then I'll just repeat recursively." [02:03:28]

**Timestamps:**
- [01:56:00] - P_Slice introduction
- [02:00:20] - Recursive worker algorithm
- [02:06:02] - Approximate vs perfect balance trade-offs

### Slice_Vector for Dense Columns [02:08:49]

**Discussion:**
When a coarse task contains a single vector with too much work, the Slice_Vector method divides that one vector into multiple fine tasks across the 3-4 input/output matrices consistently.

**Key Quote:**
> "I need some help here. I need 4 fine grain tasks. So I take this coarse task doing one vector and they go chop, chop, chop, chop, chop. That chop is Slice_Vector - it's going to cut the 3 matrices into pieces, one piece per fine task with roughly the target amount of work." [02:10:49]

**Timestamps:**
- [02:08:49] - Dense vector problem
- [02:10:51] - Slice_Vector purpose
- [02:12:27] - Multi-method usage (12 different algorithms)

## Code Style and Naming Conventions

### Greppable Function Calls [00:27:53]

**Discussion:**
Dr. Davis enforces a strict style: function calls must always have exactly one space before the opening parenthesis. This makes grepping for function usage reliable and fast.

**Key Quote:**
> "If someone contributes code that does this [no space before parenthesis], I say, no, that's bad code. There has to be one space before the parenthesis. Don't edit the code and submit code with no space. I was once typing in Matlab, and Cleve Moler was over my shoulder and said 'What is that space doing?'" [00:28:08]

**Timestamps:**
- [00:27:53] - Function call style rule
- [00:28:26] - Cleve Moler anecdote

### Memory Management Philosophy [01:36:16]

**Discussion:**
Every malloc'd pointer in GraphBLAS tracks its size consistently. This enables:
- Assertion checking during free operations
- Memory reuse without re-malloc
- Better debugging (size mismatches indicate bugs)

**Key Quote:**
> "In GraphBLAS, every single pointer I get from malloc, I keep track of the size of the thing I just malloc'd consistently. That's really useful - asserting that when I free it, it's the size I say it is. And if I want to reuse the memory space for something else, 'oh it's big enough, great!' How do you know that? I kept track." [01:36:27]

**Timestamps:**
- [01:36:16] - Malloc size tracking
- [01:37:15] - Matrix content structure example

### Type Code Enumerations [01:12:43]

**Discussion:**
GraphBLAS has two separate type code enumerations - an internal one from early development and a newer spec-defined one. They differ in numbering but both are maintained.

**Key Quote:**
> "There's an enum somewhere in the spec, and I think they specify these numbers, and they didn't match mine. So I made 2 sets of 2 enums. I don't get them confused. The user doesn't see these, and I don't see the spec ones except in very few places." [01:13:27]

**Timestamps:**
- [01:12:43] - Dual type code system
- [01:14:12] - Historical context

## Performance Considerations

### Mask Application Decisions [00:21:11]

**Discussion:**
The algorithm decides whether to apply the mask during computation or afterward based on sparsity. A complemented sparse mask is particularly expensive to apply during computation.

**Key Quote:**
> "I apply the mask in this case only if the mask is very sparse compared to A or B, because this case is fairly difficult to do - I'm doing both a set union on A and B, and then a set intersection with the mask at the same time, and that's kind of costly. It's only really worth doing if the mask is extremely sparse." [00:21:36]

**Timestamps:**
- [00:21:11] - Mask application strategy
- [00:21:46] - Performance trade-offs

### Thread and Chunk Sizing [01:06:35]

**Discussion:**
GraphBLAS dynamically adjusts thread count based on work size using a "chunk" parameter (default 64K). The ratio of work to chunk size determines threads used, up to the maximum.

**Key Quote:**
> "If I have 20,000 units of work to do and the chunk size is 64K, that says 'well, just use one thread, it's too small.' But if the work is 128,000 and the chunk is 64,000, I'll use 2 threads. I ratchet down the number of threads for every parallel region based on this decision." [01:06:44]

**Timestamps:**
- [01:06:35] - Chunk-based thread scaling
- [01:07:09] - Dynamic thread control

### Task Count Strategy [01:49:48]

**Discussion:**
GraphBLAS typically creates many more tasks than threads (32x by default) to enable load balancing through dynamic scheduling. Imperfect task balance is acceptable when there are enough tasks.

**Key Quote:**
> "I've got 32 times the number of threads for this particular algorithm. If I've got 8 threads, I've got a lot of tasks. Some are imbalanced. I'm gonna schedule them later on dynamically anyway, and that'll all even out." [02:06:07]

**Timestamps:**
- [01:49:48] - Task-to-thread ratio
- [02:06:02] - Dynamic scheduling benefit

## Broader Usage Patterns

### GB_Add Usage Beyond Element-wise [00:10:00]

**Discussion:**
The GB_add method is used in multiple contexts:
- Element-wise add/union (user-facing)
- Accumulator step in mask application
- Pending tuple summation during matrix construction
- Wait operation to flush pending updates

**Key Quote:**
> "GB_add is used as the element-wise add and union, but the element-wise add is also used as the accumulator step. It's also used in pending summation of pending tuples. When you do GB_wait, what I do is take the matrix, take pending tuples, build them into a matrix, and add them together." [00:10:19]

**Timestamps:**
- [00:10:00] - Multiple use cases
- [00:10:53] - Pending tuples explained

### Disjoint Matrices Optimization [00:09:38]

**Discussion:**
A special flag `A_and_B_disjoint` enables optimizations when matrices have no overlapping entries (common during pending tuple insertion).

**Key Quote:**
> "Sometimes I know by construction that the 2 matrices I'm adding have no entries in common at all, in which case there's no operator to apply on the set intersection because there is no intersection. The time I know when that will happen is in wait - all pending tuples are disjoint with the original matrix by definition." [00:09:51]

**Timestamps:**
- [00:09:38] - Disjoint flag introduction
- [00:11:47] - Wait operation usage

### EY_Slice Usage Across Methods [02:13:15]

**Discussion:**
The EY_slice method is used by at least 12 different GraphBLAS operations: element-wise add, element-wise multiply, element-wise union, mask application, and 8 different assign variants.

**Key Quote:**
> "Lots of algorithms need this slicing method: 8 assign methods, element-wise add, element-wise multiply, element-wise union, and the masker. That's 12 total methods." [02:13:39]

**Timestamps:**
- [02:13:15] - Usage enumeration
- [02:14:04] - Future matrix multiply teaser

## Key Takeaways

1. **Complexity of Element-wise Operations**: What seems like a simple "add two matrices" operation involves sophisticated sparsity handling, type management, and parallelization strategies

2. **32/64-bit Infrastructure**: The new integer type flexibility required pervasive changes using macro-based templating to avoid kernel explosion while maintaining performance

3. **Debug Philosophy**: Extensive assertions and reference implementations are considered integral to the algorithm, not optional extras

4. **Parallelization Challenges**: Balancing work across irregular sparse data structures requires multi-phase algorithms with coarse and fine task granularity

5. **Microsoft Compiler Impact**: Lack of VLA support forced creation of custom workspace management, affecting code throughout GraphBLAS

6. **Code Reuse**: Core infrastructure like task management, slicing algorithms, and the verk workspace are shared across many different GraphBLAS operations

**Final Note [02:14:29]:**
> "You haven't even seen the matrix multiply yet - it's even trickier than this. It uses the same task struct but builds the task in a totally different way."
