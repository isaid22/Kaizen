+++
title = 'Tiled Matrix Multiplication in CUDA'
date = 2026-09-21T00:00:00-05:00
draft = false
tags = ['cuda', 'gpu']
summary = 'How tiling reduces global memory bandwidth in CUDA matrix multiplication by loading sub-blocks into shared memory.'
katex = true
+++

# Tiled Matrix Multiplication in CUDA

Tiling is an optimization technique used in CUDA kernel development to reduce global memory bandwidth bottlenecks. By loading sub-blocks (tiles) of matrices into high-speed **shared memory**, threads within a block can reuse data multiple times rather than repeatedly querying slow global GPU memory.

## Complete CUDA Source Code

```cpp
template <int TILE_WIDTH>
__global__ void matmul_tiled(float* C, const float* A, const float* B, int M, int N, int K) {
    // Allocate tile buffers in shared memory
    __shared__ float As[TILE_WIDTH][TILE_WIDTH];
    __shared__ float Bs[TILE_WIDTH][TILE_WIDTH];

    // Global row and column indices for output matrix C
    int row = blockIdx.y * TILE_WIDTH + threadIdx.y;
    int col = blockIdx.x * TILE_WIDTH + threadIdx.x;

    float sum = 0.0f;

    // Iterate across matrix dimensions in tiled phases
    for (int phase = 0; phase < (K + TILE_WIDTH - 1) / TILE_WIDTH; ++phase) {
        int a_col = phase * TILE_WIDTH + threadIdx.x;
        int b_row = phase * TILE_WIDTH + threadIdx.y;

        // Collaborative loading of tile A into shared memory
        if (row < M && a_col < K) {
            As[threadIdx.y][threadIdx.x] = A[row * K + a_col];
        } else {
            As[threadIdx.y][threadIdx.x] = 0.0f; // Padding out-of-bounds elements
        }

        // Collaborative loading of tile B into shared memory
        if (b_row < K && col < N) {
            Bs[threadIdx.y][threadIdx.x] = B[b_row * N + col];
        } else {
            Bs[threadIdx.y][threadIdx.x] = 0.0f; // Padding out-of-bounds elements
        }

        // Synchronize to ensure the full tile is loaded
        __syncthreads();

        // Compute partial dot product from shared memory
        for (int k = 0; k < TILE_WIDTH; ++k) {
            sum += As[threadIdx.y][k] * Bs[k][threadIdx.x];
        }

        // Synchronize before loading the next phase's tiles
        __syncthreads();
    }

    // Write final accumulated sum to global memory C
    if (row < M && col < N) {
        C[row * N + col] = sum;
    }
}
```

## Concrete Example Setup

To make the calculations easy to follow, let us consider a concrete example:

* Matrix dimensions: $M = 1024$, $N = 1024$, $K = 1024$
* Tile width: `TILE_WIDTH = 32` (giving a tile size of $32 \times 32$)

With this configuration:

* **Thread Blocks:** Each thread block consists of $32 \times 32 = 1024$ threads.
* **Grid Size:** The grid requires $\frac{1024}{32} \times \frac{1024}{32} = 32 \times 32 = 1024$ blocks in total.
* **Number of Phases:** Expanding across dimension $K$ requires $\frac{1024}{32} = 32$ phases per block.

## Formal Proof: Reduction in Memory Access Complexity

To prove why global memory access complexity drops from $\mathcal{O}(N^3)$ to $\mathcal{O}\left(\frac{N^3}{T}\right)$, assume square matrices where $M = N = K$ and tile width `TILE_WIDTH = T`.

### 1. Naïve Matrix Multiplication (No Tiling)

For every element $C[i][j]$ in the output matrix, a thread must perform a dot product of row $i$ from matrix $A$ and column $j$ from matrix $B$.

* **Output elements in $C$:** $N \times N = N^2$
* **Reads required per output element:** $N$ elements from $A$ + $N$ elements from $B$ = $2N$ global memory reads.
* **Total Global Memory Accesses ($A_{\text{naive}}$):**

  $$A_{\text{naive}} = N^2 \times 2N = 2N^3$$

Thus, the global memory complexity for naive matrix multiplication is $\mathcal{O}(N^3)$.

### 2. Tiled Matrix Multiplication

In tiled execution, global memory reads occur at the thread-block level rather than individual thread level.

* **Total Number of Blocks in Grid:** $\left(\frac{N}{T}\right) \times \left(\frac{N}{T}\right) = \frac{N^2}{T^2}$
* **Number of Phases per Block:** $\frac{N}{T}$
* **Global Memory Reads per Phase per Block:**
  One tile from matrix $A$ ($T \times T$ elements) + One tile from matrix $B$ ($T \times T$ elements) = $2T^2$ reads.
* **Global Memory Reads per Block across all phases ($A_{\text{block}}$):**

  $$A_{\text{block}} = \text{Phases} \times 2T^2 = \left(\frac{N}{T}\right) \times 2T^2 = 2NT$$

* **Total Global Memory Accesses across all grid blocks ($A_{\text{tiled}}$):**

  $$A_{\text{tiled}} = \text{Total Blocks} \times A_{\text{block}} = \left(\frac{N^2}{T^2}\right) \times (2NT) = \frac{2N^3}{T}$$

### Conclusion & Speedup Factor

Comparing the total memory accesses:

$$\text{Speedup Factor} = \frac{A_{\text{naive}}}{A_{\text{tiled}}} = \frac{2N^3}{\frac{2N^3}{T}} = T = \text{TILE\_WIDTH}$$

Thus, tiling mathematically reduces global memory reads from $\mathcal{O}(N^3)$ down to $\mathcal{O}\left(\frac{N^3}{T}\right)$, where $T$ is the tile width (`TILE_WIDTH`).

### Numerical Walkthrough ($N = 1024, T = 32$)

* **Naïve Reads:**

  $$2 \times (1024)^3 = 2,147,483,648 \text{ reads } (\sim 2.15 \text{ billion})$$

* **Tiled Reads:**

  $$\frac{2 \times (1024)^3}{32} = 67,108,864 \text{ reads } (\sim 67.1 \text{ million})$$

* **Reduction:**

  $$\frac{2,147,483,648}{67,108,864} = 32\times \text{ memory bandwidth reduction}$$

## Detailed Step-by-Step Execution

### Allocating Shared Memory Tiles

Rather than loading elements directly from global memory during dot product calculations, we allocate small sub-matrices in on-chip shared memory:

```cpp
__shared__ float As[32][32];
__shared__ float Bs[32][32];
```

### Global Mapping of Result Matrix $C$

Each thread maps its 2D block and thread indices to absolute global matrix coordinates:

```cpp
int row = blockIdx.y * 32 + threadIdx.y;
int col = blockIdx.x * 32 + threadIdx.x;
```

### Iterating Through Matrix Phases

Because shared memory tiles only hold 32 columns/rows of $K$ at a time, execution is split into 32 phases using ceiling division: `(1024 + 32 - 1) / 32 = 32`.

* Tile `As` shifts **right** across matrix $A$'s columns.
* Tile `Bs` shifts **down** across matrix $B$'s rows.

### Indexing and Collaborative Shared Memory Loading

In each phase, each of the $1024$ threads in a block copies **one element** of $A$ into `As` and **one element** of $B$ into `Bs`:

```cpp
int a_col = phase * 32 + threadIdx.x;
int b_row = phase * 32 + threadIdx.y;

if (row < 1024 && a_col < 1024) As[threadIdx.y][threadIdx.x] = A[row * 1024 + a_col];
else As[threadIdx.y][threadIdx.x] = 0.0f;

if (b_row < 1024 && col < 1024) Bs[threadIdx.y][threadIdx.x] = B[b_row * 1024 + col];
else Bs[threadIdx.y][threadIdx.x] = 0.0f;
```

### Synchronization & Partial Dot-Product Computation

Because threads execute concurrently, the synchronization barrier `__syncthreads()` prevents threads from reading shared memory until all 1,024 elements of `As` and `Bs` are populated:

```cpp
__syncthreads(); // Wait for all 1,024 threads to finish tile loading

for (int k = 0; k < 32; ++k) {
    sum += As[threadIdx.y][k] * Bs[k][threadIdx.x];
}

__syncthreads(); // Wait for computation to complete before loading next phase
```

After all 32 phases complete, each thread writes its accumulated sum back to matrix $C$ in global memory.

## Further Optimization Concepts

### Shared Memory Bank Conflicts

* **Problem:** Shared memory is divided into 32 physical memory banks that process requests concurrently. When multiple threads in a warp access different addresses residing in the *same* memory bank simultaneously, accesses are serialized rather than parallelized.
* **High-Level Solution:** Padding shared memory array dimensions (e.g., declaring `__shared__ float As[32][33]`).
* **Benefit:** Shifts array row offsets so consecutive threads in a warp access different memory banks, eliminating serialization bottlenecks and maintaining peak bandwidth.

### Leveraging Tensor Cores

* **Problem:** Standard CUDA cores perform single multiply-accumulate operations ($a \times b + c$) per instruction cycle, which limits peak FLOPS during high-throughput AI and linear algebra workloads.
* **High-Level Solution:** Utilize hardware Tensor Cores via NVIDIA's **WMMA (Warp Matrix Multiply and Accumulate)** API or libraries like **CUTLASS** and **cuBLAS**. These perform specialized matrix operations ($D = A \times B + C$) directly on fixed-size sub-matrices at the hardware warp level.
* **Benefit:** Delivers massive hardware acceleration (often 4x to 16x higher throughput) for mixed-precision (FP16, BF16, INT8) matrix multiplications.