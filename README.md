# Quantum Phase Estimation with Peak Interpolation

This is my implementation of the Quantum Phase Estimation (QPE) algorithm using PennyLane. The interesting part isn't just the circuit — it's what happens when the phase you're trying to estimate doesn't sit nicely on a binary grid, and how you can still recover it accurately.

---

## What's the point of this?

QPE is used to estimate the phase $\phi$ in the eigenvalue $e^{i\phi}$ of a unitary operator $U$. The circuit works by applying controlled-$U^{2^k}$ operations to an estimation register, then using the Inverse Quantum Fourier Transform to read out a binary approximation of $\phi$.

I picked $\phi = \pi/11$ on purpose — the denominator is prime, so it can't be represented exactly in binary. This means the probability doesn't concentrate at one output state, it spreads across two peaks. If you just take the highest probability state you get a pretty bad estimate. That's the problem this project solves.

---

## Why the naive approach fails

With $m = 5$ estimation qubits you have $N = 32$ possible output states. The true phase $\phi = \pi/11 \approx 0.2856$ falls between two bins.

If you just read off the highest probability state:

$$\hat{\phi}_{\text{naive}} \approx 0.1963$$

That's a ~31% relative error. Not great.

When you look at the output you can see two peaks with similar probabilities — that's the sign that the true phase is sitting between bins and you need to do something smarter.

---

## The fix — peak interpolation

Starting from the amplitude expression from Nielsen & Chuang:

$$\alpha_l = \frac{1}{N} \left( \frac{1 - e^{2\pi i (N\delta - l)}}{1 - e^{2\pi i (\delta - l/N)}} \right)$$

where $\delta = \phi - b/N$ is how far the true phase is from the nearest bin, you can square this and simplify using $1 - \cos\theta = 2\sin^2(\theta/2)$ to get the probabilities of the two peaks:

$$p_0 = \frac{\sin^2(\pi N \delta)}{\sin^2(\pi \delta)} \cdot \frac{1}{N^2}, \qquad p_1 = \frac{\sin^2(\pi N \delta)}{\sin^2(\pi \delta_1)} \cdot \frac{1}{N^2}$$

where $\delta_1 = \frac{1}{N} - \delta$.

Taking the ratio $r = \sqrt{p_1 / p_0}$ and using the small angle approximation $\sin(x) \approx x$:

$$r \approx \frac{\delta}{\delta_1}$$

Solving for $\delta$:

$$\delta = \frac{r}{N(r + 1)}$$

Then the corrected phase is just:

$$\hat{\phi} = \delta + \frac{b}{N}$$

The handwritten derivation walking through all of this is in `derivation_for_ratio.pdf`.

---

## Results

| Method | Estimated $\phi$ | True $\phi = \pi/11$ | Relative Error |
|---|---|---|---|
| Naive (highest peak) | 0.19635 | 0.28560 | ~31% |
| Peak interpolation | 0.28561 | 0.28560 | ~0.003% |

Goes from 31% error down to basically nothing.

---

## Circuit structure

```
n qubits |v⟩ ──[U]──[U²]──[U⁴]── ··· ──[U^(2^(m-1))]── |v⟩

m qubits |0⟩ ──[H]──●────────────────────────────────────┐
         |0⟩ ──[H]────────●───────────────────────────── │ IQFT → j_m
         |0⟩ ──[H]────────────●──────────────────────── │        j_(m-1)
          ⋮                    ⋮                          │         ⋮
         |0⟩ ──[H]────────────────────────────●──────── ┘        j_1
```

Each estimation qubit gets put into superposition by a Hadamard gate, then acts as the control for a controlled-$U^{2^k}$ operation. Because the estimation qubits are in superposition when $U$ is applied, you get phase kickback — the phase gets written onto the estimation register rather than the eigenstate register. The IQFT then converts all that phase information into a readable binary number.

---

## How to run it

```bash
pip install pennylane numpy matplotlib
```

Then just run the notebook `course_work_1.ipynb` top to bottom.

---

## Files

```
├── course_work_1.ipynb       # main notebook with everything in it
├── qpe_circuit.png           # circuit diagram
├── derivation_for_ratio.pdf  # handwritten derivation of the interpolation method
└── README.md
```

---

## References

- Nielsen & Chuang — *Quantum Computation and Quantum Information*, Chapter 5
- [PennyLane PhaseShift gate](https://docs.pennylane.ai/en/stable/code/api/pennylane.PhaseShift.html)
