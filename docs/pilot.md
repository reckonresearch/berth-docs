# pilot

**The adaptive placement agent.** Watches for change, adapts the placement,
and proves what the change saved.

berth decides where a workload should run. sounding checks that the decision
was right. pilot is the part that keeps deciding, because the answer does not
hold still.

A placement is not a decision, it is a position that decays. A model version
ships and the byte profile changes. A provider moves a rate. A cell lands in
the corpus and a prior becomes a measurement. The answer that was right last
month is not right this month, and nobody checks, because checking means
re-measuring and nothing says when it is worth doing.

pilot does the checking, and adapts.

!!! note "On the name"
    A harbour pilot boards a vessel, guides it to berth, owns neither the ship
    nor the port, and is paid by the vessel. That is the position exactly: we
    route on behalf of the buyer, to any provider, and no provider pays us.

## Why static placement fails

Almost every team decides once. A benchmark, a spec sheet, a conversation, and
then the configuration stays until something breaks.

Meanwhile the inputs move continuously. Model versions ship monthly or faster.
Provider rates change. Serving stacks cut releases, and a stack upgrade alone
has been measured at over twice the throughput on identical hardware. Traffic
shape shifts as a product grows.

**A decision made once against inputs that never stop moving is wrong within
weeks, and stays wrong silently.** That is the gap adaptive placement closes,
and pilot is the part that closes it.

## What it does

**Watches.** Model registries, serving-stack releases, provider prices, and
the corpus. Four sources, polled on a schedule.

**Adapts.** When something moves, it re-runs the placement decision against
the new inputs. Not on a review cycle and not when somebody remembers: when
the input changes.

**Proposes.** When the answer changes by enough to matter, it opens a pull
request against your deployment configuration with the diff and the evidence
attached. The workload moves when you merge it.

**Proves.** Under the [Holdout Protocol](holdout.md), a declared slice of
traffic stays on the old placement so the saving can be measured rather than
claimed.

Then it starts again. A placement that was adapted last month is a placement
that can decay again, and the loop has no end state.

## What it never does

**It never touches a request.** No proxy, no gateway, no traffic. Your own
infrastructure moves the workload; pilot decides where and proves what the
change saved.

**It never merges.** `GitHubClient.merge()` exists solely to raise, so the
refusal is explicit rather than an absence. It cannot write to a default
branch and cannot touch a path you did not declare. Enforced in code rather
than in a token scope, because a scope is a promise about configuration and
this is a promise about code.

**It never owns capacity.** No resale, no reserved blocks, no inventory. The
moment inventory exists there is a reason to route toward it.

## The control plane

pilot has a control plane: the surface where you see what it is doing.

`.berth/STATUS.md`, rewritten into your repository on every pass. Every class,
what it is holding, what the estimator recommends **now** rather than when the
proposal opened, open proposals linked, settled periods, and when each source
was last polled.

A file rather than a dashboard. Version controlled, so its history is the
history of the account. Renders wherever your repository is browsed. No login,
nothing to remember to visit, and deleting it is how you turn it off.

## How you use it

**Once, at setup.** Commit `.berth/classes.yaml` declaring what to watch, and
grant write access to a branch. That is the whole integration.

```yaml
version: 1
repo:
  allowed_paths:
    - deploy/voice.yaml
classes:
  - name: voice-agent-prod
    model_id: NousResearch/Meta-Llama-3-8B
    model: llama3-8b
    running_on: h100-pcie
    config_path: deploy/voice.yaml
    slo:
      metric: p99_ttft_ms
      bound_ms: 800
    workload:
      concurrency: 8
      prompt_tokens: 512
      output_tokens: 128
      mtok_per_hour: 12.0
```

The declaration lives in **your** repository. Changing what pilot may watch is
a pull request you review, the history of what was declared is in your git
log, and revoking it is deleting a file.

[Declaring what to watch](declaring.md)

**Then nothing.** It runs on a schedule. Most passes produce no pull request,
and that is the design rather than a failure.

**When a pull request arrives**, read it and merge or close. Merging is the
deploy: your existing delivery picks up the configuration change. Closing is a
rejection, and it is respected for ninety days, because a customer saying not
this quarter means not this quarter.

## Why a pull request and not an alert

An alert creates work. A diff with the evidence attached is work already done,
arriving in a review workflow you already have, where it can be discussed,
amended, rejected or merged.

It also leaves the decision in your history rather than in a vendor dashboard,
and reverting it is reverting the commit.

The body carries the config diff, both estimates with their confidence band,
how much of the answer rests on measurement rather than a datasheet, the bound
it was optimised for, and a link to the traces. **You should be able to
disagree with the recommendation from the body alone.**

## Why it usually says nothing

Most triggers should produce no proposal. pilot stays quiet when:

- The answer did not change
- The improvement sits inside the confidence band, so it is not
  distinguishable from noise
- The improvement clears the band but not the cost of moving. The gain is real
  and smaller than collecting it
- The same proposal is already open
- You rejected this move and the cooldown has not elapsed

**An agent that opens a pull request every week gets muted, and a muted agent
is worse than none because it looks like coverage.** The suppression reasons
are recorded and reported, and they are the more informative half.

## The daily line

Silence is right about action and wrong about communication. A team that hears
nothing for three weeks does not conclude all is well, they conclude it
stopped running.

Every pass produces a short report: what was checked, against how many
sources, whether any were unreachable, and what the changes so far have been
worth.

```
pilot  2026-08-11
================================================================

Checked 4 workload classes against 5 sources. Every one is holding
its service level on the cheapest placement we can find. Nothing to do.

  SAVINGS
  realized, running total       $      32,142
    last 24 hours               $         575
    last 7 days                 $       4,025
  0% of that is VERIFIED. Every figure above is ESTIMATED.
```

**Three quantities, never summed.** *Realized* is a merged proposal accruing
from the moment it merged. *Available* is an open pull request nobody merged,
which has saved nothing. *Foregone* is a proposal you declined, reported
because a constraint has a price and the price should be visible to whoever
set it.

[Receipts and the ledger](receipts.md)

## Running it

```bash
berth pilot --classes .berth/classes.yaml --state .berth/state.json
```

Shadow by default: it decides everything and opens nothing. That is the
correct first posture. Read what it would have said for a while before letting
it say anything, and the kill criterion is computable from that alone: fewer
than one trigger in four producing a change that clears the hurdle means the
trigger set is wrong.

[Internals](pilot-internals.md)
