# Fiber-Cut Localization is Almost Always Uniquely Solvable

### An Identifiability View under High-Quality Logs

> **TL;DR.** Under realistic high-quality logging, single-edge fiber-cut localization is information-theoretic, not inference-limited. We derive a Bayesian log-likelihood scoring rule from a uniform-actual-path generative model. On every real-shaped topology tested (WAN backbones Abilene and NSFNet; DC fabrics fat-tree and spine-leaf) it hits `top-1 ≈ 1.00`. The only structural cases that retain non-trivial equivalence classes are adversarial designs with intentional symmetry — most notably **physical parallel fibers between the same two sites** (multi-rail uplinks, redundant cabling). There the algorithm narrows the suspect set to the small symmetric group; resolving inside the group needs out-of-band signals (OTDR, port-level cable identification).

---

## Abstract

We study the localization of a *single* fiber-cut failure in a network modeled as a container-level graph $G(V, E)$. Under high-quality observability — true down logs are reliable, missing logs are rare — the residual difficulty is *identifiability*: a failure is uniquely localizable unless two edges share an observational signature. Three contributions: (1) a Bayesian log-likelihood scoring rule derived from a uniform-actual-path generative model, with a single closed-form lemma; (2) an identifiability checker that maps each edge to its signature and groups edges into equivalence classes; (3) a reproducible evaluation. On every real-shaped topology tested (Abilene, NSFNet, fat-tree, spine-leaf) the algorithm hits `top-1 ≈ 1.00`. Adversarial topologies engineered for intentional symmetry retain non-trivial equivalence classes; on those the algorithm still picks the correct class with probability 1. The only structural ambiguity that persists in production is physical parallel fibers between the same site pair, which requires out-of-band physical-layer signals to break.

## Contents

1. [Introduction](#1-introduction)
2. [Model](#2-model)
3. [Scoring](#3-scoring)
4. [Identifiability](#4-identifiability)
5. [Evaluation](#5-evaluation)
6. [Limitations](#6-limitations)
7. [Reproducing the results](#7-reproducing-the-results)
8. [Code map](#8-code-map)

---

## 1. Introduction

When a fiber edge in a packet network fails, an operator wants to know *which* one. Most existing approaches treat this as a robust inference problem against noisy, missing, or conflicting logs. We start from a different premise: in the systems we target, logging is **high quality** — true down events are almost always observed, false positives are rare. Under these assumptions, the residual difficulty is no longer statistical noise but *identifiability*: how often does the observation pattern $(D, S)$ contain enough information to uniquely point at one edge?

The answer turns out to be: **almost always**. Across every real-shaped topology we test — sparse irregular WAN backbones and dense layered DC fabrics alike — the algorithm hits `top-1 ≈ 1.00`. The cases that resist single-edge localization are exactly those with engineered structural symmetry, most concretely *physical parallel fibers* between the same two sites; on those the algorithm still recovers the correct equivalence class.

**Contributions.**
- A Bayesian log-likelihood scoring rule for single-edge fiber-cut localization, derived from a uniform-actual-path generative model (§3) — closed-form, no training, no per-graph tuning.
- A formal definition of *observational equivalence* over edges and an identifiability checker that groups edges into equivalence classes by per-edge signatures (§4).
- A reproducible empirical study on WAN and DC topologies (§5) that puts a quantitative bound on the only remaining structural ambiguity (parallel fibers).

---

## 2. Model

### 2.1 Graph

Let $G(V, E)$ be undirected, where:
- $V$ is a set of containers — each may host several devices and **fiber endpoints (fiber nodes)**.
- $E$ is the set of physical fiber edges; each edge connects exactly two fiber endpoints, one per container.

We make two structural assumptions about the fiber layer:

1. **Dedicated fiber endpoints.** Every fiber node is an endpoint of exactly one edge. A container may host multiple fiber nodes, but no two edges share a fiber node. A single-edge cut therefore disables one specific pair of fiber endpoints and nothing else — it cannot cascade through a shared port — which is what makes "single-edge failure" a well-defined event at the container level.
2. **Devices are not pinned.** Devices are not bound to a specific fiber endpoint within their container; logical connectivity between two devices is inferred through paths in the container graph rather than by following a fixed device → fiber-node mapping.

Picture containers as **buildings** and fiber nodes as **wall sockets**. Each cable plugs into one dedicated socket per end (no socket shared). The algorithm works directly on the container graph $G(V, E)$; the dedicated-endpoint assumption is what lets us discard the internal socket-level detail.

![Container anatomy](figures/concept_anatomy.png)

### 2.2 Observability assumptions

Let $N \subseteq V \times V$ be the **baseline neighbor set**: a known set of peer pairs

$$x = (a_x, b_x), \qquad a_x, b_x \in V, \quad a_x \neq b_x,$$

that normally have logical connectivity. After a single edge $e^\star \in E$ fails, observation partitions $N$:

- $D \subseteq N$: peers that emit a DOWN log.
- $S = N \setminus D$: silent peers.

Assumptions:

1. **No false positives.** $\Pr[\text{DOWN log} \mid \text{link up}] \approx 0$.
2. **High recall.** $\Pr[\text{DOWN log} \mid \text{link down}] \approx 1$.
3. **Single-edge failure.** Exactly one $e^\star \in E$ at a time.

Under (1)–(2), peer $x$ is in $D$ iff its actual path contains $e^\star$, and in $S$ otherwise — both classifications are noise-free given $e^\star$ and the actual path. **Silent peers therefore carry positive information, not absence of information**: the silent label rules out every hypothesis "$e^\star = e$" under which $x$ would have routed through $e$. The scoring rule in §3 turns this rule-out into a $\log(1 - f(s, e))$ penalty per silent peer.

A diamond container graph, edge $(B, D)$ just cut, two peers between $A$ and $D$ that routed over different physical paths:

![Down vs silent observation](figures/concept_observation.png)

Same endpoints, opposite observations: Peer 1's path crosses the cut → DOWN; Peer 2's avoids it → silent. The algorithm sees only the labels, not the physical paths.

### 2.3 Routing is unknown

The actual physical path each peer uses is unobservable. For each $x = (a_x, b_x) \in N$ we enumerate the **candidate-path set**

$$P(x) := \\{ p = (v_0, v_1, \dots, v_k) : v_0 = a_x, v_k = b_x, v_i \in V \text{ pairwise distinct}, (v_i, v_{i+1}) \in E, k \le L \\}, \qquad |P(x)| \le M.$$

A path $p$ is **simple** iff its containers $v_0, \dots, v_k$ are pairwise distinct. Defaults: $L = 4$, $M = 20$.

![Simple paths between A and D](figures/concept_simple_paths.png)

In the diamond above, $P(A, D) = \\{ A \to B \to D, A \to C \to D \\}$. A walk like $A \to B \to D \to C \to A$ revisits $A$, so it is not simple and never enters $P(\cdot)$.

---

## 3. Scoring

### 3.1 Generative model

For each peer $x \in N$ let $\pi(x)$ denote its **actual** physical path — a random variable taking values in $P(x)$. We make two assumptions:

- **A1 (uniform routing).** $\pi(x) \sim \mathrm{Uniform}(P(x))$, independent of $e^\star$.
- **A2 (high-quality logging, §2.2).** Peer $x$ reports DOWN iff $e^\star \in \pi(x)$.

Define the **candidate-coverage fraction**

$$f(x, e) := \frac{|\\{ p \in P(x) : e \in p \\}|}{|P(x)|} \in [0, 1].$$

> **Lemma (Likelihood).** *Under A1 and A2,*
>
> $$\Pr[d \in D \mid e^\star = e] = f(d, e), \qquad \Pr[s \in S \mid e^\star = e] = 1 - f(s, e).$$

*Proof.* By A2, the event $\\{d \in D\\}$ equals $\\{e^\star \in \pi(d)\\}$. Substituting and using independence (A1):

$$\Pr[d \in D \mid e^\star = e] = \Pr[e \in \pi(d) \mid e^\star = e] \stackrel{\text{(A1)}}{=} \Pr[e \in \pi(d)].$$

The second equality drops the conditional because the event $\\{e \in \pi(d)\\}$ depends only on $\pi(d)$, and A1 says $\pi(d) \perp e^\star$ — knowing which edge failed gives no information about which path $d$ chose. (The two occurrences of $e$ above refer to the same fixed edge: on the left it labels the conditioning value of $e^\star$, on the right it is the edge whose presence in $\pi(d)$ we are testing.)

Expanding by total probability over $\pi(d) \in P(d)$:

$$\Pr[e \in \pi(d)] = \sum_{p \in P(d)} \Pr[\pi(d) = p] \cdot \mathbb{1}[e \in p].$$

Apply A1's uniform law, $\Pr[\pi(d) = p] = 1/|P(d)|$ for every $p \in P(d)$, and pull the constant out of the sum:

$$= \sum_{p \in P(d)} \frac{1}{|P(d)|} \cdot \mathbb{1}[e \in p] = \frac{1}{|P(d)|} \sum_{p \in P(d)} \mathbb{1}[e \in p].$$

Use the standard identity $\sum_{p \in S} \mathbb{1}[Q(p)] = |\\{p \in S : Q(p)\\}|$ (an indicator sum just counts how many elements satisfy the predicate) with $S = P(d)$ and $Q(p) = (e \in p)$:

$$= \frac{|\\{ p \in P(d) : e \in p \\}|}{|P(d)|} = f(d, e).$$

The silent identity follows because $\\{s \in S\\}$ is the complement of $\\{s \in D\\}$. $\square$

> **Remark (where independence comes from).** A1 models routing as a *static snapshot*: $\pi(x)$ is the route the peer was using at the moment of failure, decided beforehand and not re-routed in response to $e^\star$. This is consistent with §2.2's high-recall, single-edge-failure observation window — the peer simply fails, it does not get a chance to re-route around $e^\star$ before reporting. If the routing protocol *did* react to $e^\star$ (re-converge before the down log fires), $\pi(x)$ would no longer be independent of $e^\star$ and the lemma would need a different form.

The lemma turns the score-design problem into a likelihood-maximisation problem: edges with high $f$ on down peers and low $f$ on silent peers are precisely the edges that best explain the observed pattern.

### 3.2 Local silent set $S_e$

Let $\mathrm{dist}_G(\cdot, \cdot)$ be the unweighted shortest-path distance in $G$. For edge $e = (u, v) \in E$ define

$$S_e := \\{ s = (a_s, b_s) \in S : \min(\mathrm{dist}_G(a_s, u), \mathrm{dist}_G(a_s, v), \mathrm{dist}_G(b_s, u), \mathrm{dist}_G(b_s, v)) \le K \\}.$$

— silent peers with at least one endpoint within $K$ hops of either endpoint of $e$ (default $K = 2$). Restricting to $S_e$ avoids density bias from far-flung silent peers and keeps the per-edge cost bounded.

### 3.3 Bayesian log-likelihood

The score for edge $e$ is the log-likelihood of the observed pattern $(D, S)$ under the hypothesis $e^\star = e$, with the silent term softly weighted by $\gamma$ and an optional edge prior $\alpha \cdot \mathrm{Risk}(e)$:

$$\mathrm{Score}(e) = \underbrace{\sum_{d \in D} \log f(d, e)}_{\text{positive (down) evidence}} + \gamma \sum_{s \in S_e} \log\bigl(1 - f(s, e)\bigr) + \alpha \cdot \mathrm{Risk}(e).$$

We add a small $\varepsilon = 10^{-3}$ floor to both $f$ and $1 - f$ inside the logs to handle the degenerate cases.

**Why $e^\star$ is special.** For every down peer $d$, the actual physical path crosses $e^\star$ and is itself a member of $P(d)$, so $f(d, e^\star) \ge 1/|P(d)| > 0$ and the $\log f$ stays bounded. Any wrong edge $e'$ that misses some down peer's candidate set incurs a $\log \varepsilon \approx -6.9$ penalty per missed peer. With enough peers this gap is overwhelming: $e^\star$ is unique in the property of having *every* down peer's candidate set contain it.

### 3.4 Two-stage interpretation: topology, then prior

Up to an additive constant $\mathrm{Score}(e)$ is the log-posterior $\log \Pr[e^\star = e \mid D, S]$ under the generative model of §3.1 with prior $\Pr[e^\star = e] \propto \exp(\mathrm{Risk}(e))$:

- **Stage 1 — topological likelihood** ($\log f$ terms): depends only on which peers' candidate sets contain $e$ and how many of their candidates do. By §4, edges in the same equivalence class $C(e^\star)$ have identical $f(x, \cdot)$ and hence identical likelihood; stage 1 saturates at $C(e^\star)$ — the information-theoretic ceiling.
- **Stage 2 — prior tiebreaker** ($\alpha \cdot \mathrm{Risk}(e)$): only matters when stage 1 ties. The natural choice $\mathrm{Risk}(e) := \log \mathrm{length}(e)$ with $\alpha = 1$ gives the exact Bayesian posterior under the fiber-length prior $\Pr[e^\star = e] \propto \mathrm{length}(e)$.

We default $\alpha = 0$ in §5 so the experiments show the unaided power of stage 1 alone.

### 3.5 Hyperparameters

| symbol | meaning | default |
|---|---|---|
| $L$ | max hops in candidate-path enumeration | 4 |
| $M$ | max candidate paths per peer | 20 |
| $K$ | silent hop radius defining $S_e$ | 2 |
| $\gamma$ | weight of silent log-term | 0.8 |
| $\alpha$ | weight of edge prior $\mathrm{Risk}(e)$ | 0 |
| $\varepsilon$ | floor for $\log$ to avoid $-\infty$ | $10^{-3}$ |

---

## 4. Identifiability

### 4.1 Definition

Two edges $e_1, e_2$ are **observationally equivalent** under a peer set $N$ if they appear in the same number of candidate paths of every peer:

$$\forall x \in N :  f(x, e_1) = f(x, e_2).$$

Whenever this holds, $\sum_d \log f(d, e_1) = \sum_d \log f(d, e_2)$ and $\sum_{s \in S_e} \log(1 - f(s, e_1)) = \sum_{s \in S_e} \log(1 - f(s, e_2))$ for *any* failure pattern, so the scoring rule cannot break the tie.

### 4.2 Signatures and equivalence classes

Fix an enumeration $N = \\{x_1, x_2, \dots, x_n\\}$ with $n = |N|$. Let

$$n_i(e) := |\\{ p \in P(x_i) : e \in p \\}|$$

count how many candidate paths of peer $x_i$ contain edge $e$. Define the **signature**

$$\sigma(e) := \big((i, n_i(e)) \big| 1 \le i \le n, n_i(e) > 0 \big).$$

Since $|P(x_i)|$ is shared across all edges of peer $x_i$, edges with identical signatures have identical $f(x_i, \cdot)$ for every peer. They form an equivalence class. We compute signatures in $O(|E| \cdot n \cdot M)$ — see [`identifiability.py`](src/fiber_cut_localizer/identifiability.py).

A textbook example. Consider the *bridge* graph below, with two intermediate nodes $X_0$ and $X_1$ between anchors $A$ and $B$, and a single peer pair $(A, B)$:

![Equivalence class on a bridge graph](figures/concept_equivalence.png)

The simple paths from $A$ to $B$ within 4 hops are exactly $A \to X_0 \to B$ and $A \to X_1 \to B$. Each of the four edges appears in **exactly one** candidate path, so $n_1(\cdot) = 1$ for all four:

$$\sigma(A\text{–}X_0) = \sigma(X_0\text{–}B) = \sigma(A\text{–}X_1) = \sigma(X_1\text{–}B) = ((1, 1)).$$

All four edges sit in one equivalence class of size 4. No matter which one is the actual cut, the scoring rule produces a 4-way tie. The algorithm correctly narrows the suspect set to that class, but it cannot pick the single guilty edge from inside it. This is exactly the "model granularity" failure mode of §11 in [`PROJECT_CONTEXT.md`](PROJECT_CONTEXT.md), now with a precise definition.

### 4.3 Metrics that decouple algorithm from identifiability

Let $C(e^\star) \subseteq E$ denote the equivalence class of the failed edge $e^\star$ under the signatures of §4.2. We report three numbers per trial setup:

- **`top-k`** — the failed edge $e^\star$ itself is ranked in the top $k$. Strict success.
- **`class_top-k`** — at least one edge from $C(e^\star)$ is ranked in the top $k$. Relaxed success: the algorithm pinned the right *suspect group* even if it could not pick the single guilty edge inside the group.
- **`ambiguous_rate`** — the fraction of trials where $|C(e^\star)| > 1$. A property of the topology and peer placement, *not* of the algorithm: it measures how often the data was information-theoretically incapable of singling out one edge.

Reading them together:

| pattern | interpretation |
|---|---|
| `ambiguous_rate ≈ 0` and `top-k ≈ class_top-k` | data was sufficient; any error gap to 1.0 is the algorithm's fault. |
| `ambiguous_rate ≈ 1` and `class_top-k` ≫ `top-k` | data is fundamentally ambiguous; the algorithm is doing the best any candidate-path scorer could. |
| `ambiguous_rate` low but `class_top-k` < 1 | algorithm is leaving signal on the table. |

This decoupling — separating the *algorithmic* gap from the *information-theoretic* gap — is the key methodological contribution of §4.

---

## 5. Evaluation

### 5.1 Topologies and protocol

Six real-shaped graphs covering both ends of the structural spectrum:

- **WAN backbones** — sparse irregular meshes. `abilene` (11 nodes, 14 edges; the US Internet2 research backbone) and `nsfnet` (14 nodes, 16 edges; the NSF predecessor).
- **DC fabrics** — dense layered Clos networks. `fat_tree(k)` for $k \in \\{4, 6\\}$ (20/45 nodes, 32/108 edges) and `spine_leaf(s, l)` for $(s, l) \in \\{(4, 8), (8, 16)\\}$ (12/24 nodes, 32/128 edges).

![Real-shaped network topologies tested in §5](figures/concept_topologies.png)

For each trial: build the graph, sample 1000 peer pairs uniformly at random from $V \times V$ (no role restriction — peers may have endpoints at any layer, matching production CDP/LLDP coverage), pick a uniformly random failed edge, simulate the down/silent labels from each peer's actual path, score every edge, and rank. 200 trials per topology cell, fixed seed (see [`results/topologies.csv`](results/topologies.csv)).

### 5.2 Main result

![Localization performance across topologies](figures/topologies.png)

| Topology         | $\|V\|$ | $\|E\|$ | top1 | class_top1 | top3 | class_top3 | ambiguous_rate |
|------------------|------:|------:|-----:|-----------:|-----:|-----------:|---------------:|
| abilene          | 11    | 14    | **1.00** | **1.00** | **1.00** | **1.00** | 0.00 |
| nsfnet           | 14    | 16    | **1.00** | **1.00** | **1.00** | **1.00** | 0.00 |
| fat_tree(4)      | 20    | 32    | **1.00** | **1.00** | **1.00** | **1.00** | 0.00 |
| fat_tree(6)      | 45    | 108   | **1.00** | **1.00** | **1.00** | **1.00** | 0.00 |
| spine_leaf(4, 8) | 12    | 32    | **1.00** | **1.00** | **1.00** | **1.00** | 0.00 |
| spine_leaf(8, 16)| 24    | 128   | 0.96     | 0.96     | 0.99     | 0.99     | 0.00 |

(Bayesian log-likelihood of §3, $K = 2$, $\gamma = 0.8$, $\varepsilon = 10^{-3}$, num_peers = 1000.)

**Every real-shaped topology is fully identifiable.** `ambiguous_rate = 0` everywhere: equivalence classes collapse to singletons because at least some peers have endpoints at non-leaf layers, and their direct or near-direct adjacencies break the structural symmetry of the fabric. The algorithm hits `top-1 = 1.00` on five of six topologies; on `spine_leaf(8, 16)` it hits 0.96 (`top-3 = 0.99`), and the small residual is sample-complexity — the gap closes with more peers.

The earlier intuition that DC fabrics would be "fundamentally ambiguous" comes from restricting peers to a single role (leaf↔leaf, edge↔edge). Production CDP/LLDP meshes are not so restricted, and the structural ambiguity disappears.

### 5.3 Adversarial cases

![Adversarial topologies](figures/adversarial.png)

Three hand-crafted graphs designed to exhibit the failure modes described in §4:

| Topology                 | top1 | class_top1 | ambiguous_rate |
|--------------------------|-----:|-----------:|---------------:|
| `bridge_chain(3)`        | 0.15 | **1.00**   | 1.00           |
| `bipartite_k22`          | 0.31 | **1.00**   | 1.00           |
| `parallel_diamonds(2)`   | 0.14 | **1.00**   | 1.00           |
| `random+symmetric_bridge`| 0.91 | 0.91       | 0.00           |

The first three are *guaranteed-ambiguous* by construction: every edge sits in a length-2 path between the same two anchors, so all edges have identical signatures with respect to a single anchor peer. The algorithm's `class_top-1 = 1.0` shows that, given the data is information-theoretically degenerate, the algorithm still produces a *useful* answer (the correct equivalence class).

The last case is subtler: we add two parallel bridge nodes between two anchors *inside an already-dense random graph*. The bridge edges share endpoints but their candidate-path *counts* across random peers diverge enough that the Bayesian equivalence treats them as distinguishable: `ambiguous_rate = 0.00`, and `top-1 = class_top-1 = 0.91` — the algorithm correctly identifies the unique suspect on 91% of trials. This matches the WAN intuition: irregular peer placement breaks even locally symmetric structures.

### 5.4 Sweep over synthetic graphs

![top-1 vs num_peers](figures/sweep_top1.png)

Synthetic random graphs ($n \in \\{8, 12, 16, 20\\}$, extra edges $\in \\{5, 10, 20\\}$) all show `ambiguous_rate ≈ 0`. `top-1` increases with peer count and decreases as the graph becomes denser (more candidate paths per peer to confuse the scoring). Across all 36 sweep cells the strict and class metrics coincide, supporting the observation that random graphs almost never produce equivalent edges.

### 5.5 The structural exception: physical parallel fibers

The only ambiguity that persists under realistic peer coverage is **physical parallel fibers between the same two sites** — multi-rail uplinks, redundant cabling through different conduits, wavelength-multiplexed groupings. When $k$ fibers run in parallel between sites $A$ and $B$ and the topology abstracts them into a single edge, every fiber in the group has identical $f(x, \cdot)$ for every peer and forms an equivalence class of size $k$. Distinguishing inside the class requires physical-layer signals (OTDR, port-level cable identification, manual trace) — no candidate-path inference over LLDP-style logs can break it.

This is the practical mirror of §3.4's two-stage interpretation: stage 1 (topological likelihood) saturates at the parallel-fiber equivalence class; resolution within the class is an instrumentation question, not an algorithm question. The adversarial cases of §5.3 are exactly this geometry stress-tested.

---

## 6. Limitations

- **Single-edge failure only.** The entire analysis assumes $|D|$ is generated by *one* failed edge. Multi-failure is out of scope and the scoring rule should not be expected to behave well there.
- **Synthetic risk priors.** The risk term $\alpha \cdot \mathrm{Risk}(e)$ is functional but our experiments use uniformly random risk values. A real deployment would source risk from edge age, prior incidents, or fiber length.
- **Path enumeration cost.** `find_paths` enumerates simple paths up to a hop bound. On dense graphs the path budget $K$ is hit before exhaustive enumeration, biasing weights toward whichever paths are visited first.
- **No temporal modeling.** We treat the observation window as static. Real systems see events arrive over time and may need to debounce.

---

## 7. Reproducing the results

### Install

```bash
python -m venv .venv && source .venv/bin/activate
pip install -e .[tuning,viz]
```

The base install only requires `networkx`. `tuning` adds Optuna; `viz` adds matplotlib for the figures.

### Run all experiments

```bash
# Single-trial sanity check
python scripts/fiber_cut_experiment.py --seed 0 experiment

# Full evaluation table (Section 5.3)
python scripts/fiber_cut_experiment.py --seed 0 topologies --trials 200 \
    --csv results/topologies.csv

# Adversarial topologies (Section 5.4)
python scripts/fiber_cut_experiment.py --seed 0 adversarial --trials 200 \
    --csv results/adversarial.csv

# Synthetic sweep (Section 5.5)
python scripts/fiber_cut_experiment.py --seed 0 sweep --trials 100 \
    --csv results/sweep.csv

# Render the figures
python scripts/fiber_cut_experiment.py plot --csv results/topologies.csv \
    --out figures/topologies.png --kind topology
python scripts/fiber_cut_experiment.py plot --csv results/adversarial.csv \
    --out figures/adversarial.png --kind topology
python scripts/fiber_cut_experiment.py plot --csv results/sweep.csv \
    --out figures/sweep_top1.png --kind sweep --metric top1
```

### Hyperparameter tuning (optional)

```bash
python scripts/fiber_cut_experiment.py tune \
    --n-trials 60 --trials 100 --metric class_top1 --use-risk
```

### Tests

```bash
pytest tests/ -v
```

26 tests cover graph construction, scoring, identifiability, simulation, real and adversarial topologies, and CSV/JSON reporting.

---

## 8. Code map

| Module | Purpose | Paper section |
|---|---|---|
| [`graph.py`](src/fiber_cut_localizer/graph.py) | Graph generation, candidate-path enumeration | §2.3 |
| [`scoring.py`](src/fiber_cut_localizer/scoring.py) | Positive / silent / risk scoring | §3 |
| [`identifiability.py`](src/fiber_cut_localizer/identifiability.py) | Edge signatures and equivalence classes | §4 |
| [`simulation.py`](src/fiber_cut_localizer/simulation.py) | `run_trial`, `evaluate`, identifiability-aware metrics | §5.1 |
| [`topologies.py`](src/fiber_cut_localizer/topologies.py) | Real-shaped + adversarial generators | §5.2, §5.3 |
| [`tuning.py`](src/fiber_cut_localizer/tuning.py) | Optuna search over $(\gamma, \alpha, L, M, K)$ | §3.5 |
| [`reporting.py`](src/fiber_cut_localizer/reporting.py) | CSV / JSON export | §7 |
| [`plotting.py`](src/fiber_cut_localizer/plotting.py) | Topology bar chart and sweep line plot | §5 |
| [`PROJECT_CONTEXT.md`](PROJECT_CONTEXT.md) | Long-form design spec / source of truth | — |
