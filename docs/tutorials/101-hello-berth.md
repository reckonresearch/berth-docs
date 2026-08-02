# 101: Hello berth

Profile a workload, estimate it on every accelerator berth knows, and read the
placement premium off each row.

Run it:

```bash
python tutorials/101_hello_berth.py
```

## The code

```python
from berth import MODELS, PlacementClient, SimBackend, WorkloadSpec

client = PlacementClient(SimBackend(seed=0))
sig = client.profile(WorkloadSpec(model=MODELS["llama3-8b"], target_batch=8))

for e in client.estimate(sig):
    if e.feasible:
        print(f"{e.silicon:<10} ${e.cost_per_mtok:6.2f}/Mtok  "
              f"premium {e.placement_premium:5.0%}  {e.bound}-bound")
```

## What each line does

`WorkloadSpec` describes what you are serving: the model, the concurrency, and
optionally prompt and output lengths and an SLO. Nothing here is about
hardware.

`client.profile()` turns that into a **compute signature**: per-request FLOPs,
KV bytes per token, memory footprint. This is the workload expressed in the
quantities the roofline consumes, and it is hardware independent. Compute it
once and reuse it.

`client.estimate()` scores that signature against every accelerator in the
fleet and returns one `Estimate` each. Filter on `feasible`, because a
placement that cannot fit the model in memory or cannot meet a stated SLO is
excluded rather than silently ranked.

## Reading the output

Three fields carry the answer.

`cost_per_mtok` is dollars per million output tokens at the given price, driven
by decode throughput.

`placement_premium` is the cost ratio to the cheapest feasible alternative. The
cheapest row reads 0%; a row at 63% costs 63 percent more than it needs to for
that workload.

`bound` says which roofline term is binding, compute or memory. At the batch
sizes most serving runs at, decode is memory-bound by two to three orders of
magnitude, so you should expect `memory` here. If you see `compute`, something
about the workload is unusual and worth understanding before trusting the
number.

## The thing worth noticing

The ordering is not the ordering of the spec sheets. A cheaper card with less
peak compute frequently wins on cost per token, because decode is bound by
bytes moved rather than arithmetic. That is the whole reason this tool exists,
and [The Physics](../the-physics.md) explains the mechanism.

Every row is tagged MEASURED or prior. A prior row is a spec-sheet forecast, not
an observation; see [Silicon and Models](../silicon-and-models.md).

Next: [201: Policies and SLOs](201-policies-and-slos.md).
