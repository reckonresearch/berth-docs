# Self-host or API

**The first adaptive placement decision.** Before which chip and which
provider comes whether to run it yourself at all.

Most teams calling a hosted endpoint have never computed what the same work
would cost them, because the number does not exist. What an API charges per
million tokens is published. What a rented GPU would deliver under your
latency bound is not, and computing it is what this tool does.

```
berth versus --model llama3-8b --offers offers.json \
    --prompt 1469 --output 128 --slo-ms 800 \
    --requests-per-hour 40000 --engineering-per-hour 12
```

```
SELF-HOST OR API
====================================================================
  model        llama3-8b
  workload     1469 prompt, 128 output, concurrency 8
  bound        first token under 800 ms
  volume       40,000 requests an hour

  SELF-HOST    1 node of 1x mi300x at $2.80/hr plus $12.00/hr engineering
               $0.370 per 1,000 requests, running at 98% of capacity
               MEASURED

  API                     $/1k req   in $/Mtok  out $/Mtok      p99 ttft
  Groq                       0.084        0.05        0.08         260ms
  Together                   0.287        0.18        0.18         420ms
  DeepInfra               excluded   p99 first-token latency 1400 ms
                                     exceeds the 800 ms bound, so cost
                                     per compliant token is undefined

  One node saturates at 40,745 requests an hour. Past that the cost
  per request stops falling, because the next request buys a second node.

  VERDICT      use Groq, 4.4x cheaper than self-hosting at this volume
```

## The axis is cost per compliant token

Not cost per token. A provider whose p99 first-token latency exceeds your
bound delivers nothing sellable at any price, so its cost per compliant token
is not high, it is **undefined**.

That distinction is the whole reason a price sheet cannot answer this. The
cheapest row on most rate cards is the one that cannot serve the workload.

## Four things a naive comparison gets wrong

**Utilisation.** A rented GPU is paid for whether or not a request arrives; an
API is paid per token. Below some traffic level self-hosting is more expensive
at any price per hour, and the break-even is a number rather than a judgement.

**Capacity.** A node has a finite sustainable rate. Past saturation the next
request buys a second node, so cost per request plateaus rather than
continuing to fall. A graph where self-hosting gets arbitrarily cheap with
volume is a graph nobody has ever had.

**Prompt tokens.** APIs bill input and output separately and at different
rates, and prompt tokens dominate most real workloads. Ignoring the input side
understates an API by more than any placement decision is worth.

**Engineering time.** Running inference is not free even when the GPU is
cheap. It is a declared input here rather than a hidden assumption, and at
typical volumes it is the larger term.

## The result that surprises people

At the workload above, **self-hosting wins from about 33,000 requests an hour
with zero engineering cost, and does not win at any volume once an operator's
time is priced in at twelve dollars an hour.**

For most teams the deciding term is the person watching it, not the GPU. No
price sheet on either side says so, and it is worth knowing before a migration
rather than after.

## What this does not decide

Cost is one input. Vendor risk, data residency, model quality and operational
appetite decide this at least as often, and none of them are modelled here. A
tool that implied otherwise would be giving a price-sheet answer to a question
that is not about prices.

## The offers file

```json
[
  {"provider": "Groq", "model": "llama-3-8b",
   "input_per_mtok": 0.05, "output_per_mtok": 0.08, "ttft_p99_ms": 260},
  {"provider": "Together", "model": "llama-3-8b",
   "input_per_mtok": 0.18, "output_per_mtok": 0.18, "ttft_p99_ms": 420}
]
```

`ttft_p99_ms` is what makes an offer checkable against your bound. An offer
without one is ranked on price alone and flagged, because that gap is the
reason this comparison exists.

Rate cards are published. Latency mostly is not, and where a provider does
publish one it is rarely a p99 under load. **Measuring it yourself is a
morning's work and it changes the answer**, which is a reasonable use of
`sounding` even for a team with no intention of self-hosting.
