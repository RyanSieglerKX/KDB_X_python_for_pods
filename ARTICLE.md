# Pandas at Tick Scale

*KDB-X Python for Python first pods and quant teams.*

---

Pandas works well on smaller datasets but as data scales it hits its limits: the asof join that takes minutes, the tick history that doesn't fit in a DataFrame, the rolling calculation that stalls a research iteration. 

KDB-X Python (`pykx`) takes a different approach. It embeds the kdb vector engine inside your Python process. Your data lives in kdb columnar memory; your code stays Python. Tables implement the pandas API, so `head`, `dtypes`, boolean masks, `sort_values`, and `merge_asof` all work as your team already writes them, but execute where the data lives.

This article walks through a comparison of KDB-X Python and Pandas. Everything here is reproducible from the companion notebook (`KDB_X_Python_for_pods.ipynb`): run it in about 15 minutes end to end.

## Your workflow doesn't change

The notebook first generates 40M rows of synthetic market data (20M trades, 20M quotes, per-symbol price anchoring), then does exactly what a pandas user would do with them:

```python
trade[trade['sym'] == 'AAPL'].head(3)          # boolean masks
trade['notional'] = trade['price'] * trade['size']
trade.sort_values(by='notional', ascending=False).head(3)
```

The Column API reads like the pandas idiom it replaces:

```python
trade.select(
    columns={'vwap': kx.Column('size').wavg(kx.Column('price'))},
    by={'sym': kx.Column('sym')},
    where=kx.Column('price') > 100,
)
```

Round trips are cheap: `.pd()`, `.np()`, and `.pa()` hand any result back to pandas, NumPy, or Arrow when you want your existing toolchain.

## The numbers

Section 5 of the notebook benchmarks three workloads every quant team recognizes, pandas versus KDB-X Python, on the same data in the same process, after first verifying that both engines produce identical results:

| Workload (20M trades / 20M quotes) | pandas | KDB-X Python | speedup |
|---|---|---|---|
| A: asof join + slippage (100K execs) | 3.06 s | 0.13 s | **24x** |
| B: OHLC + VWAP bars, 15-min buckets | 4.01 s | 2.18 s | **1.8x** |
| C: rolling MA(20) per symbol | 18.06 s | 3.61 s | **5x** |

*Mean of 10 runs on a standard Google Colab instance; seeded data, methodology in-notebook. Run it on your own hardware.*

The asof join, the operation at the heart of TCA, signal alignment, and quote attachment, is 24x faster because kdb's `aj` binary-searches per execution rather than walking the whole feed. Simple bucketed aggregation is closer (1.8x) because pandas is genuinely decent at it. Rolling per-symbol windows sit in between at 5x.

These are single-day numbers, and the multipliers grow with the dataset. `merge_asof` scans the full quote feed, so doubling the feed roughly doubles the pandas cost, while `aj` cost tracks the much smaller executions table. The rolling and bucketed workloads widen too as the working set outgrows CPU caches and pandas' intermediate copies add memory pressure. Row count is a parameter in the notebook's first section; rerun at your own scale.

## Past the limits of a DataFrame

Pandas needs the full 300M row DataFrame in RAM before the first query runs. Section 4 moves the same data to a date-partitioned database instead, clones it to 15 days of history, and keeps querying with the same API:

```python
db = kx.DB(path=db_dir, change_dir=False)
db.create(trade_s, 'trade', day, by_field='sym', change_dir=False)
# ... 15 partitions later:
db.trade.select(...)         # touches only the partitions and columns it needs
```

Queries against 300M rows on disk return in seconds because a partitioned kdb database reads only what the query touches. Memory efficiency is the other half of the story: partitions are memory-mapped rather than loaded, so the operating system's page cache decides what stays resident, and the database can be far larger than the machine's RAM. The same point shows up in miniature in the benchmark: the pandas side only works because the notebook first materializes full DataFrame copies in Python memory (roughly 1.5 GB per 20M-row table, visible in Section 2's `.info()` output), while the kdb side queries the tables in place. 

## The ecosystem is Python-visible too

Three things the notebook demonstrates beyond the core engine:

**Parquet without an import step.** The `kx.pq` module maps parquet files as virtual tables and queries them in place:

```python
pq = kx.module.use('kx.pq')
vt = pq.pq(Path('trade_day.parquet'))
vt.select(columns={'vwap': kx.Column('size').wavg(kx.Column('price'))}, ...)
```

**Rendering where the data lives.** The `ax` module brings a grammar-of-graphics renderer (skia) into the engine. Plotting stays Python for results: plotnine on a 96-row aggregate. When you need to see every tick (bad prints, gaps, microstructure), ax renders half a million raw points to PNG in under a second with no downsampling and no conversion. At HDB scale this means charts come back as kilobytes of PNG instead of gigabytes of DataFrame.

**Connect to kdb HDB and query with the same API.** Section 8 serves the on-disk database from a separate q process and queries it over IPC. The Column API works over connections, referencing remote tables by name:

```python
with kx.SyncQConnection('localhost', 5010) as conn:
    res = conn.qsql.select('trade',
        columns={'vwap': kx.Column('size').wavg(kx.Column('price'))},
        by={'date': kx.Column('date'), 'bucket': kx.Column('time').minute.xbar(15)},
        where=kx.Column('sym') == 'AAPL',
    ).pd()
```

Only the result is returned rather than all of the data. This is the actual deployment shape for most teams: the firm's HDB already exists, and your notebook connects to it.

## What about q?

When q is the right tool, modern AI assistants are getting more fluent in it: KX publishes Claude skills for both q and KDB-X Python, covering query translation, linting, and the API patterns used throughout the notebook.

## Resources

| Resource | Where |
|---|---|
| KDB-X install + free license | [developer.kx.com](https://developer.kx.com/products/kdb-x/install) |
| KDB-X Python documentation | [code.kx.com/pykx/4.0](https://code.kx.com/pykx/4.0/) |
| Pandas API reference | [code.kx.com/pykx/4.0 — Pandas API](https://code.kx.com/pykx/4.0/user-guide/advanced/Pandas_API.html) |
| Column API / Pythonic queries | [code.kx.com/pykx/4.0 — Query](https://code.kx.com/pykx/4.0/user-guide/fundamentals/query/pyquery.html) |
| Modules from Python (`kx.module.use`) | [code.kx.com/pykx/4.0 — Module API](https://code.kx.com/pykx/4.0/api/module.html) |
| ax module (grammar of graphics, qdoc) | [github.com/KxSystems/ax](https://github.com/KxSystems/ax) |
| fusion modules (pcre2, blas, expat) | [github.com/KxSystems/fusionx](https://github.com/KxSystems/fusionx) |
| KDB-X tutorials (incl. parquet module) | [github.com/KxSystems/tutorials](https://github.com/KxSystems/tutorials) |
| Grammar of graphics reference | [code.kx.com/analyst — grammar of graphics](https://code.kx.com/analyst/libraries/grammar-of-graphics/) |
| q language reference | [code.kx.com/q](https://code.kx.com/q/) |

## Try it

Everything above is one notebook: `KDB_X_Python_for_pods.ipynb`. Python 3.10+, `pip install --upgrade pykx`, a free license, and about 15 minutes.

Run the notebook, see the performance, and note that nothing in your workflow had to fundamentally change to get it.
