# The served token

**The unit of account for AI inference.** One output token delivered inside its
service-level objective and above its quality floor.

Abbreviated **SVT**.

Everything that follows is denominated in it. The price of a placement is
**dollars per million served tokens**, written `$/MSVT`:

$$
C \;=\; \frac{\text{total spend on the placement}}
              {\text{served tokens delivered}}
$$

The specification is published and anyone may use it, including people who
compete with us. **A unit nobody else may use is not a unit.**

## Why a new unit was needed

Every existing metric drops a leg.

| | Drops |
| --- | --- |
| Dollars per GPU-hour | the output entirely |
| Dollars per raw token | the deadline and the quality floor |
| Benchmarks | the price |
| Uptime SLAs | everything except the deadline |
| Goodput | the price and the acceptance test |

Each is defensible on its own and none is commensurable with the others, which
jointly guarantees that capital cannot compare two placements.

**The served token is goodput with the invoice attached and the quality floor
declared.** That is the difference between a systems metric and a unit of
account.

The precedent is not new. Since 1988 the Transaction Processing Performance
Council has priced useful work under an enforced deadline with an audited cost
basis. Electricity had lamp-months before it had the kilowatt-hour, and
freight had negotiated rates before it had the container. **The unit comes
first. Everything that makes a market comes after.**

---

## Why the denominator counts served tokens, not tokens

A token that arrives after its deadline is not a cheap token. **It is not a
token.**

A voice agent with an 800 ms budget for its first token gets nothing usable
from a response that starts at 1,400 ms. The user has already noticed the
silence. The compute was spent, the tokens were generated, and none of it
participated in the interaction it was generated for.

The same is true of the quality floor. A response that fails the acceptance
test the customer declared is work that was paid for and cannot be sold.

So the denominator counts only output that met both conditions. Three
consequences follow, and each one is a place where a conventional metric gives
the wrong answer:

**A placement that misses the bound has no cost per served token.** Not a high
one. Undefined, because the denominator is zero. This is why a price sheet
cannot answer a placement question: the cheapest row is frequently the one
that cannot serve the workload at all.

**Spend includes idle.** A rented node is paid for whether or not a request
arrives. A placement running at 15 percent utilisation is more expensive per
unit of work than the hourly rate suggests, and the number should say so.

**A retried request carries its full spend and only its final compliant
output.** Both attempts cost money. One of them produced something sellable.

---

## What it replaces

| Metric | What it measures | Why it is not enough |
| --- | --- | --- |
| $/GPU-hour | the price of the machine | says nothing about what the machine produces |
| tokens/second | raw generation rate | can double while delivering nothing inside the bound |
| $/million tokens | price per token generated | counts late and failed output as though it were sellable |
| **$/MSVT** | **the cost of work you can actually use** | |

The first three are all available today, from every provider, and none of them
can distinguish a placement that works from one that does not.

---

## A worked example

Two placements, same model, same workload, same 800 ms first-token bound.

| | Placement A | Placement B |
| --- | --- | --- |
| Spend over the window | $2,000 | $2,000 |
| Output tokens generated | 620 M | 500 M |
| Compliance with the bound | 41% | 96% |
| **Served tokens (SVT)** | **254 M** | **480 M** |
| $/Mtok generated | $3.23 | $4.00 |
| **$/MSVT** | **$7.87** | **$4.17** |

On tokens generated, A looks 19 percent cheaper. On served tokens, B is
nearly twice as cheap.

A is a placement running above its comfortable concurrency: it generates more
and delivers less. The conventional metric rewards exactly the configuration
that fails the workload, which is not an edge case. It is what happens
whenever a node is pushed for throughput.

---

## The two bounds

The unit is defined against a service level, so the service level has to be
declared before anything is measured. Two bounds cover most workloads:

**Time to first token**, which governs whether an interaction feels
responsive. A voice agent lives or dies here.

**Time per output token**, which governs whether generation keeps pace with
reading or speaking once it starts.

Both are stated as percentiles, not means. A mean hides the tail, and the tail
is what a user notices.

---

## Provenance

Every figure derived from this unit carries a label saying where it came from:

| | |
| --- | --- |
| `MEASURED` | observed on hardware, with traces published |
| `FITTED` | recovered from measured traces |
| `CONFIG` | a declared input, such as a price or a bound |
| `SIM` | produced by replay or simulation |
| `HYPOTHESIS` | not yet tested |

**A number labelled MEASURED that contains a simulated input is the one
unrecoverable error.** Everything else can be corrected in public; that one
destroys the reason to believe any of the rest.

---

## Reading and citing

The served token is specified in full, including the sampling protocol for the
quality floor and the accounting rules for retries and partial output, in the
CUW-SLO specification. The argument for why it is needed, and the historical
cases it follows, are in **The Missing Meter**:
[themissingmeter.org](https://themissingmeter.org).

Cite it as the served token, SVT. Prices in this unit are written `$/MSVT`,
dollars per million served tokens.

If you use the unit, we would rather you used it correctly than used our tool.
The two are separable on purpose.

- [The physics](the-physics.md), how a prediction in this unit is produced
- [Validation](validation-p0.md), how those predictions were checked
- [The Holdout Protocol](holdout.md), how a change measured in this unit is
  proven to have saved money
