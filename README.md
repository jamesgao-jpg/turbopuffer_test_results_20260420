# Turbopuffer vs Zilliz Cloud — Benchmark Report

A reproducible, customer-facing comparison of **Turbopuffer** against **Zilliz Cloud Tiered** and **Zilliz Cloud Capacity** on LAION 100 M (single-namespace) and Cohere 10 M / 1,000-tenant (multi-tenant) workloads.

## Contents

- [`turbopuffer_vs_zilliz_final_report.md`](turbopuffer_vs_zilliz_final_report.md) — the report. Five sections: writes/index construction, search, single-tenant cost, multi-tenant cost, delete-consistency behavior. Every claim backed by a measured number with the operating regime spelled out.
- [`reproducibility_guide.md`](reproducibility_guide.md) — per-section canonical commands, VDBBench commit pin, and script paths so any published number can be re-run.

## Tested on

`m6i.2xlarge` (8 vCPU, 32 GB RAM) in AWS `us-west-2`, against the same region on all three systems. Datasets pulled from the public `s3://assets.zilliz.com/benchmark/` bucket.

## Pricing basis

Cost tables use the TB query-billing rate empirically reconciled against an actual TB billing-dashboard reading on 2026-04-20. See § 3 of the report for per-row derivation.
