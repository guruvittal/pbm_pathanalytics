# BigQuery Path Analytics: Therapeutic Refill DAGs & Medication Adherence

## Overview
Evaluating medication adherence (Proportion of Days Covered - PDC) under CMS Medicare Part D Star Ratings requires shifting early prescription refills forward to eliminate double-counting and credit continuous possession (CMS Attachment L). When patients transition from single-ingredient therapies to combination regimens, multi-drug dependencies form a Directed Acyclic Graph (DAG) bounded by the latest parent supply exhaustion date.

## The BigQuery Solution
This repository demonstrates scalable topological scheduling via Max-Plus recurrence inside Google Cloud BigQuery across two production-grade architectures:
1. **JavaScript UDF Array-Fold:** A single-pass $O(V+E)$ dynamic programming algorithm tracking in-memory molecule exhaustion dates, engineered for high-throughput multi-million-row nightly ETL pipelines.
2. **BigQuery Graph (ISO GQL):** Declarative `GRAPH_TABLE` queries using quantified path patterns `{0, 10}` that traverse predecessor refill chains and maximize candidate exhaustion dates for transparent compliance audits and lineage tracing.

## Verified Clinical Archetypes
Both approaches achieve 100% precision across 19 claims in 5 clinical archetypes:
- **Linear Cascades (Alice):** Cumulative early refill stockpiling (+22-day shift).
- **Combo Switches (Bob):** Multi-parent fan-in where Lisinopril and HCTZ converge into Zestoretic (lifts PDC from 79.8% to 100%, flipping performance to 5 Stars).
- **Triple Diamond DAG (Charlie):** Three-molecule fan-in and subsequent step-down (+25-day shift).
- **Care Gaps (Diana):** Genuine therapy gaps accurately preserved with zero false shifts.
- **Refill Jitter (Evan):** Realistic refill variations resolved (+20-day shift).

## Getting Started
Open [`rx_codelab.ipynb`](rx_codelab.ipynb) in Colab or BigQuery Studio to explore the complete synthetic data, graph DDL, and PDC lift analytics.
