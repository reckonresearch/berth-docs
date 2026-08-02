# 201: Policies and SLOs

A placement policy is an objective plus constraints, and both are plain Python.
There is no rule language to learn and nothing hidden.

```bash
python tutorials/201_policies_and_slos.py
```

## The code

```python
from berth import (MODELS, PlacementClient, PlacementPolicy, SimBackend,
                   WorkloadSpec, min_cost)

client = PlacementClient(SimBackend(seed=0))
sig = client.profile(WorkloadSpec(model=MODELS["llama3-70b"], target_batch=16,
                                  avg_prompt_tokens=1024, avg_output_tokens=256))

cheapest = client.place(sig, PlacementPolicy(objective=min_cost))
print(f"min-cost:   {cheapest.silicon} @ ${cheapest.estimate.cost_per_mtok:.2f}/Mtok")

snappy = client.place(sig, PlacementPolicy(
    objective=min_cost,
    constraints=(lambda e: e.ttft_ms < 300,),   # any Callable[[Estimate], bool]
))
print(f"TTFT<300ms: {snappy.silicon} @ ${snappy.estimate.cost_per_mtok:.2f}/Mtok "
      f"(TTFT {snappy.estimate.ttft_ms:.0f}ms)")
```

## Objectives

An objective is any `Callable[[Estimate], float]` that berth minimises.
`min_cost` and `min_tpot` ship with the library, and either is three lines to
write yourself:

```python
def min_cost(e):  return e.cost_per_mtok
def min_tpot(e):  return e.tpot_ms
```

Weight them however your business actually trades off:

```python
policy = PlacementPolicy(objective=lambda e: e.cost_per_mtok + 0.01 * e.tpot_ms)
```

## Constraints

A constraint is any `Callable[[Estimate], bool]`. Placements that fail one are
removed from the feasible set entirely, not penalised. That is deliberate: a
latency bound is a threshold, not a preference, and averaging across it would
report a quantity nobody buys.

```python
constraints=(
    lambda e: e.ttft_ms < 300,
    lambda e: e.n_devices <= 2,
    lambda e: e.silicon != "cpu-spr",
)
```

## What this tutorial actually demonstrates

The two calls differ only in a constraint, and they can return different
silicon at different prices. Adding a deadline shrinks the feasible set, and
the cheapest survivor may be a faster, more expensive card.

This is the inversion in miniature. A ranking by raw price is stable; a ranking
by cost of useful work under a deadline is not, because feasibility depends on
the deadline. A tool that returned one fixed answer would be telling you there
was nothing to decide.

**One caveat.** `place()` may return an accelerator berth has never measured, if
its spec-sheet prediction is cheapest. Every result is tagged, and a
prior-tagged choice is a forecast to verify rather than a measurement to act
on.

Next: [301: Calibrate from Traces](301-calibrate-from-traces.md).
