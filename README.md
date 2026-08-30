# GLM-5.3-Flash EXL3 K2/K3 mix on one NVIDIA DGX Spark

Reproducible **vLLM** recipe for **[vcruz305/GLM-5.3-Flash-EXL3-K2K3-mix](https://huggingface.co/vcruz305/GLM-5.3-Flash-EXL3-K2K3-mix)**
on a **single NVIDIA DGX Spark / GB10 (SM121)**: the K2 pack with six routed-expert
layers (24, 27, 35, 37, 42, 45) at K3, 2.14 bpw effective on the routed experts.

> Independent community engineering. Not affiliated with or endorsed by Z.ai, NVIDIA, or vLLM.

| What | Where |
|---|---|
| **Pack** | [vcruz305/GLM-5.3-Flash-EXL3-K2K3-mix](https://huggingface.co/vcruz305/GLM-5.3-Flash-EXL3-K2K3-mix) — 120 shards |
| **Runtime** | [vcruz305/GLM-5.3-Flash-EXL3-K2-spark-vllm](https://huggingface.co/vcruz305/GLM-5.3-Flash-EXL3-K2-spark-vllm) — prebuilt wheels; the plugin reads `layer_bits` |
| **K2 base recipe** | [GLM-5.3-Flash-EXL3-K2-DGX-Spark-recipe](https://github.com/vcruz305/GLM-5.3-Flash-EXL3-K2-DGX-Spark-recipe) |
| Engine | vLLM, `--quantization exl3`, TP=1, native MTP k=2. **Stock vLLM cannot load this pack** |

## Quick start

```bash
python scripts/preflight.py
bash scripts/install_prebuilt.sh
bash scripts/download_weights.sh
python scripts/patch_chat_template_thinking.py ~/models/GLM-5.3-Flash-EXL3-K2K3-mix/chat_template.jinja
SPEC_METHOD=mtp MTP_TOKENS=2 MAX_MODEL_LEN=65536 MAX_NUM_SEQS=1 bash scripts/serve_one_spark.sh
```

The load log must show `EXL3 per-layer K override ... bits=3` for the six K3 layers
and `fused_moe=exl3_moe`. A plugin without per-layer K support fails loudly with
`EXL3 load shape mismatch`.

## Results

Measured 2026-08-30 on one GB10, same fixed runtime and boot config as the K2 rows:

| | K2 | K2/K3 mix |
|---|---:|---:|
| GPU memory after load | 92.31 GiB | 98.02 GiB |
| KV cache at 64k (util 0.91) | 786,432 tokens | 337,042 tokens |
| 4-workload decode mean, 64k, MTP k=2 | 16.74 tok/s | **17.19 tok/s** (+2.7%) |
| MTP acceptance, code / math | 0.62 / 0.67 | 0.69 / 0.76 |
| 8,192-token generation, 32k prompt | pass | pass |
| sixcat 0.5.1 (think-on, 64k), same runtime | 84.17 | 83.33 (one item apart; suite noise) |
| long context | 262,144 boots; prompts verified to 163,479 tokens (see the [K2 recipe](https://github.com/vcruz305/GLM-5.3-Flash-EXL3-K2-DGX-Spark-recipe#long-context-measured-ceiling-2026-08-31)); >163k wedge under investigation | 256k run pending (smaller KV pool: 337k tokens at 64k util 0.91) |
| KLD vs BF16 teacher (512 contexts, 1.05M positions) | 0.3346 nats, top-1 0.788 | **0.3121 nats, top-1 0.795** (lower on 505/512 contexts; paired -0.0225, CI [0.0207, 0.0243]) |

Full tables: [`docs/MEASUREMENTS.md`](docs/MEASUREMENTS.md).

## Docs

- [`docs/MEASUREMENTS.md`](docs/MEASUREMENTS.md) — K2 vs mix: load, memory, dispatch, ladder, sixcat, KLD
- [`docs/KLD.md`](docs/KLD.md) — KLD method and scorer validation
- [`docs/KPOOL_TAIL_BUG.md`](docs/KPOOL_TAIL_BUG.md) — the runtime bug fixed in the 2026-08-30 wheels

## License

MIT for the scripts and notes. Weights are not redistributed here; pull them from
Hugging Face and respect the GLM-5.3-Flash license.
