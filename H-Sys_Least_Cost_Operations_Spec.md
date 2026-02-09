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

The "least cost" is not benchmark-relative. It is the optimum of a pinned contract:

```
C* = min_{legal staffing + real-time recourse}
     E[Operator wages + Leader wages + OT wages + SLA penalties]
```

If infeasible: output **UNSAT** with a minimal deficit witness.

---

## 1) Contract: indices, inputs, and the one cost formula

### 1.1 Indices
- Time buckets `t ∈ T = {1, ..., T}`, bucket length `Δ` hours (typically 15min or 1hr).
- Inquiry types / skills `k ∈ K` (product families, store zones, languages, etc.).
- Shift templates `s ∈ S` (start, duration, break placement, legal compliance).

### 1.2 Inputs

**Demand**
- `A_{t,k}`: realized inquiries.
- Forecast provides a contract (Section 2): either quantiles `d_{t,k}` or intensity bounds `λ̄_{t,k}`.

**Service**
- `μ_k`: service rate (inquiries/hour/operator) (or mean handle time `1/μ_k`).

**Costs**
- Operator wage `w_k` ($/hour) for skill k (or by level).
- OT wage `w^{OT}_k`.
- Leader wage `w_L`.
- Optional: hiring, training, attrition costs for long-term.

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

Two acceptable contracts:

### 2.1 Quantile (service guarantee) contract

For each (t,k):
```
Pr(A_{t,k} ≤ d_{t,k}) ≥ 1 - ε_{t,k}
```
Planning uses `d_{t,k}` as robust demand.

### 2.2 Intensity interval contract

For each (t,k):
```
λ_{t,k} ∈ [λ̲_{t,k}, λ̄_{t,k}],    A_{t,k} ≤ λ̄_{t,k} · Δ  (robust bound)
```

**Forecast verifier (mandatory in demo):**
- calibration check: empirical coverage ≥ `1 - ε`,
- plus hashes of dataset/model.

If calibration fails: output **Ω** ("forecast contract not satisfied").

---

## 3) Service constraints: RT and IT (makes the objective real)

We need deterministic constraints linking `n_{t,k}` to RT and IT.

### 3.1 RT (resolution time) via queue physics

Define utilization:
```
ρ_{t,k} = λ_{t,k} / (μ_k · n_{t,k})
```
Require `ρ_{t,k} < 1` (stability).

For an M/M/n approximation (standard staffing math), compute Erlang-C waiting time `W_{q,t,k}`, then:
```
RT_{t,k} = W_{q,t,k} + 1/μ_k
```

**SLA constraint:**
```
RT_{t,k} ≤ RT^{max}_k
```
(or tail form `Pr(W_q > w) ≤ ε`, also deterministic for M/M/n).

Robustify by using `d_{t,k}` or `λ̄_{t,k}`.

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

Enforce either:
- **hard idle cap:** `IdleFrac_{t,k} ≤ IT^{maxFrac}_k`, or
- **soft penalty:** add `β · Σ_{t,k} IT_{t,k}` to the objective.

---

## 4) Supply planning (solves part b) — the least cost is now well-defined

The planning problem is:

```
┌───────────────────────────────────────────┐
│  C*_plan = min_{y,o,L}  C(y,o,L)         │
└───────────────────────────────────────────┘
```

subject to:
- coverage equation for `n_{t,k}`,
- RT constraints (queue surrogate),
- IT constraint or penalty,
- labor-law constraints encoded in shift template set `S`,
- leader constraint `L_t ≥ η · Σ_k n_{t,k}`,
- any skill eligibility constraints.

This is **NP-hard** because shift selection is a form of set cover with integer counts. But now "least cost" is a number: the optimum of this pinned problem.

---

## 5) Real-time optimizer (solves part c) — recourse minimization

Planning is done ahead of time; real arrivals `A_{t,k}` differ. Real-time must minimize OT + SLA penalties, given current state.

### 5.1 Backlog dynamics (ledger state)

```
Q_{t+1,k} = max{0, Q_{t,k} + A_{t,k} - S_{t,k}},    S_{t,k} ≤ μ_k · n_{t,k} · Δ
```

### 5.2 Real-time controls
- dispatch/routing of multi-skill operators across queues,
- overtime requests `o_{t,k}`,
- optional: priority rules.

### 5.3 Recourse objective

At each time window (MPC):
```
min_{o, dispatch}  Σ_{t,k} w^{OT}_k · o_{t,k} · Δ  +  α · (RT violations)  +  β · (idle)
```
subject to backlog evolution and feasibility.

This is the deterministic real-time "match demand with supply" optimizer.

---

## 6) The unified least-cost definition (planning + real-time)

The complete least cost to run H-Sys is:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  C* = min_{y,L} [ Σ_{s,k} w_k · y_{s,k} · hours(s)                       │
│                  + Σ_t w_L · L_t · Δ                                       │
│                  + E_A  min_{o(·), dispatch(·)}  Φ(y,L; A,o,dispatch) ]    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

`Φ` includes overtime costs + SLA penalties as defined above.

**This is the exact end-to-end mathematical object for "least cost".**

---

## 7) Proof outputs (what to show in the demo)

For any given instance (store/day), output only:

### UNIQUE
- schedule `y_{s,k}`, leaders `L_t`,
- real-time policy / MPC outputs `o_{t,k}`, dispatch,
- computed RT/IT per bucket,
- total cost `C`,
- optimality proof: best incumbent = best bound (gap ≤ tolerance),
- replay receipts.

### UNSAT
- minimal deficit witness showing infeasibility (e.g., labor-hour deficit on a cut).

A canonical witness:
```
D(U,K') = Σ_{t∈U} Σ_{k∈K'} d_{t,k} / μ_k

S(U,K') = Σ_{t∈U} Σ_{k∈K'} n_{t,k} · Δ

┌──────────────────────────────────────────────┐
│  δ(U,K') = D(U,K') - S(U,K') > 0           │
└──────────────────────────────────────────────┘
```

Meaning: "you are short δ labor-hours; no schedule can meet the SLA."

### Ω
- best feasible cost, best lower bound, exact gap,
- what additional compute/capacity would close it.

---

## 8) Metrics Amazon will see (and how they're computed)

From UNIQUE outputs you compute:

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

### 9.1 Inputs
- 2-4 weeks of real inquiry logs per store: timestamps, type k, service time.
- operator roster constraints and wage tables.
- leader span-of-control policy.
- SLA targets for RT/IT.

### 9.2 Build
1. Calibrate forecast contract `d_{t,k}` (quantiles) and verify coverage.
2. Solve planning problem for each day:
   - output UNIQUE/UNSAT/Ω + receipts.
3. Run real-time MPC on replay of real arrivals `A_{t,k}`:
   - choose OT and dispatch.
4. Compare to baseline:
   - cost, RT tails, idle, OT, leader hours.

### 9.3 What makes it undeniable
- You can replay a day and reproduce identical results.
- If SLA can't be met, you produce a deficit certificate showing "how many extra people/hours are required" rather than failing silently.
- "Least cost" becomes a certified number.

---

## 10) Why no one can optimize better

Once the contract is pinned (forecast uncertainty, queue model, labor laws, wages, SLAs), "better" means lower cost while satisfying the same constraints. That is exactly what `C*` is: the minimum over the admissible set.

- If another system outputs a lower cost, it must violate a constraint (not the same problem), or it is wrong.
- If the problem is infeasible, the minimal deficit witness proves no one can satisfy it without adding capacity or relaxing SLA.

**That is the complete resolution.**
