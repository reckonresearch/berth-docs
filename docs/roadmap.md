# Roadmap

**Version 0.7.0, August 2026.**

> We define the unit, publish the instrument, and build the agents and APIs
> that act on it.

Every item carries what it is blocked on. An item with no blocker is buildable
now, and there are fewer of those than there look.

---

## The stack

| | What it is | Ecosystem role |
| --- | --- | --- |
| **The served token** | The unit: one output token delivered inside its SLO and above its quality floor | What everyone measures in |
| **berth** | The estimator. Predicts what a workload costs and how fast it runs on a given accelerator, before you rent it | What they measure with |
| **sounding** | The harness and the meter. Measures the real thing and checks the prediction | How the corpus grows |
| **pilot** | The agent and the router. Watches, decides, moves the workload, proves what the move saved | What they buy |
| **The control plane** | `.berth/STATUS.md`, the ledger, the daily report | How they see it working |
| **The Holdout Protocol** | How a saving is proven rather than claimed | What they settle under |
| **Receipts** | The conforming record for one period | What they file |
| **The Placement Index** | Public reference, workload-conditional, quarterly | How they find you |
| **versus** | Self-host or API on one axis | How people who are not customers arrive |

### The boundary that matters most

pilot routes by **programming the infrastructure**, not by carrying traffic.

A BGP speaker routes without carrying a packet's payload: it programs the
forwarding plane and the forwarding plane moves the traffic. pilot changes the
deployment configuration, merges it under a declared policy, and your gateway
or orchestrator sends traffic to the new placement.

Sitting in the request path would add latency to every call, make our
availability yours, and compromise the only thing that makes the measurement
worth anything. **A party that carries traffic cannot credibly rank the
placements it carries traffic for.**

This boundary is architectural and it is not a phase.

---

## Shipped

### berth

Closed-form roofline plus a queueing term. Roughly a page of physics, no
learned components, no training data. Prefill, decode, and the serial
admission term. Feasibility gated on TTFT p99 and TPOT p99, with a placement
missing the bound excluded rather than ranked cheaply. Key-value pressure and
concurrency-dependent key-value bandwidth. Price basis as a search dimension.
bf16 and fp8, read from traces rather than assumed. Dense, grouped-query and
mixture-of-experts attention.

Held out: fit on one accelerator, predict another it has never seen, worst
fold 9.5 percent against a 15 percent gate published before the first run.

### sounding

Sweeps with an overridable grid. Silicon identity read from the device or
recorded as self-reported, never guessed. Served model verified against the
endpoint. Quantization cross-checked against launch flags. Prefix caching
probed. Unique prompts per request. Backend-aware microbenchmark across CUDA,
ROCm, XLA and Neuron that skips rather than fabricating a ceiling. The
contamination auditor, where only physical impossibility refuses a file.
Device power at schema 5.

### pilot

The decision record. Four watchers: model registries by commit, serving-stack
releases, provider prices with an epsilon, corpus additions. Proposal as a
pull request carrying the diff and the evidence.

**Execution**, under a policy declared once in the customer's repository. Four
autonomy levels, six guardrails, automatic rollback that outranks savings.

Suppression state, so the same proposal is not opened twice and a rejection
holds for ninety days. The control plane. The ledger, with realized,
available and foregone never summed. The Holdout Protocol and conforming
receipts. Shadow mode, and a kill criterion computable from shadow data alone.

---

## Next, unblocked today

| | Why now | Effort |
| --- | --- | --- |
| **The Placement API** | Every inference pipeline is automated end to end. A decision consumable only through a repository is a decision half the market cannot reach, and pilot is not adaptive if the loop runs at the speed of a merge | 2 weeks |
| **Conformance suite** | Anyone validates their own records without contacting us. The only work that makes the specification spread without consuming founder attention, and the verification revenue line does not exist without it | 1 week |
| **Reference hashes, Go and TypeScript** | A protocol whose hardest part is a hash nobody wants to write does not spread | 3 days |
| **Serving-stack version as an axis** | A stack upgrade has measured over twice the throughput on identical hardware. Every cell is version-conditional and the analysis does not use it | 4 days |

**The API is first.** pilot executes today through a repository, which reaches
every team that deploys from git and no team that does not. The API is what
makes the decision programmatically consumable by an orchestrator, a CI
pipeline, or a router, and it is the layer every other ecosystem product
eventually calls.

**The conformance suite is second**, because it is the only work that makes
the unit reachable by people who will never be customers, and adoption by
non-customers is what turns verification into revenue.

**On the API and the boundary.** It returns a decision, not a proxied
response. A router asking "where should this class run" gets an answer; a
router asking us to serve the request does not. That distinction is the whole
architecture and the API is where it will be tested first.

---

## Next, blocked on a customer

| | Blocked on |
| --- | --- |
| **Hosted pilot** | One design partner granting repository access |
| **Live holdout period** | One customer declaring a baseline |
| **Rollback verification** | A live period. The logic exists and has never fired against a real degradation |
| **Non-GitHub delivery** | A customer who does not deploy from a repository. GitLab and a generic webhook are the obvious two |
| **Public register of conforming records** | The conformance suite, then adopters |

---

## The financial layer

This is where the specification becomes revenue rather than reputation.

| | Requires |
| --- | --- |
| **Verification service** | The specification adopted by two parties who are not us. Settling periods on contracts we did not originate |
| **Settlement for delivered-work contracts** | Contracts denominated in served tokens existing at all |
| **Corpus subscription** | More cells, and allocators finding the software-deficit closure series. That series is the one nobody else can produce |

The precedent is energy performance contracting. IPMVP is free and public; the
firms measuring savings under it are not. The protocol turned an unobservable
counterfactual into a contractible number and an industry formed above it.

---

## Measurement

| | Blocked on |
| --- | --- |
| **A third architecture** | An operator with TPU or Trainium access. This is the cell that turns the 9.5 percent transfer from a memory-technology result into a cross-architecture one. Brief written, unsent |
| **MoE decode term** | The dense control cell, pre-registered and unrun |
| **Speculative decode** | The MoE term above |
| **Disaggregated prefill and decode** | A cell where the two run on different hardware. The regime most likely to break the closed form, which is why it is worth measuring |
| **Energy in the estimate** | Power telemetry in three or more cells. Captured at schema 5. Energy is in the unit and not yet in the meter |
| **Interconnect and multi-node** | Multi-node access. The tensor-parallel scaling factor has never been checked above one node |
| **Quality floor evaluation** | The specification's evaluation section. The harness records the declaration rather than evaluating it |

---

## Research

| | Blocked on |
| --- | --- |
| **Portfolio placement across classes** | A multi-class account. Solving each class alone leaves the headroom a tight-deadline class reserves unused by a loose one. This is where Insull's diversity factor becomes a feature rather than an analogy |
| **Deadline-class mixing** | The pre-registered mixing run |
| **Forward placement** | A premium time series, several quarters. What capacity to commit and at what term |
| **Learned residual** | The second engineer, and the boundary of the closed form being mapped first |

---

## Sequencing constraints

Not preferences. Each is a case where building in the wrong order produces a
result that cannot be interpreted.

**MoE decode before speculative decode.** A speculative verify pass with k+1
tokens routes to more experts than a single token does. Measuring both in one
cell confounds them and neither term can be recovered.

**The closed-form boundary before the learned residual.** A learned model
fitted over a regime nobody has characterised does not tell you where the
physics stopped working. It tells you nothing, expensively.

**Shadow before execution, per account.** The kill criterion is computable
from shadow data alone. Executing before that data exists means the first
evidence of a bad trigger set is a bad move.

**Measurement before autonomy, per cell.** An unmeasured placement is never
executed by default. The corpus gates what the agent may do unattended, which
is the clearest reason the estimator and the agent are the same company.

**The conformance suite before the verification service.** You cannot sell
verification of a standard nobody else can check.

---

## Kill criteria

Pre-registered conditions under which the work stops.

**The agent.** Fewer than one trigger in four producing a change that clears
the hurdle means the trigger set is wrong and the agent is noise.

**Execution.** If rollback fires on more than one move in ten across the first
fifty executions, the guardrails are wrong and the default returns to propose.

**Portfolio placement.** If solving classes jointly does not beat solving them
independently by more than the confidence band, it does not ship.

**Deadline-class mixing.** Below 1.2x improvement in delivered work per unit
capital, the claim is smaller than stated and the external copy changes.

**The learned residual.** If the closed form holds across disaggregated
serving and speculative decoding, there is no boundary to map. That is good
news and it publishes.

---

## Deliberately not on this roadmap

**A data plane.** Not now, not later. The measurement is worth something
because we do not carry the traffic.

**Owning capacity.** No resale, no reserved blocks, no inventory. The moment
inventory exists there is a reason to route toward it.

**Provider-paid placement.** No provider pays for routing, ranking, or
position in the Index, at any price. This is structural rather than a values
statement: a measurement its seller can tune is not one anyone should price
against.

**A dashboard.** The control plane is a file in the customer's repository.
Version controlled, no login, and deleting it turns the thing off.
