# Measurements: K2/K3 mix vs K2

One DGX Spark (GB10), vLLM with the K-pool tail fix (2026-08-30 wheels),
`--quantization exl3`, TP=1, `EXL3_FUSED_MOE=1`, native MTP k=2, KV fp8,
`max-model-len` 65536, `gpu-memory-utilization` 0.91, `max-num-seqs` 1, CUDA
graphs on. The K2 rows are the same build and the same boot configuration,
measured the same day, so the only variable is the six K3 layers.

## Load, memory, dispatch

| | K2 | K2/K3 mix |
|---|---:|---:|
| weights on disk | 97.73 GB | 103.16 GB (+5.4 GB) |
| GPU memory after load (weights + non-torch) | 92.31 GiB | 98.02 GiB (+5.7 GiB) |
| GPU KV cache at 64k, util 0.91 | 786,432 tokens (12.0x) | 337,042 tokens (5.1x) |
| K3 layers dispatched | none | 24, 27, 35, 37, 42, 45 |
| load shape mismatches | 0 | 0 |

The extra 5.7 GiB comes straight out of the KV pool: at 64k the mix keeps 5.1
concurrent full-length sequences instead of 12. Dispatch is proven by the load
itself: the K3 layers' `[256, 128, 48]` trellis tensors only fit parameters
allocated at K3, and the same per-layer `bits` value feeds the fused kernel's
`k` argument.

## Stability (same boot)

| probe | K2 | mix |
|---|---|---|
| 64-token prompt, 4,096-token generation | pass | pass |
| 64-token prompt, 8,192-token generation | pass | pass |
| 32,000-token prompt | pass | pass |

## Decode speed: four-workload ladder at 64k

One warm-up plus three measured runs, 400 completion tokens, thinking off,
temperature 0, per-workload medians.

| Workload | K2 tok/s | mix tok/s | delta | K2 accept | mix accept |
|---|---:|---:|---:|---:|---:|
| prose | 14.8064 | 14.4393 | -2.5% | 0.5500 | 0.5309 |
| structured | 20.2678 | 20.5853 | +1.6% | 0.9599 | 0.9925 |
| code | 15.6495 | 16.5362 | +5.7% | 0.6208 | 0.6935 |
| math | 16.2294 | 17.2118 | +6.1% | 0.6735 | 0.7579 |
| arithmetic mean | 16.7383 | **17.1931** | **+2.7%** | | |

The mix is faster on three of four workloads despite carrying more bytes, and
the acceptance column says why: the native MTP draft is accepted more often
against a better target (code 0.62 to 0.69, math 0.67 to 0.76). That is the
first end-model signal that the six promoted layers improve the model, not just
its size. Prose is the exception at -2.5%, inside the 2-3% run-to-run spread
seen on this box.

## sixcat 0.5.1 (think-on, 64k)

pending

## KLD against the BF16 teacher

pending; see [`KLD.md`](KLD.md).
