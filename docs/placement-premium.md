# The Placement Premium

The placement premium is berth's headline quantity: how much more a workload
costs where you were about to run it than on the cheapest accelerator that still
meets the same p99 latency target.

## Definition

    premium = cost($/Mtok) of chosen placement
              / cost($/Mtok) of cheapest feasible placement at equal p99

A premium of 1.0 means you already chose the cheapest option that clears your
SLO. A premium of 1.6 means the placement you chose costs 60 percent more than
the cheapest feasible alternative, and equivalently that the alternative serves
the same workload for 37.5 percent less. Those are the same fact stated from the
two ends; quote whichever direction you mean, and do not mix them.

## The measured result

For Llama-3-8B decode, measured on both cards, at matched concurrency and with
no per-token latency floor declared:

| Concurrency | L40S $/Mtok | H100 PCIe $/Mtok | Premium of the pricier choice |
|---|---|---|---|
| batch 1 | 6.07 | 9.91 | 1.63x |
| batch 8 | 0.885 | 1.40 | 1.59x |
| batch 32 | 0.317 | 0.535 | 1.69x |

The L40S is cheaper at every batch size, by 1.57 to 1.69 times. The mechanism:
the H100 PCIe decodes about 2.1 times faster per stream, but costs about 3.4
times more per hour, so on a per-token basis the cheaper card wins.

## The condition that table is under

That table compares cost at matched batch. It is not a comparison at equal p99,
and the two cards are nowhere near equal p99: at batch 1 the L40S emits 45.3
tokens per second per stream against the H100's 93.9.

The distinction decides the answer. Declare a floor of 100 output tokens per
second, the kind an interactive voice agent needs, and neither card clears it at
batch 1, so the cheaper card's cost per unit of compliant work is undefined
rather than low. Loosen the floor and the L40S wins by the margins above. Between
those two regimes the ranking inverts, and no metric that omits the deadline can
see the inversion happen.

So the premium is defined at equal p99, and the table above is the special case
where the p99 constraint is not binding. When you run it on your own workload,
declare the SLO. That is the field that decides which placements are even in the
feasible set.

## Why it is not a constant

The premium depends on the workload and the SLO. A latency target tight enough to
require the faster card changes the feasible set; a throughput-bound workload at
high batch shifts the balance again. A tool that returned a single fixed premium
would be telling you there was nothing to decide. The premium varies precisely
because placement is a real decision, and that variation is the product.

## How to read it for your own workload

Run `berth premium` on your actual model, batch and SLO, or call
`client.estimate(sig)` and read `placement_premium` off each candidate. If the number is
close to 1, your current placement is already near-optimal for that workload. If
it is well above 1, berth is pointing at savings available at the same latency,
and the estimate tells you which accelerator captures them before you rent
anything.

## A caveat stated plainly

The measured premium above is decode and cost driven, at provider list prices on
one day. It is a ratio, so it is robust to uniform price changes but not to
differential discounting: a reserved-capacity deal on one card can move it. And
it is a cost quantity, not a latency-under-load guarantee; first-token latency
under heavy concurrency is modeled separately. Read the premium as what it is: a
sharp, auditable estimate of cost headroom, not a contract.
