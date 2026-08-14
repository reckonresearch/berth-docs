# Executing a placement change

pilot decides where a workload should run. This page is how it moves it, and
what stops it moving something it should not.

## The authorization model

Nothing here is granted at the moment of a change. It is granted once, in
advance, by a commit you make to your own repository.

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
    autonomy:
      level: window
      windows: [[6, 2, 5]]
      min_margin_to_execute: 0.30
```

**That commit is the authorization.** It is in your history, under your
credential, reviewable by your team, and revocable by deleting a file. pilot
has no standing permission to write to your default branch. It has permission
to merge a pull request it opened, for a class whose declared policy allows
it, when every guardrail passes.

This is the same model as Terraform auto-apply, ArgoCD auto-sync and
Dependabot auto-merge. It works because the human decision moves from
"approve this change" to "approve this class of change under these
conditions", and the second is the decision a human is actually good at.

## The four levels

| Level | What pilot may do |
| --- | --- |
| `frozen` | nothing. Not a proposal, not an execution |
| `propose` | open a pull request and stop. **The default for any class without a policy** |
| `window` | execute inside declared windows, propose outside them |
| `execute` | execute whenever every guardrail passes |

A class with no `autonomy` block is `propose`. A policy nobody thought about
behaves like a policy that says ask first, because the failure mode of a
permissive default is a workload moving at three in the morning on a four
percent estimate.

## What happens on an execution

1. pilot re-runs the placement decision against the changed input.
2. The margin is checked against the confidence band plus the switching cost.
   If it does not clear that, nothing happens at all.
3. Every guardrail is evaluated. Any one of them can refuse.
4. A pull request is opened against the declared config path, carrying the
   diff, both estimates with their band, how much rests on measurement, the
   bound it was optimised for, and a link to the traces.
5. If the policy permits, pilot merges it. **The merge commit contains the
   policy rule that allowed it.**
6. Your delivery pipeline picks up the config change and moves the workload.
7. The holdout leg stays on the old placement, so the saving can be measured.

**Steps 4 through 6 are the routing.** pilot programs the infrastructure; the
infrastructure moves the traffic. No request passes through us at any point.

## The guardrails, in detail

### Blast radius

`max_moves_per_pass`, default 1.

A single pass sees every class. Without a cap, one bad corpus update or one
mispriced provider could move every workload you have in one pass. The cap
means the worst case is one wrong move rather than a fleet.

### Rate limit

`min_hours_between_moves`, default 168.

Two placements a few percent apart trade the lead every time a rate card
moves. Each swap costs real money while the estimate that justified it was
never wrong. **A placement that moves twice in a week is oscillating, not
adapting**, and the rate limit is what makes that distinction enforceable
rather than aspirational.

### Blackout dates

`blackout_dates`, default empty.

A change freeze is real. A placement system that moves a workload during one
gets uninstalled, and rightly.

### A higher bar to act than to ask

`min_margin_to_execute`, default 0.25.

The hurdle for a proposal is the confidence band plus the switching cost. The
hurdle for an unattended execution is that, and this on top. A margin can
clear the first and not the second, in which case pilot opens a pull request
and waits.

The gap is deliberate. **The bar for moving production without being asked
should exceed the bar for asking.**

### A measurement requirement

`allow_unmeasured`, default false.

An unmeasured placement is never executed unattended, regardless of how good
the estimate looks. A spec-sheet prior has been observed above 40 percent
error in this corpus, and moving production onto one without a human looking
is the single worst thing this system could do.

Turning it on requires stating a reason in the declaration. The parser
refuses the flag without one.

### Rollback

`rollback_after_breaches`, default 1.

Two triggers, and the first outranks everything:

**A breached service level reverts the move regardless of what it saved.** A
placement that misses its bound is not a cheap placement. It is not a
placement.

**An estimate that did not survive contact reverts too.** If the observed cost
exceeds the estimate that justified the move by more than the tolerance, the
model was wrong about this cell, and acting on it further would compound the
error. The cell is flagged for measurement.

The tolerance is not zero. An estimate a few percent optimistic is an
estimate, not a failure, and reverting on noise would make the system
oscillate through its own rollback path.

## Turning it off

```bash
rm .berth/classes.yaml
```

Nothing runs. No API call, no support ticket, no notice period.

An autonomy system whose off switch requires contacting the vendor is not one
anyone should install, and saying so in the documentation is cheaper than
being asked.

## What an executed change looks like afterwards

A squashed merge commit in your repository, on your default branch, containing
the configuration diff and a message naming the policy rule that permitted it.

Reverting the move is reverting that commit.

The execution is recorded in `.berth/STATUS.md` with its margin, its basis,
and its outcome, and it enters the [ledger](receipts.md) as realized saving
accruing from the moment it merged.
