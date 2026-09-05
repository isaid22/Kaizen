+++
title = 'CUDA Memory Hierarchy: A Practical Guide'
date = 2026-09-05T09:00:00-05:00
draft = false
tags = ['cuda', 'gpu']
summary = 'A practical guide to how CUDA memory hierarchy works.'
+++

# CUDA Memory Hierarchy: A Practical Guide

## `cudaMalloc` Allocates Only One Kind of Memory

`cudaMalloc` **always allocates device global memory** (also called **device memory** or **off-chip GDDR/HBM memory**). It does **not** let you choose which level of the memory hierarchy to use.

```cpp
float *d_A, *d_B, *d_C;
cudaMalloc((void**)&d_A, size_A);  // global memory
cudaMalloc((void**)&d_B, size_B);  // global memory
cudaMalloc((void**)&d_C, size_C);  // global memory
```

> Note: `cudaMalloc` expects `void**`, so `(void**)&d_A` is the explicit-cast form.
> `cudaMalloc(&d_A, size)` may compile with a warning because `float**` does not implicitly convert to `void**`.

To use faster memory levels (shared, registers, constant), you do **not** use `cudaMalloc`. Instead, you move data explicitly inside the kernel using qualifiers like `__shared__` or by reading into local variables that the compiler places in registers.

---

## Mapping the CUDA Memory Hierarchy

| API / qualifier | Memory level | Lifetime | Typical use |
|-----------------|--------------|----------|-------------|
| `cudaMalloc` | Global / device memory | Until `cudaFree` | Large arrays, inputs/outputs |
| `cudaMallocHost` / `cudaHostAlloc` | Pinned host memory | Until `cudaFreeHost` | Faster H→D/D→H transfers, zero-copy |
| `cudaMallocManaged` | Unified / managed memory | Until `cudaFree` | Simplified programming, oversubscription |
| `cudaMallocPitch` / `cudaMalloc3D` | Global memory with padding | Until `cudaFree` | 2D/3D arrays for coalescing |
| `__shared__` inside kernel | Shared memory | Block lifetime | Tile/block data reuse |
| `__constant__` | Constant memory | Program lifetime | Read-only, broadcast data |
| Local variables / registers in kernel | Registers / spill to local memory | Thread lifetime | Per-thread scalars, accumulators |
| `__device__` qualifier | Global memory | Program lifetime | Static global device arrays |
| `__managed__` / `__device__ __managed__` | Unified memory | Until freed | Static unified variables |

### Memory bandwidth order (fastest → slowest)

1. **Registers** — fastest, private to each thread, very limited capacity.
2. **Shared memory** — fast on-chip memory, shared by all threads in a block, used for intra-block reuse.
3. **Constant / Texture / Read-only cache** — cached, broadcast-friendly, read-only.
4. **Global memory (L2 cached)** — large, off-chip, slow unless accesses are coalesced and cached.
5. **Host memory** — accessed over PCIe/NVLink, much slower than device memory.

---

## Applying This to Your Matrix-Multiplication Code

In `matrix_mult.cu`:

- `d_A`, `d_B`, `d_C` live in **global memory** because they are allocated with `cudaMalloc`.
- In the **naive kernel**, every thread reads from global memory `K` times per output element. There is no reuse, so the kernel is **global-memory bound** and slow.
- In the **tiled kernel**, each block loads a tile of `A` and `B` into `__shared__` memory once, then reuses those tiles across the inner loop. This is the standard way to move hot data up the hierarchy from **global → shared**.
- The accumulator `sum` is a local variable, so it is kept in a **register**.
- The final result is written back to global memory exactly once per output element.

```cpp
__global__ void matmul_tiled(float* C, const float* A, const float* B, ...) {
    __shared__ float As[TILE_WIDTH][TILE_WIDTH];  // shared memory
    __shared__ float Bs[TILE_WIDTH][TILE_WIDTH];  // shared memory

    float sum = 0.0f;  // register

    for (int phase = 0; phase < ...; ++phase) {
        As[...] = A[...];   // global → shared
        Bs[...] = B[...];   // global → shared
        __syncthreads();

        for (int k = 0; k < TILE_WIDTH; ++k) {
            sum += As[...] * Bs[...];  // shared → register
        }
        __syncthreads();
    }

    C[...] = sum;  // register → global
}
```

---

## Is the Code Optimized for the Memory Hierarchy?

**Short answer:** partially optimized, but not fully.

### What is good

- The tiled kernel uses **shared memory** to reduce redundant global reads.
- The accumulator is kept in a **register**.
- The output is written to global memory only once.
- cuBLAS is included as a reference for a fully optimized implementation.

### What could still be improved

| Issue | Why it matters |
|-------|---------------|
| No `__restrict__` on pointer arguments | The compiler cannot safely cache loads through the read-only data cache (`LDG`) because it must assume `A`, `B`, and `C` may alias. |
| No vectorized global loads (`float4`) | Global memory is accessed one `float` at a time, wasting memory-transaction bandwidth. |
| One output per thread, no register blocking | Each thread computes only one dot product. Computing a small tile per thread (e.g. `4x4`) keeps more partial results in registers and exposes more parallelism. |
| No padding on shared-memory arrays | Current tile sizes (8, 16, 32) happen to avoid bank conflicts, but other sizes would not. Adding `+1` padding makes the kernel robust. |
| No use of Tensor Cores / warp-level primitives | Modern GPUs get far higher throughput from mixed-precision tensor cores and warp shuffles. cuBLAS uses these; the hand-written kernels do not. |

---

## Summary

- `cudaMalloc` always gives you **global memory**.
- To get better performance, move hot data up the hierarchy explicitly:
  - Global → **shared** (tile/block reuse)
  - Shared → **registers** (per-thread accumulators)
  - Use **cuBLAS** when you need production performance.
