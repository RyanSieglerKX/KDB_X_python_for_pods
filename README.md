# KDB-X Python for Python-First Quant Teams

**Keep your notebook. Keep your pandas. Lose the wait.**

This package demonstrates [KDB-X Python](https://code.kx.com/pykx/4.0/) (`pykx`) — the kdb vector engine embedded in your Python process — on the workloads quant pods run daily: asof joins, bucketed bars, rolling windows, at tick scale. One runnable notebook, three claims, each backed by code you can audit:

1. **Your workflow doesn't change** — same notebook, same Python, largely the same pandas API.
2. **The core quant operations get much faster** — measured live in Section 5, not asserted.
3. **q is optional** — the Python API covers the work; your AI assistant covers the rest.

## Measured (Google Colab, seeded data — rerun it yourself)

| Workload (20M trades / 20M quotes) | pandas | KDB-X Python | speedup |
|---|---|---|---|
| A: asof join + slippage (100K execs) | 3.06 s | 0.13 s | **24×** |
| B: OHLC + VWAP bars, 15-min buckets | 4.01 s | 2.18 s | **1.8×** |
| C: rolling MA(20) per symbol | 18.06 s | 3.61 s | **5×** |

Both engines produce identical results (verified in-notebook before timing). Mean of 10 runs.

## Run it

- Python 3.10+, `pip install --upgrade pykx`
- Free KDB-X license — [developer.kx.com](https://developer.kx.com/products/kdb-x/install) (the notebook's install cells walk through it, including Colab)
- ~10 GB free disk for the 300M-row section (tunable via `DAYS`)
- Open `KDB_X_Python_for_pods.ipynb`, run top-to-bottom (~10 min compute + ~5 min partition build)
- The final cell deletes everything the notebook wrote to disk

## Contents

| Section | What it shows |
|---|---|
| 1–2 | 40M rows generated in seconds; the pandas API you already know, on kdb memory |
| 3 | VWAP buckets and the asof join — the time-series operations pandas fights |
| 4 | 300M rows on disk, partitioned, same API |
| 5 | The benchmark above, run live with sanity checks |
| 6 | Plotting stays Python (plotnine); optional: 500K raw ticks rendered in-engine in <1s |
| 7 | Parquet files queried in place — no import step |
| 8 | Connecting to a production kdb server — same queries over IPC |
| 9–10 | q with an AI assistant that knows it; getting started |

## Learn more

[`ARTICLE.md`](ARTICLE.md) — the narrative version, plus a full resource guide (docs, tutorials, modules).
