# Fiber Cut Localization Research Context

## 1. Problem Overview

We are designing a system to localize a single fiber edge failure in a network.

The network is modeled as:

- A graph G(V, E)
  - V = containers (each may contain multiple devices and fiber nodes)
  - E = physical fiber edges (each edge connects exactly two fiber nodes)

Key constraints:

- Each fiber edge has exactly two fiber nodes (one per container)
- Fiber nodes are **dedicated**: every fiber node belongs to exactly one edge
  → no two edges share a fiber node
  → a single-edge cut isolates one node pair, no cascade through shared ports
- Devices do NOT have fixed attachment to specific fiber nodes
- Connectivity is inferred through paths in the container-level graph

---

## 2. Observability Assumptions (CRITICAL)

We assume high-quality logging:

1. If a logical neighbor (peer link) is truly down:
   → We almost always observe a log

2. If we observe a down log:
   → It is almost certainly true (no false positives)

3. Logs may be incomplete (rarely missing some events), but:
   → Missing logs are rare enough to treat "no log" as meaningful

This leads to:

- Positive evidence (down logs) → strong signal
- Silent evidence (no log) → strong negative evidence

---

## 3. Input Data

We observe:

### Baseline neighbors
A set of device pairs that normally have logical connectivity:

N = {n₁, n₂, ..., n_k}

Each neighbor pair is between two containers.

---

### Observed failures

D = subset of N:
- Neighbor pairs that are observed DOWN

S = N - D:
- Neighbor pairs with no down log (silent)

---

## 4. Key Modeling Challenge

We do NOT know:

- Which physical path each neighbor uses
- Which fiber edge a device is attached to

We only know:

- The container-level topology
- The neighbor relationships
- The observed failures

---

## 5. Core Insight

Each neighbor pair may use one or more candidate paths:

P(x) = set of candidate paths between the two containers

We approximate unknown routing by enumerating bounded simple paths.

---

## 6. Candidate Path Constraints

To avoid combinatorial explosion:

- max_hops (e.g. ≤ 4–6)
- max_paths (e.g. ≤ 10–20)

---

## 7. Scoring Model

The scoring rule is the BAYESIAN log-likelihood of the observed (D, S)
pattern under a uniform-actual-path generative model.

### Generative model

Each peer x picks its actual physical path uniformly at random from P(x),
independently of the failure. A peer reports DOWN iff its actual path
crosses e*. Under this model, the per-peer fraction

  f(x, e) := |{ p ∈ P(x) : e ∈ p }| / |P(x)|

is exactly:

  P(d ∈ D | e* = e) = f(d, e)
  P(s ∈ S | e* = e) = 1 - f(s, e)

---

### Local Silent Set

S_e is the local silent set for edge e — the silent peers that are
topologically close to e in the container graph.

For edge e = (u, v) and silent peer s = (a, b):

  s ∈ S_e  iff  min(dist(a, u), dist(a, v), dist(b, u), dist(b, v)) ≤ K

where dist(·, ·) is unweighted shortest-path distance in G(V, E)
and K is the local silent hop radius (e.g. 1 or 2).

Rationale:

- Restricting to S_e prevents density bias from far-flung silent peers
- Avoids unnecessary computation over the entire silent set per edge
- Captures the intuition that only nearby silent peers could plausibly
  witness a failure of e

---

### Score (Bayesian log-likelihood)

Score(e) = Σ_{d ∈ D} log f(d, e)
         + γ · Σ_{s ∈ S_e} log(1 - f(s, e))
         + α · Risk(e)

ε = 1e-3 is added as a floor inside the logs to avoid -∞ on degenerate
cases where f(d, e) = 0 (wrong edge, peer has no candidate through e)
or 1 - f(s, e) = 0 (every candidate of a silent peer crosses e, which
would have made it DOWN if e were really failed).

---

### Why e* dominates

For the true failed edge e*:
- Every down peer d has its actual path crossing e*, and that path is in
  P(d) → f(d, e*) ≥ 1/|P(d)| > 0 → log f stays bounded.
- Wrong edges that miss any down peer's candidate set incur a log ε ≈ -6.9
  penalty per missed peer.

With enough peers this gap is overwhelming: e* is the only edge whose
log f stays bounded across all of D.

---

### Final Score

Score(e) =
  Σ_{d ∈ D} log f(d, e)
  + γ · Σ_{s ∈ S_e} log(1 - f(s, e))
  + α · Risk(e)

Where:

- γ = silent log-term weight (default 0.8)
- α = log-prior weight (default 0; set Risk(e) := log(length(e)) and α=1
      to recover the exact Bayesian posterior under fiber-length priors)

---

## 8. Interpretation of Signals

- Positive → strong support
- Silent → strong contradiction
- Risk → prior probability

In our setting:

- Positive dominates
- Silent removes false candidates
- Risk breaks remaining ties

---

## 9. Identifiability Analysis (Key Contribution)

Under assumptions:

1. Single-edge failure
2. High-quality observability
3. Sufficient baseline neighbor coverage

We find:

### Almost always uniquely identifiable

Ambiguity occurs ONLY when:

Two edges e₁ and e₂ satisfy:

∀ neighbor pairs x:
  e₁ ∈ P(x) ⇔ e₂ ∈ P(x)

Meaning:

- They affect all observations identically
- They are observationally indistinguishable

---

## 10. Important Negative Results

We explicitly reject these as common issues:

### NOT a problem:

- Symmetric topology
  → broken by single-edge failure

- Silent symmetry
  → broken because real failures create asymmetric down patterns

- Missing logs
  → negligible due to system design

---

## 11. Real Causes of Ambiguity

Only two practical cases:

### 1. Model Granularity Issue

Multiple physical edges collapse into one logical edge:

A --E1-- B
A --E2-- B

Model cannot distinguish them.

---

### 2. Observational Equivalence

Edges affect all observed neighbor pairs identically:

No available observation can separate them.

---

## 12. Conclusion

Under high-quality logs:

- The problem is NOT inference-limited
- It is an information-theoretic identifiability problem

Key statement:

"Fiber cut localization is almost always uniquely solvable,
except when edges are observationally indistinguishable."

---

## 13. Research Direction

We are implementing:

- Simulation-based evaluation
- Scoring-based inference model
- Hyperparameter tuning via simulation
- Ablation studies
- Sensitivity analysis

---

## 14. Next Tasks

1. Modularize prototype code
2. Add Optuna hyperparameter tuning
3. Add identifiability checker
4. Run large-scale synthetic experiments
5. Prepare paper sections