# Gustafson's Law Practice — Tasks 01–07

## Table of Contents
- Task 01 — Strong vs. Weak Scaling
- Task 02 — Formal Derivation
- Task 03 — Serial Fraction Invariance
- Task 04 — Asymptotic Efficiency
- Task 05 — nCUBE Case Reconstruction
- Task 06 — Parametric Table
- Task 07 — Maximum Tolerable Serial Fraction


---

# Task 01 — Strong Scaling vs. Weak Scaling: Amdahl (1967) vs. Gustafson (1988)

## 1. Core assumptions

| Aspect | Amdahl (1967) | Gustafson–Barsis (1988) |
|---|---|---|
| Scaling type | Strong scaling | Weak scaling |
| Question asked | "How much faster can I solve *this* problem?" | "How much *bigger* a problem can I solve in the *same time*?" |
| Held constant | Total problem size W | Parallel wall-clock time T_P |
| Varies with P | Time to solution (decreases) | Problem size (increases) |
| Serial fraction measured on | Uniprocessor (s = T_serial / T_1) | Parallel machine (α = T_serial / T_P) |
| Serial *time* as P grows | Fixed absolute value | Fixed absolute value (but a shrinking share of the *scaled* work) |

## 2. Behaviour of problem dimension as P → ∞

**Amdahl.** Problem size is a constant, W(P) = W₁. Each processor receives W₁/P of the parallelisable work, so the parallel part shrinks toward zero and runtime tends to the serial floor `s · T₁`:

    S_Amdahl(P) = 1 / (s + (1 − s)/P)  →  1/s      as P → ∞

The *serial share* of the run grows toward 100 %. Speedup is hard-capped (s = 0.05 ⇒ S ≤ 20).

**Gustafson.** Problem size grows with P: the parallel work is W_par(P) = P · W_par(1), while the serial work stays (approximately) constant. Per-processor work is constant, so runtime is constant:

    S(P) = P − α(P − 1)  →  ∞  (linearly, slope 1 − α)    as P → ∞

The *serial share* of the (parallel) run stays fixed at α. There is no ceiling.

## 3. Why "fixed-size" vs. "fixed-time"

* **Amdahl = fixed-size**: the workload is the invariant. The model asks how the clock changes when the workload is spread over more workers. Speedup = T₁(W) / T_P(W).
* **Gustafson = fixed-time**: the *clock* is the invariant (e.g. a 1-hour weather forecast, an overnight batch). Extra processors are spent on resolution, fidelity, or domain size. Speedup is the ratio of the work done (equivalently, the time a single processor would need for the *scaled* work) to the work of the base problem: S = T₁(W_scaled) / T_P(W_scaled) with T_P held constant.

## 4. Philosophical vs. physical divergence

* **Philosophical.** Amdahl's view is that of a user with a *finished question* who wants an answer sooner; extra hardware has diminishing returns. Gustafson's view is that of a scientist with an *open-ended appetite*: problems expand to fill the available machine, so hardware is never "wasted" on a fixed-size job.
* **Physical.** Real simulations (grids, particle counts, sample sizes) have a tunable resolution parameter. Serial work such as setup, I/O, and program load tends to be roughly independent of that parameter, while parallel work (the inner loops) grows with it. Hence the serial *fraction* of the scaled workload falls as P rises, which is the mechanism behind Gustafson's optimism.
* **Mathematical equivalence.** The two laws are not contradictory; they describe different experiments. Task 03 shows that Gustafson's α and Amdahl's s are the same physical serial time normalised by different baselines (T_P vs. T_1).

## 5. Caveats
Gustafson's law presumes the workload *can* be scaled meaningfully, that memory per node is sufficient, and that serial/communication costs do not themselves grow with P (see Tasks 10, 12, 13). For jobs with a fixed problem (e.g. a compiler run, a single real-time deadline on fixed data), Amdahl remains the right model.


---

# Task 02 — Formal Derivation of Gustafson's Scaled Speedup

## Setup
Run the *scaled* problem on P processors and normalise the parallel wall-clock time to 1:

    s + p = 1                                              (1)

* s = time spent in the serial section (one processor works, the rest idle)
* p = time spent in the parallel section (all P processors busy)

Justification: normalisation fixes the unit of time; it loses no generality because speedup is a ratio.

## Step 1 — Work done on the parallel machine
During the parallel section each of the P processors does p units of work, so total parallel work = p·P. The serial section contributes s. Total work (in units of "one processor-time"):

    W_total = s + p·P                                      (2)

## Step 2 — Time for one processor to do the same scaled problem
A single processor executes the serial part in s and the parallel part sequentially in p·P:

    T₁ = s + p·P                                           (3)

Assumption: the one-processor run performs the same operations at the same rate (no cache/memory effects), so time = work.

## Step 3 — Speedup
    S(P) = T₁ / T_P = (s + p·P) / (s + p) = s + p·P        (4)

using (1) in the denominator.   ∎ (first form)

## Step 4 — Eliminate p to get the α-form
Let α ≡ s (serial fraction *observed on the parallel machine*). From (1), p = 1 − α:

    S(P) = α + (1 − α)·P
         = α + P − αP
         = P − α(P − 1)                                    (5)   ∎

## Step 5 — Sanity checks
* P = 1: S = 1 − 0 = 1 ✔
* α = 0: S = P (ideal) ✔
* α = 1: S = 1 (fully serial) ✔
* Slope dS/dP = 1 − α, so every added processor contributes (1 − α) of a processor.

## Step 6 — Efficiency
    E(P) = S(P)/P = 1 − α(P − 1)/P                         (6)

## Interpretation
The "equivalent serial workload" is T₁ = α + (1 − α)P: the serial part counts once, the parallel part counts P times because it is replicated across P workers' worth of work. Compared with Amdahl, the key difference is that the parallel portion of the workload is allowed to grow by a factor of P.


---

# Task 03 — Serial Fraction Invariance: Frame of Reference

**To:** Practice repository  **Re:** Why measuring the serial fraction on the parallel machine removes the apparent contradiction

## 1. Two definitions of "serial fraction"
Let the scaled problem have absolute serial time σ and absolute parallel work π (in one-processor time units).

* **Gustafson's α** (measured on the P-processor run): 
  T_P = σ + π/P,  α = σ / T_P.
* **Amdahl's s** (measured on a uniprocessor run of the same problem): 
  T₁ = σ + π,  s = σ / T₁.

Same physical σ, different denominators.

## 2. Converting between frames
Normalise T_P = 1 so σ = α and π = (1 − α)P. Then T₁ = α + (1 − α)P and

    s = α / (α + (1 − α)P)                                  (1)

Inverting: α = sP / (1 + s(P − 1)).

## 3. Showing the two laws are the same statement
Plug (1) into Amdahl's formula for the *same* scaled problem on P processors:

    1/(s + (1 − s)/P)
    = T₁ / (σ + π/P)        [multiply numerator/denominator by T₁]
    = (α + (1 − α)P) / 1
    = P − α(P − 1)

which is Gustafson's S(P). So the speedup of the scaled problem is identical under either law; the "contradiction" is a mismatch of baselines.

## 4. Numeric illustration (α = 0.04)
| P | s = α/(α+(1−α)P) |
|---|---|
| 16 | 0.04/(0.04+15.36) = 0.00260 |
| 1024 | 0.04/(0.04+983.04) = 4.07 × 10⁻⁵ |
| 16384 | 0.04/(0.04+15728.64) ≈ 2.54 × 10⁻⁶ |

The uniprocessor serial fraction of the *scaled* problem collapses as P grows, so Amdahl's 1/s ceiling (≈ 24,600 at P = 1024, and far beyond P) never binds.

## 5. Resolution of the "paradox"
* Amdahl's s is a property of a *fixed* problem; if the problem grows, s itself changes (shrinks) — it is not invariant.
* Gustafson's α is observed on the machine actually used, at the scaled size, and is the quantity that stays roughly constant when serial work (setup, I/O, reductions of fixed size) does not grow with the problem.
* Using a uniprocessor s measured on the *small baseline* problem and extrapolating it to a huge problem is the error that creates the apparent ceiling.

## 6. Critical remark
The "invariance" of α is an empirical assumption, not a theorem. It holds when serial work is O(1) in problem size and P. If σ grows with P (serial setup per rank, centralised I/O, O(P) or O(log P) collectives) then α(P) rises and the law over-predicts (see Task 12).


---

# Task 04 — Asymptotic Analysis of Parallel Efficiency

## 1. Gustafson efficiency
    E_G(P) = S(P)/P = [P − α(P − 1)]/P = 1 − α(P − 1)/P = 1 − α + α/P

### Limit
    lim_{P→∞} E_G(P) = lim (1 − α + α/P) = 1 − α

**Proof.** α/P → 0 as P → ∞ for fixed α. ∎

Additional facts:
* E_G is strictly decreasing in P (derivative −α/P² < 0) for α > 0, from E(1) = 1 down to the asymptote 1 − α.
* Bounds: 1 − α ≤ E_G(P) ≤ 1. 
* Convergence is O(1/P): E_G(P) − (1 − α) = α/P.

## 2. Amdahl efficiency
    S_A(P) = 1/(s + (1 − s)/P)
    E_A(P) = S_A/P = 1/(sP + 1 − s)

### Limit
    lim_{P→∞} E_A(P) = lim 1/(sP + 1 − s) = 0   (for s > 0)

**Proof.** The denominator → ∞. ∎ 
Rate: E_A ≈ 1/(sP), i.e. O(1/P). The efficiency falls to half of its P = 1 value when sP + 1 − s = 2, i.e. P = (1 + s)/s ≈ 1/s.

## 3. Contrast
| | Gustafson | Amdahl |
|---|---|---|
| lim S | ∞ (linear, slope 1−α) | 1/s (finite) |
| lim E | **1 − α** (positive constant) | **0** |
| Example α = s = 0.05 | E → 0.95 | E → 0; at P=1000, E ≈ 0.0196 |

## 4. Deductions
* Under weak scaling a fixed fraction (1−α) of the machine is always productive, because the parallel work grows with P and keeps every processor busy; the loss α is a constant "tax" (the idle time of P−1 processors during the serial phase, as a fraction of total capacity, tends to α).
* Under strong scaling the parallel work per processor vanishes while the serial section stays fixed, so the machine is eventually idle almost all the time.
* Practical message: a code with a few percent serial time can remain highly efficient on very large machines only if the problem is scaled with the machine.


---

# Task 05 — Historical Reconstruction: the nCUBE/10 Result (1988)

> Note: figures below are from my recollection of the Gustafson–Montry–Benner work at Sandia and the 1988 CACM paper "Reevaluating Amdahl's Law". Please verify exact numbers against the paper before submitting.

## 1. Context
* In 1967 Amdahl argued that a serial fraction caps speedup at 1/s. By the 1980s conventional wisdom (a "10× limit" for massive parallelism) held that machines with ~1000 processors were pointless: a 1 % serial fraction implies ≤ 100×.
* At Sandia National Laboratories, Gustafson, Montry and Benner ran real applications on a **1024-processor nCUBE/10** hypercube (distributed memory, small local memory per node).
* Three applications were reported: a **beam stress analysis**, a **baffled surface-wave simulation**, and an **unstable fluid-flow** simulation. Scaled speedups of about **1,016–1,021** on 1,024 processors were reported.

## 2. Why this "should" have been impossible under Amdahl
The measured serial fraction on the parallel machine was small but non-zero (on the order of 0.4–0.8 %). Applying Amdahl with s = 0.004:

    S_A(1024) = 1/(0.004 + 0.996/1024) ≈ 201
    ceiling 1/s = 250

Observed ≈ 1,000+ is 4–5× above the Amdahl value and above the 250 ceiling.

## 3. How problem expansion dismantled the ceiling
1. **Problem sizes were scaled with P.** Each processor worked on a fixed-size subdomain (grid points / elements); the 1024-node problem was ~1024× larger than the 1-node problem. Runtime stayed roughly constant.
2. **Serial work did not scale.** Program load, setup, and sequential portions are almost independent of grid size, so serial *time* stayed small while parallel *work* grew ×1024.
3. **Serial fraction measured on the parallel run.** Writing α ≈ 0.004 and using the scaled formula:

       S = 1024 − 0.004 × 1023 ≈ 1019.9

   which matches the observed ~1,020 (and α = 0.008 gives ≈ 1,015.8, matching the lower result).
4. **Equivalent Amdahl s.** From Task 03, s = α/(α + (1−α)P) = 0.004/(0.004 + 1020.0) ≈ 3.9 × 10⁻⁶. The "serial fraction" of the *scaled* problem on one processor was a few parts per million, so Amdahl's ceiling (~255,000) was nowhere near binding.

## 4. Why a fixed problem could not be run that way
A single nCUBE node had only a small local memory, so the 1-processor *scaled* problem would not fit. The authors estimated the single-processor time by operation counts rather than direct measurement — a methodological caveat that later motivated memory-bounded models (Sun–Ni, Task 11).

## 5. Takeaway
The result did not refute Amdahl's arithmetic; it refuted the *assumption* that the problem size is fixed. Amdahl's law answers a strong-scaling question; the nCUBE experiments answered a weak-scaling one. This reframed how supercomputer use was evaluated (e.g., the Gordon Bell prize results of the era).


---

# Task 06 — Parametric Speedup, α = 0.04

Formulas: S(P) = P − α(P − 1),  E(P) = S(P)/P.

## Worked log
| P | α(P−1) | S(P) = P − α(P−1) | E(P) |
|---|---|---|---|
| 16 | 0.04×15 = 0.6 | 16 − 0.6 = **15.4** | 15.4/16 = **0.962500** |
| 64 | 0.04×63 = 2.52 | 64 − 2.52 = **61.48** | **0.960625** |
| 256 | 0.04×255 = 10.2 | 256 − 10.2 = **245.8** | **0.960156** |
| 1024 | 0.04×1023 = 40.92 | 1024 − 40.92 = **983.08** | **0.960039** |
| 4096 | 0.04×4095 = 163.8 | 4096 − 163.8 = **3932.2** | **0.960010** |
| 16384 | 0.04×16383 = 655.32 | 16384 − 655.32 = **15728.68** | **0.960002** |

## Evaluation
* Efficiency E(P) = 0.96 + 0.04/P, so it converges to **1 − α = 0.96** from above.
* Even as P grows by 1024× (16 → 16384), efficiency drops only 0.9625 → 0.9600 (−0.26 percentage points). Degradation is *not* exponential; it saturates at 96 %.
* Speedup grows essentially linearly with slope 0.96: roughly 96 % of every added core is useful.
* The loss in absolute cores is large (655 cores' worth at P = 16,384) but constant in *fraction*.
* Contrast: Amdahl with s = 0.04 would cap speedup at 25 for every row.
* Caveat: this assumes α stays at 4 % at every scale (see Task 12 for growing overhead).


---

# Task 07 — Maximum Tolerable Serial Fraction

Given: P = 8,192, requirement S(P) ≥ 7,500.

## Derivation
    S(P) = P − α(P − 1) ≥ 7500
    −α(P − 1) ≥ 7500 − P
    α(P − 1) ≤ P − 7500        (multiply by −1, inequality flips)
    α ≤ (P − 7500)/(P − 1)     (P − 1 > 0)

## Numbers
    α_max = (8192 − 7500)/(8192 − 1) = 692/8191
          ≈ 0.084483

**α_max ≈ 0.0845 (≈ 8.45 % serial time on the parallel machine).**

## Verification
S = 8192 − 0.084483 × 8191 = 8192 − 692.00 = 7500.0 ✔

## Implied efficiency
E = 7500/8192 = 0.9155 (≈ 91.6 %), consistent with E = 1 − α(P−1)/P.

## Notes
* Because the slope is 1 − α, the budget is generous: nearly 8.5 % serial time is allowed.
* Equivalent Amdahl-frame fraction: s = α/(α + (1−α)P) ≈ 0.0845/(0.0845 + 7499.9) ≈ 1.13 × 10⁻⁵ — i.e., the uniprocessor serial fraction of the scaled problem would have to be only ~11 ppm. The tolerance only holds if the problem is truly scaled to fill the machine.
