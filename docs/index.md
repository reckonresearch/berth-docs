# Adaptive placement infrastructure for AI inference

**To measure what compute produces, and move every workload to where it
produces most.**

Compute is sold by the hour. It is consumed as work delivered under a
deadline. No standard unit connects the two.

So there is nothing to be efficient *per*. Tokens per second can double while
a service delivers nothing sellable, because a token that arrives after its
deadline is not a token. Dollars per GPU-hour is the price of the machine, not
the cost of the work.

We built the unit, the instrument that measures it, and the agent that acts on
it.

---

## The unit

**One output token delivered inside its latency target and above a declared
quality floor.** A served token.

Everything here measures in that unit. The specification is published and
anyone may use it, including people who compete with us. A unit nobody else
may use is not a unit.

## Three tools

| | |
| --- | --- |
| **berth** | predicts what a workload costs and how fast it runs on a given accelerator, before you rent it |
| **sounding** | measures the real thing and checks the prediction |
| **pilot** | watches for change, re-decides, and proves what the change saved |

All open source, Apache-2.0. All reproducible from published traces.

```bash
pip install berth-placement
berth place --workload-class voice --model llama3-8b \
    --incumbent h100-pcie --slo-ms 800
```

---

## Adaptive placement

Deciding which chip, which provider, and at which price a model runs under a
stated p99, then moving it as prices, capacity, and traffic shift.

**It is workload-conditional.** The answer depends on your token lengths and
your concurrency, not on a benchmark average. The right chip can invert inside
a single workload as concurrency changes.

**It is bounded by your service level.** A placement that misses your p99 has
no price, not a low one. That is the distinction a price sheet cannot express,
and the reason a league table is the wrong object.

**It constantly adapts.** Prices move, models ship, capacity changes. So does
the answer, and most teams decide once and never look again.

---

## What we found

Fit the decode constant on one accelerator, then predict a different
accelerator it has never seen, on different memory technology.

**Worst held-out fold: 9.5 percent, against a 15 percent gate published before
the first run.**

The residual is the real spread between the two cards. One constant across
both costs about nine percent, which is the price of generality and is stated
rather than fitted away.

Two more from the same corpus:

**Prefill admission is serial, and not as a quirk of one server.** Effective
parallelism 1.09 under vLLM and 1.01 under SGLang. Modelling that one term
drops first-token error from roughly 65 percent to under 9.

**Decode is stack-independent to three decimals.** 0.854 under SGLang against
0.850 under vLLM, same card, same model.

Every trace is downloadable. Every number is reproducible.

[The full validation record](validation-p0.md)

---

## Arithmetic, not a black box

A closed-form roofline plus a queueing term. Roughly a page of maths, no
learned components. Every prediction ships with the physical term that
produced it and a label saying whether that term rests on a measurement or a
datasheet.

It runs in your environment. Nothing leaves your machine.

When it is wrong, [we publish that
too](https://github.com/reckonresearch/berth/blob/main/DEFECTS.md): eleven
instrument failures, what caused each, and the test that fails if it returns.
A measurement tool that has never been caught lying has not been used hard
enough.

---

## Start here

**New to this** · [Get started](quickstart.md) in two minutes ·
[Tutorials](tutorials/index.md), four runnable programs

**Evaluating it** · [The physics](the-physics.md), term by term ·
[Validation](validation-p0.md), how it was checked against hardware ·
[The placement premium](placement-premium.md)

**Deciding whether to self-host** · [Self-host or API](versus.md), both on one
axis

**Running it continuously** · [pilot](pilot.md) ·
[Declaring what to watch](declaring.md) ·
[The Holdout Protocol](holdout.md), how a saving is proven

**Contributing** · [Verify and contribute](verify-and-contribute.md) a cell we
have not measured

---

Built by [Reckon Research](https://reckonresearch.com).
