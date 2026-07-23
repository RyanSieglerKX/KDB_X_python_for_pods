# KDB-X Python for Python-First Quant Teams

**Keep your notebook. Keep your pandas. Lose the wait.**

This package demonstrates [KDB-X Python](https://code.kx.com/pykx/4.0/) (`pykx`) - the kdb vector engine embedded in your Python process - on common quant workloads: asof joins, bucketed bars, rolling windows, at tick scale. It covers three concepts:

1. **Your workflow doesn't change:** same notebook, same Python, largely the same pandas API.
2. **The core quant operations get much faster:** measured in Section 5.
3. **q is optional:** the Python API covers the work; your AI assistant covers the rest.

## Measured performance

Section 5 benchmarks three workloads (20M trades / 20M quotes), pandas versus KDB-X Python on the same data in the same process, after verifying both engines produce identical results. Across our runs (Google Colab x86 with pandas 2.x; Apple Silicon with pandas 3.0):

- **Asof join + slippage: 8-24x faster**
- **Rolling MA(20) per symbol: 5-11x faster**
- **OHLC + VWAP bars: ~2x faster**

Absolute times vary by machine and pandas version, so the notebook measures them live on yours. The multipliers grow with data size (`n` is a parameter; rerun at your scale).


## Run it

- Python 3.10+, `pip install --upgrade pykx`
- Free KDB-X license: [developer.kx.com](https://developer.kx.com/products/kdb-x/install) (the notebook's install cells walk through it, including Colab)
- ~10 GB free disk for the 300M-row section (tunable via `DAYS`)
- Open `KDB_X_Python_for_pods.ipynb`, run top-to-bottom (~10 min compute + ~5 min partition build)
- The final cell deletes everything the notebook wrote to disk

## Contents

| Section | What it shows |
|---|---|
| 1-2 | 40M rows generated in seconds; the pandas API you already know, on kdb memory |
| 3 | VWAP buckets and the asof join |
| 4 | 300M rows on disk, partitioned |
| 5 | The benchmark |
| 6 | Plotting stays Python (plotnine); optional: 500K raw ticks rendered in-engine in <1s |
| 7 | Parquet files queried in place, no import step |
| 8 | Connecting to a production kdb server, same queries over IPC |
| 9-10 | q with an AI assistant that knows it, more resources |

## Learn more

[`ARTICLE.md`](ARTICLE.md): the narrative version, plus a full resource guide (docs, tutorials, modules).
