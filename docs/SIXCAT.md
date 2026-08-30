# sixcat 0.5.1 on the K2/K3 mix

Served on the fixed 2026-08-30 runtime, native MTP k=2, 65,536 context, thinking
on, vendor policy (`glm-5.x`, temperature 1.0, top_p 0.95, reasoning_effort
high), 20 items per category, `--request-timeout 3600`, no engine faults.
120/120 completed, not timed out.

| Category | Score | trunc | loop | empty | ctok p50 | ctok max | total ctok | wall s | tps_mean |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| knowledge | 65.0 | 1 | 0 | 1 | 88 | 8,192 | 10,457 | 769.4 | 12.42 |
| math | **100.0** | 0 | 0 | 0 | 107 | 223 | 2,391 | 141.3 | 16.76 |
| truth | 85.0 | 0 | 0 | 0 | 30 | 178 | 942 | 75.3 | 11.61 |
| instruct | 75.0 | 1 | 1 | 1 | 520 | 32,768 | 44,035 | 2,524.1 | 14.77 |
| code | 85.0 | 0 | 1 | 0 | 386 | 28,853 | n/a | n/a | n/a |
| tools | 90.0 | 0 | 0 | 0 | 23 | 113 | n/a | n/a | n/a |
| **overall[vendor]** | **83.33** | | | | | | | | |

Flags: `truncated:knowledge`, `trunc-in-think:knowledge`, `truncated:instruct`,
`trunc-in-think:instruct`, `loop-failures:instruct`, `loop-failures:code`.

Suite throughput (four categories, 80 items): 57,825 ctok in 3,510 s,
**16.47 tok/s**, `tps_mean` 13.89. Code and tools report no per-item throughput.

## Reading it against K2

| | K2 (container runtime, 2026-08-29) | K2 (fixed runtime, rerun) | K2/K3 mix |
|---|---:|---:|---:|
| knowledge | 65.0 | 70.0 | 65.0 |
| math | 100.0 | 100.0 | 100.0 |
| truth | 85.0 | 80.0 | 85.0 |
| instruct | 75.0 | 70.0 | 75.0 |
| code | 90.0 | 90.0 | 85.0 |
| tools | 90.0 | 95.0 | 90.0 |
| overall | 84.17 | **84.17** | **83.33** |

The two budget-hitting items are the same ones every run on this model hits:
`mmlu:8` (8,192-token budget, empty answer; also reported failing on K2 by an
independent operator) and `ifeval:1300` (32,768-token loop, the K-pool
reproducer prompt). The code loop failure is one 28,853-token item. With 20
items per category at temperature 1.0, one item is five points. The paired
comparison on the same fixed runtime is K2 84.17 against mix 83.33: one item.
K2's own two runs, on different runtimes, also differ by one item in four
categories (knowledge +5, truth -5, instruct -5, tools +5) while landing on the
same overall; that is the size of this suite's noise. On 120 items sixcat cannot
resolve a K2-vs-mix difference; the KLD panel (1,048,064 positions per
checkpoint) is the measurement for that, see [`KLD.md`](KLD.md).

K2 fixed-runtime receipt: knowledge 70 (1 truncated/empty: `mmlu:8`), math 100,
truth 80, instruct 70 (1 truncated, 1 loop: `ifeval:1300`), code 90, tools 95;
suite 16.48 tok/s over 62,768 ctok in 3,808 s.
