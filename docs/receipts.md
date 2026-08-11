# Receipts and the ledger

Two different questions. A **receipt** settles one measurement period under the
[holdout protocol](holdout.md). The **ledger** says what everything so far has
been worth.

## Receipts

A period closes and produces one conforming record: the declaration verbatim,
both legs, all six stationarity checks with their values, the settlement, and
a pointer to the traces.

```python
from berth.receipt import build_receipt, render

receipt = build_receipt(declaration, baseline, treatment,
                        trace_pointer="s3://traces/2026-03/voice",
                        serving_stack={"baseline": "vLLM 0.9.1",
                                       "treatment": "vLLM 0.9.1"},
                        warmup_excluded=3140)
print(render(receipt))
```

### What the settlement does that a naive one would not

**The holdout cost is its own line.** The held-out slice runs on the worse
placement by design and costs real money. Netting it silently would make every
receipt an overstatement.

**A negative period carries forward.** The loss offsets the next period before
the share is applied. Capturing a share of every gain and none of any loss is
an asymmetry a counterparty is right to refuse, and carrying it costs nothing
when the recommendation was correct.

**A tripped circuit breaker voids rather than counting as a loss.** A
placement that never held the bound was never a measurement.

**Billing uses the conservative end of the interval.** The upper bound on
compliance gives the lower bound on cost per served token, which gives the
smaller saving.

### Periods that produce no invoice

Stationarity failing, the breaker tripping, or a negative saving all produce a
receipt and no invoice. **A period that cannot be billed is a result, and
publishing it is what makes the ones that can be billed credible.**

## The ledger

```python
from berth.ledger import ClassEconomics, build_ledger, daily_report

ledger = build_ledger(agent_state, economics, decisions=decisions,
                      receipts=receipts)
print(daily_report(ledger=ledger, classes=classes, sources_polled=5,
                   triggers=7, proposals=1))
```

### Three quantities, never summed

**Realized** is a merged proposal accruing from the moment it merged. The
workload moved.

**Available** is an open pull request nobody merged. Nothing has been saved,
and it accrues to no one while it waits. Reporting it as saved would be the
single most tempting lie this system could tell, so it is reported per hour
rather than as a total: stating a cumulative figure would make waiting look
like earning.

**Foregone** is a declined proposal. Not a loss, and not free. It is what the
constraints behind that decision cost per hour, reported so whoever set them
can see the price.

### What is never counted

A class checked and left alone saved nothing. The gap between its placement
and the worst in the fleet is not a saving, it is a comparison nobody was
going to make. **Systems that report that number are reporting their own
existence as value.**

### Windows do not double count

A change merged in March did not earn its March money again in August. The
day, week, month and quarter figures accrue inside their window, which is why
they differ rather than repeating one total.

### Provenance

Every figure reads **ESTIMATED** until a holdout period settles it, at which
point it becomes **VERIFIED** and names the receipt. The report states what
share is which, starting at zero and stated as zero rather than omitted.

An estimate presented as a measurement is the one unrecoverable error in this
project, and this is where it would happen.

### A class with no declared volume is skipped

Not defaulted. A percentage is not money until somebody says how much work
there is, and inferring it would let the reported saving be adjusted by
changing an assumption.
