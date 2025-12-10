# Session 2 Summary

## Table of Contents
- [Overview](#overview)
- [Code Organization and Matrix Object Structure](#code-organization-and-matrix-object-structure)
  - [Source Code Structure](#source-code-structure-000109---000903)
  - [Unified Object Model](#unified-object-model-000415---000700)
- [Core Data Structure Components](#core-data-structure-components)
  - [Basic Object Header](#basic-object-header-000940---001200)
  - [Type and Format Information](#type-and-format-information-001210---001400)
- [Matrix Storage Formats](#matrix-storage-formats)
  - [The Four Base Formats](#the-four-base-formats-001630---002230)
  - [Format Unification - The P-H-B-I-X Arrays](#format-unification---the-p-h-b-i-x-arrays-002300---003400)
  - [Full Format](#full-format-002430---002700)
  - [Bitmap Format](#bitmap-format-002710---002900)
  - [Sparse Format (CSR/CSC)](#sparse-format-csrcsc-003140---003400)
  - [Hypersparse Format](#hypersparse-format-003410---004000)
- [Advanced Features and Optimizations](#advanced-features-and-optimizations)
  - [Iso-Valued Matrices](#isovalued-matrices-004250---004630)
  - [Format-Agnostic Kernels](#format-agnostic-kernels-001440---001530)
  - [Unified Iteration Pattern](#unified-iteration-pattern-004700---005800)
- [Dynamic Graph Capabilities](#dynamic-graph-capabilities)
  - [Zombies (Pending Deletions)](#zombies-pending-deletions-010550---011520)
  - [Pending Tuples (Pending Insertions)](#pending-tuples-pending-insertions-011600---012800)
  - [Relationship Between Zombies and Pending Tuples](#relationship-between-zombies-and-pending-tuples-013150---013300)
  - [Limited Dynamic Capability](#limited-dynamic-capability-013230---013500)
- [Additional Pending Work](#additional-pending-work)
  - [Jumbled Matrices](#jumbled-matrices-013500---014300)
  - [Non-Empty Vector Count](#non-empty-vector-count-010620---010730)
- [Future Topics](#future-topics)
  - [Hyper Hash](#hyper-hash-014300---014600)
  - [Sparsity Control](#sparsity-control-014400---014430)
  - [Shallow and Static Headers](#shallow-and-static-headers-014430---014500)
- [Design Philosophy](#design-philosophy)
  - [Performance vs. Flexibility Trade-offs](#performance-vs-flexibility-trade-offs-013400---013500)
  - [Format Family Design](#format-family-design-001700---001830)
  - [Algorithmic Tolerance Patterns](#algorithmic-tolerance-patterns-013900---014200)
  - [Code Readability and Grep-ability](#code-readability-and-grep-ability-013520)
- [Release Information](#release-information)

---

## Overview
This session covers the internal data structures used in SuiteSparse GraphBLAS, focusing on how matrices, vectors, and scalars are represented and manipulated. Dr. Tim Davis provides a detailed walkthrough of the code structure, data formats, and design decisions.

---

## Code Organization and Matrix Object Structure

### Source Code Structure [00:01:09 - 00:09:03]

The matrix data structure is defined in the GraphBLAS/Source/builtin folder, specifically in the `GrB_opaque.h` file. Key organizational points:

- Core operations for matrices are in the matrix folder (create, reallocate, query operations)
- Vector and scalar folders exist but are minimal since they share the same underlying structure
- The actual data structure definition is in the GraphBLAS/Source/builtin folder, not with the core operations

**Key Quote**: "A scalar is just a 1-by-1 matrix. Everything you can do with
the matrix you can do with a 1-by-1 matrix and scalar. A `GrB_scalar` is a 1-by-1
matrix internally, and a `GrB_Vector` is just an n-by-1 `GrB_Matrix`."

### Unified Object Model [00:04:15 - 00:07:00]

All three GraphBLAS objects (matrix, vector, scalar) use the same internal structure:
- Uses a single include file with #include to define all three types
- This workaround is necessary because C's `_Generic` macro requires distinct types
- Would prefer a single struct definition, but the C compiler needs three separate struct definitions for polymorphic functions

---

## Core Data Structure Components

### Basic Object Header [00:09:40 - 00:12:00]

Every GraphBLAS object starts with:
- **Magic number** (64-bit integer): Used to detect uninitialized, freed, or invalid objects
- **Header size**: Size of the malloc'd header (0 if it's a static header on the stack)
- **Name pointer**: User-settable name for the matrix
- **Error message pointer**: Stores error information from operations

**Key Quote**: "Every object my objects always start with this stuff right here. I can take any of my objects and look at the first integer, first 64 bit integer is a magic number. This tells because GraphBLAS I'm supposed to detect if it's an uninitialized object, if it's a freed object, if it's invalid object."

### Type and Format Information [00:12:10 - 00:14:00]

- **Type**: Pointer to `GrB_Type` (handles user-defined types, not just built-in codes)
- **Vector dimension and length**: These define the matrix shape
  - For CSR: vector dimension = number of rows, vector length = row length (# of columns)
  - For CSC: vector dimension = number of columns, vector length = column length (# of rows)
- **is_csc flag**: Determines if stored by column (true) or row (false)

**Key Quote**: "I don't store the number of rows or number of columns in the matrix. I call the number of vectors the `vector dimension`. So if I have a matrix that's stored by column that's 10 by 4, it's the same data structure as a matrix stored by row that's 4-by-10. These 2 are the same data structures. I just have a flag that says, 'Well, who are you?'"

---

## Matrix Storage Formats

### The Four Base Formats [00:16:30 - 00:22:30]

GraphBLAS uses four primary storage formats, each available in both by-row and by-column orientations (8 total):

1. **Full**: Dense storage, all entries present
2. **Bitmap**: Dense arrays with a boolean mask indicating presence
3. **Sparse**: Traditional CSR/CSC format
4. **Hypersparse**: Doubly compressed format for very sparse matrices

**Key Quote**: "I have a total of 8 formats. I have sparse, hypersparse, bitmap, and full, and all of these are by row or by column. "

### Format Unification - The P-H-B-I-X Arrays [00:23:00 - 00:34:00]

Five arrays (P, H, B, I, X) are used across all formats in different combinations:

- **P**: Pointer array (like CSR/CSC pointers)
- **H**: Hypersparse list (names of non-empty vectors)
- **B**: Bitmap array (uint8 presence/absence flags)
- **I**: Index array (row/column indices)
- **X**: Values array (the actual numerical data)

Different formats use different subsets:
- Full: Only X
- Bitmap: B and X
- Sparse: P, I, and X
- Hypersparse: P, H, I, and X

**Key Quote**: "With these 5 macros, every matrix has all 5 components effectively, I can pretend. That could be an expensive ternary test to put all the way down the innermost gut of the code. So the JIT doesn't do this; it compiles a kernel for each case."

### Full Format [00:24:30 - 00:27:00]

- Only stores values in the X array
- No gaps in storage (unlike LAPACK's leading dimension concept)
- Can handle matrices up to 2^60 by 2^60 when iso-valued
- For iso-valued full matrices, uses constant O(1) space

**Key Quote**: "I can create a 2 to the 60 by 2 to the 60 full matrix that's
iso-valued in O(1) time and space.  How many values? That can be a hard
question. If you ask how many entries in a 2 to the 60 by 2 to 60 full matrix,
I have to return infinity. I have to return uint64 max because I've
overflowed."

### Bitmap Format [00:27:10 - 00:29:00]

- Two dense arrays: B (bitmap) and X (values)
- B array is uint8 (not truly a bitmap, more like a "bool map")
- Internally may use values other than 0/1 for masks during computation
- Any user-facing matrix has only 0s and 1s in the bitmap

### Sparse Format (CSR/CSC) [00:31:40 - 00:34:00]

- Traditional compressed sparse row/column format (CSR and CSC)
- Uses P (pointers), I (indices), and X (values)
- Can be iso-valued, making X array size 1
- Access pattern uses macros to handle both sparse and full cases uniformly

### Hypersparse Format [00:34:10 - 00:40:00]

- Doubly compressed format for extremely sparse matrices
- Adds H array to sparse format to track which vectors are non-empty
- Vector dimension can be massive (e.g., 10^30) while only storing a few vectors
- H array always kept sorted for efficient lookup
- Enables efficient representation of graphs with huge node IDs but few edges

**Key Quote**: "Imagine a matrix here that's not 5 by 10, but make it 10 to the 30 by 10 stored by row but with 5 non-empty rows. How would I do that? I have 5 vectors in this matrix that are present that have something potentially in them. And I have to have their 5 names like this is row 1,000, this is row 1 million, this is row 3 billion. That's the H array."

---

## Advanced Features and Optimizations

### Iso-Valued Matrices [00:42:50 - 00:46:30]

When all present entries have the same value:
- X array becomes size 1 instead of size N
- Critical for unweighted graphs where all edges = 1
- Full iso-valued matrices can represent arbitrarily large matrices in O(1) space
- Used internally for operations like matrix-vector reductions

**Key Quote**: "Unweighted graphs are all over the place in graph algorithms. And so matrices where all the entries that are present have the same value, whether it's 1 or pi, or whatever are really important."

### Format-Agnostic Kernels [00:14:40 - 00:15:30]

Once inside kernels, the code becomes CSR/CSC agnostic:
- Pretends everything is by column internally
- A CSR matrix with transpose descriptor = CSC matrix = same thing to the kernel
- Reduces 4 input cases down to 2 cases
- Simplifies kernel implementation significantly

**Key Quote**: "I'm CSR CSC agnostic. If you see the word agnostic in GraphBLAS source code, that means I don't care if it's by row or by column anymore. I pretend it's all by column internally."

### Unified Iteration Pattern [00:47:00 - 00:58:00]

A single iteration pattern works across all four formats using macros:

```c
for (K = 0; K < nvec; K++) {
    J = GBH(H, K);  // Get vector name
    P_start = GBP(P, K);
    P_end = GBP(P, K+1);
    for (P = P_start; P < P_end; P++) {
        if (!GB_BITMAP_CHECK(B, P)) continue;
        I = GBI(I, P);
        AIJ = GBX(X, P, iso);
        // work with entry at (I, J) with value AIJ
    }
}
```

These macros hide the differences between formats:
- **GBH**: Get vector name (K for most, H[K] for hypersparse)
- **GBP**: Get vector start position (P[K] or `K*vlen` for full)
- **GBI**: Get row index (I[P] or P % vlen for full)
- **GBX**: Get value (X[P] or X[0] if iso)

**Key Quote**: "What that means is I can write a single loop to move through all 4 formats. I get the vector length. I walk through all my vectors. I want to know the name of my Kth vector. Well, for sparse bitmap and full it's just K, but hypersparse it's H of K. Same line of code that does all 4 cases."

---

## Dynamic Graph Capabilities

### Zombies (Pending Deletions) [01:05:50 - 01:15:20]

Zombies are entries marked for deletion but not yet removed from the data structure:

- Only exist in sparse/hypersparse formats (bitmap/full can delete immediately)
- Implemented using "flip" operation: index i becomes -(i+2)
- Flip is its own inverse: flip(flip(i)) = i
- Special handling for 0: 0 → -2, -2 → 0
- Maintains sorted order, enabling binary search on rows/columns with zombies
- Created by delete operations and some assign operations
- Killed in batch during matrix wait operations

**Key Quote**: "If it's a 3, and I want to mark it for deletion, I put a minus 3 there. Minus 3 means yeah, you were row 3, but you've been deleted because it's a negative. It's like a tombstone. Your name is 3, but you're dead. That's a zombie. You're still in the land of the living but you're dead. (actually it would not be a minus three, but flip(3) = -5 but it's easier to explain the tombstone if I say minus 3 for row index 3."

**Key Quote**: "Zombies are really useful. If I have a bunch of deletions happening all at same time, I'll do all the deletions, and I may postpone the actual construction of the actual compacted matrix with the deletions been applied. I can do that later."

**Storm Trooper Effect**: "One zombie coming after you is really nasty. If you have a million, it's really easy, because the same work to kill a million zombies versus killing one."

### Pending Tuples (Pending Insertions) [01:16:00 - 01:28:00]

Pending tuples are insertions that haven't been added to the sparse structure yet:

- Only for sparse/hypersparse formats
- Stored as three arrays: row indices, column indices, values
- Includes a dup operator for handling duplicate updates
- Disjoint from existing entries (updates are applied immediately)
- Assembled using `GrB_Matrix_build` + element-wise add
- Enables batching of insertions

**Key Quote**: "A pending tuple is an entry that I tried to insert into the matrix, and I couldn't, because it's not there. Stick this number AIJ into the matrix, oops it's not there already. So what do I do? I'll do it later."

**Non-Associative Operators**: A dup operator is provided to "sum" up duplicates.  Unlike the spec, SuiteSparse/GraphBLAS allows non-associative dup operators like "second" to support "take the last value" semantics:

**Key Quote**: "In GraphBLAS, the GraphBLAS API for the GrB matrix build function says that the dup operator must be associative. But I allow my dup operator for GrB matrix build to be non associative, because I want to use things like the first or second operator. Second means if you see 2 duplicates take the last one in the list."

### Relationship Between Zombies and Pending Tuples [01:31:50 - 01:33:00]

Critical invariant: zombies, live entries, and pending tuples are mutually disjoint in their sparsity patterns:
- Never a pending tuple into a zombie
- Never a pending tuple into a live entry
- If you set an element at a zombie location, it resurrects (unflips the zombie)

**Key Quote**: "What this means is that zombies and pending tuples are disjoint. There's never a pending tuple into a zombie ever. There's never a pending tuple into an entry that's there that's live never, ever. So the zombies, the live entries, the zombies, and the pending tuples are all disjoint from one another in any given matrix in terms of their sparsity pattern."

### Limited Dynamic Capability [01:32:30 - 01:35:00]

The design provides a middle ground between static and fully dynamic:

**Advantages**:
- Batch many small changes efficiently
- Delay expensive restructuring until necessary
- Some algorithms are zombie/tuple tolerant

**Limitations**:
- Must eventually wait (rebuild) before heavy operations
- Not as dynamic as structures like Stinger
- Performance penalty compared to pure static CSR/CSC when fully dynamic

**Key Quote**: "I've hit a spot in between a completely static data structure and a completely dynamic data structure. I've got something in the middle. I've got somewhat dynamic capability, but not fully. Eventually I'd really like to be able to handle arbitrarily dynamically modifying graphs that are changing wildly."

**Use Case**: "It's useful for incrementally making a new matrix or changing a matrix until you then need to do something significant with it."

**Louvain Challenge**: The worst case is algorithms like Louvain where "every loop tries to do an update on a row, or tries to remove a row, and then do a multiply, and then put that row back and remove another row."

---

## Additional Pending Work

### Jumbled Matrices [01:35:00 - 01:43:00]

Jumbled is a special property of sparse/hypersparse matrices indicating indices within vectors may be out of order:

- The H array (hypersparse list) is always kept sorted
- But the I array (row/column indices within each vector) may be unsorted
- Flag indicates uncertainty: jumbled=true means "might be unsorted" (could actually be sorted)
- Special name "jumbled" chosen over "unsorted" for easier grep-ability

**Benefits**:
- Many algorithms don't care about order (jumble-tolerant)
- Algorithms that naturally produce unsorted output can skip sorting
- If never consumed by order-sensitive algorithm, sorting is never performed
- Significant performance savings for temporary intermediate matrices

**Examples**:
- Breadth-first search frontiers are naturally jumbled
- Gustavson-style matrix multiply produces jumbled output
- Reduction operations don't care about order
- Binary search requires unjumbled matrices

**Key Quote**: "If I have an algorithm that's producing a jumbled output, and I don't ever have to wait on it because nobody cares on input with that result, and then I delete that vector, delete that matrix, I never sorted it. And I saved all that time. I never had to compute that answer. I never had to sort all the indices."

### Non-Empty Vector Count [01:06:20 - 01:07:30]

The number of non-empty vectors (nvec_nonempty):
- May be unknown (set to -1)
- Can differ from the number stored in hypersparse list (some could be empty)
- Computation delayed until needed
- Another form of pending work

---

## Future Topics

The session concluded with a preview of remaining topics for future sessions:

### Hyper Hash [01:43:00 - 01:46:00]

- Data structure for hypersparse matrices
- Enables fast inverse lookup: given J (vector name), find K (position in H array)
- Alternative to binary search on H array
- Implemented as a matrix where each row is a hash bucket
- Construction is itself pending work - built only when needed by an algorithm
- GPU-friendly design

**Key Quote**: "My hash buckets are just a matrix. Single `GrB_Matrix` contains
all the hash buckets. Every row is a hash bucket. The construction of the hyper
hash is pending work for a hyper sparse matrix. You can have a hyper sparse
matrix that doesn't have it yet because the next algorithm says I don't care
I'm not going to use it."

### Sparsity Control [01:44:00 - 01:44:30]

Parameters and heuristics for automatically converting between the four storage formats as the sparsity pattern changes.

### Shallow and Static Headers [01:44:30 - 01:45:00]

How matrices can share data without copying:
- Arrays may be owned by another matrix (shallow)
- Headers can be stack-allocated for temporary internal matrices (static)
- Never exposed to end users - only for internal temporary objects

---

## Design Philosophy

### Performance vs. Flexibility Trade-offs [01:34:00 - 01:35:00]

**Current Position**: "I've hit a spot in between a completely static data structure and a completely dynamic data structure."

**Static Advantages**: Compact CSR/CSC is faster when not changing
**Dynamic Disadvantages**: More pointer dereferencing, worse memory traffic

**Future Direction**: Could add more dynamic formats as new opaque types without API changes

### Format Family Design [00:17:00 - 00:18:30]

The four formats form a coherent family that share code:
- Sometimes sparse/hypersparse grouped vs bitmap/full
- Sometimes all three non-bitmap formats grouped together
- Adding a radically different format (like B-tree based) would be more disruptive

**Key Quote**: "These families of 8, this family of 8 is fortunate, and they all kind of look alike. It would be a much more breaking change to add a new format that's totally different."

### Algorithmic Tolerance Patterns [01:39:00 - 01:42:00]

Every algorithm has an "input guard" that checks for tolerable pending work:
- Zombie tolerant or wait if zombies present
- Tuple tolerant or wait if tuples present
- Jumble tolerant or wait if jumbled
- Different algorithms have different tolerance profiles
- Wait means "finish everything" not just one type of pending work

**Key Quote**: "Many of my algorithms will start off looking at their input matrices and asking, 'Is there pending work in this matrix that I cannot tolerate? If so, do a GrB matrix wait on that input.' So finish the work."

### Code Readability and Grep-ability [01:35:20]
[![01:35:20](https://img.youtube.com/vi/Am0QfqAFHqs/default.jpg)](https://www.youtube.com/watch?v=Am0QfqAFHqs&t=5720s)

Intentional naming choices for searchability:
- "zombie" instead of generic marker
- "jumbled" instead of "unsorted"
- Enables focused code search without false positives

---

## Release Information

The session began with the announcement that GraphBLAS 9.3.0 beta 1 has been released, containing the source code restructuring discussed in Session 1. [00:01:30]
