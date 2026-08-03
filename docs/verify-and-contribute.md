# Verify and Contribute

berth is an estimator, and an estimate you cannot check is just an assertion. So
berth ships with the measurement harness that produced its own validation data.
You run it on your hardware, confirm the estimate against reality, and if it is
useful, contribute the trace so the public corpus gets denser. This is the whole
loop: estimate, verify, contribute.

## sounding, the meter

`sounding` (`bench/sounding.py`) drives an OpenAI-compatible completion server (vLLM or
SGLang) with streaming requests, measures time to first token and time per output
token per concurrency and context cell, and writes trace records that
`berth.calibrate` consumes directly. It is standard-library Python, so the box
you rent needs nothing beyond a Python interpreter and a running model server.

## Verify on your own hardware

Start a model server on your GPU, then point the harness at it:

    python -m bench.sounding \
        --base-url http://localhost:8000 \
        --silicon h100-pcie \
        --model llama3-8b \
        --model-id meta-llama/Meta-Llama-3-8B \
        --out traces.jsonl

If the endpoint requires a key, pass `--api-key` or set `BERTH_API_KEY` in the
environment. Do not redeploy the server without auth to work around a 401: a
differently configured server is a different measurement, and the numbers will
not compare to anything.

Prompts are unique per request, and that is load-bearing rather than cosmetic.
Identical prompts plus automatic prefix caching, which is on by default in
current vLLM and SGLang, means the first request prefills and the rest are
served from cache. TTFT then stops measuring prefill entirely, and nothing
errors. The harness probes for caching at startup and records what it finds,
because a cell measured on a deployment with caching on does not transfer to
one without it.

The sweep runs a grid of batch sizes, prompt lengths, and output lengths and
writes one JSON line per cell. Then check berth's predictions against what your
hardware actually did:

    python -m bench.validate traces.jsonl --prefill-overhead-ms 54.6

You now know berth's error on your silicon, for your workload, in your
environment.

The floor is per setup, not per model of card. 54.6 ms is the value fitted on the
H100 PCIe cell above; 74.6 ms is the L40S value. Do not carry either to a
different accelerator, driver or server version. Fit your own with
`python -m bench.fit_overhead traces.jsonl`, which inverts it from your batch-1
cells. Reusing someone else's floor is the same error as reusing someone else's
measurement, and berth does not do that.

## Try the whole pipeline with no GPU

Mock mode generates measurements from the analytical model plus noise, so the
full path (sweep to JSONL to calibrate to error report) runs with zero GPUs. It
is also the CI smoke test. On real hardware the only thing that changes is where
the numbers come from.

    python -m bench.sounding --mock --silicon h100-sxm --model llama3-8b --out traces.jsonl

Mock traces are stamped `"source": "mock"`. They are for exercising the pipeline
and for CI. They are not measurements, they must never be contributed, and
`bench.validate` refuses a file that mixes the two. Calibrating on mock output
recovers the model that generated it, which tells you the plumbing works and
nothing whatsoever about hardware.

## The trace schema

Each line is one measured cell:

    {"schema": 2, "source": "measured", "silicon": "h100-pcie",
     "model_name": "llama3-8b", "batch": 1, "avg_prompt_tokens": 2048,
     "avg_output_tokens": 128, "measured_ttft_ms": 67.03,
     "measured_tpot_ms": 6.50}

`source` is `measured` or `mock` and is written by the harness, not by hand. It
exists because a mock trace and a hardware trace are otherwise indistinguishable
on disk, and one contributed mock file would silently corrupt the public corpus.

`measured_ttft_ms` is wall-clock to the first streamed token. For a batch
submitted concurrently it is the batch tail. `measured_tpot_ms` is decode-phase
wall-clock divided by output tokens, the mean inter-token interval, not the
inter-chunk transport gap.

## Contribute

If berth's estimate was useful on your hardware, send the anonymized trace back.
A trace carries no prompt content, only the cell coordinates and two timing
numbers, so it is safe to share. Disputes that arrive with traces attached
outrank everything else: we would rather be corrected with data than be right in
private.

Open a pull request adding your `traces.jsonl` under `data/contributed/`, or open
an issue and attach it. CI rejects any contributed file containing a record with
`"source": "mock"`, and any file without a `source` field at all. Every contributed cell makes the public estimate more
accurate for everyone serving that workload shape, and measured cells replace
spec-sheet priors in the fleet.

## Provenance

A contributed trace is tagged MEASURED for the exact (silicon, model, server)
combination it was captured on. berth never promotes a measurement on one setup
to a different one, and never presents a spec-sheet prior as a measurement. That
discipline is what makes a berth number worth trusting.
