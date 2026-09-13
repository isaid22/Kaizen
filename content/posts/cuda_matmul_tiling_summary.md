+++
title = 'CUDA Matrix Multiplication: Thread Indexing & Tiling'
date = 2026-09-13T00:00:00-05:00
draft = false
tags = ['cuda', 'gpu']
summary = 'How CUDA thread indexing works and why shared-memory tiling speeds up matrix multiplication.'
+++

## 1. How `row` and `col` start at 0

In the naive kernel:

```cuda
int row = blockIdx.y * blockDim.y + threadIdx.y;
int col = blockIdx.x * blockDim.x + threadIdx.x;
```

The starting thread receives:

```text
blockIdx.x = 0, blockIdx.y = 0
threadIdx.x = 0, threadIdx.y = 0
```

So:

```text
row = 0 * blockDim.y + 0 = 0
col = 0 * blockDim.x + 0 = 0
```

CUDA automatically assigns these indices starting from zero. The kernel launch creates a grid of threads, and each thread computes its own unique `(row, col)` coordinate based on those indices.

### Key CUDA index variables

| Variable | Meaning |
|----------|---------|
| `blockIdx.x`, `blockIdx.y` | Block position in the grid (starts at 0) |
| `blockDim.x`, `blockDim.y` | Threads per block |
| `threadIdx.x`, `threadIdx.y` | Thread position inside its block (starts at 0) |

## 2. Why tiling speeds up matrix multiplication

### The naive kernel's bottleneck

Each thread in the naive kernel computes one element of `C`:

```cuda
for (int k = 0; k < K; ++k) {
    sum += A[row * K + k] * B[k * N + col];
}
```

This causes a lot of redundant global memory access:

- The same row of `A` is read by every thread computing that row of `C`.
- The same column of `B` is read by every thread computing that column of `C`.

Global memory is slow, so repeatedly fetching the same data becomes the main bottleneck.

### What tiling does differently

Tiling uses **shared memory**, which is much faster and located on the SM (Streaming Multiprocessor):

```cuda
__shared__ float As[TILE_WIDTH][TILE_WIDTH];
__shared__ float Bs[TILE_WIDTH][TILE_WIDTH];
```

The kernel works in phases:

1. Load a tile of `A` and a tile of `B` from global memory into shared memory **once**.
2. Compute partial dot products from the shared memory tiles.
3. Move to the next tile and repeat.

### Why it's faster

| Aspect | Naive Kernel | Tiled Kernel |
|--------|--------------|--------------|
| Reads each element of `A`/`B` | Many times from slow global memory | Once per phase into fast shared memory |
| Data reuse | Re-reads from global memory every time | Reuses data already in shared memory |
| Memory bottleneck | High global memory traffic | Lower global memory traffic |

The speedup comes from **data reuse**: a `32x32` tile loaded once can be used by 32 threads in the block before being discarded. This reduces global memory bandwidth pressure, which is usually the limiting factor in matrix multiplication performance.

## 3. Simple mental model

- **Naive**: every thread fetches its own row and column independently, causing massive redundant global memory reads.
- **Tiled**: a block of threads cooperatively loads a small square of data into fast shared memory, then reuses that data to compute many partial results.

Tiling trades a small amount of extra synchronization (`__syncthreads`) for much better memory efficiency, which is why it outperforms the naive approach.
