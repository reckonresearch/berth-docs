# berth is a placement estimator for inference engineers

berth predicts what an LLM inference workload will cost and how fast it will run,
on a given accelerator, under a given latency target, before you rent the GPU.
You describe the workload. berth returns dollars per million tokens, time to
first token, and time per output token, plus the placement premium: how much you
would save by moving to the cheapest silicon that still meets your p99.

It is a closed-form roofline, not a learned black box. Every number is arithmetic
you can audit by hand, and every prediction ships with the physical term that
produced it. berth is open source (Apache-2.0) and runs in your environment.
Nothing leaves your machine.

Here is the division of labor in practice:

| You focus on | You write | berth handles |
|---|---|---|
| Your workload: model, batch, prompt and output lengths, SLO | A short Python call or one CLI line, on your laptop | The roofline: compute-bound and memory-bound decode, prefill with its fixed floor, serial batch admission |
| Which placements to compare | The silicon and provider list to evaluate | $/Mtok, TTFT, TPOT per placement, and the premium between them |
| The decision | Nothing you cannot inspect | Auditable math, every term tagged and checkable |

## Why this exists

Compute is priced by the GPU-hour but consumed as useful work under a latency
constraint, and no standard unit bridges the two. A GPU-hour is a list price. The
cost of a token delivered within its SLO is what you actually buy, and it varies
by multiples across accelerators that quote similar hourly rates. berth measures
that gap so placement becomes a decision backed by physics rather than habit.

The result is often counterintuitive. On measured hardware, an L40S delivers
Llama-3-8B decode tokens 1.6x cheaper per token than an H100 PCIe at matched
concurrency: the H100 decodes about twice as fast but costs about three times as
much per hour. The premier chip is the wrong chip for that workload.

Read that with its condition attached. It is a cost result at matched batch, not
a result at matched latency. Declare a tight enough per-token deadline and the
L40S stops being feasible at all, at which point its cost per unit of compliant
work is not low, it is undefined. Which card wins is a function of your SLO, and
that is the whole reason a placement estimator exists rather than a league
table.

## What berth is, precisely

berth is an ESTIMATOR. It predicts from a model; it does not require the workload
to be running. The measurement harness that validates it (see Validation) is a
separate component: it observes real hardware to check the estimator. Keep the
distinction in mind. You use the estimator; the harness is how we earned your
trust in it.

## A quick look at functionality

berth's core is a handful of calls:

- `estimate(sig, silicon, price_hr)`: return cost_per_mtok, TTFT, TPOT for one placement, plus its placement_premium
- `PlacementClient.place(sig, policy)`: choose the cheapest placement meeting an SLO
- `PlacementClient.estimate(sig)`: score every candidate placement, filter on feasibility

## Start here

- [Get Started](quickstart.md): install and run your first estimate in two minutes
- [Tutorials](tutorials/index.md): four runnable programs, 101 through 401
- [The Physics](the-physics.md): the roofline, term by term, so you can audit it
- [Validation (P0)](validation-p0.md): how berth's predictions were checked against real hardware
- [The Placement Premium](placement-premium.md): the headline quantity and how to read it
- [Worked Examples](examples/index.md): end-to-end, reproducible from published data
- [Verify and Contribute](verify-and-contribute.md): check the estimate on your own silicon
- [API Reference](api-reference.md)

berth is a Python library with a command-line front end. `berth estimate`,
`berth premium` and `berth list` wrap the same public API, so the fastest path
to a number needs no Python.
