# Get Started

## Install

    pip install berth-placement          # the CLI command is berth

Or from source, to run the tests and the harness:

    git clone https://github.com/reckonresearch/berth
    cd berth
    python -m pip install -e .

berth's core is standard-library Python. No GPU, no account, no API key. It runs
on the laptop you are reading this on.

Verify:

    python -m pytest tests/ -q      # expect: 61 passed

## Your first estimate, from the command line

The fastest path to a number needs no Python:

    berth estimate --model llama3-8b --batch 32        # scores the whole fleet, cheapest first
    berth premium  --model llama3-8b --prices l40s=0.99 h100-pcie=3.35
    berth list                                          # known silicon and models, provenance-tagged

Every line prints whether the silicon is MEASURED or a spec-sheet prior.

## Your first estimate, in Python

    from berth.workload import WorkloadSpec, profile, MODELS
    from berth.silicon import FLEET
    from berth.estimate import estimate

    spec = WorkloadSpec(
        model=MODELS["llama3-8b"],
        avg_prompt_tokens=2048,
        avg_output_tokens=256,
        target_batch=32,
    )
    sig = profile(spec)
    e = estimate(sig, FLEET["h100-pcie"], price_hr=3.35)

    print(f"{e.cost_per_mtok:.3f} $/Mtok")
    print(f"{e.ttft_ms:.1f} ms TTFT (single-request service time)")
    print(f"{e.tpot_ms:.2f} ms TPOT")

## Choose the cheapest feasible placement

berth compares a fleet of placements and picks the cheapest that meets your SLO,
via a placement client backed by an estimation backend:

    from berth import PlacementClient, PlacementPolicy, min_cost, SimBackend

    client = PlacementClient(SimBackend(seed=1))
    choice = client.place(sig, PlacementPolicy(objective=min_cost))
    print(choice.silicon, f"{choice.estimate.cost_per_mtok:.3f} $/Mtok")

Each returned Estimate carries a placement_premium field: the cost ratio to the
cheapest feasible alternative at equal p99. Feasibility is what the SLO buys you,
so set `p99_ttft_ms` and `p99_tpot_ms` on the WorkloadSpec. Without them every
placement is feasible and the premium reduces to a raw price ranking, which is
the metric these docs exist to argue against. Inspect every candidate with
client.estimate(sig) and filter on e.feasible.

Note: `place()` may return an unmeasured (prior) accelerator if its spec-sheet
prediction is cheapest. berth tags each silicon MEASURED or prior (see Silicon
and Models); treat a prior-based choice as a forecast to verify, not a
measurement.

## Check the estimate against your own hardware

berth ships `sounding`, the harness that produced its own validation data:

    python -m bench.sounding --base-url http://localhost:8000 \
        --silicon h100-pcie --model llama3-8b \
        --model-id meta-llama/Meta-Llama-3-8B --out traces.jsonl
    python -m bench.fit_overhead traces.jsonl          # fit your own prefill floor
    python -m bench.validate     traces.jsonl --prefill-overhead-ms <fitted>

Every record carries `source`, `measured` or `mock`. See Verify and Contribute.

## What to read next

- The Physics: understand every term in the number you just got
- Validation (P0): the evidence these estimates match real hardware
- Silicon and Models: the full fleet and model registry
