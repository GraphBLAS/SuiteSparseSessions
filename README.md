# Tim Davis GraphBLAS Interview Series Summary

This repository contains 14 video interview sessions with Dr. Tim Davis (Texas A&M University), author of the SuiteSparse GraphBLAS library. The series provides an in-depth exploration of GraphBLAS internals, covering code architecture, data structures, algorithms, and the JIT compilation system used to achieve high-performance graph processing through sparse linear algebra.

## Series Overview

The interviews span from May 2024 to January 2025, progressively building understanding from foundational concepts to advanced implementation details. Topics include matrix storage formats, the five kernel types, JIT compilation infrastructure, and specific algorithm implementations for operations like transpose, reduce, apply, and element-wise operations.
More sessions are planned in the future.

---

## Table of Contents

### Foundational Sessions

| Video | Session | Date | Topics | Link |
|-------|---------|------|--------|------|
| [![Session 1](https://img.youtube.com/vi/TOwiv1yDcwk/default.jpg)](https://www.youtube.com/watch?v=TOwiv1yDcwk) | **Session 1** | May 28, 2024 | Code organization, 5 kernel types, JIT system intro, data structures overview | [Summary](Session1/README.md) |
| [![Session 2](https://img.youtube.com/vi/Am0QfqAFHqs/default.jpg)](https://www.youtube.com/watch?v=Am0QfqAFHqs) | **Session 2** | June 4, 2024 | Matrix/vector object structure, storage formats (Full/Bitmap/Sparse/Hypersparse), zombies & pending tuples | [Summary](Session2/README.md) |
| [![Session 3](https://img.youtube.com/vi/Pfy5hAsc85Q/default.jpg)](https://www.youtube.com/watch?v=Pfy5hAsc85Q) | **Session 3** | June 11, 2024 | Hyper hash data structure, format conversions, matrix multiply variants overview | [Summary](Session3/README.md) |

### JIT Compilation Deep Dive

| Video | Session | Date | Topics | Link |
|-------|---------|------|--------|------|
| [![Session 4](https://img.youtube.com/vi/4ZHeaThKReA/default.jpg)](https://www.youtube.com/watch?v=4ZHeaThKReA) | **Session 4** | June 17, 2024 | JIT architecture (encodify/jitify/execute), kernel caching, Pre-JIT, CUDA integration | [Summary](Session4/README.md) |
| [![Session 5](https://img.youtube.com/vi/vUKnfYX-o1Y/default.jpg)](https://www.youtube.com/watch?v=vUKnfYX-o1Y) | **Session 5** | June 25, 2024 | JIT control modes, kernel loading, macrofication process, binary operators & monoids | [Summary](Session5/README.md) |
| [![Session 6](https://img.youtube.com/vi/Mb0AspQ2IhE/default.jpg)](https://www.youtube.com/watch?v=Mb0AspQ2IhE) | **Session 6** | August 1, 2024 | JIT synchronization, multi-threading/multi-process, macrofy system, error handling | [Summary](Session6/README.md) |
| [![Session 7](https://img.youtube.com/vi/libeAi866NQ/default.jpg)](https://www.youtube.com/watch?v=libeAi866NQ) | **Session 7** | August 7, 2024 | Kernel validation, macro system deep dive, type casting, compilation methods | [Summary](Session7/README.md) |

### Algorithm Implementations

| Video | Session | Date | Topics | Link |
|-------|---------|------|--------|------|
| [![Session 8](https://img.youtube.com/vi/-taSMF1om0Y/default.jpg)](https://www.youtube.com/watch?v=-taSMF1om0Y) | **Session 8** | August 13, 2024 | GrB_apply operation, parallel execution, factory kernel system, three-tier execution | [Summary](Session8/README.md) |
| [![Session 9](https://img.youtube.com/vi/JG-gh5q1o6o/default.jpg)](https://www.youtube.com/watch?v=JG-gh5q1o6o) | **Session 9** | August 19, 2024 | Reduce operations, monoid vs binary operator debate, panel reduction algorithm | [Summary](Session9/README.md) |
| [![Session 10](https://img.youtube.com/vi/vDMndSZwSN4/default.jpg)](https://www.youtube.com/watch?v=vDMndSZwSN4) | **Session 10** | September 18, 2024 | Transpose algorithms (builder vs bucket), shallow copies, format selection | [Summary](Session10/README.md) |
| [![Session 11](https://img.youtube.com/vi/InpR7kaKKbY/default.jpg)](https://www.youtube.com/watch?v=InpR7kaKKbY) | **Session 11** | October 2, 2024 | Matrix build process, duplicate handling, 5-phase parallel algorithm, hyperhash | [Summary](Session11/README.md) |
| [![Session 12](https://img.youtube.com/vi/g3F8Q8KlzFc/default.jpg)](https://www.youtube.com/watch?v=g3F8Q8KlzFc) | **Session 12** | October 16, 2024 | Transpose/apply integration, 8 transpose methods, iso-valued optimization | [Summary](Session12/README.md) |

### Element-wise Operations

| Video | Session | Date | Topics | Link |
|-------|---------|------|--------|------|
| [![Session 13](https://img.youtube.com/vi/YrpTs_qm9MU/default.jpg)](https://www.youtube.com/watch?v=YrpTs_qm9MU) | **Session 13** | November 8, 2024 | Element-wise add vs union, format-agnostic design, debugging infrastructure, factory vs JIT | [Summary](Session13/README.md) |
| [![Session 14](https://img.youtube.com/vi/uYqRvag62sc/default.jpg)](https://www.youtube.com/watch?v=uYqRvag62sc) | **Session 14** | January 10, 2025 | 32/64-bit integer infrastructure, workspace management (Werk), parallelism strategies, task slicing | [Summary](Session14/README.md) |

---

## Key Themes Across Sessions

### Data Structures
- **Four matrix formats**: Full, Bitmap, Sparse (CSR/CSC), Hypersparse
- **P-H-B-I-X arrays**: Unified storage model across formats
- **Zombies & pending tuples**: Dynamic graph support through deferred operations
- **Hyper hash**: O(1) lookup for hypersparse matrices
- **Iso-valued matrices**: Single-value optimization for structural matrices

### Kernel Architecture
- **Five kernel types**: Factory, JIT, Pre-JIT, Generic, CUDA
- **Three-tier execution**: Factory → JIT → Generic fallback
- **Macrofication**: Template specialization through macro expansion
- **Switch factory pattern**: C-based runtime templating

### JIT System
- **Encodify/Jitify/Execute**: Three-phase kernel generation
- **28-bit encoding**: Compact representation of operator/type combinations
- **Hash table caching**: In-memory and on-disk kernel storage
- **Five control modes**: OFF, PAUSE, RUN, LOAD, ON
- **Multi-process safety**: File locking for shared cache directories

### Parallelization Strategies
- **Coarse vs fine tasks**: Adaptive granularity based on work distribution
- **EK_slice / EY_slice**: Work partitioning for sparse operations
- **P_slice**: Recursive load balancing
- **Atomic vs non-atomic bucket sort**: Trade-offs in transpose algorithms

### Design Philosophy
- **Format-agnostic algorithms**: CSR/CSC unified through descriptor handling
- **Operation fusion**: Transpose + typecast + operator in single pass
- **Shallow copies**: O(1) array sharing for temporary matrices
- **Greppable code style**: Consistent formatting for searchability
- **Debug as documentation**: Reference implementations in debug code

---

## Session Participants

- **Dr. Tim Davis** - Creator of SuiteSparse GraphBLAS, Professor at Texas A&M University
- **Michel Pelletier** - Primary interviewer
- **Piotr Luszczek** - Participating researcher (some sessions)

---

## File Structure

```
TimSessions/
├── README.md           # This file - series overview
├── CLAUDE.md           # Claude Code guidance
├── Session1/
│   ├── README.md       # Session summary with timestamps
│   ├── *.vtt           # WebVTT transcript
│   ├── *.m4a           # Audio recording
│   └── *.mp4           # Video recording
├── Session2/
│   └── ...
└── Session14/
    └── ...
```

---

## Searching the Content

To find discussions of specific topics:

```bash
# Search all transcripts for a keyword
grep -r "hypersparse" Session*//*.vtt

# Search all summaries
grep -r "JIT" Session*/README.md
```

Each session summary includes timestamps in `[HH:MM:SS]` format for jumping to specific sections in the video recordings.
