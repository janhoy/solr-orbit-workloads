# nyc_taxis — Test Procedures

This file documents the test procedures available in the `nyc_taxis` workload.
Run any procedure with:

```bash
solr-benchmark run --workload=nyc_taxis --test-procedure=<name>
```

---

## append-no-conflicts *(default)*

Indexes the full corpus using append-only writes (all document IDs are unique), then runs the
complete set of search, aggregation, and sort queries.

**Schedule:**
1. Delete and re-create the `nyc_taxis` collection
2. Check cluster health (CLUSTERSTATUS)
3. Index all documents (`bulk-index`, 240 s warmup)
4. Hard commit
5. Optimize (force-merge to `max_segments`)
6. Run all search/aggregation/sort operations sequentially

**Additional parameters** (beyond the common ones in [README.md](README.md)):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `match_all_warmup_iterations` | `50` | Warmup iterations for `match-all` |
| `match_all_iterations` | `100` | Measured iterations for `match-all` |
| `range_warmup_iterations` | `50` | Warmup for `range` |
| `range_iterations` | `100` | Measured iterations for `range` |
| `distance_amount_facet_iterations` | `50` | Measured iterations for `distance_amount_facet` |
| *(all other search ops)* | see README | Follow the `<op>_warmup_iterations` / `<op>_iterations` pattern |

---

## append-no-conflicts-index-only

Indexes the full corpus (append-only) without running any search queries. Use this to measure
pure indexing throughput in isolation.

**Schedule:**
1. Delete and re-create the `nyc_taxis` collection
2. Check cluster health
3. Index all documents
4. Hard commit
5. Optimize

---

## append-sorted-no-conflicts-index-only

Identical to `append-no-conflicts-index-only` but intended for use when the Solr schema is
configured to sort documents at index time (e.g. descending by `pickup_datetime`).

Sort-at-index-time is configured in `configsets/nyc_taxis/schema.xml` (or `solrconfig.xml`),
not via workload parameters — edit the configset before running this procedure.

**Schedule:** same as `append-no-conflicts-index-only`.

---

## update

Indexes the corpus while deliberately reusing document IDs to simulate real-world update
workloads. The conflict strategy and rate are configurable.

**Schedule:**
1. Delete and re-create the `nyc_taxis` collection (with configurable shards/replicas)
2. Check cluster health
3. Index with conflicts (`bulk-index` using `update` operation, 1200 s warmup)
4. Hard commit
5. Optimize

**Additional parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `conflicts` | `random` | ID conflict strategy: `sequential` or `random` |
| `conflict_probability` | `25` | Probability (0–100) of reusing an existing ID |
| `on_conflict` | `update` | Action on conflict: `index` (overwrite) or `update` (partial update) |
| `recency` | `0` | Bias towards recent IDs (0 = uniform, 1 = most-recent) |
| `number_of_shards` | `1` | Shards for the collection (mapped to `num_shards`) |
| `number_of_replicas` | `1` | Replicas (mapped to `replication_factor`) |

---

## search-longevity-test

A durability/longevity test that first indexes half the corpus, optimizes, then concurrently
runs indexing and all search operations for a configurable time period.

**Schedule:**
1. Delete and re-create the `nyc_taxis` collection
2. Check cluster health
3. Index 50 % of the corpus (30 s warmup)
4. Hard commit
5. Optimize
6. Run in parallel for `time_period` seconds:
   - Index remaining 50 % of the corpus
   - All search/aggregation/sort operations
   - Periodic CLUSTERSTATUS calls (every 10 s)
   - Periodic hard commits (every 30 s)

**Additional parameters:**

| Parameter | Default | Description |
|-----------|---------|-------------|
| `time_period` | `300` | Duration in seconds for the parallel phase |
| `<op>_time_period` | `time_period` | Override time period per search operation |

---

## indexing-search-longevity-test

A longevity test that indexes and queries entirely concurrently from the start (no initial
sequential index phase).

**Schedule:**
1. Delete and re-create the `nyc_taxis` collection
2. Check cluster health
3. Run in parallel for `time_period` seconds:
   - Index the full corpus in a loop (`looped: true`)
   - All search/aggregation/sort operations
   - Periodic CLUSTERSTATUS calls (every 10 s)
   - Periodic hard commits (every 30 s)

**Additional parameters:** same `time_period` parameters as `search-longevity-test`.
