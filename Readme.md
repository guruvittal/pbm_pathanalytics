# BigQuery Path Analytics: Therapeutic Refill DAGs & Medication Adherence

## Overview
Evaluating medication adherence (Proportion of Days Covered - PDC) under CMS Medicare Part D Star Ratings requires shifting early prescription refills forward to eliminate double-counting and credit continuous possession (CMS Attachment L). When patients transition from mono-therapy to combination pills, multi-drug dependencies form a Directed Acyclic Graph (DAG) bounded by the latest parent supply.

## Why Traditional SQL Fails
Standard SQL recursive CTEs (`WITH RECURSIVE`) fail on multi-ingredient DAGs. GoogleSQL forbids `GROUP BY` and `MAX()` inside recursive terms, preventing parent synchronization. Without barrier reduction, converging paths fire prematurely, generating exponential duplicate rows and spurious care gaps that depress Star Ratings.

## The BigQuery Solution
This repository demonstrates topological scheduling via Max-Plus recurrence across two architectures:
1. **JavaScript UDF:** Single-pass $O(V+E)$ dynamic programming fold optimal for multi-million-row nightly production ETL.
2. **BigQuery Graph (ISO GQL):** Declarative `GRAPH_TABLE` queries with bounded path quantification `{0, 10}` maximizing exhaustion dates across converging paths for auditability.

## Verified Clinical Archetypes
Tested across 19 claims in 5 clinical scenarios:
- **Linear Cascades (Alice):** Cumulative early refill stockpiling (+22d shift).
- **Combo Switches (Bob):** Multi-parent fan-in (79.8% to 100% PDC; flips to 5 Stars).
- **Triple Diamond DAG (Charlie):** Three-molecule fan-in and step-down (+25d shift).
- **Care Gaps (Diana):** Genuine therapy gaps preserved (0d shift).
- **Refill Jitter (Evan):** Timing variations resolved (+20d shift).

## Getting Started
Open [`rx_codelab.ipynb`](rx_codelab.ipynb) in Colab or BigQuery Studio to explore synthetic data, graph DDL, and side-by-side PDC lift analytics.
