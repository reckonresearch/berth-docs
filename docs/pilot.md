# pilot

**The adaptive placement agent and router.** Watches for change, decides where
a workload should run, moves it, and proves what the move saved.

berth predicts what a placement costs. sounding checks the prediction against
hardware. pilot is the part that keeps deciding, and acts on the answer.

!!! note "On the name"
    A harbour pilot boards a vessel, guides it to berth, owns neither the ship
    nor the port, and is paid by the vessel. That is the position exactly: we
    route on behalf of the buyer, to any provider, and no provider pays us.

## What routing means here

pilot routes at the **workload level**, not the request level.

A BGP speaker routes without carrying a single packet's payload. It programs
the forwarding plane and the forwarding plane moves the traffic. pilot works
the same way: it changes the deployment configuration, and your existing
gateway, load balancer or orchestrator sends traffic to the new placement.

**This is a deliberate architectural boundary, not a limitation.**

Sitting in the request path would add latency to every call, make our
availability your availability, and compromise the only thing that makes the
measurement worth anything. A party that carries traffic cannot credibly rank
the placements it carries traffic for.

So: pilot decides, executes, and proves. It does not proxy.

## The loop

**Watch.** Model registries by commit, serving-stack releases, provider
prices, and the corpus. Four sources, polled on a schedule.

**Decide.** When an input moves, re-run the placement decision. A placement
that misses your bound is excluded rather than ranked cheaply, because its
cost per served token is undefined rather than high.

**Move.** Where your declared autonomy policy permits, pilot executes the
change: it opens a pull request against your deployment configuration and
merges it, and your delivery pipeline moves the workload. Where the policy
does not permit, it opens the pull request and stops.

**Prove.** Under the [Holdout Protocol](holdout.md), a declared slice of
traffic stays on the old placement so the difference can be measured rather
than claimed.

Then it starts again. A placement adapted last month can decay again, and the
loop has no end state.

## Autonomy

A system that only proposes is a recommendation engine with extra steps. If
every change waits on someone noticing a pull request, the loop is as slow as
the human in it.

**Authority is declared once, in advance, in your repository.** Not per
change. You commit an autonomy policy and that commit is the authorization: a
change you reviewed, in your history, under your credential. The human
decision moves from approving a change to approving a class of change under
stated conditions, which is the decision a human is good at.

```yaml
    autonomy:
      level: window                    # frozen | propose | window | execute
      windows: [[6, 2, 5]]             # Sunday 02:00 to 05:00 UTC
      min_margin_to_execute: 0.30
      min_hours_between_moves: 168
      max_moves_per_pass: 1
      blackout_dates: ["2026-12-20"]
```

| Level | Behaviour |
| --- | --- |
| `frozen` | nothing, not even a proposal |
| `propose` | open a pull request and stop. **The default** |
| `window` | execute inside declared windows, propose outside them |
| `execute` | execute whenever the guardrails pass |

## The guardrails

Six, and every one can only refuse. There is no path where a guardrail permits
something the autonomy level did not already allow, and that property is
pinned by test.

**Blast radius.** A cap on executions per pass. One bad corpus update must not
be able to move a fleet.

**Rate limit.** A minimum interval between moves for one class. A placement
that moves twice in a week is oscillating, not adapting.

**Blackout dates.** A freeze is a real thing in most organisations, and a
placement system that ignores one gets uninstalled.

**A higher bar to act than to ask.** The margin required to move production
unattended exceeds the margin required to open a pull request. That gap is
deliberate.

**A measurement requirement.** An unmeasured placement is never executed by
default, regardless of how good the estimate looks. A spec-sheet prior has
been observed above 40 percent error in this corpus.

**Rollback, which outranks savings.** A move that breaches the service level
reverts regardless of what it saved, because a placement that misses its bound
is not a cheap placement. A move whose observed cost exceeds its estimate by
more than the tolerance also reverts, and the cell is flagged for measurement.

## The kill switch

Delete `.berth/classes.yaml`. Nothing runs.

That is deliberate. An autonomy system whose off switch requires contacting
the vendor is not one anyone should install.

## The control plane

pilot has a control plane: the surface where you see what it is doing.

`.berth/STATUS.md`, rewritten into your repository on every pass. Every class,
what it is holding, what the estimator recommends **now** rather than when the
last proposal opened, executions and their outcomes, settled periods, and when
each source was last polled.

A file rather than a dashboard. Version controlled, so its history is the
history of the account. Renders wherever your repository is browsed. No login,
nothing to remember to visit.

## What it never does

**It never proxies a request.** No data plane, no traffic, no latency added to
any call.

**It never writes outside the paths you declared.** Enforced in code rather
than in a token scope, because a scope is a promise about configuration and
this is a promise about code.

**It never merges without a stated policy.** `merge()` refuses
unconditionally; `merge_proposal` requires the autonomy rule that permitted it
and writes that rule into the merge commit. A change that landed unattended
says on its face which rule allowed it, and reverting it is reverting a
commit.

**It never owns capacity.** No resale, no reserved blocks, no inventory. The
moment inventory exists there is a reason to route toward it.

## Why it usually says nothing

Most passes produce no proposal and no execution. pilot stays quiet when the
answer did not change, when the improvement sits inside the confidence band,
when the gain is real and smaller than the cost of collecting it, when the
same proposal is already open, and when you rejected this move and the
cooldown has not elapsed.

**An agent that acts every week gets muted, and a muted agent is worse than
none because it looks like coverage.** The suppression reasons are recorded
and reported, and they are the more informative half.

## The daily line

Silence is right about action and wrong about communication. A team that hears
nothing for three weeks does not conclude all is well, they conclude it
stopped running.

```
pilot  2026-08-12
================================================================

Checked 4 workload classes against 5 sources. One placement moved.
Every other class is holding its service level on the cheapest
placement we can find.

  MOVED
  voice-agent-prod   h100-pcie to mi300x   31% margin, executed 02:14 UTC

  SAVINGS
  realized, running total       $      32,142
    last 24 hours               $         575
  0% of that is VERIFIED. Every figure above is ESTIMATED.
```

[Receipts and the ledger](receipts.md)

## Running it

```bash
pip install berth-placement
pilot --classes .berth/classes.yaml --state .berth/state.json
```

Shadow by default: it decides everything and changes nothing. That is the
correct first posture, and the kill criterion is computable from shadow data
alone. Fewer than one trigger in four producing a change that clears the
hurdle means the trigger set is wrong.

[Declaring what to watch](declaring.md) · [Internals](pilot-internals.md)
