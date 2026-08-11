# pilot internals

How the adaptive placement agent works, for anyone reading the source.

The loop, for anyone reading the source or running it themselves. Everything
here is in `berth/` and Apache-2.0.

## `berth.watch`: noticing

Four sources, each a pure function of a fetched payload and stored state, so
the network layer can be swapped for a fixture and the logic tested without
polling anything.

| Source | What moves |
| --- | --- |
| Model registries | the repository commit, not the timestamp. A README edit is not a placement event |
| Serving stacks | a release tag. vLLM 0.5.5 to 0.25 was worth 1.48x at batch 1 and 2.70x at batch 32 on one card |
| Provider prices | beyond an epsilon. Rates move by fractions of a percent constantly and none of it changes a placement |
| Corpus cells | a prior becoming a measurement, the only trigger where the answer changes without anything in your world changing |

**Three properties worth knowing.** The first sight of a source fires nothing,
because we do not yet know whether it changed. An unreachable source is not a
change, so an outage does not fire the agent. And the detector polls once per
pass rather than once per class, since twenty classes watching one model would
hit a registry twenty times.

**Unreachable is recorded, not swallowed.** A source that has never succeeded
must not look like one that succeeded and did not change. The first shadow
deployment of this system had every remote source dead behind a certificate
failure and the output read as a normal quiet pass, which is the worst failure
a monitor has.

The price source is a callable, so point it at your own negotiated rates. Your
contracted rate decides your placement and it is usually not the published one.

## `berth.agent`: deciding whether to speak

```python
from berth.agent import AgentState, run
result = run(classes, resolve_decision, detect_triggers,
             state=AgentState(), shadow=True)
```

A trigger becomes a proposal only if the answer changed **and** the margin
clears the hurdle:

```
hurdle = confidence band + switching cost
```

Two terms because they fail differently. A gain inside the band might not
exist. A gain smaller than the switching cost exists and is not worth
collecting. Without the second the loop is an oscillator waiting for a price
feed: two placements a few percent apart trade the lead every time a rate card
moves, and each swap costs real money while the estimate was never wrong.

**State stops it repeating.** The same proposal is not opened twice. A
rejection suppresses it for ninety days. A new proposal supersedes an open one,
so nobody holds two pull requests pointing different ways. Several triggers
reaching one conclusion in a pass produce one proposal.

**Two kill criteria.** Proposal rate below one in four means the trigger set is
wrong. Merge rate consistently low means the arithmetic is right and nobody
wants it, which is a product problem rather than a threshold one.

## `berth.github`: the smallest surface that opens a pull request

Read a file, write a branch, open a pull request. It **cannot** merge, cannot
write to a default branch, and cannot touch an undeclared path. Enforced in
code rather than in a token scope: a scope is a promise about configuration
and this is a promise about code. `merge()` exists solely to raise.

An edit matching zero lines or two is refused rather than guessed. Zero means
the estimate is stale and re-estimating is correct; two is ambiguous. An agent
that guesses at a deployment file has its access revoked once.

See [gateway templates](https://github.com/reckonresearch/berth/tree/main/contrib/gateway)
for running the holdout assignment at Envoy, NGINX, ALB or Istio, which needs
no application code.

## `berth.status`: one place to look

`.berth/STATUS.md`, rewritten every pass into your repository. Every class,
what it holds, what the estimator recommends **now** rather than when the
proposal opened, open proposals linked, settled periods, and when each source
was last polled.

A file rather than a dashboard: version controlled, so its history is the
history of the account; renders wherever the repository is browsed; no login;
and deleting it is how you turn it off, which the page says on itself.

A recommendation resting on unmeasured silicon is lifted out of the table into
its own callout, because a table column is where it would be missed.

## `berth.quantities`: types where two numbers share a shape

```python
from berth.quantities import Ceiling
peak = Ceiling.datasheet("h100-pcie", bandwidth_tbs=2.0)
peak.efficiency_of(measured_tbs=1.7)          # 0.85
```

Two instrument defects in this project were a float handed to a function
expecting a different float. A microbenchmark passed where a datasheet peak
was expected reported a clean file as contaminated. An efficiency fitted on one
card's timings while dividing by another card's bandwidth reported 109.5
percent error with a fitted value of 1.761.

`Ceiling` carries its device and its source. `efficiency_of` refuses a
microbenchmark denominator and refuses a measurement from another device.
Carrying a constant between cards is a separate named operation,
`transfer_efficiency`, which refuses any value that is not a fraction of a
peak.

Both defects were caught only because the output was absurd. At 0.62 instead
of 1.761 the second would have been published as a finding.
