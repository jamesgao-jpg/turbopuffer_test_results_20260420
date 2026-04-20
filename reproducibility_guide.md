# Reproducibility Guide

Companion to [`turbopuffer_vs_zilliz_final_report.md`](turbopuffer_vs_zilliz_final_report.md). Each section below matches a section of the report.

## Global setup

- **VDBBench fork / branch:** [`jamesgao-jpg/VectorDBBench` @ `tp_vs_zilliz_0415`](https://github.com/jamesgao-jpg/VectorDBBench/tree/tp_vs_zilliz_0415) — pinned commit [`a177dbf`](https://github.com/jamesgao-jpg/VectorDBBench/tree/a177dbf/experiments/tpuf_zilliz)
- **Driver host:** `m6i.2xlarge`, `aws-us-west-2`
- **Standalone scripts:** `VectorDBBench/experiments/tpuf_zilliz/`
- **Dataset cache:** `/tmp/vectordb_bench/dataset/{laion/laion_large_100m,cohere/cohere_large_10m}/`. VDBBench auto-downloads on first `--case-type` reference.

## 1 · Writes and index construction

Scripts: `experiments/tpuf_zilliz/insertion/{batch100_bp_on_reattach.py, wait_for_index_drain.py}`

```bash
# Zilliz Cloud Tiered 8 CU
NUM_PER_BATCH=1000 vectordbbench zillizautoindex \
  --uri <tiered-uri> --user-name <user> --password <pw> --token <token> \
  --case-type Performance768D100M --collection-name LAION100M \
  --load-concurrency 0 --drop-old --load \
  --skip-search-serial --skip-search-concurrent \
  --db-label laion100m_insertion_tiered

# Zilliz Cloud Capacity 12 CU — same, with --uri <capacity-uri> and NUM_PER_BATCH=5000

# Turbopuffer bp=OFF
DISABLE_BACKPRESSURE=1 NUM_PER_BATCH=50000 vectordbbench turbopuffer \
  --api-key <key> --region aws-us-west-2 --namespace laion100m_bulk \
  --case-type Performance768D100M --load-concurrency 0 \
  --skip-search-serial --skip-search-concurrent \
  --db-label laion100m_bulk

# Turbopuffer bp=ON (wall-clock-capped)
cd experiments/tpuf_zilliz/insertion && python batch100_bp_on_reattach.py
```

## 2 · Search

VDBBench-only (no standalone scripts).

```bash
# Unfiltered QPS + recall
vectordbbench <db> ... \
  --case-type Performance768D100M \
  --skip-drop-old --skip-load \
  --search-serial --search-concurrent \
  --num-concurrency 1,5,20,40,60,80 --concurrency-duration 30 \
  --db-label laion100m_search_<label>

# Filter-rate sweep (swap --filter-rate 0.5 / 0.9 / 0.99 / 0.999)
vectordbbench <db> ... \
  --case-type NewIntFilterPerformanceCase \
  --dataset-with-size-type "Large LAION (768dim, 100M)" \
  --filter-rate 0.9 \
  --skip-drop-old --skip-load \
  --search-serial --search-concurrent \
  --num-concurrency 1,5,20,40,60,80 --concurrency-duration 30 \
  --db-label laion100m_filter_0.9_<label>

# True days-idle cold
TPUF_SKIP_CACHE_WARM=1 vectordbbench turbopuffer ... \
  --num-concurrency 1 --concurrency-duration 5 \
  --db-label laion100m_cold_filter0.9_tpuf
```

## 3 · Single-tenant cost

Script: `experiments/tpuf_zilliz/cost/cost_calc.py`

```bash
cd experiments/tpuf_zilliz/cost && python cost_calc.py
# Toggle dim_bytes=2 (fp16) or dim_bytes=4 (fp32) in the file.
```

## 4 · Multi-tenant cost

Scripts: `experiments/tpuf_zilliz/multi_tenant/*`

```bash
cd experiments/tpuf_zilliz/multi_tenant

# One-time tenant assignment
python generate_tenant_labels.py

# Turbopuffer path (1000 namespaces)
python split_by_tenant.py
python load_multitenant_tpuf.py
python search_multitenant_tpuf.py

# Zilliz path (one collection + partition_key, 64 buckets)
python load_multitenant_zilliz.py
python optimize_collection.py
python search_multitenant_zilliz.py
```

## 5 · Turbopuffer delete-consistency

Script: `experiments/tpuf_zilliz/delete_consistency/delete_consistency_test_cohere.py`

```bash
cd experiments/tpuf_zilliz/delete_consistency

# 60-minute run (matches report trace)
POLL_INTERVAL_SEC=60 POLL_ITERATIONS=60 python delete_consistency_test_cohere.py

# 24-hour run (matches "no recovery after 24h")
POLL_INTERVAL_SEC=600 POLL_ITERATIONS=144 python delete_consistency_test_cohere.py
```

## Env vars referenced above (all default off)

| Env var | Purpose |
|---------|---------|
| `NUM_PER_BATCH` | bulk batch size |
| `DISABLE_BACKPRESSURE` | Turbopuffer `disable_backpressure=True` |
| `TPUF_SKIP_CACHE_WARM` | skip Turbopuffer `hint_cache_warm` (§ 2 true cold) |
| `POLL_INTERVAL_SEC`, `POLL_ITERATIONS` | delete-consistency timing (§ 5) |

## Pricing assumptions

- Turbopuffer: $1/PB queried + $0.05/GB returned + $2/GB written (up to 50 % batch discount) + $0.33/GB-mo storage. Tiered discount: 0-32 GB full, 32-128 GB 80 % off, >128 GB 96 % off. Query billing confirmed at **fp32 basis** empirically.
- Zilliz Cloud (`us-west-2`): Zilliz Tiered $0.372/CU-hr, Zilliz Capacity $0.248/CU-hr, $0.025/GB-mo storage.
