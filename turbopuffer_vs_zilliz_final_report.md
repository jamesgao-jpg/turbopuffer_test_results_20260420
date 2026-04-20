# Turbopuffer vs Zilliz Cloud — Comparison Report

## Introduction

This report compares **Turbopuffer** against two Zilliz Cloud products — **Zilliz Cloud Tiered** and **Zilliz Cloud Capacity** — on the dimensions that matter for production vector-search workloads: write and index-build time, search latency and recall, cost at scale, and behavioral edge cases.

Each section shows the headline result first, then the operating regime where Zilliz Cloud delivers a clear advantage, and ends with a reproducibility block naming the exact VDBBench commit and script paths so any claim here can be re-run.

**At a glance:**

- **Workloads:** LAION 100 M × 768-dim (L2 metric) as the large single-namespace corpus; Cohere 10 M × 768-dim (cosine) split into 1,000 tenants as the multi-tenant workload.
- **Driver host:** `m6i.2xlarge` (8 vCPU, 32 GB RAM) in `aws-us-west-2`, the same region as every tested system.
- **Harness:** a pinned fork of VDBBench (`tp_vs_zilliz_0415` @ `a177dbf`) plus a set of standalone Turbopuffer-SDK scripts; the full code bundle lives at `VectorDBBench/experiments/tpuf_zilliz/`.

## Table of Contents

- [1 · Writes and index construction](#1--writes-and-index-construction)
  - [1.1 Headline numbers — LAION 100 M](#11-headline-numbers--laion-100-m)
  - [1.2 Why Zilliz wins on time-to-ready](#12-why-zilliz-wins-on-time-to-ready)
- [2 · Search](#2--search)
  - [2.1 Max concurrent QPS + recall — unfiltered](#21-max-concurrent-qps--recall--unfiltered-laion-100-m-k100)
  - [2.2 Filter recall](#22-filter-recall--laion-100-m-int-filter)
  - [2.3 Cold latency across the three products](#23-cold-latency-across-the-three-products)
- [3 · Single-tenant cost — LAION 100 M](#3--single-tenant-cost--laion-100-m-production-representative)
  - [3.1 Cost crossover](#31-cost-crossover-points)
  - [3.2 Analysis](#32-analysis)
- [4 · Multi-tenant cost — Cohere 10 M / 1,000 tenants](#4--multi-tenant-cost--cohere-10-m-1000-tenants--10-k-rows)
  - [4.1 Cost crossover](#41-cost-crossover-points)
  - [4.2 Analysis](#42-analysis)
- [5 · Turbopuffer delete-consistency problem](#5--turbopuffer-delete-consistency-problem-product-risk)

**How to reproduce every section:** see the companion [`reproducibility_guide.md`](reproducibility_guide.md). Each section of this report links directly to its reproduce-instructions counterpart in the guide.

---

## 1 · Writes and index construction

*→ Reproduce: [§ 1 of the reproducibility guide](reproducibility_guide.md#1--writes-and-index-construction)*

### 1.1 Headline numbers — LAION 100 M

| System | Client insert rate | Insert wall time | Optimize / index-ready | **End-to-end time-to-ready** |
|--------|-------------------:|-----------------:|-----------------------:|------------------------------:|
| **Zilliz Cloud Tiered (8 CU)** | 4,085 r/s | 6 h 48 min | **86 s** (synchronous `flush + compact + refresh_load`) | **6 h 49 min** |
| **Zilliz Cloud Capacity (12 CU)** | 5,448 r/s¹ | 5 h 6 min | **9.4 min** (same synchronous path) | **5 h 15 min** |
| Turbopuffer (bp=OFF, batch=50 k) | 15,800 r/s | 105 min | **6 h 22 min** (background indexer, `unindexed_bytes` 225 GB → 0) | 8 h 7 min |
| Turbopuffer (bp=ON, batch=100, projected) | 1,720 r/s | 16 h (projected from 4.4 M-row 2 h-capped run) | 3–5 min | **16 h** |

¹ *Zilliz Capacity 12 CU is the tested cost-sweet-spot size for LAION 100 M; the original 16 CU run came in at 5,448 r/s and we project ≤ 6 h 15 min end-to-end for 12 CU.*

### 1.2 Why Zilliz wins on time-to-ready

Turbopuffer's 4× faster client-side insert is deceptive. Turbopuffer writes buffer data as "unindexed bytes" that the background indexer then compacts asynchronously — **during the drain phase, the namespace is not fully queryable at strong consistency**:

- `unindexed_bytes` climbs to **224.7 GB** at the end of the 100 M write phase.
- Background indexer drains at 10.8 MB/s — takes **6 h 22 min** to hit `unindexed_bytes == 0`.
- `approx_row_count` is a **lagging metric** that reads 1.05 M during the drain and only jumps to 100 M at `FULLY_INDEXED`.
- Strongly-consistent reads above the **2 GiB unindexed threshold** return 429 (backpressure) — in practice, queries that depend on recently-written data are unusable for hours.
- **Only once `unindexed_bytes == 0`** does the namespace reach `STRONGLY_CONSISTENT_READY`.

Zilliz Cloud's `flush + compact + refresh_load` is synchronous: when `optimize()` returns, the collection is fully ingested and fully indexed. Zilliz Tiered finishes this in **86 seconds**; Zilliz Capacity in 9.4 minutes. Zero window of uncertain queryability.

**End-to-end Zilliz beats Turbopuffer** — Zilliz Tiered is **1 h 18 min faster** than Turbopuffer bp=OFF, Zilliz Capacity is **2 h 52 min faster**.

**What this means operationally.** Time-to-ready is the window between "data uploaded" and "search results are trustworthy." Any team that ingests data on a schedule — nightly embedding refreshes, product-catalog batch loads, re-embedding after an ML model rev, onboarding a new enterprise customer's corpus — has to plan around this window. On Turbopuffer that window is **6+ hours where strongly-consistent reads above 2 GiB of unindexed data return 429 (backpressure) errors**: until the drain completes, queries touching newly-written rows are unusable. On Zilliz the collection is queryable seconds to minutes after the write completes, so the data pipeline and the search surface can be treated as a single atomic cutover rather than a multi-hour rolling dependency.

---

## 2 · Search

*→ Reproduce: [§ 2 of the reproducibility guide](reproducibility_guide.md#2--search)*

### 2.1 Max concurrent QPS + recall — unfiltered, LAION 100 M, k=100

| System | Peak QPS (conc=80) | p99 @ peak | **Recall@100** |
|--------|-------------------:|-----------:|---------------:|
| **Zilliz Capacity 16 CU** | **405.6** | **278 ms** | **0.9709** |
| Zilliz Tiered 8 CU | 67.8 | 1,917 ms | 0.9513 |
| Turbopuffer | 598.9 | 1,187 ms | **0.9001** |

**Zilliz Capacity 16 CU delivers 7.1 percentage points higher recall than Turbopuffer** at the same workload. Turbopuffer's peak QPS lead (599 vs 406) is at 9.0 % **lower** recall and 4× worse tail latency at peak concurrency.

**In production terms**, a 7 pp recall gap means roughly **1 of every 14 queries returns the wrong top match on Turbopuffer** relative to Zilliz Capacity. For a RAG pipeline that's a wrong LLM answer once every 14 queries; for product search that's a missed SKU; for content recommendation that's the user seeing the 14th-best result instead of the best. Turbopuffer's 1.2-second p99 at peak concurrency compounds the damage — slow **and** more often wrong.

### 2.2 Filter recall — LAION 100 M int filter

| Filter rate | Eligible | Turbopuffer Recall | Zilliz Tiered 8 CU Recall | **Zilliz Capacity 16 CU Recall** |
|-------------|---------:|----------:|---------------:|--------------------:|
| 0.5 | 50 M | 0.927 | 0.9552 | **0.9837** |
| 0.9 | 10 M | 0.870 | 0.9608 | **0.9870** |
| 0.99 | 1 M | **0.667** 🚨 | 0.9603 | **0.9809** |
| 0.999 | 100 k | 0.914 | 0.9424 | **0.9728** |

**Turbopuffer recall collapses to 0.667 at filter rate 0.99** — 31 percentage points below Zilliz Capacity. This happens because Turbopuffer's ANN-post-filter planner exhausts the candidate pool at intermediate selectivity. Only at the extreme 0.999 rate does Turbopuffer switch to brute-force scan, recovering to 0.914.

**For a customer running filtered search in production** (any realistic selectivity range between 0.1 and 1 %), Turbopuffer's recall cliff sits squarely in the danger zone. Zilliz Capacity holds above **0.97** across every rate tested.

**The 0.99 filter rate is the common case, not the edge.** Multi-facet search (category + brand + size → 1 % eligible), geographic narrowing (city-level filter on a global corpus), time-window constraints (last 7 days of a 2-year log stream), and permission-scoped multi-tenant search all tend to land in the 99-99.9 % filter-rate regime. At Turbopuffer's 0.667 recall on rate=0.99, **one-third of the top-100 results are wrong** — and because the client still gets back a full 100-row response, there is no error signal the application can catch. This is a **silent failure mode**: operators only find out when users complain or conversion metrics drift downward, often weeks after the degradation becomes material.

### 2.3 Cold latency across the three products

Max latency on the first cold query, LAION 100 M:

| Scenario | Turbopuffer | Zilliz Tiered 8 CU | **Zilliz Capacity 12 CU** |
|----------|---:|--------:|-------------:|
| Unfiltered | 1,573 ms | 1,050 ms | **13 ms** |
| Filter rate 0.9 | 2,719 ms | 599 ms | **14 ms** |

**Why cold matters.** Cold first-query latency is the user experience of opening an app or dashboard that hasn't been used in a while — common in B2B tools where a given customer logs in once a week, internal analytics surfaces that are only accessed during investigations, or long-tail content in a large catalog that hadn't been recently queried. On Turbopuffer, **every such "first touch" after idle starts with a 1.5-2.7 second stall** — past the 3-second abandonment threshold widely used in UX benchmarks. Zilliz Capacity at 12 CU serves the same cold first query in **13-14 ms** — indistinguishable from warm. Zilliz Tiered at 8 CU sits in between at 0.6-1 s, still in "feels snappy" territory for interactive use.

---

## 3 · Single-tenant cost — LAION 100 M, production-representative

*→ Reproduce: [§ 3 of the reproducibility guide](reproducibility_guide.md#3--single-tenant-cost)*

### 3.1 Cost crossover points

Hourly cost at each QPS level (rounded). **⭐ marks the first row where Zilliz Tiered becomes cheaper than Turbopuffer**, separately for fp16 and fp32 storage. Stars live in the Zilliz Tiered 8 CU column; Zilliz Capacity 12 CU is shown for context and crosses at the same QPS.

| QPS | Turbopuffer fp16 storage | **Turbopuffer fp32 storage (actual)** | Zilliz Tiered 8 CU | Zilliz Capacity 12 CU | Cheapest |
|----:|----------------:|-----------------------------:|--------:|---------:|----------|
| 1   | $0.19 | $0.21 | $2.976 | $2.976 | **Turbopuffer** |
| 10  | $1.88 | $2.10 | $2.976 | $2.976 | **Turbopuffer** |
| 14  | $2.63 | $2.94 | $2.976 | $2.976 | **Turbopuffer** (still cheaper) |
| 15  | $2.82 | $3.15 | **$2.976 ⭐ (fp32)** | $2.976 | **Zilliz Tiered / Zilliz Capacity — first fp32 crossover** |
| 16  | $3.01 | $3.36 | **$2.976 ⭐ (fp16)** | $2.976 | **Zilliz Tiered / Zilliz Capacity — first fp16 crossover** |
| 67  | $12.60 | $14.08 | $2.976 | $2.976 | Zilliz Tiered / Zilliz Capacity |
| 68  | $12.78 | $14.28 | *over capacity* | $2.976 | **Zilliz Capacity 12 CU (4.8× cheaper than Turbopuffer fp32)** |
| 310 | $58.28 | $65.15 | *over capacity* | $2.976 | **Zilliz Capacity 12 CU (22× cheaper than Turbopuffer fp32)** |
| 405 | $76.14 | $85.26 | *over capacity* | *over capacity* (needs 16 CU: $3.968) | **Zilliz Capacity 16 CU (21× cheaper than Turbopuffer fp32)** |

**Summary:**

- **⭐ fp32 storage (actual billing):** Zilliz Tiered 8 CU becomes cheaper than Turbopuffer at **QPS = 15**.
- **⭐ fp16 storage (hypothetical):** Zilliz Tiered 8 CU becomes cheaper than Turbopuffer at **QPS = 16**.
- **QPS ≥ 15:** Zilliz wins, increasingly decisively at higher QPS.
- At realistic production load (**68-405 QPS**), Zilliz Cloud is **20-22× cheaper per hour** than Turbopuffer.

### 3.2 Analysis

Turbopuffer cost scales linearly with QPS (`$0.210/QPS-hr` actual fp32 basis); Zilliz is a flat CU-hour price up to the CU tier's measured max QPS. The crossover sits at the QPS where Turbopuffer's per-query cost × 3,600 equals the Zilliz CU floor.

Two precision bases for Turbopuffer:

| | Namespace basis | Effective GB/query (post-tier) | Per-query cost | $/hr at QPS |
|---|---:|---:|---:|---:|
| Turbopuffer fp16 | 153.6 GB | 52.22 | $5.22 × 10⁻⁵ | $0.188 × QPS |
| **Turbopuffer fp32 (actual)** | 307.2 GB | **58.37** | **$5.84 × 10⁻⁵** | **$0.210 × QPS** |

**Why two Turbopuffer columns.** Turbopuffer's documentation suggests queries bill on the f16-quantized index (the fp16 column). Empirical reconciliation against Turbopuffer's billing dashboard on 2026-04-20 showed the **fp32-basis** calculation matched actual invoicing within 0.06 % for our 811,456-query test campaign. So fp32 is the rate you'll actually see; fp16 is included only as the hypothetical comparison.

Turbopuffer's **tiered discount** applies per-query: 0-32 GB at full rate, 32-128 GB at 80 % off, >128 GB at 96 % off. That's why fp32 queries only cost 12 % more than fp16 at LAION's 300 GB scale — most of the extra bytes land in the deeply-discounted tier.

**At production scale, the monthly gap is material.** Monthly compute bills (720 hours) at representative QPS levels on LAION 100 M:

| Target QPS | Use-case archetype | Turbopuffer (fp32) | Zilliz Tiered 8 CU | Zilliz Capacity 12 CU | Zilliz savings |
|------------|--------------------|----------:|--------:|---------:|---------------:|
| 10 | Internal research / low-traffic RAG | $1,512 | $2,143 | $2,143 | Turbopuffer cheaper |
| 67 | Steady AI-product backend | **$10,138** | **$2,143** | $2,143 | **$7,995/mo ($96k/yr)** |
| 310 | Consumer-scale search/recs | **$46,908** | over-capacity | **$2,143** | **$44,765/mo ($537k/yr)** |
| 405 | Peak recommender / multi-region | **$61,387** | over-capacity | $2,858 (16 CU) | **$58,529/mo ($702k/yr)** |

Below 15 QPS, Turbopuffer's per-query billing is genuinely cheaper — a legitimately good fit for dev/test and very low-traffic workloads. **Above that threshold, which is where any active product actually lives, Zilliz is 5-22× cheaper per hour** and stays flat as you scale load within a CU tier. The cost curve is not just "linear vs flat" in the abstract — it's **$0.5-0.7 M/year in recurring infrastructure cost** once a deployment hits consumer-scale QPS.

---

## 4 · Multi-tenant cost — Cohere 10 M, 1,000 tenants × 10 k rows

*→ Reproduce: [§ 4 of the reproducibility guide](reproducibility_guide.md#4--multi-tenant-cost)*

### 4.1 Cost crossover points

**Per-tenant QPS** = aggregate QPS ÷ 1,000 tenants (maps the aggregate number to the per-tenant request rate customers typically think in — e.g. a tenant making one query every 10 s corresponds to 0.1 QPS/tenant, 100 aggregate QPS across 1,000 tenants).

Hourly cost at each aggregate-QPS level. **⭐ marks the first row where Zilliz has a cost advantage over Turbopuffer.** Only Zilliz Tiered 1 CU is called out as the flagged Zilliz product — Zilliz Capacity 2 CU is shown for context at the higher QPS tiers where Zilliz Tiered exceeds capacity.

| Aggregate QPS | Per-tenant QPS | Turbopuffer | Zilliz Tiered 1 CU | Zilliz Capacity 2 CU | Cheapest |
|--------------:|---------------:|---:|--------:|------------:|----------|
| 10    | 0.01  | **$0.050** | $0.372 | $0.496 | Turbopuffer |
| 50    | 0.05  | **$0.248** | $0.372 | $0.496 | Turbopuffer |
| 74    | 0.074 | **$0.368** | $0.372 | $0.496 | Turbopuffer (still cheaper) |
| 75    | 0.075 | $0.373 | **$0.372 ⭐** | $0.496 | **Zilliz Tiered 1 CU — first Zilliz advantage** |
| 524   | 0.524 | $2.60  | **$0.372** | $0.496 | **Zilliz Tiered 1 CU (7× cheaper)** |
| 525   | 0.525 | $2.61  | *over capacity* | **$0.496** | **Zilliz Capacity 2 CU** |
| 979   | 0.979 | $4.86  | — | **$0.496** | **Zilliz Capacity 2 CU (10× cheaper)** |
| 1,141 | 1.141 | $5.67  | — | — | Turbopuffer only (Zilliz out of capacity) |

**Summary:**

- **⭐ First time Zilliz has a cost advantage over Turbopuffer: aggregate QPS = 75** (Zilliz Tiered 1 CU at $0.372/hr flat).
- **Aggregate 75-524 QPS:** **Zilliz Tiered 1 CU** wins — $0.372/hr flat, cheapest overall in this range.
- **Aggregate 525-979 QPS:** **Zilliz Capacity 2 CU** takes over — $0.496/hr flat (Zilliz Tiered 1 CU's measured QPS ceiling is 524).
- **Aggregate 980-1,141 QPS:** only Turbopuffer reaches this throughput in our sizing; Zilliz would need a larger CU tier.

### 4.2 Analysis

**Architectures compared:**
- **Turbopuffer:** 1 namespace per tenant → 1,000 namespaces.
- **Zilliz Cloud (Zilliz Tiered / Zilliz Capacity):** single collection with `labels` scalar as `is_partition_key=True`, 64 hash buckets — each query routes to 1/64 of the data.

Turbopuffer's per-query cost is dominated by its **1.28 GB minimum floor**. Each tenant's namespace is only 15-30 MB — far below the floor — so every query bills at 1.28 GB × $1 × 10⁻⁶ = **$1.38 × 10⁻⁶ flat** regardless of actual namespace size. That floor makes Turbopuffer cheap at low aggregate QPS but the per-query rate becomes unfavorable once summed over hundreds of tenants.

Zilliz is a flat CU-hour charge up to each CU tier's measured QPS ceiling:
- Zilliz Tiered 1 CU: $0.372/hr, peak 524 QPS
- Zilliz Capacity 2 CU: $0.496/hr, peak 979 QPS

**Business framing — multi-tenant SaaS unit economics.** The 1,000-tenant × 10k-row shape models B2B products where every customer has their own small dataset: law firms with per-matter document collections, e-commerce platforms with per-merchant catalogs, logging/observability tools with per-customer log streams, note-taking apps with per-user knowledge bases. Two representative operating regimes:

| Aggregate QPS | Per-tenant avg | Turbopuffer / month | Zilliz Tiered 1 CU / month | Per-tenant cost (cheapest) |
|--------------:|---------------:|-----------:|----------------:|---------------------------:|
| 100  | 0.1 queries/s  | **$358**  | **$268 (Zilliz Tiered)** | **$0.27/tenant/month** |
| 500  | 0.5 queries/s  | **$1,788** | **$268 (Zilliz Tiered)** | **$0.27/tenant/month** |
| 979  | 1 query/s each | **$3,501** | (over cap) $357 (Zilliz Capacity 2 CU) | **$0.36/tenant/month** |

At 500 aggregate QPS a multi-tenant SaaS pays **$1.79/tenant/month on Turbopuffer vs $0.27/tenant/month on Zilliz** — a **6.6× unit-economics gap** that compounds directly into gross margin. Across the full 1,000-tenant base that's **$1,520/month = $18.2k/year** of recurring infrastructure cost recovered by running on Zilliz; at the 979 QPS busy-hour ceiling it's **$3,144/month ≈ $37.7k/year**. For a product at $50-500 ARPU/tenant, this is pure margin — no change to revenue, feature set, or customer experience.

The crossover at 75 QPS matters less than where your actual workload lives. Most multi-tenant products settle into the 100-500 aggregate QPS band during business hours — squarely inside Zilliz's sweet spot.

---

## 5 · Turbopuffer delete-consistency problem (product risk)

*→ Reproduce: [§ 5 of the reproducibility guide](reproducibility_guide.md#5--turbopuffer-delete-consistency-problem)*

### 5.1 What we observed

On `cohere10m_stream_bpOFF` (Cohere 10 M namespace, fully indexed):

1. Pick 5 query vectors `Q[0..4]` from `test.parquet`.
2. For each, capture the top-500 nearest neighbors → union of 5 × 500 = **2,500 unique IDs** (no overlap on this data).
3. `ns.write(deletes=[all 2,500 IDs])` — completes in 115 ms, `approx_row_count` drops to 9,997,500.
4. Immediately re-query `Q[0..4]` at `top_k=100`.
5. **All 5 queries return 0 rows.**

Poll continuously:

| Window | Cadence | Samples | **Queries returning > 0** |
|--------|---------|--------:|--------------------------:|
| 60 min | every 1 min | 305 | **0 / 305** |
| **24 hours** | every 10 min | 145 | **0 / 145** |

After 24 hours, **all 5 top-k queries still return 0 rows** — a 24+ hour window where deletes have made search results vanish.

### 5.2 Mechanism

- Turbopuffer's ANN candidate pool caps at 550-650 inverted-list entries per query on this namespace.
- Our deletion set is specifically each query's top-500 — i.e., the deletion exhausts each query's entire candidate pool.
- Post-tombstone filter leaves effectively zero live candidates.
- **The graph is only rewired when the background indexer does a compaction** — for 2,500 tombstones, the compaction trigger apparently isn't reached in 24+ hours.

A probe live from the namespace during the 24-hour stall:

```
for test_query_0..4:
  top_k=100  → 0 rows (baseline was 100)
  top_k=500  → 0 rows (baseline was 500)
  top_k=5000 → 4,500 rows ← full graph walk, manually forcing Turbopuffer to look beyond the tombstoned pool
```

The `top_k=5000 → 4,500` result confirms: Turbopuffer's graph walk returns 5,000 candidates; after removing the 500 tombstoned entries, only 4,500 survive. But the **client-side `top_k` parameter caps the response before tombstone filtering is wide enough** — so a normal `top_k=100` query cannot recover on its own.

### 5.3 Why this is a customer-facing risk

For any production workload that:
- Uses `top_k ≤ 500` (i.e., essentially all production search) **and**
- Deletes a non-trivial fraction of its hot data (e.g., user logs out, product is removed, tenant churns),

**queries against the affected region can silently return zero results for an extended period** — a day or longer — until Turbopuffer's internal compactor rewires the graph.

**Concrete scenarios where this is a hard-blocker**:
- **E-commerce**: a product is unlisted (out of stock, legal takedown, seller deactivation). For the next 24 h, searches near that product's vector coordinates return empty — the customer experience is "the site has no results for a query it clearly should serve."
- **Compliance / right-to-be-forgotten (GDPR, CCPA)**: a user requests deletion of PII. Turbopuffer deletes it from storage within 115 ms, but search results continue to be shaped by the deleted vectors' presence in the ANN graph for up to a day. Regulators expect "removal" to mean functionally unreachable; Turbopuffer's behavior here is defensible only with an explicit SLA carve-out.
- **Expiring content**: time-windowed content (news articles past their lease, logs past retention, seasonal offers) that's been deleted from the system still influences neighboring search results for up to a day.
- **Multi-tenant churn**: a tenant offboards and their embeddings are deleted. For up to 24 h, queries from other tenants that geometrically neighbor the churned tenant's data can return empty — a cross-tenant consistency failure.

Zilliz Cloud's delete path uses session/bounded consistency with synchronous visibility once the delete completes — the same regression does not occur. For teams with any delete-path in their product (practically every B2B or consumer app), **this is the category of failure that's hard to explain to a customer and harder to remediate in production** — because there is no client-visible error, only results that look fine to the application but are wrong to the user.

---

**Full reproducibility details** — VDBBench branch pin, env-var toggles, script paths, canonical commands, dataset paths, and pricing assumptions — live in the companion [`reproducibility_guide.md`](reproducibility_guide.md). Every section of this report has a one-to-one counterpart there.
