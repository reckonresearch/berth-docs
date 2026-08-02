# Worked Example: Reproduce the Placement Premium

This page walks the L40S-versus-H100 placement premium end to end, with the exact
numbers from berth's measured traces. Every figure here comes from real hardware,
and the steps are reproducible against the published trace data.

## The question

You are serving Llama-3-8B decode. You can run it on an NVIDIA H100 PCIe, the
premier inference card, or an NVIDIA L40S, a cheaper part. The H100 is faster.
Which is cheaper per token?

Intuition says the faster premier card. The measurement says otherwise.

## The measured numbers

From 60 traces on each card (Llama-3-8B, bf16, vLLM):

| Concurrency | L40S $/Mtok | H100 PCIe $/Mtok | Premium of choosing H100 | L40S tok/s/stream | H100 tok/s/stream |
|---|---|---|---|---|---|
| batch 1 | 6.065 | 9.908 | 1.63x | 45.3 | 93.9 |
| batch 8 | 0.885 | 1.404 | 1.59x | 38.8 | 82.9 |
| batch 32 | 0.317 | 0.535 | 1.69x | 27.1 | 54.4 |

The L40S is cheaper per output token at every batch size, by 1.57 to 1.69 times.

One condition, stated before the explanation rather than after it: this is a
comparison at matched batch, not at matched latency. The H100 is roughly twice as
fast per stream, so under a per-token deadline tight enough to exclude the L40S,
the L40S delivers no compliant output and this table does not apply. The premium
is a quantity conditional on an SLO, and the SLO here is loose.

## Why the faster card loses

The H100 PCIe decodes about 2.1 times faster per stream (93.9 versus 45.3 tokens
per second at batch 1). But it costs about 3.4 times more per hour ($3.35 versus
$0.99 on-demand at time of measurement). Cost per token is price-per-hour divided
by throughput, so:

    H100:  ($3.35 / 3600) / (93.9 tok/s)  scaled to a million tokens = $9.91/Mtok
    L40S:  ($0.99 / 3600) / (45.3 tok/s)  scaled to a million tokens = $6.07/Mtok

The 3.4x price gap swamps the 2.1x speed gap. The premier chip is the wrong chip
for this workload, by 63 percent at batch 1.

## Reproduce it yourself

The arithmetic is transparent. Given a card's measured decode rate and its hourly
price, the cost per million output tokens is:

    def usd_per_mtok(price_per_hour, tokens_per_second_aggregate):
        return (price_per_hour / 3600) / tokens_per_second_aggregate * 1e6

At batch b, aggregate throughput is b divided by TPOT in seconds. berth predicts
that TPOT from the roofline; the traces measure it. Both agree to within a few
percent (see Validation), which is why the estimate can stand in for the
measurement when you are deciding before you rent.

## Get the estimate without renting anything

    from berth.workload import WorkloadSpec, profile, MODELS
    from berth.silicon import FLEET
    from berth.estimate import estimate

    sig = profile(WorkloadSpec(
        model=MODELS["llama3-8b"],
        avg_prompt_tokens=2048, avg_output_tokens=256, target_batch=32,
    ))
    l40s = estimate(sig, FLEET["l40s"],      price_hr=0.99)
    h100 = estimate(sig, FLEET["h100-pcie"], price_hr=3.35)

    print(f"L40S      {l40s.cost_per_mtok:.3f} $/Mtok")
    print(f"H100 PCIe {h100.cost_per_mtok:.3f} $/Mtok")
    print(f"premium   {h100.cost_per_mtok / l40s.cost_per_mtok:.2f}x")

## The honest boundary

This premium is decode and cost driven, at list prices, on one day. It is a
ratio, so it is robust to uniform price changes but not to differential
discounting: a reserved-rate deal on the H100 could narrow it. It is a cost
result, not a latency-under-load guarantee. And it holds for this workload shape;
a different model, context, or SLO can move it, which is the entire reason a
placement engine exists rather than a lookup table.
