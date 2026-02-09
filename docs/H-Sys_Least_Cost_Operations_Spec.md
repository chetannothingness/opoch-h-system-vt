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

### 3.0 Pinned contract (removes all mode ambiguity)

To make "least cost" a single unambiguous number, we commit to **one** contract:

**Contract C (pinned):**

- **RT** is a hard tail-SLA per bucket/type:
```
Pr(W_{q,t,k} > w_k^{max}) ≤ ε_k
```

- **IT** is not a penalty weight; it is a **lexicographic tie-breaker** after cost is minimized:
  1. Minimize total wage cost.
  2. Among cost-optimal schedules, minimize idle hours (equivalently, maximize utilization).

So "least cost" is a single, unambiguous number `C*`.

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

#### 3.1.1 Queue model (used only to generate requirements)

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

This exists and is unique because the LHS decreases with `n` once stable.

**Robust demand:**

If forecast provides a quantile `d_{t,k}` for arrivals per bucket, convert to rate:

```
λ_{t,k}^{rob} := d_{t,k} / Δ
```

If forecast provides an interval `[λ̲_{t,k}, λ̄_{t,k}]`, set:

```
λ_{t,k}^{rob} := λ̄_{t,k}
```

Then the bucket requirement is:

```
┌──────────────────────────────────────────────────────┐
│  n_{t,k} ≥ N_req(λ_{t,k}^{rob}; k)    ∀ t, k      │
└──────────────────────────────────────────────────────┘
```

**This is the key normalization:** RT feasibility is reduced to a precomputed monotone staffing curve. No nonlinear constraints remain in the optimizer.

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

**Pinned IT treatment:** Per the pinned contract (Section 3.0), IT is **not** an objective penalty or a hard cap. It is a **lexicographic tie-breaker**: among all schedules achieving the minimum cost `C*`, we select the one that minimizes total idle hours `Σ_{t,k} IT_{t,k}`. This removes all "penalty weight" ambiguity.

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

### 4.4 Objective (pinned, no penalties)

**Stage 1 (least cost):**

```
┌──────────────────────────────────────────────────────────────┐
│  C* = min_{y,o,L}  C(y,o,L)   s.t. all constraints above   │
└──────────────────────────────────────────────────────────────┘
```

**Stage 2 (tie-break IT, only among cost-optimal schedules):**

Define idle hours using robust demand:

```
IT_{t,k} = n_{t,k} · Δ - (λ_{t,k}^{rob} · Δ) / μ_k
```

Then:

```
min  Σ_{t,k} IT_{t,k}   s.t. all constraints and C(y,o,L) = C*
```

This removes all "weights" ambiguity and makes "least cost" unique and review-proof.

This is **NP-hard** because shift selection is a form of set cover with integer counts. But now "least cost" is a number: the optimum of this pinned problem.

---

## 5) Real-time optimizer (solves part c) — recourse minimization

Planning is done ahead of time; real arrivals `A_{t,k}` differ. Real-time must minimize OT, given current state.

### 5.1 Backlog dynamics (ledger state)

```
Q_{t+1,k} = max{0, Q_{t,k} + A_{t,k} - S_{t,k}},    S_{t,k} ≤ μ_k · n_{t,k} · Δ
```

### 5.2 Real-time controls
- dispatch/routing of multi-skill operators across queues,
- overtime requests `o_{t,k}`,
- optional: priority rules.

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

`Φ` includes overtime costs as defined above.

**This is the exact end-to-end mathematical object for "least cost".**

Under the pinned contract (Section 3.0), the three structural gaps are closed:

1. **RT constraint is computable:** it is a precomputed monotone staffing curve `N_req` and a linear constraint `n ≥ N_req`. No nonlinear constraints remain in the optimizer.
2. **UNSAT witness matches feasibility:** infeasibility is about failing `n ≥ N_req`, and IIS/deficits certify exactly that (Section 7).
3. **Least cost is unambiguous:** single cost objective, plus lexicographic tie-break for IT. No penalty weights, no mode selection.

**This is the full closure: one contract → one computable optimization → one matching witness system.**

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

Because the RT SLA is now exactly the linear requirement `n_{t,k} ≥ N_req(λ_{t,k}^{rob}; k)`, infeasibility can be certified by a standard IIS (irreducible infeasible subset) from the MILP.

**Simple human-readable witness:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ∃ (t,k):  max n_{t,k}^{possible}  <  N_req(λ_{t,k}^{rob}; k)           │
└─────────────────────────────────────────────────────────────────────────────┘
```

where `max n_{t,k}^{possible}` is the maximum coverage achievable in that bucket given the legal shift universe `S` and headcount caps.

**Cut-style witness (in staffing-hours form, keyed to the same RT-normal form):**

Let required staffing-hours in a window `U ⊆ T` be:

```
D(U,k) = Σ_{t∈U} N_req(λ_{t,k}^{rob}; k) · Δ
```

Let maximum schedulable staffing-hours be:

```
S(U,k) = Σ_{t∈U} n_{t,k}^{max} · Δ
```

If `D(U,k) > S(U,k)`, deficit:

```
┌──────────────────────────────────────────────┐
│  δ(U,k) = D(U,k) - S(U,k) > 0              │
└──────────────────────────────────────────────┘
```

Meaning: "you are short δ staffing-hours; no schedule can meet the RT SLA."

**This now certifies infeasibility for the same RT constraint the optimizer enforces.**

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
- Operator roster constraints and wage tables.
- Leader span-of-control policy.
- SLA targets for RT/IT.
- Service rates `μ_k`, SLA parameters `(w_k^{max}, ε_k)`.
- Forecast contract `d_{t,k}` or `λ̄_{t,k}`.
- Shift template set `S` (labor-law compliant).
- Wages `w_k`, `w_k^{OT}`, `w_L`, leader span `η`.

### 9.2 Build

**Step A: Precompute `N_req(λ; k)`**
- For each `k`, tabulate `N_req` over a grid of `λ` values.
- Store the table and hash it (this is part of the contract).

**Step B: Solve planning MILP**
- Calibrate forecast contract `d_{t,k}` (quantiles) and verify coverage.
- Solve for each day: output `C*`, rosters `y`, leaders `L`, overtime plan `o`.
- Emit optimality gap certificate (MILP bound).
- Output UNIQUE/UNSAT/Ω + receipts.

**Step C: Replay real-time**
- Replay historical arrival traces `A_{t,k}`, run minimal overtime recourse using `N_req`.
- Choose OT and dispatch.
- Report RT SLA pass rate and overtime used.

**Step D: Compare to baseline**
- Cost, RT tails, idle, OT, leader hours.

### 9.3 What makes it undeniable
- You can replay a day and reproduce identical results.
- If SLA can't be met, you produce a deficit certificate showing "how many extra people/hours are required" rather than failing silently.
- "Least cost" becomes a certified number.
- RT SLA (p95/p99) is verified (since it's encoded by `N_req`).
- IT (idle) reported as tie-break result, not a tunable weight.

---

## 10) Why no one can optimize better

Once the contract is pinned (forecast uncertainty, queue model, labor laws, wages, SLAs), "better" means lower cost while satisfying the same constraints. That is exactly what `C*` is: the minimum over the admissible set.

- If another system outputs a lower cost, it must violate a constraint (not the same problem), or it is wrong.
- If the problem is infeasible, the minimal deficit witness proves no one can satisfy it without adding capacity or relaxing SLA.

**That is the complete resolution.**
