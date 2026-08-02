# 401: Tail-Aware Sizing

Add an arrival rate and a p99 target, and an estimate becomes a fleet size with
queueing headroom priced into the cost.

```bash
python tutorials/401_tail_aware_sizing.py
```

## The code

```python
from berth import MODELS, PlacementClient, SimBackend, WorkloadSpec

client = PlacementClient(SimBackend(seed=0))
sig = client.profile(WorkloadSpec(
    model=MODELS["llama3-70b"], target_batch=16,
    avg_prompt_tokens=1024, avg_output_tokens=256,
    p99_ttft_ms=500.0,        # the deadline
    arrival_rps=40.0,         # the load
))

for e in client.estimate(sig):
    if e.feasible:
        print(f"{e.silicon:<10} {e.replicas:>3} replicas  util {e.utilization:4.0%}  "
              f"p99 TTFT {e.p99_ttft_ms:4.0f}ms  ${e.cost_per_mtok:.2f}/Mtok")
```

`arrival_rps` is the switch. Without it you get single-replica service times.
With it, three new fields populate: `replicas`, `utilization` and `p99_ttft_ms`.

## The model

The replica pool is an M/M/c queue on batch slots, where `c = replicas x batch`
and service time is request residence, `TTFT + out_tokens x TPOT`. Erlang-C via
the numerically stable Erlang-B recurrence gives the probability of waiting; the
M/M/c wait tail is exponential, so p99 TTFT is closed-form rather than
simulated.

`size_replicas` finds the smallest fleet meeting the p99 target, or falls back
to an 0.85-utilisation convention when no SLO is stated. Cost then prices the
headroom in: total fleet cost over goodput at the offered arrival rate, not over
theoretical peak.

## Why this changes the answer

Tightening the TTFT target from 500ms to 300ms flips which silicon is cheapest
and knocks two classes out entirely. A card that wins on cost per token at
saturation can lose once you have to hold a tail, because holding a tail means
running with headroom, and headroom is capacity you pay for and do not sell.

This is the same inversion as [201](201-policies-and-slos.md), now with the
mechanism visible: the deadline determines the utilisation you can run at, and
the utilisation determines the cost.

## Two disclosed limitations

**Residence times are modelled exponential.** Real output lengths are
heavy-tailed, which makes true tails worse than this predicts. The
Allen-Cunneen correction slots into the calibration layer once measured
residence variance exists. Until then, read these p99 figures as optimistic.

**p99 TPOT is still checked against the mean.** Prefill interference under
continuous batching is not modelled. p99 TTFT is queueing-aware; p99 TPOT is
not yet.

Both are stated here rather than in a footnote because a sizing you cannot
trust at the tail is a sizing you cannot use.
