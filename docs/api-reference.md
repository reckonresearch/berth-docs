# API Reference

berth is used as a Python library. Every symbol below is importable from the
top-level `berth` package or the submodule shown. Signatures match the shipping
code.

## Describing a workload

### `WorkloadSpec`

    WorkloadSpec(
        model: ModelSpec,
        avg_prompt_tokens: int,
        avg_output_tokens: int,
        target_batch: int,
        p99_ttft_ms: float | None = None,
        p99_tpot_ms: float | None = None,
        arrival_rps: float | None = None,
    )

The workload you want priced. `p99_*` fields set SLO constraints used by
placement; `arrival_rps` engages the queueing path for fleet sizing.

### `profile(spec) -> ComputeSignature`

Turns a `WorkloadSpec` into the compute signature the estimator consumes
(per-request FLOPs, KV bytes, and so on). Call it once, pass the result to
`estimate` or the placement client.

## Estimating one placement

### `estimate(sig, hw, price_hr) -> Estimate`

    from berth import estimate, FLEET
    e = estimate(sig, FLEET["h100-pcie"], price_hr=3.35)

Returns an `Estimate` for one (workload, silicon, price) triple.

### `Estimate` fields

| Field | Meaning |
|---|---|
| `silicon` | the silicon key priced |
| `feasible` | does this placement meet the workload's SLO |
| `reason` | why infeasible, when applicable |
| `n_devices` | accelerators required (memory-driven) |
| `ttft_ms` | single-request time to first token (service time) |
| `tpot_ms` | time per output token |
| `tokens_per_s` | aggregate decode throughput |
| `cost_per_mtok` | dollars per million output tokens |
| `bound` | which roofline term binds, compute or memory |
| `placement_premium` | cost ratio to the cheapest feasible alternative |
| `replicas`, `utilization`, `p99_ttft_ms` | populated when `arrival_rps` is set |

## Choosing a placement

### `PlacementClient`

    from berth import PlacementClient, PlacementPolicy, min_cost, SimBackend
    client = PlacementClient(backend)
    handle = client.place(sig, PlacementPolicy(objective=min_cost))
    candidates = client.estimate(sig)   # every placement, score yourself

`place` returns the objective-minimizing feasible placement. `estimate` returns
the full candidate list so you can filter on `feasible` and inspect
`placement_premium` yourself.

### `PlacementPolicy`

    PlacementPolicy(
        objective: Callable,           # min_cost or min_tpot
        constraints: tuple = (),       # e.g. (lambda e: e.ttft_ms < 400,)
        min_improvement: float = 0.0,  # hysteresis for migration decisions
    )

Objectives `min_cost` and `min_tpot` are provided; constraints are predicates on
an `Estimate`.

## Modeling concurrency

### `concurrent_prefill_ttft_ms(single_req_prefill_ms, overhead_ms, batch, c_prefill=1) -> float`

The measured serial-admission term: batch-tail first-token time when a
synchronous batch contends for prefill. `c_prefill=1` is the measured default
(prefill is serial). Lives in `berth.queueing`.

### `size_replicas(...)` and `erlang_c(...)`

Fleet sizing under a Poisson arrival process, in `berth.queueing`. Engaged when
the workload carries `arrival_rps`.

## Calibrating against measured traces

### `calibrate(prior_fleet, traces, holdout_frac=0.25) -> (fitted_fleet, CalibrationReport)`

    from berth import calibrate, FLEET
    fitted, report = calibrate(FLEET, traces)

Fits per-silicon parameters from measured `TraceRecord`s and reports held-out
error. Note: prefill MFU is inverted from batch-1 cells only, and `_mape` warns
when scoring single-request TTFT against batched traces. See Validation for why.

## Silicon and models

- `FLEET: dict[str, SiliconProfile]` - the accelerator registry (see Silicon and Models)
- `MODELS: dict[str, ModelSpec]` - the model registry
- Each `SiliconProfile` carries `peak_tflops`, `hbm_bw_tbs`, `mem_gb`,
  `base_price_hr`, and `prefill_overhead_ms` (ships 0.0; a calibration output)

## Command-line interface

Installed as the `berth` entry point. Wraps the public API for a first number
without writing Python.

    berth estimate --model KEY [--silicon KEY] [--batch N] [--prompt N] [--output N] [--price-hr F] [--json]
    berth premium  --model KEY [--silicon KEY ...] [--batch N] [--prices KEY=USD ...]
    berth list

`estimate` scores one silicon or the whole fleet, cheapest first. `premium`
ranks feasible placements and shows the cost ratio to the cheapest; its output is
PREDICTED (roofline) and accepts real per-silicon prices via `--prices`. Every
command tags each silicon MEASURED or prior.

## sounding, the measurement instrument

A module entry point (see Verify and Contribute):

    python -m bench.sounding --base-url URL --silicon KEY --model KEY --model-id HF_ID --out traces.jsonl
    python -m bench.validate traces.jsonl --prefill-overhead-ms FLOAT
    python -m bench.fit_overhead traces.jsonl          # fit the prefill floor from batch-1 cells

`bench.run_sweep` is retained as a deprecation shim for the old path.

Every record carries `source`, either `measured` or `mock`, written by the
harness. `TraceRecord.source` defaults to `measured` when reading a schema-1
file, and `bench.validate` refuses a file that mixes the two.
