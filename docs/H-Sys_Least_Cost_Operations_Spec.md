# H-Sys: Least-Cost Operations Under RT/IT Targets

**Complete math + verifiable demo**

---

## 0) What H-Sys is, as a single optimization object

H-Sys is a stochastic service system with:
- random arrivals (inquiries) by type,
- staffed servers (operators) with skills,
- queueing delay (RT),
- idle capacity (IT),
- legal shift constraints,
- overtime recourse.

The "least cost" is not benchmark-relative. It is the optimum of a day-ahead planning problem given fixed headcount caps and pinned SLA physics:

```
C* = min_{legal staffing}  [Operator wages + Leader wages]
```

Real-time overtime recourse is a separate execution report (Section 5), not part of `C*`.

If infeasible: output **UNSAT** with a minimal deficit witness.

### 0.1 Output meanings (pinned)

- **UNIQUE-COST**: the scalar optimum `C*` is unique (a number).
- **UNIQUE-PLAN**: a single canonical plan is selected among potentially many cost-optimal plans by a deterministic tie-break chain (Section 4.4).
- **UNSAT**: infeasible under the pinned contract; output a deterministic deficit witness (human-readable) plus an IIS artifact.
- **Ω**: solver budget-limited; output `[LB, UB]` and exact gap.

Contract choice: We promise **UNIQUE-PLAN** in the demo, not just UNIQUE-COST.

---

## 1) Contract: indices, inputs, and the one cost formula

### 1.1 Indices
- Time buckets `t ∈ T = {1, ..., T}`, bucket length `Δ` hours (typically 15min or 1hr).
- Inquiry types / skills `k ∈ K` (product families, store zones, languages, etc.).
- Shift templates `s ∈ S` (start, duration, break placement, legal compliance).

### 1.2 Inputs

**Demand**
- `A_{t,k}`: realized inquiries.
- Forecast provides a contract (Section 2): quantiles `d_{t,k}` or intensity bounds `λ̄_{t,k}` (see Section 2).

**Service**
- `μ_k`: service rate (inquiries/hour/operator) (or mean handle time `1/μ_k`).

**Costs**
- Operator wage `w_k` ($/hour) for skill k (or by level).
- OT wage `w^{OT}_k`.
- Leader wage `w_L`.
- Optional: hiring, training, attrition costs for long-term.

**Caps**
- Headcount cap `H^{cap}_k` per skill type (required for daily planning; hiring decisions are a separate outer-loop problem).
- Overtime cap `o^{cap}_{t,k}` per bucket/type (optional).
- Leader cap `L^{cap}_t` per bucket (optional).

**Leadership**
- Span-of-control parameter `η` so leaders required:

```
L_t ≥ η · Σ_k n_{t,k}
```

### 1.3 Decision variables

**Shift staffing:**
- `y_{s,k} ∈ Z≥0`: number of skill-k operators assigned shift s.

**Overtime/flex:**
- `o_{t,k} ∈ Z≥0`.

**Effective staffing:**
```
n_{t,k} = Σ_{s∈S} y_{s,k} · 1[t ∈ cov(s)] + o_{t,k}
```

**Leaders:**
- `L_t ∈ Z≥0`.

### 1.4 The exact operational cost expression

The operational cost (operators + leaders + OT) is:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  C(y,o,L) = Σ_{s,k} w_k · y_{s,k} · hours(s)                         │
│           + Σ_{t,k} w^{OT}_k · o_{t,k} · Δ                           │
│           + Σ_t     w_L · L_t · Δ                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**This is the mathematical expression to calculate H-Sys operating cost.**

---

## 2) Forecasting as a contract, not a guess (solves part a)

Forecast output must be a **witnessable constraint object** that planning can satisfy.

We use the **quantile contract (Section 2.1)**. The intensity interval contract (Section 2.2) is an alternative documented for reference but not used.

### 2.1 Quantile (service guarantee) contract

For each (t,k):
```
Pr(A_{t,k} ≤ d_{t,k}) ≥ 1 - ε^{dem}_{t,k}
```
Planning uses `d_{t,k}` as robust demand.

Then robust arrival rate is:
```
┌────────────────────────────────────────┐
│  λ_{t,k}^{rob} := d_{t,k} / Δ        │
└────────────────────────────────────────┘
```

### 2.2 Intensity interval contract (alternative)

For each (t,k):
```
λ_{t,k} ∈ [λ̲_{t,k}, λ̄_{t,k}],    A_{t,k} ≤ λ̄_{t,k} · Δ  (robust bound)
```

If used, `λ_{t,k}^{rob} := λ̄_{t,k}`.

**Forecast verifier (mandatory in demo):**
- Calibration check: empirical coverage on held-out days ≥ `1 - ε^{dem}_{t,k}`.
- Plus hashes of dataset/model.

**Forecast failure fallback (deterministic):**

If calibration fails, the pipeline does **not** output Ω and stop. Instead, it replaces the model forecast with a conservative empirical fallback:

```
┌────────────────────────────────────────────────────────────────────────────┐
│  d^{fallback}_{t,k} := max( d^{model}_{t,k},  Q^{emp}_{1-ε}(A_{t,k}) ) │
└────────────────────────────────────────────────────────────────────────────┘
```

where `Q^{emp}_{1-ε}(A_{t,k})` is the empirical quantile from the last `D` days of actual arrival logs (same `D` and winsorization rule as the `μ_k` procedure, Section 3.0.3).

Planning then proceeds with `λ_{t,k}^{rob} := d^{fallback}_{t,k} / Δ`. This makes the demand contract more conservative (possibly increasing staffing cost), which is correct. Forecast failure never becomes a way to dodge the least-cost computation.

---

## 3) Service constraints: RT and IT (makes the objective real)

We need deterministic constraints linking `n_{t,k}` to RT and IT.

### 3.0 Pinned contract (removes all mode ambiguity)

To make "least cost" a single unambiguous number, we commit to **one** contract:

**Contract C (pinned):**

- **RT** is a hard tail-SLA per bucket/type:
```
Pr(W_{q,t,k} > w_k^{max}) ≤ ε_k
```
with pinned `w_k^{max}` and `ε_k`.

- **IT** is not a penalty weight; it is resolved by a **canonical tie-break chain** after cost is minimized (Section 4.4). The tie-break guarantees a single canonical plan (**UNIQUE-PLAN**), not just a unique cost.

So "least cost" is a single, unambiguous number `C*`, and the emitted schedule is a single, deterministic plan.

#### 3.0.1 Pinned SLA physics model

The RT constraint is compiled into a staffing requirement (Section 3.1.2) under the following declared physics model. These are **contract assumptions**, not implementation details — they are the official SLA physics under which the optimizer certifies feasibility:

- **Bucket size:** `Δ` (pinned, e.g., 15 min or 1 hr).
- **Arrival process:** Poisson with rate `λ_{t,k}` within each bucket (from forecast contract, Section 2).
- **Service times:** Exponential with mean `1/μ_k`.
- **Server model:** Dedicated servers per queue type `k` (see Section 3.0.2).

The resulting queue model is **M/M/n per (t,k)**.

Monotonicity of `N_req` is a theorem under this model: as `n` increases, waiting probability and decay rate improve; the LHS decreases for `n` with `ρ < 1`. This is standard and can be verified numerically for the tabulated domain.

#### 3.0.2 Dedicated skill pools

Each operator belongs to exactly one skill type `k` during a bucket. So the per-k queue model is correct and `N_req(λ; k)` is separable per type.

**Operational mapping of `k`:** `k` corresponds to the actual workforce segmentation used in current planning. Typical definitions:

| `k` | Example |
|---|---|
| Language pool | English, Spanish, Japanese operators |
| Store zone pool | Fresh, Electronics, General Merchandise |
| Inquiry family | Product identification, checkout review, theft review |
| Region/compliance pool | US, EU, JP (different labor rules) |

If planning already staffs these pools separately — even if dispatch later does minor cross-skill routing — then the dedicated-pool model is the correct abstraction of how staffing decisions are actually made.

If multi-skill pooling is introduced, the per-k model generalizes as follows. Let pools be `p ∈ P`, each pool serves a set of types `K(p)`. Let effective arrivals:

```
λ_{t,p}^{rob} = Σ_{k∈K(p)} λ_{t,k}^{rob}
```

and effective service rate `μ_p` (or compute from mix), then:

```
n_{t,p} ≥ N_req(λ_{t,p}^{rob}; p)
```

plus routing constraints as a bipartite flow each bucket.

#### 3.0.3 Service rate measurement procedure

`μ_k` is not a tunable parameter — it is a deterministic function of historical logs. This matters because `N_req` depends on `μ_k`; if `μ_k` is unstable, so is feasibility.

For each type `k`, collect handle times `h_i` over the last `D` days (e.g., `D = 14`):

1. Winsorize at `[p05, p95]` to remove outliers.
2. Take the median handle time `m_k`.
3. Define:

```
┌──────────────────────┐
│  μ_k = 1 / m_k      │
└──────────────────────┘
```

**Verifier:** the `μ_k` bundle includes hashes of the raw handle-time samples and the winsorization rule.

### 3.1 RT (resolution time) via queue physics

Define utilization:
```
ρ_{t,k} = λ_{t,k} / (μ_k · n_{t,k})
```
Require `ρ_{t,k} < 1` (stability).

For the pinned M/M/n model (Section 3.0.1), compute Erlang-C waiting time `W_{q,t,k}`, then:
```
RT_{t,k} = W_{q,t,k} + 1/μ_k
```

**SLA constraint (pinned as tail-SLA):**
```
Pr(W_{q,t,k} > w_k^{max}) ≤ ε_k
```

Robustify by using `d_{t,k}` from the pinned quantile forecast contract (Section 2.1).

#### 3.1.1 Queue model (official SLA physics, not an approximation)

For each type `k`, service rate `μ_k`. For bucket `t`, arrival rate `λ_{t,k}` (from forecast contract).

Define `n` servers and utilization `ρ = λ/(nμ)`. For M/M/n, the waiting tail is:

```
Pr(W_q > w) = P^{wait}(λ, μ, n) · e^{-(nμ - λ)w}
```

where `P^{wait}` is Erlang-C (standard formula).

The SLA is:

```
P^{wait}(λ, μ, n) · e^{-(nμ - λ)w_k^{max}} ≤ ε_k
```

This is the **declared physics model** under which RT feasibility is defined. It is not an approximation to be debated — it is the contract under which the optimizer certifies feasibility.

#### 3.1.2 Compiling RT to a solver-ready staffing requirement (N_req)

The Erlang-C tail-SLA is nonlinear, but it is **monotone in `n`**, so it can be compiled into an integer coverage requirement per bucket/type.

Define the minimal required integer staffing:

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  N_req(λ; k) := min{ n ∈ Z≥0 :                                                 │
│      P^{wait}(λ, μ_k, n) · e^{-(nμ_k - λ)w_k^{max}} ≤ ε_k }                  │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

This exists and is unique because the LHS decreases with `n` once stable (theorem under pinned M/M/n model, Section 3.0.1).

**Robust demand:**

From the pinned quantile forecast contract (Section 2.1), convert to rate:

```
λ_{t,k}^{rob} := d_{t,k} / Δ
```

Then the bucket requirement is:

```
┌──────────────────────────────────────────────────────┐
│  n_{t,k} ≥ N_req(λ_{t,k}^{rob}; k)    ∀ t, k      │
└──────────────────────────────────────────────────────┘
```

**This is the key normalization:** RT feasibility is reduced to a precomputed monotone staffing curve. No nonlinear constraints remain in the optimizer.

**Implementation note:** `N_req` is computed once and hashed; the model and parameters are pinned in the contract so the curve is not arbitrary. Reproducibility is guaranteed by the hash.

### 3.2 IT (idle time) from utilization

Expected busy hours in bucket:
```
busy_{t,k} = λ_{t,k} · Δ / μ_k
```

Paid capacity:
```
cap_{t,k} = n_{t,k} · Δ
```

Idle hours:
```
IT_{t,k} = n_{t,k} · Δ - λ_{t,k} · Δ / μ_k
```

Idle fraction:
```
IdleFrac_{t,k} = 1 - ρ_{t,k}
```

**Pinned IT treatment:** Per the pinned contract (Section 3.0), IT is **not** an objective penalty or a hard cap. It is the first stage of a **canonical tie-break chain** (Section 4.4): among all schedules achieving the minimum cost `C*`, we select the one that minimizes total idle hours `Σ_{t,k} IT_{t,k}`, then further tie-break on subsequent criteria. This removes all "penalty weight" ambiguity.

---

## 4) Supply planning (solves part b) — the least cost is now well-defined

The planning problem is a **MILP** (mixed-integer linear program). "Least cost" is the optimum of this pinned problem.

### 4.1 Variables

- `y_{s,k} ∈ Z≥0`: number of skill-k operators assigned to shift template `s`.
- `o_{t,k} ∈ Z≥0`: overtime/flex operators in bucket `(t,k)`.
- `n_{t,k} ∈ Z≥0`: total staff in bucket `(t,k)`.
- `L_t ∈ Z≥0`: leaders in bucket `t`.

### 4.2 Constraints

**Coverage:**
```
n_{t,k} = Σ_{s∈S} y_{s,k} · 1[t ∈ cov(s)] + o_{t,k}
```

**RT (solver-ready):**
```
n_{t,k} ≥ N_req(λ_{t,k}^{rob}; k)    ∀ t, k
```

**Leaders:**
```
L_t ≥ η · Σ_k n_{t,k}
```

**Caps:**
```
Σ_s y_{s,k} ≤ H^{cap}_k             (headcount cap per skill, required)
o_{t,k} ≤ o^{cap}_{t,k}             (overtime cap per bucket/type, if applicable)
L_t ≤ L^{cap}_t                      (leader cap per bucket, if applicable)
```

All labor laws are already encoded by which `s ∈ S` exist (and any additional linear constraints on `y`).

Any skill eligibility constraints apply as further linear restrictions.

### 4.3 Cost (exact expression)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  C(y,o,L) = Σ_{s,k} w_k · y_{s,k} · hours(s)                         │
│           + Σ_{t,k} w^{OT}_k · o_{t,k} · Δ                           │
│           + Σ_t     w_L · L_t · Δ                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.4 Objective (pinned, no penalties) and canonical plan selection

**Stage 1 (least cost, day-ahead with no planned overtime):**

```
┌──────────────────────────────────────────────────────────────────────────┐
│  C* = min_{y,L}  Σ_{s,k} w_k · y_{s,k} · hours(s) + Σ_t w_L · L_t · Δ│
│       s.t. all constraints above, o_{t,k} = 0  ∀ t, k                   │
└──────────────────────────────────────────────────────────────────────────┘
```

The roster is built to satisfy `N_req` via shift staffing alone, without pre-planned overtime. Overtime arises only in real-time recourse (Section 5) when realized demand deviates from forecast.

**Stages 2–6 (canonical tie-break chain for UNIQUE-PLAN):**

After finding minimal cost `C*`, let `F(C*)` be the feasible set with cost fixed: `C(y,o,L) = C*`.

Solve successive refinement problems in order:

1. **Min idle hours** (computed under the same demand contract):
```
J_1 = Σ_{t,k} ( n_{t,k} · Δ - (λ_{t,k}^{rob} · Δ) / μ_k )
```

2. **Min overtime hours:**
```
J_2 = Σ_{t,k} o_{t,k} · Δ
```

3. **Min total headcount-shift count:**
```
J_3 = Σ_{s,k} y_{s,k}
```

4. **Min leader-hours:**
```
J_4 = Σ_t L_t · Δ
```

5. **Canonical lexicographic hash order** over the integer vector `(y, o, L)` (smallest lexicographic ordering of a fixed serialization of variables).

The canonical plan is:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  (y*, o*, L*) = argmin_{(y,o,L) ∈ F(C*)}  (J_1, J_2, J_3, J_4, lex-hash) │
└──────────────────────────────────────────────────────────────────────────────┘
```

This is fully pinned: there is exactly one output plan. Demo output label is **UNIQUE-PLAN** (with `C*` and tie-break receipts).

This is **NP-hard** because shift selection is a form of set cover with integer counts. But now "least cost" is a number and the plan is a single deterministic object: the optimum of this pinned problem.

---

## 5) Real-time optimizer (solves part c) — recourse minimization

Planning is done ahead of time; real arrivals `A_{t,k}` differ. The real-time optimizer is a separate execution stage — its OT cost is not part of the day-ahead `C*` but is reported alongside it for full transparency.

### 5.1 Backlog dynamics (ledger state)

```
Q_{t+1,k} = max{0, Q_{t,k} + A_{t,k} - S_{t,k}},    S_{t,k} ≤ μ_k · n_{t,k} · Δ
```

### 5.2 Real-time controls
- dispatch within each pool `k` (cross-skill routing across pools is described in the pooling extension, Section 3.0.2),
- overtime requests `o_{t,k}`,
- optional: priority rules within each pool.

### 5.3 Recourse in the same contract normal form (N_req)

At runtime, observe `A_{t,k}`. Maintain backlog `Q_{t,k}` and choose overtime `o_{t,k}` and dispatch to keep the same RT contract, but now with realized demand.

Use the same compiled requirement:

- Compute a short-horizon realized rate `λ̂_{t,k}` from arrivals/backlog.
- Compute `N_req(λ̂_{t,k}; k)`.
- Enforce `n_{t,k} ≥ N_req` by adding overtime minimally.

**Real-time decision per bucket:**

```
min_{o_{t,k}}  Σ_k  w^{OT}_k · o_{t,k} · Δ

s.t.   n_{t,k}^{planned} + o_{t,k}  ≥  N_req(λ̂_{t,k}; k)    ∀ k
```

If infeasible (caps), output **UNSAT** with the same deficit witness (Section 7).

---

## 6) The unified least-cost definition

### 6.1 C* is the day-ahead planning MILP

`C*` is the optimum of the day-ahead planning MILP (Section 4):

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  C* = min_{y,L}  Σ_{s,k} w_k · y_{s,k} · hours(s)                    │
│                + Σ_t     w_L · L_t · Δ                                 │
│       s.t. all constraints (Section 4.2), o_{t,k} = 0                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

The roster is built to satisfy `N_req` under forecast demand, without pre-planned overtime.

### 6.2 Real-time OT is a separate execution report

The real-time optimizer (Section 5) adds minimal overtime when actual arrivals exceed the forecast contract. The full cost picture for a given day is:

```
C_total = C* (day-ahead roster cost)  +  C_OT (real-time overtime cost)
```

`C_OT` is observable only after replay or live execution. It is not embedded in the planning objective.

### 6.3 Closure

Under the pinned contract (Section 3.0), all structural gaps are closed:

1. **RT constraint is computable:** it is a precomputed monotone staffing curve `N_req` and a linear constraint `n ≥ N_req`, under a declared M/M/n physics model (Section 3.0.1). No nonlinear constraints remain in the optimizer.
2. **UNSAT witness matches feasibility:** infeasibility is about failing `n ≥ N_req`, and IIS/deficits certify exactly that (Section 7). The witness is keyed to the same `N_req` the optimizer enforces.
3. **Least cost is unambiguous:** single cost objective `C*`, plus a canonical tie-break chain (Section 4.4) that selects a single deterministic plan (**UNIQUE-PLAN**). No penalty weights, no mode selection.
4. **Demand contract is chosen:** quantile contract, so `λ^{rob}` is unambiguous.
5. **Skill model:** dedicated pools per type `k`, so per-k `N_req` is correct.
6. **Service rate `μ_k`:** deterministic measurement procedure (Section 3.0.3), so `N_req` is stable and reproducible.
7. **Headcount caps `H^{cap}_k`:** required, so UNSAT is meaningful. Hiring is a separate outer-loop problem.
8. **Forecast failure has a deterministic fallback** (Section 2): conservative empirical quantile replaces the model forecast. The pipeline never dodges the computation via Ω.

**One contract → one computable MILP → one canonical plan → one matching witness system.**

---

## 7) Proof outputs (what to show in the demo)

For any given instance (store/day), output only:

### UNIQUE-PLAN (only when MILP optimality gap ≤ tolerance)
- Canonical schedule `y*_{s,k}`, leaders `L*_t`.
- Total cost `C*`.
- Tie-break receipts: values of `(J_1, J_2, J_3, J_4)` from the canonical chain (Section 4.4).
- Computed RT/IT per bucket.
- Optimality proof: best incumbent = best bound (gap ≤ tolerance).
- Replay receipts.

### UNSAT

Because the RT SLA is now exactly the linear requirement `n_{t,k} ≥ N_req(λ_{t,k}^{rob}; k)`, infeasibility can be certified by a standard IIS (irreducible infeasible subset) from the MILP.

The witness is produced in two steps:

1. **IIS from MILP solver** (machine-readable, machine-minimal infeasible constraint set).
2. **Human-readable deficit witness** derived by projecting the IIS to buckets/types (below).

The human-readable witness is derived from the IIS — it is not an independently computed globally minimal window.

#### 7.1 Deriving the human-readable witness from IIS

The MILP solver produces an IIS: a minimal set of constraints that are jointly infeasible. Project the IIS constraints involving RT coverage (`n_{t,k} ≥ N_req`) onto their bucket/type indices to obtain a window `U ⊆ T` and type set `K' ⊆ K`.

Then report the deficit over that window:

```
D(U, K') = Σ_{t∈U} Σ_{k∈K'} N_req(λ_{t,k}^{rob}; k) · Δ
```

```
S(U, K') = maximum schedulable staffing-hours in the window given caps and templates
```

```
┌──────────────────────────────────────────────────────┐
│  δ(U, K') = D(U, K') - S(U, K') > 0                │
└──────────────────────────────────────────────────────┘
```

Interpretation: "You are short by δ operator-slots over this time window for these queues, even at maximum feasible staffing."

If leaders are capped (`L^{cap}_t`), leader feasibility can create additional infeasibility — the IIS will include the leader constraint and the witness will reflect it.

**This certifies infeasibility for the same RT constraint the optimizer enforces.**

### Ω (solver budget-limited, optimality not proven)
- Best feasible cost (upper bound), best lower bound, exact gap.
- The canonical incumbent plan (same tie-break chain applies to the best known feasible solution).
- What additional compute/capacity would close the gap.

---

## 8) Metrics Amazon will see (and how they're computed)

From UNIQUE-PLAN outputs you compute:

**OPEX**
```
OPEX = C(y,o,L)
```

**Idle reduction**
```
H_idle = Σ_{t,k} IT_{t,k},    IT% = H_idle / Σ_{t,k} n_{t,k} · Δ
```

**RT improvement**
Compute `RT_{t,k}` from queue model; report p50/p95/p99 across time.

**Overtime reduction**
```
H_OT = Σ_{t,k} o_{t,k} · Δ,    OT cost = Σ_{t,k} w^{OT}_k · o_{t,k} · Δ
```

**Leader right-sizing**
```
H_L = Σ_t L_t · Δ
```

**Savings**
If baseline cost is `C^{base}`:
```
ΔC = C^{base} - C*
```

---

## 9) Demo plan (engineering real, end-to-end)

### 9.1 Inputs (one day)
- 2-4 weeks of real inquiry logs per store: timestamps, type k, service time.
- Operator roster constraints and wage tables.
- Leader span-of-control policy.
- SLA targets for RT/IT.
- Pinned parameters: `Δ`, `w_k^{max}`, `ε_k`, `ε^{dem}_{t,k}`.
- Last `D` days of handle-time logs per type `k` → compute `μ_k` via measurement procedure (Section 3.0.3).
- Forecast contract: quantile `d_{t,k}` with calibration report.
- Shift template set `S` (labor-law compliant).
- Wages `w_k`, `w_k^{OT}`, `w_L`, leader span `η`.
- Headcount caps `H^{cap}_k` (required). Overtime/leader caps if applicable.

### 9.2 Build

**Step A: Precompute `N_req(λ; k)`**
- For each `k`, tabulate `N_req` over a grid of `λ` values using the pinned SLA physics (Section 3.0.1).
- Store the table and hash it (this is part of the contract).

**Step B: Solve planning MILP**
- Calibrate forecast contract `d_{t,k}` (quantiles) and verify coverage (`≥ 1 - ε^{dem}_{t,k}` on held-out days).
- Solve for each day: output `C*` and canonical plan `(y*, o*, L*)` via tie-break chain.
- Emit optimality gap certificate (MILP bound).
- Output UNIQUE-PLAN/UNSAT/Ω + receipts.

**Step C: Replay real-time**
- Replay historical arrival traces `A_{t,k}`, run minimal overtime recourse using `N_req`.
- Choose OT and dispatch.
- Report RT SLA pass rate and overtime used.

**Step D: Compare to baseline**
- Cost, RT tails, idle, OT, leader hours.

### 9.3 Outputs
- Day-ahead cost `C*` vs current benchmark.
- Real-time OT cost `C_OT` (from replay, reported separately).
- Canonical plan with tie-break receipts `(J_1, J_2, J_3, J_4)`.
- RT compliance (tail bound satisfied by construction via `N_req`).
- Idle hours (tie-break metric, not a tunable weight).
- UNSAT days: deficit witness `δ(U, K')` + IIS artifact.

### 9.4 What makes it undeniable
- You can replay a day and reproduce identical results.
- If SLA can't be met, you produce a deficit certificate showing "how many extra people/hours are required" rather than failing silently.
- "Least cost" becomes a certified number.
- The emitted plan is a single deterministic object (UNIQUE-PLAN), not one of many.
- RT SLA (p95/p99) is verified (since it's encoded by `N_req`).
- IT (idle) reported as tie-break result, not a tunable weight.
- No ambiguity remains because every object is pinned: physics model, demand contract, skill scope, tie-break chain.

---

## 10) Why no one can optimize better

Once the contract is pinned (forecast uncertainty, queue model, labor laws, wages, SLAs), "better" means lower cost while satisfying the same constraints. That is exactly what `C*` is: the minimum over the admissible set.

- If another system outputs a lower cost, it must violate a constraint (not the same problem), or it is wrong.
- If the problem is infeasible, the minimal deficit witness proves no one can satisfy it without adding capacity or relaxing SLA.

**That is the complete resolution.**
