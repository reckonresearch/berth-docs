# Validation (P0)

P0 is berth's hardware-calibration gate, not a separate product. It answers one
question: do berth's predictions match real silicon within tolerance? This page
reports what P0 found, including what failed, because a validation that reports
only its passes is not a validation.

## What was measured

Two accelerators, each running Llama-3-8B in bf16 under vLLM, 60 traces apiece
spanning batch sizes 1 to 32 and prompt lengths 514 to 7682:

- NVIDIA L40S (GDDR6, 864 GB/s): microbenched GEMM 211 TFLOPS, bandwidth 652 GB/s
- NVIDIA H100 PCIe (HBM2e, 2.0 TB/s): microbenched GEMM 388 TFLOPS, bandwidth
  1591 GB/s

These two cards differ in memory technology, microarchitecture, and roofline
regime, so a model that predicts both is being tested across a real difference,
not repeated on similar parts.

## What was validated

| Term | L40S | H100 PCIe |
|---|---|---|
| Decode bandwidth (batch 1) | 4.5% error | 1.7% error |
| KV-cache slope | 1.3 to 3.2% | 6.3 to 16.6% |
| Decode latency (TPOT) from untuned priors | 10.6% | 4.2% |

The headline is the last row. Those predictions use spec-sheet priors with no
fitting to either card. The default bandwidth prior implies 648 GB/s on the L40S
against a microbenched 652, and 1500 on the H100 against 1591: within 1 and 6
percent of achievable bandwidth on cards it never saw. This is the mechanism
behind untuned decode prediction.

## The systems finding

Prefill of a synchronous batch is serial: no request emits a first token until
every prompt in the batch has been prefilled. Measured effective parallelism is
1.09 on both cards under vLLM. A pre-registered cell on an L40S under SGLang
returned 1.01, inside the 0.9 to 1.4 interval and far from the 1.8 falsification
bound. Serial admission is a property of batched inference rather than of one
scheduler, and the constant transfers as well as the form.

Fitted per batch level rather than pooled, it is near 1.0 from batch 4 to 16 and
falls to about 0.75 at batch 32 on both stacks: serial at moderate concurrency,
partially overlapped at high. That is the leading explanation for the batch-32
residual structure disclosed above, and it is open.

The same cell found decode unchanged across stacks: effective bandwidth 0.854
under SGLang against 0.850 under vLLM, same card, same model, agreement to
three decimals. Bytes over bandwidth does not care which scheduler queued the
request, and that is the strongest evidence so far that the decode term is
physics rather than a fit. This was confirmed against the sharpest objection: the
harness records the median first-token time across concurrent streams, and if
each request emitted on its own prefill the implied parallelism would be near
two. It is near one. The recorded median is the batch tail.

## What failed, and what it meant

Three reported failures each carried an incorrect label, and each turned out to
be an instrument defect that understated the model rather than a model error:

1. The prefill check reported an attention-accounting error. The real cause was
   the fixed floor being attributed to compute. Subtracting the fitted floor
   flattens the trend.
2. A ceiling check reported passes for accelerators that were never measured,
   using their untouched priors. Scoping it to measured silicon removed the
   vacuous verdicts and surfaced a genuine open discrepancy (below).
3. An end-to-end error of about 30 percent came from scoring a single-request
   service time against a batched tail. Scored against the quantity actually
   measured, the same model reads 4.4 and 4.7 percent, and under the target
   before any fitting.

None of the three inflated a result. A validation whose every discovered defect
made the model look worse than it was is the opposite of a fragile one.

## What remains open

Two genuine discrepancies are disclosed rather than resolved:

- On the L40S, the decode fit implies 707 GB/s against a microbenched 652, an
  8.4 percent overshoot. The H100 passes the same check. Unresolved.
- Residuals at batch 32 run about 10 to 14 percent, while every other batch sits
  within a few percent. High-concurrency structure is not fully modeled.

One design weakness is also disclosed: the held-out split separates repetitions
rather than distinct cells, which weakens every error figure in the release.

## Scope

Two cards, one vendor, one dense model. The claim P0 supports is that a
closed-form roofline predicts decode latency across two memory technologies and
two microarchitectures without per-card tuning, together with the serial-prefill
finding. It does not yet support a claim of generalization across vendors or
architectures. Predictions for AMD and Google TPU parts are published separately
as pre-registered forecasts with stated falsification conditions, not as
validated results.
