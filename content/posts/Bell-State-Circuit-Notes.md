+++
title = 'Bell-State Circuit: A Qubit Journey into Entanglement'
date = 2026-10-19T00:00:00-05:00
draft = false
tags = ['quantum-computing', 'bell-state', 'entanglement']
summary = 'Follow a two-qubit circuit through initialization, a Hadamard gate, and CNOT to understand how a separable state becomes an entangled Bell state.'
katex = true
+++

# Bell-State Circuit: Derivation and Refined Notes

This summary explains how a Bell-state circuit creates quantum entanglement. 

## 0. Approach

We will follow the two qubits through time—their “qubit journey”—and examine how the circuit’s quantum state evolves at three stages: $t_0$, before any gates are applied; $t_1$, after the Hadamard gate; and $t_2$, after CNOT. This step-by-step analysis shows how an initially separable state becomes an entangled Bell state.

Then I need to use this in markdown

![Bell-state circuit with dotted lines marking t0, t1, and t2](/images/Bell-State-Circuit.svg)


## 1. Circuit and conventions

The circuit starts with two qubits in $|0\rangle$, applies a Hadamard gate to $q_0$, and then applies CNOT with $q_0$ as control and $q_1$ as target.

| Time | Operation completed |
|---|---|
| $t_0$ | Initialize both qubits to $|0\rangle$ |
| $t_1$ | Apply $H$ to $q_0$ |
| $t_2$ | Apply $\mathrm{CNOT}_{q_0\to q_1}$ |

Throughout these notes, the ket ordering is **$|q_0q_1\rangle$**, and the ordered computational basis is

$$
\bigl(|00\rangle,\ |01\rangle,\ |10\rangle,\ |11\rangle\bigr).
$$

Thus, the entries in a state vector are the amplitudes of these four basis states, in that order:

$$
|\psi\rangle=
\begin{pmatrix}
a_{00}\\a_{01}\\a_{10}\\a_{11}
\end{pmatrix}
=a_{00}|00\rangle+a_{01}|01\rangle+a_{10}|10\rangle+a_{11}|11\rangle.
$$

For a normalized state, $\sum_{i,j}|a_{ij}|^2=1$. Amplitudes can be complex; probabilities are their squared magnitudes.

> **Ordering refinement:** Software and textbooks may use different qubit ordering conventions. State the ordering before interpreting vector entries or gate matrices.

## 2. At $t_0$: initial state

Each qubit starts in

$$
|0\rangle=\begin{pmatrix}1\\0\end{pmatrix}.
$$

The two-qubit state is the tensor product:

$$
|\psi_0\rangle
=|0\rangle_{q_0}\otimes|0\rangle_{q_1}
=|00\rangle
=\begin{pmatrix}1\\0\\0\\0\end{pmatrix}.
$$

For column vectors, the tensor product expands as

$$
\begin{pmatrix}a\\b\end{pmatrix}
\otimes
\begin{pmatrix}c\\d\end{pmatrix}
=\begin{pmatrix}ac\\ad\\bc\\bd\end{pmatrix}.
$$

## 3. At $t_1$: Hadamard on $q_0$

The Hadamard matrix is

$$
H=\frac{1}{\sqrt2}
\begin{pmatrix}1&1\\1&-1\end{pmatrix}.
$$

Applying it to the first qubit gives

$$
\begin{aligned}
H|0\rangle
&=\frac{1}{\sqrt2}
\begin{pmatrix}1&1\\1&-1\end{pmatrix}
\begin{pmatrix}1\\0\end{pmatrix}\\
&=\frac{1}{\sqrt2}\begin{pmatrix}1\\1\end{pmatrix}\\
&=\frac{|0\rangle+|1\rangle}{\sqrt2}
\equiv|+\rangle.
\end{aligned}
$$

The state $|+\rangle$ points along the **positive $x$-axis** of the Bloch sphere. The second qubit remains in $|0\rangle$.

The complete two-qubit operation is $H\otimes I$, where $I$ is the $2\times2$ identity matrix:

$$
\begin{aligned}
|\psi_1\rangle
&=(H\otimes I)|\psi_0\rangle\\
&=(H|0\rangle)\otimes|0\rangle\\
&=\frac{1}{\sqrt2}
\begin{pmatrix}1\\1\end{pmatrix}
\otimes
\begin{pmatrix}1\\0\end{pmatrix}\\
&=\frac{1}{\sqrt2}
\begin{pmatrix}1\\0\\1\\0\end{pmatrix}\\
&=\frac{|00\rangle+|10\rangle}{\sqrt2}.
\end{aligned}
$$

**This state is still separable:** it factors as $|+\rangle\otimes|0\rangle$. A superposition does not by itself imply entanglement.

## 4. CNOT: control, target, and matrix

In the circuit, the solid dot on $q_0$ denotes the control, and the circled plus on $q_1$ denotes the target.

CNOT leaves the control unchanged and flips the target when the control is $1$:

$$
|a,b\rangle\longmapsto|a,b\oplus a\rangle,
\qquad a,b\in\{0,1\}.
$$

Here $\oplus$ means XOR (addition modulo two).

| Input | Output | Output vector |
|---|---|---|
| $\lvert 00\rangle$ | $\lvert 00\rangle$ | $(1,0,0,0)^T$ |
| $\lvert 01\rangle$ | $\lvert 01\rangle$ | $(0,1,0,0)^T$ |
| $\lvert 10\rangle$ | $\lvert 11\rangle$ | $(0,0,0,1)^T$ |
| $\lvert 11\rangle$ | $\lvert 10\rangle$ | $(0,0,1,0)^T$ |

The matrix columns are the output vectors for the corresponding input basis states:

$$
\mathrm{CNOT}=
\begin{pmatrix}
1&0&0&0\\
0&1&0&0\\
0&0&0&1\\
0&0&1&0
\end{pmatrix}.
$$

> **CNOT refinement:** The circled plus is the target of a controlled Pauli-$X$ operation. XOR describes its action on computational basis labels. On a superposition, CNOT acts linearly and coherently; it does not measure the control.

## 5. At $t_2$: apply CNOT

Using matrix multiplication:

$$
\begin{aligned}
|\psi_2\rangle
&=\mathrm{CNOT}|\psi_1\rangle\\
&=
\begin{pmatrix}
1&0&0&0\\
0&1&0&0\\
0&0&0&1\\
0&0&1&0
\end{pmatrix}
\frac{1}{\sqrt2}
\begin{pmatrix}1\\0\\1\\0\end{pmatrix}\\
&=\frac{1}{\sqrt2}
\begin{pmatrix}1\\0\\0\\1\end{pmatrix}.
\end{aligned}
$$

Equivalently, apply CNOT to each term:

$$
\begin{aligned}
\mathrm{CNOT}\frac{|00\rangle+|10\rangle}{\sqrt2}
&=\frac{\mathrm{CNOT}|00\rangle+\mathrm{CNOT}|10\rangle}{\sqrt2}\\
&=\frac{|00\rangle+|11\rangle}{\sqrt2}.
\end{aligned}
$$

The resulting Bell state is

$$
\boxed{|\Phi^+\rangle=\frac{|00\rangle+|11\rangle}{\sqrt2}}.
$$

This state cannot be written as a tensor product of two individual qubit states. That is what establishes entanglement.

Entanglement is created when CNOT acts on the superposition prepared by the Hadamard gate. Before CNOT, the joint state is separable; afterward, it cannot be expressed as a product of individual qubit states.

The input matters: CNOT does not create entanglement for every input—for example, it leaves \(|00\rangle\) unchanged.

## 6. Measurement probabilities

For measurement in the computational ($Z$) basis:

| Outcome $q_0q_1$ | Amplitude | Probability |
|---|---|---|
| $00$ | $1/\sqrt2$ | $1/2$ |
| $01$ | $0$ | $0$ |
| $10$ | $0$ | $0$ |
| $11$ | $1/\sqrt2$ | $1/2$ |

Each individual qubit has a 50% chance of yielding $0$ or $1$, while the two results always agree in an ideal experiment.

If $q_0$ is measured first, outcome $0$ leaves the joint state $|00\rangle$, and outcome $1$ leaves $|11\rangle$. The first outcome is random; these correlations cannot be used to send a controllable message to the other qubit.

## 7. Why the final state is entangled

A separable pure state would have the form

$$
(a|0\rangle+b|1\rangle)\otimes(c|0\rangle+d|1\rangle)
=ac|00\rangle+ad|01\rangle+bc|10\rangle+bd|11\rangle.
$$

Matching $|\Phi^+\rangle$ would require

$$
ac=bd=\frac{1}{\sqrt2},\qquad ad=bc=0.
$$

The nonzero products $ac$ and $bd$ require all four coefficients to be nonzero, contradicting $ad=bc=0$. Therefore the state cannot factor into two individual pure-qubit states: **it is entangled**.

> **Bloch-sphere refinement:** Before CNOT, $q_0$ has the pure state $|+\rangle$. After CNOT, neither qubit has its own pure state. Each has reduced density matrix $I/2$, represented by the center of the Bloch sphere, while the joint state remains pure.

## 8. Bell state versus classical correlation

The Bell state is a coherent superposition, not merely a classical 50/50 mixture of $|00\rangle$ and $|11\rangle$.

Both give identical computational-basis statistics. They differ in other measurement bases. Define

$$
|+\rangle=\frac{|0\rangle+|1\rangle}{\sqrt2},
\qquad
|-\rangle=\frac{|0\rangle-|1\rangle}{\sqrt2}.
$$

Then

$$
|\Phi^+\rangle=\frac{|++\rangle+|--\rangle}{\sqrt2}.
$$

In the $X$ basis, the Bell state again gives perfectly matching results: $++$ or $--$, each with probability $1/2$. The classical mixture instead gives all four $X$-basis outcomes with probability $1/4$ each.

Computational-basis correlations alone therefore do not establish entanglement.

## 9. Complete derivation at a glance

$$
|00\rangle
\xrightarrow{H\otimes I}
\frac{|00\rangle+|10\rangle}{\sqrt2}
\xrightarrow{\mathrm{CNOT}_{q_0\to q_1}}
\frac{|00\rangle+|11\rangle}{\sqrt2}
=|\Phi^+\rangle.
$$

The handwritten calculation is correct under the stated ordering convention. The main refinements are to distinguish single-qubit and two-qubit operators, amplitudes and probabilities, superposition and entanglement, and coherent Bell correlations from a classical mixture.
