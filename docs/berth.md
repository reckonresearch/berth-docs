# berth

**The estimator.** Predicts what a workload costs and how fast it runs on a
given accelerator, before you rent it.

```bash
pip install berth-placement
berth place --workload-class voice --model llama3-8b \
    --incumbent h100-pcie --slo-ms 800
```

You describe the workload: model, concurrency, prompt and output lengths, and
the latency bound you have to hold. berth returns dollars per million served
tokens, time to first token, and time per output token, for every placement it
knows about.

A placement that misses your bound is **excluded**, not ranked cheaply. Its
cost per compliant token is not high, it is undefined, and ranking it first is
what a price sheet does.

## It is arithmetic

A closed-form roofline plus a queueing term. Roughly a page of maths, no
learned components, no training data.

Every number is auditable by hand, and every prediction ships with the
physical term that produced it plus a label saying whether that term rests on
a measurement or a datasheet. Six cells are measured today; everything else in
the fleet is a spec-sheet prior and says so on every line.

[The physics, term by term](the-physics.md)

## What it predicts, and how well

Fit the decode constant on one accelerator, then predict a different
accelerator it has never seen, on different memory technology: **worst
held-out fold 9.5 percent, against a 15 percent gate published before the
first run.**

[The full validation record](validation-p0.md)

## What it does not do

**It does not measure.** It predicts from a model, and the workload does not
have to be running. The harness that checks it, [sounding](sounding.md), is a
separate component and never imports the estimator: evidence that cannot be
separated from the thing it evaluates is not evidence.

**It does not act.** [pilot](pilot.md) is the agent that watches for change,
re-decides, and proves what a change saved.

**It does not touch your traffic.** It runs in your environment and nothing
leaves your machine.

## Start here

- [Get started](quickstart.md), a first estimate in two minutes
- [Tutorials](tutorials/index.md), four runnable programs
- [The placement premium](placement-premium.md), the headline quantity
- [Silicon and models](silicon-and-models.md), what is in the fleet
- [API reference](api-reference.md)
