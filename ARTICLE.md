# KDB-X Python for Quant Teams Who Aren't Going to Rewrite Their Stack

*And shouldn't have to.*

---

Many quant pods have the same shape of problem. The team thinks in pandas; it's the dialect of every notebook, every backtest, every hire. And every team eventually hits the same wall at tick scale. The usual responses (downsample, distribute, or rewrite in something faster) all trade away the thing that made the team productive in the first place.

[KDB-X Python](https://code.kx.com/pykx/4.0/) (`pykx`) takes a different trade. It embeds the kdb vector engine, the database behind tick infrastructure at the world's largest trading firms, *inside* your Python process. Your data lives in kdb columnar memory; your code stays Python. Tables implement the pandas API, so `head`, `dtypes`, boolean masks, `sort_values`, and `merge_asof` all work as your team already writes them, but execute where the data lives.

This article walks through what that means in practice, with numbers. Everything here is reproducible from the companion notebook (`KDB_X_Python_for_pods.ipynb`): seeded data, sanity-checked results, about 10 minutes end to end.

## The Wall

Three pains, and every pod recognizes at least two of them:

1. **The asof join that takes minutes.** Attaching quotes to executions is the heart of TCA and signal alignment, and pandas walks the entire feed to do it.
2. **The tick history that doesn't fit in a DataFrame.** A single day is workable. Fifteen days is a 22 GB allocation before the first query runs.
3. **The rolling calculation that kills iteration.** Per-symbol windows over tens of millions of rows turn a research loop into a coffee break.

## The Solution: Same Notebook, Different Engine

The notebook makes three claims, in order, each backed by code you can audit:

1. **Your workflow doesn't change.** Same notebook, same Python, largely the same pandas API.
2. **The core quant operations get much faster.** Measured live in Section 5, not asserted.
3. **q is optional.** The Python API covers the work; your AI assistant covers the rest.


Let's take them in order.

## Your Workflow Doesn't Change

The notebook's first act generates 40M rows of realistic market data (20M trades, 20M quotes, per-symbol price anchoring) in a few seconds, then does exactly what a pandas user would do with them:

```python
trade[trade['sym'] == 'AAPL'].head(3)          # boolean masks
trade['notional'] = trade['price'] * trade['size']
trade.sort_values(by='notional', ascending=False).head(3)
```

No new mental model, no schema ceremony. These are the idioms your team already types, running against kdb columnar memory instead of a DataFrame.

When you want a real query, the [Column API](https://code.kx.com/pykx/4.0/user-guide/fundamentals/query/pyquery.html) reads like the pandas pattern it replaces:

```python
trade.select(
    columns={'vwap': kx.Column('size').wavg(kx.Column('price'))},
    by={'sym': kx.Column('sym')},
    where=kx.Column('price') > 100,
)
```

That's a filtered, grouped VWAP in one expression, evaluated inside the engine. And round trips are cheap: `.pd()`, `.np()`, `.pa()` hand any result back to pandas, NumPy, or Arrow when you want your existing toolchain. Nothing locks you in.

## The Numbers

Section 5 of the notebook benchmarks three workloads, pandas versus KDB-X Python, on the same data in the same process, after first verifying both engines produce identical results:

| Workload (20M trades / 20M quotes) | pandas | KDB-X Python | speedup |
|---|---|---|---|
| A: asof join + slippage (100K execs) | 3.06 s | 0.13 s | **24x** |
| B: OHLC + VWAP bars, 15-min buckets | 4.01 s | 2.18 s | **1.8x** |
| C: rolling MA(20) per symbol | 18.06 s | 3.61 s | **5x** |

*Mean of 10 runs on a standard Google Colab instance; seeded data, methodology in-notebook. Run it on your own hardware. That's the point.*

The pattern is worth reading closely. The asof join is 24x faster because kdb's `aj` binary-searches per execution rather than walking the whole feed. Simple bucketed aggregation is closer (1.8x) because pandas is genuinely decent at it. Rolling per-symbol windows sit in between at 5x.

The gaps grow with data size. These are single-day numbers, and pandas needs the entire working set in Python memory before the first query runs.

## Which Brings Us to the Cliff

At 300M rows, the pandas conversation ends; that's a roughly 22 GB DataFrame before you've computed anything. Section 4 persists the day to a date-partitioned database, clones it to 15 days of history, and keeps querying with the same API:

```python
db = kx.DB(path=db_dir, change_dir=False)
db.create(trade_s, 'trade', day, by_field='sym', change_dir=False)
# ... 15 partitions later:
db.trade.select(...)         # touches only the partitions and columns it needs
```

Queries against 300M rows on disk come back in seconds, because a partitioned kdb database reads only what the query touches. This is the same layout production tick stores use, which means the notebook's toy is structurally your firm's HDB.

## The Ecosystem Is Python-Visible Too

Three things the notebook demonstrates beyond the core engine.

**Parquet without an import step.** The `kx.pq` module maps parquet files as virtual tables. You query in place; there is no load:

```python
pq = kx.module.use('kx.pq')
vt = pq.pq(Path('trade_day.parquet'))
vt.select(columns={'vwap': kx.Column('size').wavg(kx.Column('price'))}, ...)
```

**Rendering where the data lives.** The [ax module](https://github.com/KxSystems/ax) brings a grammar-of-graphics renderer (skia) into the engine. Plotting stays Python for results (plotnine on a 96-row aggregate), but when you need to *see every tick* (bad prints, gaps, microstructure), ax renders half a million raw points to PNG in under a second. No downsampling, no conversion. At HDB scale this means charts come back as kilobytes of PNG instead of gigabytes of DataFrame.

**Your production kdb, same API.** Section 8 serves the on-disk database from a separate q process and queries it over IPC. The Column API works over connections, referencing remote tables by name:

```python
with kx.SyncQConnection('localhost', 5010) as conn:
    res = conn.qsql.select('trade',
        columns={'vwap': kx.Column('size').wavg(kx.Column('price'))},
        by={'date': kx.Column('date'), 'bucket': kx.Column('time').minute.xbar(15)},
        where=kx.Column('sym') == 'AAPL',
    ).pd()
```

Only the result crosses the wire. For a pod, this is the actual deployment shape: the firm's HDB already exists; your notebook connects to it.

## What About q?

The honest answer: the notebook contains exactly one line of q, and it's a module import. Everything else (joins, windows, buckets, parquet, IPC, plotting) is Python.

When q *is* the right tool (it remains the most concise time-series language ever built), modern AI assistants are getting more fluent in it: KX publishes Claude skills for both q and KDB-X Python. q went from prerequisite to power-up.

## When This Isn't Worth It

If your working set is a few million rows and your queries already come back in seconds, pandas is fine and you should keep using it; the 1.8x on bucketed bars alone doesn't justify a new dependency. The wins concentrate where the wall is: asof joins, rolling windows per key, and anything that pushes past Python memory. There's also a setup cost, small but real: a pip install, a free license, and a first hour of learning where the Column API diverges from pandas. The notebook is designed to compress that hour.

## Wrapping Up

The interesting thing about these benchmarks isn't the 24x. It's that the speedup arrives without a migration. The industry has spent a decade telling pandas-fluent teams that scale means leaving their tools behind: new language, new cluster, new mental model. The notebook makes the opposite argument. The engine moves to where the team already works, the API meets them at the idioms they already type, and the same code path extends from a 40M-row experiment to the firm's production HDB over IPC.

That changes what "adopting kdb" means for a pod. It's no longer a rewrite decision made above your head; it's a pip install you can evaluate in an afternoon, against your own workloads, with results you can audit line by line.

## Try It

Everything above is one notebook: `KDB_X_Python_for_pods.ipynb`. Python 3.10+, `pip install --upgrade pykx`, a free license, and about 15 minutes. The final cell cleans up after itself.

## Resources

| Resource | Where |
|---|---|
| KDB-X install + free license | [developer.kx.com](https://developer.kx.com/products/kdb-x/install) |
| KDB-X Python documentation | [code.kx.com/pykx/4.0](https://code.kx.com/pykx/4.0/) |
| Pandas API reference | [code.kx.com/pykx/4.0: Pandas API](https://code.kx.com/pykx/4.0/user-guide/advanced/Pandas_API.html) |
| Column API / Pythonic queries | [code.kx.com/pykx/4.0: Query](https://code.kx.com/pykx/4.0/user-guide/fundamentals/query/pyquery.html) |
| Modules from Python (`kx.module.use`) | [code.kx.com/pykx/4.0: Module API](https://code.kx.com/pykx/4.0/api/module.html) |
| ax module (grammar of graphics, qdoc) | [github.com/KxSystems/ax](https://github.com/KxSystems/ax) |
| fusion modules (pcre2, blas, expat) | [github.com/KxSystems/fusionx](https://github.com/KxSystems/fusionx) |
| KDB-X tutorials (incl. parquet module) | [github.com/KxSystems/tutorials](https://github.com/KxSystems/tutorials) |
| Grammar of graphics reference | [code.kx.com/analyst: grammar of graphics](https://code.kx.com/analyst/libraries/grammar-of-graphics/) |
| q language reference | [code.kx.com/q](https://code.kx.com/q/) |
