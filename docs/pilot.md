# Pilot

**berth tells you where a workload should run. Pilot keeps it there.**

A placement is not a decision, it is a position that decays. A model version
ships and the byte profile changes. A provider moves a rate. A cell lands in
the corpus and a prior becomes a measurement. The answer that was right last
month is not right this month, and nobody checks, because checking means
re-measuring and nothing says when it is worth doing.

Pilot is the control plane that does the checking.

!!! note "On the name"
    A harbour pilot boards a vessel, guides it to berth, owns neither the ship
    nor the port, and is paid by the vessel. That is the position exactly: we
    route on behalf of the buyer, to any provider, and no provider pays us.

    **Pilot is the control plane. The agent is the component inside it that
    notices.** They are not synonyms, and the distinction matters when reading
    the source: `berth/agent.py` is one of six modules.

## What it never does

**It never touches a request.** No proxy, no gateway, no traffic. Your own
infrastructure moves the workload; Pilot decides where and proves what the
change saved.

**It never merges.** `GitHubClient.merge()` exists solely to raise, so the
refusal is explicit rather than an absence. It cannot write to a default
branch and cannot touch a path you did not declare. Those are enforced in code
rather than in a token scope, because a scope is a promise about configuration
and this is a promise about code.

**It never owns capacity.** No resale, no reserved blocks, no inventory. The
moment inventory exists there is a reason to route toward it.

## The parts

| Module | What it does |
| --- | --- |
| `berth.place` | the decision record: what is recommended, what it beat, by how much, and whether that clears the uncertainty |
| `berth.agent` | watch, detect, re-estimate, propose. And decide when to stay quiet |
| `berth.watch` | model registries, provider prices, serving-stack releases, corpus additions |
| `berth.github` | opens the pull request and refuses everything else |
| `berth.holdout` | the protocol under which a saving can be proven |
| `berth.receipt` | settles a measurement period into a conforming record |
| `berth.status` | one page in your repository showing every class at once |
| `berth.ledger` | what this has been worth, and what it has not |

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

The declaration lives in **your** repository, not ours. Changing what the agent
may watch is a pull request you review, the history of what was declared and
when is in your git log, and revoking us is deleting a file.

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

Most triggers should produce no proposal. Pilot stays quiet when:

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
PILOT  2026-08-11
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
from the moment it merged. *Available* is an open pull request nobody has
merged, which has saved nothing. *Foregone* is a proposal you declined,
reported because a constraint has a price and the price should be visible to
whoever set it.

A class we checked and left alone contributes nothing. The gap between its
placement and the worst in the fleet is not a saving, it is a comparison
nobody was going to make.

## Running it

```
pip install berth-placement

# One pass. Shadow by default: it decides everything and opens nothing.
berth pilot --classes .berth/classes.yaml --state .berth/state.json

# With the report.
python -m bench.pilot_shadow --status --report
```

Shadow is the correct first posture. Read what it would have said for a while
before letting it say anything, and the kill criterion is computable from that
alone: **fewer than one trigger in four producing a change that clears the
hurdle means the trigger set is wrong.**

## Where a saving gets proven

A pull request that improves an estimate is not a saving. Proving one needs a
declared baseline, a held-out slice of traffic, and an agreed method for
comparing them. That is the [Placement Holdout Protocol](holdout.md), and it
is published openly so a counterparty can check it before agreeing to be
measured.
