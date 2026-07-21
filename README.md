# GPU-Accelerated Boolean Operations on IC Layout Geometry

CUDA/C++ engine that performs boolean operations on integrated-circuit layout
polygons — the geometric primitive that Design Rule Checking (DRC) is built on.
Graduation project, Computer & Systems Engineering, Ain Shams University.

**Scope note:** this is a boolean-operations engine, not a complete DRC checker.
It outputs geometry (`.oas`), not rule violations. Rule-level operations such as
minimum-area and spacing checks were developed by teammates on separate branches
and are not yet wired into the dispatcher.

---

## Results

One UNION operation, measured at six input sizes. Baseline is [gdstk](https://github.com/heitzmann/gdstk),
a widely used CPU layout library.


|
 Polygons/layer 
|
 GPU     
|
 gdstk (CPU) 
|
 Ratio             
|
|
---------------:
|
--------:
|
------------:
|
-------------------
|
|
 100            
|
 1.295 s 
|
 —           
|
 —                 
|
|
 2,500          
|
 1.505 s 
|
 —           
|
 —                 
|
|
 10,000         
|
 1.461 s 
|
 —           
|
 —                 
|
|
 40,000         
|
 1.552 s 
|
 0.593 s     
|
**
gdstk 2.6× faster
**
|
|
 250,000        
|
 2.330 s 
|
 11.074 s    
|
**
GPU 4.75×
**
|
|
 490,000        
|
 4.153 s 
|
 38.188 s    
|
**
GPU 9.2×
**
|

Two honest caveats:

- **The GPU loses below the crossover.** Note the flat ~1.5 s from 100 to 40,000
  polygons — that is fixed overhead (GPU init, transfers), not computation. Real
  kernel time only becomes visible above it. The crossover point lies somewhere
  between 40K and 250K polygons; it was not measured precisely.
- **This is one operation, not a rule suite.** A single boolean UNION, repeated at
  six input sizes.

Docker adds roughly 0.8–1.4 s of container startup on top of GPU init (2.1–2.7 s
via the GUI vs. ~1.3 s running the binary directly on identical small inputs).

**Test environment:** GTX 1660 Ti (Turing, SM 7.5, 24 SMs, 6 GB) · i7-10750H
@ 2.60 GHz (6C/12T) · CUDA 13.2 · WSL2, Ubuntu 24.

---

## Pipeline

```
.oas / .gds
   │
   ├─ gdstk read → flatten hierarchy → normalize layers
   ├─ ProofRead: parse operations file into an expression tree
   │              (operator precedence, parentheses)
   │
   ├─ quadtree partition into leaves        ← coarse spatial index
   │
   └─ per leaf:
        7 GPU preprocessing stages          ← rule-agnostic
        boolean filter                      ← the only rule-dependent step
        face reconstruction
   │
   └─ merge leaves → gdstk write → result.oas
```

### Two-level spatial indexing

The two levels serve different purposes and are easy to confuse:

- **Quadtree (coarse)** — splits the layout into GPU-sized batches, so layouts
  larger than the 6 GB of device memory can still be processed.
- **500×500 uniform grid (fine)** — generates candidate polygon pairs *within* a
  single batch.

### The 7 preprocessing stages

Identical regardless of which boolean operation was requested:

1. Build a 500×500 grid over the bounding box
2. Map polygons to cells, emit candidate pairs
3. Deduplicate pairs, build the candidate mapping
4. Convert polygons to edge segments
5. Compute segment–segment intersections between candidates
6. Split segments at intersection points
7. Compute inside/outside fill flags per segment

The requested operation enters at exactly one narrow point: the **boolean filter**,
which keeps or drops each segment based on its fill flags. Everything upstream is
operation-agnostic.

---

## Correctness

- **Differential testing against gdstk** (`compare.py`) — outputs identical.
- **CPU reference mode** (`--test`) — 300 segments in, 200 filtered, all 200 final
  segments matched the GPU path.
- **Unit tests** — I wrote 30 tests covering BinaryTree and the ProofRead
  expression parser; the full suite has 2 known pre-existing failures.

---

## My contributions

This was a team project. The core CUDA kernels live in a private university
repository. My work covered:

- **Build system** — Makefile, and a Dockerized build for reproducible runs
- **Test suite** — 30 unit tests (BinaryTree, ProofRead expression parser)
- **Output validation** — differential testing harness against gdstk
- **Performance benchmarking** — the measurements above, including baseline
  methodology and identification of the fixed-overhead floor
- **GUI** — Python/Tkinter front end over the engine


---

## Stack

C++ · CUDA · CUB · Python · Tkinter · Docker · gdstk
