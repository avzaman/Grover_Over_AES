# Grover's Algorithm Attack on Mini-AES

A quantum key-recovery attack on a 4-bit AES variant, built for CS558 Quantum Cryptography. The circuit runs Grover's algorithm to find the secret encryption key with **95.7% probability in just 3 iterations** — compared to the 8 classical guesses a brute-force search would average over 16 candidates.

---

## What this is

AES is the world's most widely deployed symmetric cipher. Grover's algorithm offers a theoretical quadratic speedup for searching unstructured spaces, which means a quantum computer running Grover on full 128-bit AES would halve its effective security to 64 bits. This project builds a concrete demonstration of that attack from scratch.

Because a full-scale AES circuit would require thousands of logical qubits, the implementation uses a **4-bit Mini-AES** — a single-round cipher with the same structural steps as real AES (AddRoundKey, SubBytes, ShiftRows, MixColumns) but a key space of only 16 candidates. This makes it simulable on a laptop while preserving the exact circuit architecture described in the academic literature.

The attack is a **known-plaintext key-recovery**: given one plaintext–ciphertext pair, the circuit recovers the secret key with near-certainty.

---

## How it works

```
|0⟩⁴  key  ──── H⊗⁴ ──┬──────────────────┬──┬──── M ──▶  key bits
                       │  × k iterations  │  │
|0⟩⁴ state ──────────── │ Oracle  Diffusion │──│
                       │  Uf       D       │  │
|1⟩  oracle ── H ────── └──────────────────┘  │
```

Each Grover iteration applies two operations:

1. **Oracle** `Uf` — runs Mini-AES in superposition over all 16 candidate keys, compares the output to the known ciphertext, and phase-flips the amplitude of the correct key via phase kickback onto the `|−⟩` oracle qubit. The state register is then uncomputed back to `|plaintext⟩` so the workspace stays clean.

2. **Diffusion** `D` — reflects the amplitude distribution about its mean, amplifying the marked state and suppressing all others. Implemented as `H X MCZ X H` on the key register.

The optimal iteration count is `k = ⌊(π/4)√(N/M)⌋ = 3` for N=16, M=1. After 3 iterations the secret key has ~96% probability amplitude.

---

## Mini-AES design

| Parameter | Value |
|---|---|
| Block size | 4 bits |
| Key size | 4 bits |
| Rounds | 1 |
| S-box | PRESENT cipher 4-bit S-box |
| Key space | 16 candidates |

The round structure mirrors FIPS-PUB 197:

```
plaintext ─▶ AddRoundKey ─▶ SubBytes ─▶ ShiftRows ─▶ MixColumns ─▶ AddRoundKey ─▶ ciphertext
                ⊕ key      PRESENT S-box  rotate ⟳1   GF(2) linear      ⊕ key
```

The quantum circuit verifies against the classical reference across all 256 `(key, plaintext)` pairs before running the attack — zero mismatches.

---

## Results

**Single run (4096 shots):**

| Outcome | Probability |
|---|---|
| `\|0111⟩` — secret key ✓ | **95.7%** |
| All other candidates | < 0.4% each |

**Multi-trial (10 runs × 4096 shots):**

```
mean success rate: 0.957 ± 0.002
```

The amplitude trajectory matches the theoretical `sin²((2k+1)θ)` curve and shows the characteristic Grover overshoot when iterated past k=3.

---

## Circuit stats

| Metric | Value |
|---|---|
| Qubits | 9 (4 key + 4 state + 1 oracle) |
| Classical bits | 4 |
| Pre-transpile depth | 87 |
| Post-transpile depth | 56 |
| Gate count (post) | 136 (66 CX, 19 U2, 18 X, ...) |

---

## Running it

**Requirements:** Python 3.9+, Jupyter (or Google Colab)

```bash
pip install qiskit==2.4.1 qiskit-aer==0.17.2 pylatexenc matplotlib
jupyter notebook Minimal-Grover-AES-Attack.ipynb
```

The notebook is structured as self-contained cells:

| Cell | Contents |
|---|---|
| 1–2 | Install dependencies, imports |
| 3 | Classical Mini-AES reference implementation |
| 4 | Quantum gates for each AES step |
| 5 | Grover oracle (encrypt + compare + uncompute) |
| 6 | Grover diffusion operator |
| 7 | Full circuit assembly |
| 8 | Aer simulator run + measurement histogram |
| 9 | Amplitude growth visualization across iterations |
| A–F | Publication-quality figure generators |

---

## References

- Grover, L. K. (1996). *A fast quantum mechanical algorithm for database search.* STOC '96.
- Grassl, M., Langenberg, B., Roetteler, M., & Steinwandt, R. (2016). *Applying Grover's Algorithm to AES: Quantum Resource Estimates.* PQCrypto 2016.
- Preston, C. (2022). *Applying Grover's Algorithm to Hash Functions.*
- PRESENT cipher S-box: Bogdanov et al. (2007). *PRESENT: An Ultra-Lightweight Block Cipher.* CHES 2007.
