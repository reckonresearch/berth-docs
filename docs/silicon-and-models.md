# Silicon and Models

## Silicon fleet

berth ships spec-sheet profiles for a range of accelerators. Each profile carries
peak compute, memory bandwidth, memory capacity, and a list price. The two cards
below marked MEASURED have been validated against real hardware traces; the rest
carry spec-sheet priors and are labeled accordingly.

Six cells are measured today across three memory technologies and two vendors:
L40S under vLLM and again under SGLang, H100 PCIe, H100 SXM at bf16 and at fp8,
A100 80GB with a mixture-of-experts model, and MI300X. That last one is the
first non-NVIDIA cell in the corpus.

A cell is one accelerator running one model under one serving stack at one
precision. A card marked MEASURED has at least one cell; it does not mean every
model on it has been measured, and the CLI says which on every line.

| Key | Accelerator | Peak TFLOPS (bf16) | HBM BW (TB/s) | Memory | Status |
|---|---|---|---|---|---|
| `l40s` | NVIDIA L40S | 362 | 0.864 | 48 GB | MEASURED |
| `h100-pcie` | NVIDIA H100 PCIe | 756 | 2.00 | 80 GB | MEASURED |
| `h100-sxm` | NVIDIA H100 SXM | 989 | 3.35 | 80 GB | MEASURED |
| `h200-sxm` | NVIDIA H200 SXM | 989 | 4.80 | 141 GB | prior |
| `a100-80g` | NVIDIA A100 80GB | 312 | 2.00 | 80 GB | MEASURED |
| `mi300x` | AMD MI300X | 1307 | 5.30 | 192 GB | MEASURED |
| `tpu-v6e` | Google TPU v6e | 918 | 1.64 | 32 GB | prior |
| `tpu-v5e` | Google TPU v5e | 197 | 0.819 | 16 GB | prior |
| `trn1` | AWS Trainium | 190 | 0.820 | 32 GB | prior |
| `trn2` | AWS Trainium2 | 650 | 2.90 | 96 GB | prior |
| `cpu-spr` | Intel Sapphire Rapids | 8 | 0.30 | 512 GB | prior |

A profile with prior status predicts from the spec sheet alone. Predictions for
those parts are honest forecasts, not measurements, and berth labels them so.

Two things to know about the peak compute column. It carries the vendors' 2:4
structured-sparsity ratings, which are roughly twice the dense bf16 figure, and
the microbenchmarked dense numbers on the measured cards came in far lower:
211 TFLOPS on the L40S against a rated 362, and 388 on the H100 PCIe against 756.
On measured silicon this is absorbed by the fitted MFU, so it does not move the
published results. On prior silicon there is no fit to absorb it, so a
compute-bound prediction on a prior row can be optimistic by up to a factor of
two. Decode is memory-bound by two to three orders of magnitude and is therefore
unaffected; prefill on an unmeasured accelerator is the exposed case, and the
pre-registered forecasts for AMD and TPU parts cover decode only for exactly this
reason.

The price column is a list-price default and moves constantly. The measured
premium in these docs uses the prices actually paid at capture time ($0.99/hr
L40S, $3.35/hr H100 PCIe), which differ from the shipped defaults. Pass your own
with `--prices` or `price_hr=`; a premium computed on stale list prices is a
stale premium.

## Model registry

| Key | Model | Params | Architecture |
|---|---|---|---|
| `llama3-8b` | Llama-3-8B | 8B | dense |
| `llama3-70b` | Llama-3-70B | 70B | dense |
| `qwen3-235b-moe` | Qwen3-235B-A22B | 235B total, 22B active | MoE |
| `mixtral` | Mixtral-8x7B | 47B total, 13B active | MoE |

## Provenance status

berth never presents a spec-sheet prior as a measurement. Every accelerator is
tagged MEASURED or prior, and every number a prediction produces inherits that
status. The distinction is a hard rule, not a stylistic choice: a prediction you
can trust is one you can tell apart from a measurement.
