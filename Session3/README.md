# Session 3 Summary

## Table of Contents
- [Overview](#overview)
- [Data Structures and Matrix Formats](#data-structures-and-matrix-formats)
  - [Matrix Format Variants](#matrix-format-variants-000008---000900)
- [Hyper Hash Data Structure](#hyper-hash-data-structure)
  - [Hyper Hash Concept and Motivation](#hyper-hash-concept-and-motivation-001206---002606)
  - [Hyper Hash Implementation](#hyper-hash-implementation-002606---003520)
- [Hyper Hash Heuristics and Management](#hyper-hash-heuristics-and-management)
  - [When to Build Hyper Hash](#when-to-build-hyper-hash-003900---004900)
  - [Hyper Hash and User API](#hyper-hash-and-user-api-005500---010000)
- [Matrix Format Conversions](#matrix-format-conversions)
  - [Conform and Convert Mechanisms](#conform-and-convert-mechanisms-010200---010900)
  - [Hysteresis and Format Stability](#hysteresis-and-format-stability-010441---010509)
- [Implications for Dense Operations](#implications-for-dense-operations)
  - [Dense Vector Results](#dense-vector-results-010509---011100)
- [Matrix Multiply Variants](#matrix-multiply-variants)
  - [Saxpy Methods Overview](#saxpy-methods-overview-011200---011305)
- [Next Steps and JIT Discussion](#next-steps-and-jit-discussion)
  - [Transition to JIT Focus](#transition-to-jit-focus-011305---012000)
- [Session Conclusion](#session-conclusion)

---

## Overview
This session focused on SuiteSparse GraphBLAS data structures, specifically exploring the hyper hash implementation and data format conversions. The discussion covered algorithmic variants, performance optimizations, and the design philosophy behind GraphBLAS's multiple data formats.

---

## Data Structures and Matrix Formats

### Matrix Format Variants [00:00:08 - 00:09:00]
[![00:00:08](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=8s)

**Discussion:** Dr. Davis explained the core data structures in SuiteSparse GraphBLAS, emphasizing the complexity introduced by multiple matrix formats and parallelism strategies.

**Key Points:**
- Four fundamental matrix formats: full, bitmap, sparse, and hyper-sparse
- Four parallelism variants for mxm operations (in saxpy-3 method): coarse Gustavson, fine Gustavson, coarse hash, fine hash
- The Saxpy-3 method implements Gustavson's algorithm modified for sparse matrix multiplication
- Different variants handle workspace management differently: coarse grain gives each thread its own workspace, fine grain uses atomics for shared access

**Key Quote:**
> "How many matrix multiplies do I have? I don't even know how to count that high. Do you count these as different algorithms? They're kind of different algorithms. Do you have the mask present? Is the mask complemented or not? Yeah, it's a totally different algorithm now."
[00:07:38]

**Technical Details:**
- Template-based factory system handles variations at compile time without JIT for data-type-independent operations
- Extensive debug scaffolding code validates parallel task execution
- Each variant combination produces different factory methods (e.g., saxpy3_sim files)

---

## Hyper Hash Data Structure

### Hyper Hash Concept and Motivation [00:12:06 - 00:26:06]
[![00:12:06](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=726s)

**Discussion:** Introduction to the hyper hash data structure, which accelerates lookups in hyper-sparse matrices by inverting the hyper list.

**Key Points:**
- Hyper list maps K (position) → J (vector name): `J = A.h[K]`
- Hyper hash inverts this: given J, find K (or -1 if not present)
- Traditional approach used binary search across the hyper list
- Binary search is expensive on CUDA and for large hyper lists

**Key Quote:**
> "I came up with this idea of well, gee, I could build a hash table, but you see, it's a static hash table. There's no inserts or deletions into this hash table because the hashing is just across the vectors that are present in the matrix."
[00:17:39]

**Design Decision:**
- Hyper hash is a sparse CSC matrix where:
  - Rows represent J values (vector dimension)
  - Columns represent hash buckets (64 in the example)
  - Values are pairs of (J, K)
- Load factor kept at ≤25% to minimize collisions
- Hash table size always power of 2 for efficient modulo operations

### Hyper Hash Implementation [00:26:06 - 00:35:20]
[![00:26:06](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=1566s)

**Discussion:** Technical implementation details of the hyper hash structure as a GrB_Matrix object.

**Key Points:**
- Hyper hash stored as a GrB_Matrix Y inside the matrix structure
- Y is always sparse (never hyper-sparse itself)
- Hash collisions handled by searching short bucket lists
- If bucket has >256 entries, falls back to binary search within bucket

**Key Quote:**
> "I have a very, very fast, very stupid hash function, frankly, because it's just supposed to be super fast. Yeah, there could be collisions. I make the hash table size a power of 2 so I can do bit-and operations to mask out the modular thing to know which hash bucket I'm in."
[00:24:16]

**Lookup Algorithm:**
1. Compute hash function: `F(J)` to get bucket number
2. Access column `F(J)` of hyper hash matrix Y
3. Search bucket for J (linear or binary search depending on size)
4. Return K value from (J,K) pair if found, else -1

**Code Location:** [00:27:20 - 00:32:00]
- Build: `builder/GB_hyper_hash_build.c`
- Lookup: `hyper/GB_hyper_hash_lookup` (static inline function)
- Free: `matrix/` folder

---

## Hyper Hash Heuristics and Management

### When to Build Hyper Hash [00:39:00 - 00:49:00]
[![00:39:00](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=2340s)

**Discussion:** Decision logic for when to construct a hyper hash versus using simple binary search.

**Key Points:**
- Default threshold: 1024 non-empty vectors (configurable via `GrB_get/set`)
- Hyper hash only built when needed by specific algorithms
- Not built during matrix construction, only when operations require it
- If matrix not hyper-sparse, no hyper hash needed

**Algorithms that use hyper hash:** [00:49:00 - 00:54:00]
- Assign operations (zombie assignments)
- Some add operations
- Element-wise multiply (emult algorithm 8)
- Submatrix operations
- Saxpy-3 method (for M and A matrices, not B)
- Dot product methods
- Extract operations

**Key Quote:**
> "The hyper hash build might decide, yes, that's a hyper-sparse matrix. You could in theory build a hyper hash for it. But why bother? You've only got 10 vectors in there. Just do the binary search across the 10, and you're fine."
[00:48:05]

**Performance Trade-offs:**
- Build time versus lookup time
- For small vectors (<1024), binary search sufficiently fast
- Hysteresis prevents thrashing when vector count oscillates near threshold

### Hyper Hash and User API [00:55:00 - 01:00:00]
[![00:55:00](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=3300s)

**Discussion:** How hyper hash interacts with pack/unpack functionality and API design.

**NOTE:** the pack/unpack methods were the primary methods for moving data in and out of
a GrB_Matrix in O(1) time and space in GraphBLAS v9.  Since this video was taken, I have
since added a more flexible set of methods: load/unload into/from a GxB_Container.
For that method, the hyper-hash is loaded/unloaded as its own GrB_Matrix.

**Key Points:**
- Pack/unpack provides non-opaque access to matrix internals
- Separate `unpack_hyper_hash` function added to preserve hyper hash across pack/unpack
- Original design would delete hyper hash on unpack, requiring expensive rebuild
- Decision made to add separate function rather than break API compatibility

**Key Quote:**
> "I added a unpack hyper hash. If you're really desperate to preserve my hyper-sparse matrix object, you can unpack as a hyper-sparse matrix and also unpack its hyper hash, keep it, fiddle with it, and then put it back nicely, and then I've got back to where we started."
[00:56:02]

**API Design Consideration:**
- Could have broken backward compatibility with extended unpack parameters
- Instead chose two separate functions for cleaner API
- Pack/unpack somewhat experimental, difficult to standardize in spec


---

## Matrix Format Conversions

### Conform and Convert Mechanisms [01:02:00 - 01:09:00]
[![01:02:00](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=3720s)

**Discussion:** How GraphBLAS automatically selects and converts between matrix formats.

**Key Points:**
- "Conform" = heuristic decision about which format to use
- Matrix format chosen dynamically after computation completes
- 16 possible format combinations (4 formats, each can be enabled/disabled)
- Default: all formats enabled, automatic selection

**Conversion Examples:**
- Sparse → Full: Just free integer arrays if all entries present
- Full → Sparse: Add integer arrays, data stays in place
- Full → Bitmap: Create bitmap of all true values, no data movement
- Sparse → Bitmap: Requires moving data (needs templating for different types)

**Key Quote:**
> "GraphBLAS must always respect the structural computation of its result. You always have to say it's not there or it's there, and you get the right answer."
[01:03:36]

### Hysteresis and Format Stability [01:04:41 - 01:05:09]
[![01:04:41](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=3881s)

**Discussion:** How GraphBLAS prevents format thrashing.

**Key Points:**
- Hysteresis region prevents constant conversion at format boundaries
- If matrix near boundary (e.g., sparse vs hyper-sparse), keep current format
- Only convert when clearly in one regime or another
- User can constrain formats via `GrB_set` (though "full only" becomes "bitmap or full")

---

## Implications for Dense Operations

### Dense Vector Results [01:05:09 - 01:11:00]
[![01:05:09](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=3909s)

**Discussion:** Why GraphBLAS sometimes produces sparse results even with dense inputs.

**Key Points:**
- `Y = A*X` where A is sparse, X is dense does NOT guarantee dense Y
- If A has empty rows, Y must have missing entries (cannot fake with zeros)
- GraphBLAS builds output in sparse/bitmap form, then checks if full during conform
- Converting sparse to full only requires freeing integer arrays (no data movement)

**Performance Impact:**
- Extra overhead computing indices that may be discarded
- Necessary for semantic correctness
- Optimized path: use accumulator to ensure no deletions

**Key Quote:**
> "What if it happens that you have no empty rows? Well, I have to compute that fact. I'll produce an output Y, and then I'm like, 'Oh look, vector Y, you're full.' I convert it to full by just freeing its integers."
[01:05:56]

**Workaround for Dense Output:**
- Use accumulator: `Y += A*X` where Y starts dense
- With accumulator, no deletions possible
- Allows use of specialized dense kernels (saxpy4, saxv5)
- Faster execution, direct writes to output

---

## Matrix Multiply Variants

### Saxpy Methods Overview [01:12:00 - 01:13:05]
[![01:12:00](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=4320s)

**Discussion:** Different matrix multiply algorithms in GraphBLAS.

**Key Points:**
- Two major families: dot-product style and Saxpy style
- Saxpy-3: Main sparse × sparse algorithm, uses Gustavson method
- Saxpy-4: `C += A*B` where C is bitmap/full
- Saxpy-5: `C += A*B` where A is bitmap/full, B is sparse
- Saxpy-1 and Saxpy-2 deleted (old algorithms)
- Dot-1 deleted, Dot-2 and Dot-3 remain
- XP Dot: Additional dot product methods

**Complexity:**
- Saxpy-3 is "one of the more complex matrix multiply methods"
- Handles all combinations of sparse/hyper-sparse inputs
- Different parallel strategies mixed in single operation

---

## Next Steps and JIT Discussion

### Transition to JIT Focus [01:13:05 - 01:20:00]
[![01:13:05](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q&t=4385s)

**Discussion:** Planning next session's topics and identifying JIT as critical foundation.

**Key Points:**
- Matrix multiply too complex without understanding JIT first
- JIT system shared between CPU (OpenMP) and CUDA
- 63-bit enumeration space for mxm variants (8 bits just for matrix formats)
- Fast enumeration critical for hash table lookup of compiled kernels
- Template system uses pound-define macros and pound-include

**JIT Components:**
- Enumify: Fast encoding of kernel variants
- Codify: Generate source code
- Macrify: Build macro definitions
- Almost identical between CUDA and CPU JIT
- Functions all have "fy" suffix in jitifer folder

**Compilation Flow:**
1. Enumerate kernel signature (very fast)
2. Check hash table for existing kernel
3. If not found: generate source, compile, load, cache
4. If found: retrieve function pointer (fast path)

---

## Session Conclusion

**Topics Covered:**
- Hyper hash data structure and implementation
- Format conversion mechanisms
- Pack/unpack API considerations
- Performance heuristics and trade-offs
- Matrix multiply algorithm variants

**Next Session Plan:** [01:19:25]
- Deep dive into JIT system
- Factory kernels vs JIT kernels
- Macro system and template mechanisms
- Enumeration and code generation
- CUDA JIT integration

