# Session 9 Summary

## Table of Contents
- [Overview](#overview)
- [Reduce Operations: High-Level Architecture](#reduce-operations-high-level-architecture)
  - [Reduce-to-Vector Implementation](#reduce-to-vector-implementation-001000---002500)
- [The Monoid vs Binary Operator Controversy](#the-monoid-vs-binary-operator-controversy)
  - [The Spec's Binary Operator Requirement](#the-specs-binary-operator-requirement-000600---001000)
  - [Why Monoids Are Computationally Essential](#why-monoids-are-computationally-essential-002300---002700)
- [Reduce-to-Scalar Implementation](#reduce-to-scalar-implementation)
  - [Overview and Special Cases](#overview-and-special-cases-003300---004000)
  - [Iso-Valued Matrix Reduction](#iso-valued-matrix-reduction-003700---004000)
  - [GPU Recursive Reduction](#gpu-recursive-reduction-003500---003630)
- [Panel Reduction Algorithm](#panel-reduction-algorithm)
  - [Purpose and Design](#purpose-and-design-004200---004700)
  - [Terminal Monoids and Short-Circuiting](#terminal-monoids-and-short-circuiting-004700---004900)
  - [Parallel Panel Reduction](#parallel-panel-reduction-004900---005300)
- [Two Reduction Templates](#two-reduction-templates)
  - [Template 1: Panel-Based Reduction](#template-1-panel-based-reduction-004300---005300)
  - [Template 2: Simple Reduction](#template-2-simple-reduction-005400---005700)
- [Kernel Selection Strategy](#kernel-selection-strategy)
  - [Factory Kernels](#factory-kernels-004000---004200)
  - [JIT Kernels](#jit-kernels-004200---004300)
  - [Generic Kernels](#generic-kernels-005900---010000)
- [The Accumulator Problem for Scalars](#the-accumulator-problem-for-scalars)
  - [Unique Design Challenge](#unique-design-challenge-010000---010300)
- [Cast Factory](#cast-factory)
  - [Purpose](#purpose-010259---010430)
- [User-Defined Types and First Operator](#user-defined-types-and-first-operator)
  - [Creating First Operator for User Types](#creating-first-operator-for-user-types-001700---001900)
- [Code Organization and Style](#code-organization-and-style)
  - [Function Call Formatting](#function-call-formatting-002000---002130)
- [Complexity Comparison](#complexity-comparison)
  - [Reduce Operations vs Other Kernels](#reduce-operations-vs-other-kernels-000000---000200)
- [Key Takeaways](#key-takeaways)

---

## Overview
This session covers the implementation of reduce operations in SuiteSparse GraphBLAS, including reduce-to-vector and reduce-to-scalar operations. Dr. Davis demonstrates how these operations are implemented efficiently by reusing existing matrix-vector multiply machinery and discusses the design controversies around monoids versus binary operators in the GraphBLAS specification.

---

## Reduce Operations: High-Level Architecture

### Reduce-to-Vector Implementation [00:10:00 - 00:25:00]

Dr. Davis explains how reduce-to-vector is elegantly implemented as a matrix-vector multiply operation.

**Key Implementation Strategy:**
The reduce-to-vector operation is implemented in only 218 lines of code by converting it into a matrix-vector multiply problem.

> "This is so powerful and so simple. It takes 218 lines to express it. And that's the entire algorithm. We're done. It's that simple and that powerful." [00:11:11]

**Technical Approach:**
- Creates an iso-valued dense vector B (constant time to construct, ~300 bytes)
- Uses first operator as the multiplicative operator in a semiring
- The monoid provided by the user becomes the additive operator
- Delegates to the full matrix-vector multiply machinery (15,000 lines of optimized code)

> "Why implement all of that algorithm for reduce to vector when I'm already doing all that work for matrix vector multiply? Let's just use that to do reduce the vector and then get all this richness of different algorithms and different optimizations and parallelizations." [00:12:47]

**Benefits:**
- Automatic parallelization
- All mask operations supported
- Transpose operations work naturally
- Leverages highly optimized matrix-vector multiply kernels

[00:11:00]

---

## The Monoid vs Binary Operator Controversy

### The Spec's Binary Operator Requirement [00:06:00 - 00:10:00]

Dr. Davis passionately explains why `GrB_matrix_reduce` with a binary operator (instead of monoid) is problematic.

> "This is an abomination. GrB matrix reduce with binary operator should not be in the spec." [00:06:30]

**The Only Place Identity Matters:**
The identity value is only mathematically required in one place in the entire GraphBLAS spec: when reducing an empty matrix/vector to a C scalar.

> "That's it in the entire spec. That's the one place that's actually required mathematically speaking, in the spec, logically speaking." [00:08:08]

**Tim's Implementation Workaround:**
When given a binary operator instead of monoid, he uses `GB_binop_to_monoid()` to find the corresponding built-in monoid. If it's a user-defined operator with no known monoid, he refuses to implement it.

> "If you have a user defined operator, I don't know what monoid it corresponds to. I say sorry, go away. You can't do this. I refuse." [00:09:25]

### Why Monoids Are Computationally Essential [00:23:00 - 00:27:00]

**Performance Argument:**
> "It's mathematically nonsense to give it to me for the matrix matrix multiply and not reduce to vector when mathematically they're so similar and computationally so similar." [00:26:27]

**Parallel Computing Needs:**
The identity value is crucial for efficient parallel reduction. Without it:
- GPU threads can't use identity values for empty results
- Parallelization becomes significantly harder
- Performance suffers unnecessarily

> "If you have the monoid and the identity, it's really super computationally powerful to have that extra rich mathematical structure." [00:24:13]

**Tim's Position:**
> "Give me the stinking identity value. Just give me a stinking monoid. It's probably the most insane part of the spec, honestly." [00:26:36]

[00:06:30]

---

## Reduce-to-Scalar Implementation

### Overview and Special Cases [00:33:00 - 00:40:00]

The reduce-to-scalar operation has several special cases and optimizations:

1. **Empty matrix case**: Returns identity value of monoid
2. **Iso-valued matrices**: Logarithmic time algorithm
3. **GPU reduction**: May return partial results (one per thread block)
4. **CPU reduction**: Two template algorithms (with/without panels)

### Iso-Valued Matrix Reduction [00:37:00 - 00:40:00]

For iso-valued matrices, even massive ones, reduction is computed in logarithmic time.

> "You can create a matrix that's 2 to the 60 by 2 to the 60, it's all equal to one, and you could say, Tim, give me the sum of that. Well, that's actually easy to do. I can do that in log 2 to the 120 time." [00:37:05]

**Algorithm:**
Uses recursive doubling - repeatedly applying the monoid operator to halves of the data, similar to computing X^N with log(N) multiplications.

**Implementation:**
```
GB_reduce_to_scalar_iso_worker()
```
Recursively reduces: if N=1, done; otherwise reduce N/2 and combine.

[00:37:05]

### GPU Recursive Reduction [00:35:00 - 00:36:30]

**Challenge:**
Not all data types/monoids support atomic operations for cross-thread-block reduction on GPU.

**Solution:**
The CUDA kernel may return a partial result - a vector of length equal to the number of thread blocks.

> "I've determined if I do that, if I have a monoid or a data type that I don't know how to get all the thread blocks to cooperate atomically to reduce the thing to a scalar... every thread block produces its own scalar, and I'll reduce it later." [00:35:05]

The function then recursively calls itself to finish the reduction (usually on CPU since the vector is small).

[00:35:00]

---

## Panel Reduction Algorithm

### Purpose and Design [00:42:00 - 00:47:00]

**Why Panels?**
Direct scalar reduction (s += s += s) is inefficient even on single threads. Panel reduction enables vectorization.

**Panel Size:**
- Default: 16 elements
- Float with plus: 64 elements (tuned for performance)
- Defined at compile time per operator/type combination

> "It's better to do it this way. I have an array of size, whatever the panel is, which is by default 16 entries." [00:44:52]

**Vectorization Benefits:**
> "This inner loop here, that is easy for the compiler to vectorize. For many operators this is literally a panel of K plus equals the entry from the matrix." [00:46:23]

[00:42:00]

### Terminal Monoids and Short-Circuiting [00:47:00 - 00:49:00]

**Terminal Monoids:**
Some monoids can short-circuit when they reach a terminal value:
- Boolean OR (stops at true)
- Boolean AND (stops at false)
- Integer times (stops at 0)

**Non-Terminal Monoids:**
Floating-point operations generally cannot short-circuit:
> "Floating point times, if you reach a 0, you're done right? Well, no, technically 0 times NaN is not 0. 0 times NaN or 0 times infinity is NaN." [00:47:31]

**Panel-Based Terminal Checking:**
Instead of checking after every value, checks every 256 panels for performance.

> "I check the terminal condition, and I just count how many terminal. This is also easy to vectorize. Even a break out of this is not so good. It's actually faster just to check to see if I found any this way." [00:48:41]

[00:47:02]

### Parallel Panel Reduction [00:49:00 - 00:53:00]

**Task-Based Parallelism:**
- Each task gets its own panel (allocated on stack)
- Tasks are independent until final combination
- Typically creates 64x more tasks than threads for load balancing

**Atomic Early Exit:**
When a task finds a terminal value, it sets an atomic flag so other tasks can skip work.

> "If a task is already started, it won't get terminated unless it finds itself a terminal condition. But you see, I have 64, I typically allocate a lot of tasks. If you have 10 threads, I'll create 64 times that number of tasks, so I have 640 tasks." [00:51:52]

**Relaxed Termination:**
Tasks that have already started continue to completion, but new tasks check the atomic flag and skip if terminal value found.

[00:49:22]

---

## Two Reduction Templates

### Template 1: Panel-Based Reduction [00:43:00 - 00:53:00]

**File:** `GB_reduce_to_scalar_template.c`

**When Used:**
- No zombies present
- Matrix is not bitmap format
- Optimized for performance through vectorization

**Characteristics:**
- Uses panel of size 16 (or 64 for certain operators)
- Easier for compiler to vectorize
- Handles terminal monoids efficiently
- Both sequential and parallel variants

### Template 2: Simple Reduction [00:54:00 - 00:57:00]

**File:** `GB_reduce_to_scalar_jit.c`

**When Used:**
- Matrix has zombies
- Matrix is in bitmap format
- Fallback when panel method can't be used

**Characteristics:**
- More flexible - can skip zombies and bitmap gaps
- Simpler algorithm
- Slightly slower (not vectorizable)
- Still handles terminal monoids

> "I realized in some cases it wasn't as fast as it should be. And so that's why I wrote this faster method, which is easier, more easily vectorizable." [00:58:23]

[00:54:10]

---

## Kernel Selection Strategy

### Factory Kernels [00:40:00 - 00:42:00]

**Pre-compiled Kernels:**
Only available for specific built-in monoids:
- min, max, plus, times, any
- Boolean operations

**Implementation:**
Big switch case based on monoid opcode, calling pre-compiled optimized kernels.

**Location:** `FactoryKernels/` folder, e.g., `GB_red_*` files

[00:41:12]

### JIT Kernels [00:42:00 - 00:43:00]

Use the same two templates as factory kernels but compile at runtime for:
- Custom type combinations
- User-defined types (with built-in monoids)
- Any monoid not covered by factory kernels

### Generic Kernels [00:59:00 - 01:00:00]

**Fallback Implementation:**
When JIT is disabled, uses function pointers and memcpy operations.

**Performance:**
> "Everything's mem copies and function pointers... It's gonna be slow. Well, yeah, but it's scalar code. It's calling everything once, so it doesn't matter." [01:02:00]

[00:59:07]

---

## The Accumulator Problem for Scalars

### Unique Design Challenge [01:00:00 - 01:03:00]

Reduce-to-scalar is an outlier in GraphBLAS operations:

**The Problem:**
- Can pass an accumulator operator
- Cannot pass a mask (it's a single scalar)
- Uses C scalar (not GrB_Scalar)
- C scalar is always present (not sparse)

> "GrB reduced to scalar is an outlier. It uses a C scalar, which is evil for one... you can apply an accumulator, but you cannot pass a mask." [01:00:41]

**Different from Standard Accumulator:**
> "There's no bypass the accumulator operator, because there's a C scalar which is always present. When you have an accumulator and there's no entry present, it behaves different. This is different. It's always present." [01:02:25]

**Implementation Choice:**
Rather than reusing the general accumulator-mask machinery, Tim hard-coded a special path using function pointers.

> "I could hoist this up and call my accumulator mask process - that felt like overkill. And so I just hard coded it into this algorithm." [01:01:35]

[01:00:13]

---

## Cast Factory

### Purpose [01:02:59 - 01:04:30]

**Function:**
Returns function pointers for type casting between all built-in types (13x13 matrix of cast functions).

**Special Cases:**
- User-defined to user-defined: just memcpy
- Same type to same type: just memcpy
- Otherwise: actual type conversion function

> "What the cast factory does, it returns a function pointer to one of like 13 squared functions. It's just a table of functions that cast from int 8 to double, from double to int 16..." [01:03:25]

**Usage:**
Used in generic kernels and accumulator operations where performance of a single call doesn't matter.

> "If you reduce a matrix of a billion entries down to a scalar, and I call one function pointer a few times, you'll never notice." [01:04:54]

[01:02:59]

---

## User-Defined Types and First Operator

### Creating First Operator for User Types [00:17:00 - 00:19:00]

**Challenge:**
For reduce-to-vector, need a first operator for the multiplicative part of the semiring.

**Solution for User-Defined Types:**
Creates a special operator with opcode `GB_FIRST_opcode` but applied to user-defined type.

> "I have an OP code for this. So I create a new operator. It's not user defined, it's got its own operator code called first. I'm using a built-in opcode for a user-defined type." [00:18:27]

**Implementation:**
Just a memcpy operation of the appropriate byte size for the user-defined type.

> "It's just a copy. It's easy. You know what it is. It's copy those bytes over. If it's 12 bytes, copy the 12 bytes. If it's 20 bytes, copy the 20 bytes." [00:19:00]

[00:17:43]

---

## Code Organization and Style

### Function Call Formatting [00:20:00 - 00:21:30]

Dr. Davis is very particular about code formatting for searchability:

**Rule:**
Function calls always formatted as: `function_name<SPACE>(`

> "Whenever I call a function in my code, it's always function space left paren, full stop. Why do I care about that? Because if I know in my code where do I call a certain function, I can grep for that pattern." [00:20:59]

**Benefit:**
Using `grep "GB_mxm ("` finds exactly where the function is called, with no false positives.

[00:20:30]

---

## Complexity Comparison

### Reduce Operations vs Other Kernels [00:00:00 - 00:02:00]

Dr. Davis's assessment of operation complexity:

**Simple Operations:**
- Apply: Very simple
- Reduce: Simple (with some subtleties)

**Complex Operations:**
- Assign: Huge codebase
- Matrix multiply: Huge codebase
- Element-wise operations: Complex
- Transpose: Fairly complex (multiple algorithms, sometimes uses build)

**Kronecker Product:**
> "My GrB Kronecker isn't very fast. It's not a high performance product. I don't think everybody uses it, so I didn't put a lot of effort into it. It just works." [00:02:00]

[00:00:45]

---

## Key Takeaways

1. **Code Reuse**: Reduce-to-vector brilliantly reuses matrix-vector multiply machinery, saving thousands of lines of code while maintaining high performance

2. **Design Philosophy**: The monoid requirement isn't just mathematical pedantry - it's essential for performance, especially in parallel/GPU contexts

3. **Performance Tuning**: Multiple algorithm variants (panel-based vs simple, sequential vs parallel) selected at runtime based on matrix properties

4. **Vectorization**: Panel-based reduction enables SIMD vectorization that would be impossible with naive scalar accumulation

5. **Spec Criticism**: GraphBLAS spec's insistence on binary operators for reduce (while requiring monoids for matrix multiply) creates unnecessary implementation complexity and performance limitations

6. **Terminal Monoids**: Careful optimization for short-circuiting with terminal values, but balanced against the cost of checking terminal conditions

7. **Edge Cases**: Special handling for iso-valued matrices, empty matrices, GPU partial reductions, and the unique accumulator semantics for scalar results
