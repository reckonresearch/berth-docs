# sounding

**The harness.** Measures the real thing and checks the prediction.

berth predicts. sounding drives a live endpoint, records what actually
happened, and produces the traces that say whether the prediction was right.

## It never imports the estimator

Enforced by test. Evidence that cannot be separated from the thing it
evaluates is not evidence, and a harness that shares code with the model it
scores can agree with it for reasons that have nothing to do with the
hardware.

## What it captures

A sweep across batch sizes, prompt lengths and output lengths, with the run
conditions recorded rather than assumed:

**Silicon identity**, read from the device where the server is local and
recorded as self-reported where it is not. An operator once ran a sweep on one
card while declaring another, and every timing was real.

**The served model**, verified against the endpoint rather than trusted. A
server started with the wrong checkpoint produces clean timings for a model
that is not the one being attributed.

**Quantization**, cross-checked against the launch flags. A run declaring
half-precision weights without the corresponding server flag is refused.

**Prefix caching**, probed and recorded, because a cell measured with caching
enabled does not transfer to one without it.

**Unique prompts per request.** Identical prompts against a default-on cache
once produced apparent prefill throughput of twenty-one times a card's peak
FLOPS, and nothing errored.

## The auditor

Before anything is fitted, `bench.audit_traces` checks a file against physics:
prefill within peak, first-token latency responding to token count, effective
bandwidth constant across context where the device has one access pattern.

**Only physical impossibility refuses a file.** Everything else prints and
passes. Seven of the eleven defects in this project were checks that rejected
correct data, and one nearly discarded the most valuable cell in the corpus.

## Contributing

If you run a cell we have not measured, send it. The corpus grows by
measurement and every contributed cell is published with its provenance.

[Verify and contribute](verify-and-contribute.md) ·
[Validation record](validation-p0.md) ·
[Defect register](https://github.com/reckonresearch/berth/blob/main/DEFECTS.md)
