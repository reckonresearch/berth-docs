# The Physics

berth is a closed-form roofline. Every prediction decomposes into terms you can
compute by hand and check against your own hardware. This page states each term
so nothing in the estimate is opaque.

## Decode: a race between compute and memory

Per output token, decode does a fixed amount of arithmetic and moves a fixed
number of bytes (the model weights, plus the KV cache for the context so far).
The time per token is the larger of the two:

    TPOT = max( flops_per_token / achievable_flops,
                bytes_per_token / achievable_bandwidth )

At the batch sizes most serving runs at, decode is memory-bound by a wide margin.
For Llama-3-8B the arithmetic intensity at batch 1 is about 1.05 FLOP/byte, while
the ridge points of current accelerators are in the hundreds. Decode is therefore
memory-bound by two to three orders of magnitude, and the binding term is bytes
divided by bandwidth. This is why an untuned bandwidth prior predicts decode
latency well, and why the premier chip, which wins on compute, does not
necessarily win on decode cost.

## Prefill: compute plus a fixed floor

Prefill processes the whole prompt before the first token. Its compute scales
with prompt length, but there is also a fixed cost that does not:

    TTFT = fixed_floor + prompt_compute / achievable_flops

The fixed floor is scheduler admission, detokenization, sampler setup, and the
first forward pass through a compiled graph. It is prompt-length independent and
hardware-and-server specific. On measured hardware it was 74.6 ms on an L40S and
54.6 ms on an H100 PCIe, and it is not predictable from a spec sheet. At short
prompts it dominates TTFT; at long prompts it amortizes away.

### Subtract the floor before taking any ratio derived from TTFT

This is the rule that has cost us the most to learn, so it is stated as a rule
rather than left as an implication.

A constant added to a measured quantity survives subtraction but not division.
Any ratio taken from raw TTFT inherits the floor, and because the floor does
not scale with prompt length, batch size or anything else, it distorts every
such ratio in a way that looks like a physical finding.

Three separate checks in this project have made the same mistake, and each one
reported it as a failure of the model rather than of the arithmetic:

- The prefill validator inverted TTFT to an implied MFU without removing the
  floor, and reported that attention accounting was wrong. It was not: the
  floor was being attributed to compute.
- A check on the serial-prefill finding divided batch ratio by time ratio to
  recover effective parallelism. At short prompts, where the floor is most of
  TTFT, it reported parallelism of 3.2 on a clean file and declared it
  contaminated.
- A trace auditor computed prefill throughput from raw TTFT and reported 1.19
  times the card's peak FLOPS on the same clean file.

None of these were subtle in hindsight and all three shipped. The floor is 54.6
ms on an H100 PCIe and 74.6 ms on an L40S; at a 60 ms TTFT there is almost
nothing left that scales, so the ratio is measuring the constant.

In practice:

    implied_mfu = prompt_compute / ((ttft_ms - floor_ms) / 1000 * peak_flops)

and never `ttft_ms` alone. Fit the floor first with
`python -m bench.fit_overhead`, pass it to anything downstream, and prefer the
longest prompt in a sweep when a ratio has to be judged, because that is where
the floor is the smallest share of the measurement.

The same caution applies to any other term that is constant with respect to
the thing being varied. The floor is the one that has bitten repeatedly, but
the general form is: a ratio is only interpretable once every constant term
has been removed from both sides.

## Concurrency: serial batch admission

When a batch of requests arrives together, they do not each emit a first token
when their own prefill finishes. Measured behavior is that the whole batch's
first tokens are gated on the last prefill: prefill runs serially through one
pipeline, so the batch-tail request waits behind all of them.

    TTFT_tail = fixed_floor + batch * single_request_prefill

Measured effective prefill parallelism is about 1.09 across every batch size and
prompt length on two different cards, i.e. essentially one. Both measurements
were taken under vLLM 0.6.3. Serial admission is a scheduling decision, so this
is a property of that scheduler until a second serving stack is measured. That
run is pre-registered. This is deterministic
admission, not a stochastic queue, so the wait is exact. Modeling it reduces
first-token-latency error from roughly 65 percent to under 9 percent.

This term lives in the queueing layer, not in the base estimate. The base TTFT is
a single request's service time by deliberate design; the batched tail is a
queueing quantity composed on top. Keeping them separate is what prevents
double-counting contention.

## Cost

Cost per million tokens is throughput driven:

    $/Mtok = (price_per_hour / 3600) / aggregate_tokens_per_second * 1e6

where aggregate throughput at batch b is b divided by TPOT. Because TPOT is the
decode term above, cost inherits decode's memory-bound physics: a card that is
fast on compute but ordinary on bandwidth, and expensive per hour, can lose on
cost per token to a cheaper card.

## The placement premium

The premium is the ratio between the cost of a chosen placement and the cheapest
placement that still meets the same p99. It is not a constant. It depends on the
workload and the SLO, which is precisely why a placement engine is worth having:
a single fixed premium would mean there was nothing to decide.
