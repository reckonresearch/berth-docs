# Tutorials

Four runnable programs, each in the repository under `tutorials/`, each printing
its expected output. They are smoke-tested in CI, so if one does not run the
build is red.

Work through them in order. Each introduces exactly one idea and depends only
on the ones before it.

| | What it introduces |
|---|---|
| [101: Hello berth](101-hello-berth.md) | Profile a workload, estimate it across the whole fleet, read the placement premium |
| [201: Policies and SLOs](201-policies-and-slos.md) | Objectives and constraints as plain Python; how an SLO changes the answer |
| [301: Calibrate from Traces](301-calibrate-from-traces.md) | Fit efficiency factors from measured traces, with held-out error and confidence intervals |
| [401: Tail-Aware Sizing](401-tail-aware-sizing.md) | Arrival rate plus a p99 target becomes a fleet size, with queueing headroom priced in |

All four run on a laptop. No GPU, no account, no API key: the core is
standard-library Python, and 301 uses the mock harness so the calibration path
runs with zero accelerators.

```bash
pip install berth-placement
git clone https://github.com/reckonresearch/berth && cd berth
python tutorials/101_hello_berth.py
```

When you are ready to point the estimator at your own hardware rather than the
simulator, go to [Verify and Contribute](../verify-and-contribute.md).
