*KDB-X Python for quant teams who aren't going to rewrite their stack — and shouldn't have to.*

---

Every quant pod has the same shape of problem. The team thinks in pandas — it's the dialect of every notebook, every backtest, every hire. And every team has hit the same wall: the asof join that takes minutes, the tick history that doesn't fit in a DataFrame, the rolling calculation that turns a research iteration into a coffee break. The usual responses — downsample, distribute, or rewrite in something faster — all trade away the thing that made the team productive.

KDB-X Python (`pykx`) takes a different trade. It embeds the kdb vector engine — the database under most of the world's tick infrastructure — *inside* your Python process. Your data lives in kdb columnar memory; your code stays Python. Tables implement the pandas API, so `head`, `dtypes`, boolean masks, `sort_values`, and `merge_asof` all work as your team already writes them — but execute where the data lives.

This article walks through what that means in practice, with numbers. Everything here is reproducible from the companion notebook (`KDB_X_Python_for_pods.ipynb`) — seeded data, sanity-checked results, ~10 minutes end to end.

## Your workflow doesn't change

The notebook's first act generates 40M rows of realistic market data (20M trades, 20M quotes, per-symbol price anchoring) in a few seconds, then does exactly what a pandas user would do with them:

```python
trade[trade['sym'] == 'AAPL'].head(3)          # boolean masks
trade['notional'] = trade['price'] * trade['size']
trade.sort_values(by='notional', ascending=False).head(3)
```

No new mental model, no schema ceremony. When you want a real query, the Column API reads like the pandas idiom it replaces:

```python
trade.select(
    columns={'vwap': kx.Column('size').wavg(kx.Column('price'))},
    by={'sym': kx.Column('sym')},
    where=kx.Column('price') > 100,
)
```

And round trips are cheap: `.pd()`, `.np()`, `.pa()` hand any result back to pandas, NumPy, or Arrow when you want your existing toolchain. Nothing locks you in.

## The numbers

Section 5 of the notebook benchmarks three workloads every pod recognizes, pandas versus KDB-X Python, on the same data in the same process — after first verifying both engines produce identical results:

| Workload (20M trades / 20M quotes) | pandas | KDB-X Python | speedup |
|---|---|---|---|
| A: asof join + slippage (100K execs) | 3.06 s | 0.13 s | **24×** |
| B: OHLC + VWAP bars, 15-min buckets | 4.01 s | 2.18 s | **1.8×** |
| C: rolling MA(20) per symbol | 18.06 s | 3.61 s | **5×** |

*Mean of 10 runs on a standard Google Colab instance; seeded data, methodology in-notebook. Run it on your own hardware — that's the point.*

The pattern is worth reading closely. The asof join — the operation at the heart of TCA, signal alignment, and quote attachment — is 24× faster because kdb's `aj` binary-searches per execution rather than walking the whole feed. Simple bucketed aggregation is closer (1.8×) because pandas is genuinely decent at it. Rolling per-symbol windows sit in between at 5×. The gaps grow with data size: these are single-day numbers, and pandas needs the entire working set in Python memory before the first query runs.

## Which brings us to the cliff

At 300M rows, the pandas conversation ends — that's a ~22 GB DataFrame before the first query. Section 4 persists the day to a date-partitioned database, clones it to 15 days of history, and keeps querying with the same API:

```python
db = kx.DB(path=db_dir, change_dir=False)
db.create(trade_s, 'trade', day, by_field='sym', change_dir=False)
# ... 15 partitions later:
db.trade.select(...)         # touches only the partitions and columns it needs
```

Queries against 300M rows on disk come back in seconds, because a partitioned kdb database reads only what the query touches. This is the same layout production tick stores use — which means the notebook's toy is structurally your firm's HDB.


## The ecosystem is Python-visible too

Three things the notebook demonstrates beyond the core engine:

**Parquet without an import step.** The `kx.pq` module maps parquet files as virtual tables — query in place, no load:

```python
pq = kx.module.use('kx.pq')
vt = pq.pq(Path('trade_day.parquet'))
vt.select(columns={'vwap': kx.Column('size').wavg(kx.Column('price'))}, ...)
```

**Rendering where the data lives.** The `ax` module brings a grammar-of-graphics renderer (skia) into the engine. Plotting stays Python for results — plotnine on a 96-row aggregate — but when you need to *see every tick* (bad prints, gaps, microstructure), ax renders half a million raw points to PNG in under a second, no downsampling, no conversion. At HDB scale this means charts come back as kilobytes of PNG instead of gigabytes of DataFrame.

**Your production kdb, same API.** Section 8 serves the on-disk database from a separate q process and queries it over IPC — the Column API works over connections, referencing remote tables by name:

```python
with kx.SyncQConnection('localhost', 5010) as conn:
    res = conn.qsql.select('trade',
        columns={'vwap': kx.Column('size').wavg(kx.Column('price'))},
        by={'date': kx.Column('date'), 'bucket': kx.Column('time').minute.xbar(15)},
        where=kx.Column('sym') == 'AAPL',
    ).pd()
```

Only the result crosses the wire. For a pod, this is the actual deployment shape: the firm's HDB already exists; your notebook connects to it.

## What about q?

The honest answer: the notebook contains exactly one line of q, and it's a module import. Everything else — joins, windows, buckets, parquet, IPC, plotting — is Python. When q *is* the right tool (it remains the most concise time-series language ever built), modern AI assistants are getting more fluent in it: KX publishes Claude skills for both q and KDB-X Python. q went from prerequisite to power-up.

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

Everything above is one notebook: `KDB_X_Python_for_pods.ipynb`. Python 3.10+, `pip install --upgrade pykx`, a free license, and ~15 minutes. The final cell cleans up after itself.


Run the notebook, see the performance, and realize that nothing in your workflow has to fundamentally change to get the performance boost.
