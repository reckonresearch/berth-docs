# 301: Calibrate from Traces

The estimator ships with priors. Calibration replaces them with values fitted
from measured traces, and reports how much that helped on data it did not see.

```bash
python tutorials/301_calibrate_from_traces.py
```

This runs with zero GPUs. The mock harness generates traces from the analytical
model plus noise, which exercises the whole path from sweep to error report. On
real hardware the only thing that changes is where the numbers come from.

## The code

```python
from types import SimpleNamespace
from bench.sounding import DEFAULT_GRID, run_sweep
from berth import FLEET, calibrate

args = SimpleNamespace(mock=True, silicon="h100-sxm", model="llama3-8b",
                       seed=7, grid=DEFAULT_GRID, base_url=None, model_id=None)
traces = run_sweep(args)

fleet, report = calibrate(FLEET, traces)
(mfu_lo, mfu_hi), (bw_lo, bw_hi) = report.ci95["h100-sxm"]

print(f"holdout MAPE: {report.mape_prior:.1%} -> {report.mape_calibrated:.1%}")
print(f"mfu    {fleet['h100-sxm'].mfu:.3f}  95% CI [{mfu_lo:.3f}, {mfu_hi:.3f}]")
print(f"bw_eff {fleet['h100-sxm'].bw_eff:.3f}  95% CI [{bw_lo:.3f}, {bw_hi:.3f}]")
```

## What is being fitted

Two numbers per accelerator, and only two.

`mfu` is the fraction of peak compute a real prefill achieves. `bw_eff` is the
fraction of peak memory bandwidth a real decode achieves. Everything else in the
roofline comes from public specifications, which is what makes an estimate
auditable by hand.

They are recovered by direct roofline inversion, not by an optimiser: TTFT
inverts to `mfu`, memory-bound TPOT inverts to `bw_eff`, aggregated with robust
medians and iterated twice for bound reclassification. Every fitted value traces
back to specific observations, and there is no dependency to install.

## Two details that matter

**Prefill MFU is inverted from batch-1 cells only.** Batched cells carry the
serial-admission term, so including them attributes contention to compute and
the fit gets worse rather than better. This was a real defect, found and fixed;
see [Validation](../validation-p0.md).

**The holdout is the number to read.** `mape_prior` against `mape_calibrated` is
scored on traces held out of the fit. A calibration that improves the training
fit and not the holdout has learned the noise.

## Confidence intervals, not point estimates

`report.ci95` gives a 95% interval per parameter. A fit whose interval spans
half the plausible range is telling you the sweep was too thin, not that the
card is unusual. Read the interval before you read the value.

## On real hardware

Swap the mock harness for a sweep against a live server:

```bash
python -m bench.sounding --base-url http://localhost:8000 \
    --silicon h100-pcie --model llama3-8b \
    --model-id meta-llama/Meta-Llama-3-8B --out traces.jsonl
```

Then calibrate from `traces.jsonl` instead. Every record carries `source`,
either `measured` or `mock`, and a file mixing the two is refused: a fit over
simulated data recovers the model that generated it and tells you nothing about
hardware. See [Verify and Contribute](../verify-and-contribute.md).

Next: [401: Tail-Aware Sizing](401-tail-aware-sizing.md).
