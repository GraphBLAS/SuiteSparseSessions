# Session 11 Summary

## Table of Contents
- [Introduction and Overview](#introduction-and-overview-000003)
- [Matrix States and Completion](#matrix-states-and-completion-000047)
  - [Finished vs. Unfinished Matrices](#finished-vs-unfinished-matrices)
  - [The Build Process Role](#the-build-process-role-000129)
- [Duplicate Operators](#duplicate-operators-000612)
  - [Spec Requirements vs. Implementation](#spec-requirements-vs-implementation)
  - [Order-Preserving Duplicate Handling](#order-preserving-duplicate-handling-000730)
  - [Motivation: Pending Updates](#motivation-pending-updates-000836)
- [The GB_builder Function Interface](#the-gb_builder-function-interface-001101)
  - [Parameters and Design](#parameters-and-design)
  - [Workspace vs. Input Arrays](#workspace-vs-input-arrays-001957)
- [Five-Phase Build Algorithm](#five-phase-build-algorithm-002900)
  - [Phase 1: Copy and Sanitize](#phase-1-copy-and-sanitize-002930)
  - [Phase 2: Sorting](#phase-2-sorting-003704)
  - [Type-Agnostic Sorting](#type-agnostic-sorting-003841)
  - [Phase 3: Counting and Analysis](#phase-3-counting-and-analysis-004000)
  - [Phase 4: Allocating Output](#phase-4-allocating-output-004200)
  - [Phase 5: Numerical Assembly](#phase-5-numerical-assembly-005000)
- [Handling Duplicates Across Thread Boundaries](#handling-duplicates-across-thread-boundaries-010900)
  - [Ownership Rules](#ownership-rules-010903)
  - [Critical Assertion](#critical-assertion-011106)
- [The Switch Factory Pattern](#the-switch-factory-pattern-005730)
  - [Templating in C](#templating-in-c-005737)
  - [Factory Kernels](#factory-kernels-010200)
- [JIT Compilation](#jit-compilation-012037)
  - [Enumification](#enumification-012108)
  - [Type Encoding](#type-encoding-012325)
- [Typecasting Complexity](#typecasting-complexity-012700)
  - [Five Different Types](#five-different-types-012717)
  - [The GB_BUILD_DUPE Macro](#the-gb_build_dupe-macro-012829)
- [Performance Optimizations](#performance-optimizations-003540)
  - [Fast Build Detection](#fast-build-detection-003603)
  - [Greased Lightning Case](#greased-lightning-case-010000)
- [The Hyperhash Data Structure](#the-hyperhash-data-structure-013033)
  - [Purpose and Design](#purpose-and-design-013050)
  - [When It's Built](#when-its-built-013207)
  - [Storage Requirements](#storage-requirements-013344)
- [Matrix Dimensions in Build](#matrix-dimensions-in-build-013849)
  - [Pre-allocated Output](#pre-allocated-output-013922)
  - [Output Not Empty Error](#output-not-empty-error-014242)
- [Build Template and Code Structure](#build-template-and-code-structure-010518)
  - [Template Design](#template-design-010525)
  - [ISO-valued Optimization](#iso-valued-optimization-010612)
- [Error Handling and Edge Cases](#error-handling-and-edge-cases-001229)
  - [Input Validation](#input-validation-003304)
  - [Invalid Matrix State](#invalid-matrix-state-014517)
- [Session Participants](#session-participants)
- [Technical Scope](#technical-scope)

---

## Introduction and Overview [00:00:03]

Session 11 focuses on the builder sub-package in SuiteSparse GraphBLAS, specifically how vectors and matrices are built from tuples. Dr. Davis explains that the builder is closely tied to the matrix data structure and its four fundamental formats (sparse, hypersparse, bitmap, full).

**Key Quote:** "This will be closely tied to the matrix data structure. There's 4 fundamental formats: sparse, hypersparse, bitmap and full."

## Matrix States and Completion [00:00:47]

### Finished vs. Unfinished Matrices

Dr. Davis describes the concept of a "finished" matrix that has no zombies, no pending operations, and is sorted. Once a matrix is "waited upon," it becomes a read-only object that can be safely shared across threads.

**Key Quote:** "A finished matrix has none of those, no zombies, no pending operations. And it's in order. Because once you wait on the matrix, it's got to be a read-only object or some other potentially graph user thread that's calling GraphBLAS wants to share that matrix."

### The Build Process Role [00:01:29]

The build operation is fundamental to handling pending updates. It takes a list of pending inserts (row, column, value tuples) and an operator that determines how to handle duplicates.

## Duplicate Operators [00:06:12]

### Spec Requirements vs. Implementation

The GraphBLAS spec states that duplicate operators must be associative and commutative. However, Dr. Davis's implementation relaxes this requirement.

**Key Quote:** "The spec says that this duplicate operator must be associative and commutative. Full stop. And it's an operator. It's not a monoid. So that's unfortunate. We don't necessarily know what the mono identity value is."

### Order-Preserving Duplicate Handling [00:07:30]

Dr. Davis's implementation guarantees that if there are duplicates, the operator is applied exactly in order. This enables non-associative and non-commutative operators like "first" and "second."

**Key Quote:** "I allow the dupe to be non-associative and not commutative, and I guarantee that if there are duplicates, I apply the operator exactly in order. Just so any one duplicate cannot be summed in parallel."

### Motivation: Pending Updates [00:08:36]

The ordered duplicate handling is crucial for pending updates. When you do multiple assigns to the same position, the last one should win (equivalent to using the "second" operator).

**Key Quote:** "If you do 3 inserts in order, it's the last one that wins right. So it overwrites the first through 2, and so that corresponds in here to the second operator."

## The GB_builder Function Interface [00:11:01]

### Parameters and Design

The builder function takes:
- Output matrix (pre-allocated)
- Three C arrays: row indices, column indices, and values
- Length of arrays
- Duplicate operator
- Type information

**Key Quote:** "The build method takes in the output matrix that already has to be allocated. It's got 3 C arrays: indices, row and column indices and values."

### Workspace vs. Input Arrays [00:19:57]

The builder supports two ways of providing input:
1. As workspace that will be consumed and freed
2. As read-only input that won't be modified

This design enables performance optimization for internal use while maintaining safety for user-facing APIs.

## Five-Phase Build Algorithm [00:29:00]

### Phase 1: Copy and Sanitize [00:29:30]

The first phase copies user input into modifiable space and performs basic validation:
- Range checking
- Detecting if input is sorted
- Finding obvious duplicates

**Key Quote:** "Every thread goes through and does the copy of its space, and it also does some simple analysis and some error handling. It detects if it's out of range."

### Phase 2: Sorting [00:37:04]

If the input isn't already sorted, a parallel merge sort is performed on the tuples. To ensure stability, each tuple is numbered (K value) before sorting.

**Key Quote:** "I need this sort to be stable. I need I, J and the K value and K is not the K-th vector of the matrix. It's just the Kth tuple."

### Type-Agnostic Sorting [00:38:41]

The sort only operates on integer indices, not values, making it type-agnostic.

**Key Quote:** "This is type agnostic. I'm not sorting the data types as I'm doing a sort, just the integers. It's sorting a pair of integers, or sorting a tuple of 3 integer arrays."

### Phase 3: Counting and Analysis [00:40:00]

After sorting, the algorithm counts:
- Number of unique vectors per thread
- Number of entries after removing duplicates
- Detection of which entries are duplicates (marked with -1)

### Phase 4: Allocating Output [00:42:00]

The output matrix is allocated as hypersparse format, and the P and H arrays are constructed using cumulative sums across thread results.

**Key Quote:** "I've allocated the matrix. It's always hypersparse."

### Phase 5: Numerical Assembly [00:50:00]

The final phase copies values and applies duplicate operators. This phase handles:
- Greased lightning case: no duplicates, no typecast, sorted input can be transplanted directly
- Parallel memcpy for simple copies
- Factory kernels for built-in types
- JIT compilation for combinations not covered by factory
- Generic kernel for complex typecasting scenarios

## Handling Duplicates Across Thread Boundaries [01:09:00]

### Ownership Rules [01:09:03]

A subtle aspect of the parallel algorithm is handling duplicates that span thread boundaries:

1. If a thread's first entry is marked as duplicate (row index = -1), it skips it, letting the thread to the left handle it
2. A thread scans forward into its right neighbor's region to collect all duplicates for its entries

**Key Quote:** "If your first row index is minus one, then you're just gonna skip it. Let the guy to your left do it."

### Critical Assertion [01:11:06]

An important assertion in the code verifies the loop invariant: at a certain point, you're either at a unique entry or at the first of all duplicates for that position.

**Key Quote:** "At this point in time I know I am not, I'm at the I'm either don't have a duplicate, or I have the first of all the duplicates for this entry."

## The Switch Factory Pattern [00:57:30]

### Templating in C [00:57:37]

Dr. Davis uses a "switch factory" pattern for C templating, with consistent code structure:

```
#define the worker for the switch factory
launch the switch factory
```

**Key Quote:** "I always first say: define the worker for the switch factory, launch the switch factory. I'm very consistent, exceedingly consistent with this code style."

### Factory Kernels [01:02:00]

Factory kernels are pre-compiled for common combinations of operators and types. The build template is used for many type combinations with different duplicate operators (plus, times, min, max, first, second).

## JIT Compilation [01:20:37]

### Enumification [01:21:08]

When factory kernels aren't available, the JIT system:
1. Encodes the problem (operator + types) into a 28-bit code
2. Looks up the kernel in a hash table
3. Compiles it if not found
4. Calls the compiled kernel

**Key Quote:** "It's 28 bits. So there's 2 to the 28 builds I could do for JIT, not counting different user defined operators."

### Type Encoding [01:23:25]

- 8 bits (2 hex digits) for the operator (254 possible operators)
- 4 bits per type (up to 5 different types in complex typecasting scenarios)
- One special code for all user-defined types (suffix distinguishes which UDT)

## Typecasting Complexity [01:27:00]

### Five Different Types [01:27:17]

In the most complex case, the build operation may involve 5 different types:
1. Duplicate operator input type 1
2. Duplicate operator input type 2
3. Duplicate operator output type
4. Input matrix value type
5. Output matrix value type

**Key Quote:** "The generic kernel is an ugly beast, because these 5 types, everything has to be cast to everything."

### The GB_BUILD_DUPE Macro [01:28:29]

The duplicate operator macro handles all necessary typecasting:
- Cast S to Y (operator input type 1)
- Cast T to X (operator input type 2)
- Apply the operator
- Cast Z back to T (output type)

## Performance Optimizations [00:35:40]

### Fast Build Detection [00:36:03]

The algorithm includes several fast paths:
1. If input is sorted with no duplicates: 10x speedup possible
2. Early detection during the copy phase
3. Direct transplant of arrays when possible

**Key Quote:** "It's a huge difference, like order 10x speed up. If I can avoid duplicates building construction and avoid sorting, it's easily 10x performance difference."

### Greased Lightning Case [01:00:00]

The fastest case: sorted, no duplicates, no typecasting. The input workspace can be transplanted directly into the output matrix with virtually no data movement.

**Key Quote:** "This is the greased lightning case. Greece lightning."

## The Hyperhash Data Structure [01:30:33]

### Purpose and Design [01:30:50]

The hyperhash provides fast inverse lookup for hypersparse matrices:
- Forward lookup: J = H[K] (constant time via hyperlist)
- Inverse lookup: K = find(J) (requires search)

Instead of binary search, the hyperhash uses a hash table implemented as a CSR matrix.

**Key Quote:** "I can take the one I'm looking for, hash its name, look through the bucket. Those set of buckets, they're not changing dynamically. That's just a CSR matrix, one bucket per row. So I just use a GraphBLAS matrix as the set of hash tables."

### When It's Built [01:32:07]

- Not built for small hypersparse matrices (fewer than ~1000 vectors)
- Binary search is sufficient for small cases
- Critical for GPU performance
- Not serialized (rebuilt on deserialization for compact storage)

**Key Quote:** "If I have a hypersparse matrix with fewer than a thousand vectors unique vectors in it, I think I never build it. It's not worth it."

### Storage Requirements [01:33:44]

The hyperhash is very compact: O(K) where K is the number of non-empty vectors. It stores only indices, not values, with typically 4x as many buckets as vectors to minimize collisions.

**Key Quote:** "It's order of the number of non-empty vectors in the matrix, which is N less than N. It's very small."

## Matrix Dimensions in Build [01:38:49]

### Pre-allocated Output [01:39:22]

A key design decision: the build function doesn't create the matrix, it populates a pre-allocated one. The user must create an empty matrix of the desired dimensions and type before calling build.

**Key Quote:** "You give it the matrix you're building, and it's not star C there, it's not a constructor. You give it the matrix of a given size and a given type, and that's the size of your output."

### Output Not Empty Error [01:42:42]

The spec requires that the output matrix be empty, though Dr. Davis notes this is somewhat redundant since the first operation frees the content anyway.

**Key Quote:** "The very first thing is, thank you for this empty suitcase. Now I know the size of the suitcase. And now I'll throw away all the data structures in it, because I'm gonna replace it with mine anyway."

## Build Template and Code Structure [01:05:18]

### Template Design [01:05:25]

Many templates are not full functions but code blocks starting with a curly bracket, designed to be embedded in functions. The build template is only 113 lines with two versions: one with duplicates, one without.

### ISO-valued Optimization [01:06:12]

When building ISO-valued matrices (all values the same), the numerical assembly is skipped entirely, processing only the structural pattern.

## Error Handling and Edge Cases [00:12:29]

### Input Validation [00:33:04]

Error reporting is complicated by the algorithm's agnostic treatment of CSC vs CSR:
- Internally, J is always the vector index and I is the index within vector
- Error messages must translate back to rows/columns based on actual format
- Ternary operators handle this translation

**Key Quote:** "When I report an error back to the caller, I have to say that, oh, this is a row, and this is a column, and it's bad, it's out of bounds. I handle the flip algorithmically internally."

### Invalid Matrix State [01:45:17]

After GB_phybix_free, the matrix is in an invalid state with bad magic number. If the build fails (e.g., out of memory), the user is left with an invalid matrix.

**Key Quote:** "If by chance something happens later on, like the builder runs out of memory, you were returned with an invalid matrix. You ask to try, you look at it, and it'll say invalid object if you try to use it."

## Session Participants

- **Dr. Tim Davis**: Presenter, creator of SuiteSparse GraphBLAS
- **Michel Pelletier**: Primary participant asking questions
- **Piotr Luszczek**: Participating researcher

## Technical Scope

This session provides deep insight into:
- Parallel matrix construction algorithms
- Trade-offs between spec compliance and practical implementation
- Performance optimization strategies
- The complexity of supporting arbitrary typecasting
- How JIT compilation integrates with factory kernels
- Design decisions in the GraphBLAS API
