# BigQuery Path Analytics: Therapeutic Refill DAGs & Medication Adherence (PDC)

[![BigQuery](https://img.shields.io/badge/Google_Cloud-BigQuery_Graph-blue.svg)](https://cloud.google.com/bigquery)
[![ISO GQL](https://img.shields.io/badge/Standard-ISO_GQL_2024-green.svg)](https://www.iso.org)
[![CMS Star Ratings](https://img.shields.io/badge/CMS_Part_D-Attachment_L_PDC-orange.svg)](https://www.cms.gov)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

An enterprise-grade reference architecture and codelab demonstrating how **Google Cloud BigQuery Graph (ISO GQL)** and **Topological Scheduling (Max-Plus Algebra)** solve complex multi-ingredient prescription refill cascades for Pharmacy Benefit Managers (PBMs) and Medicare Part D health plans.

---

## 1. Executive Summary & Clinical Background

Under **CMS Medicare Part C & Part D Star Ratings** and **Pharmacy Quality Alliance (PQA)** measure specifications (Attachment L: *Medication Adherence Measure Calculations*), adherence is evaluated using **Proportion of Days Covered (PDC)**.

### The Regulatory Rule: Shifting Overlapping Days of Supply
When a patient refills a medication before their previous supply is exhausted, the subsequent fill’s start date must be **shifted forward** to the day after the previous supply ends. This prevents double-counting and credits the beneficiary with continuous possession. 

When a member transitions from mono-therapy (e.g., Lisinopril 20mg and HCTZ 12.5mg) to a combination pill (e.g., Zestoretic / Lisinopril+HCTZ), the combination fill is constrained by **both** ancestor drug supplies. Its coverage start date cannot begin until the **latest** parent supply is exhausted:

$$T_{\text{adj\_start}}(v) = \max\Big(\text{fill\_date}(v), \;\; \max_{u \in \text{Parents}(v)} T_{\text{adj\_stop}}(u)\Big)$$
$$T_{\text{adj\_stop}}(v) = T_{\text{adj\_start}}(v) + \text{days\_supply}(v)$$
$$\text{time\_shift}(v) = T_{\text{adj\_start}}(v) - \text{fill\_date}(v)$$

---

## 2. Why Traditional Relational SQL Fails

Traditional SQL databases and data warehouses fail on this calculation due to fundamental relational and compiler limitations:

1. **`WITH RECURSIVE` Compiler Aggregation Ban:**  
   Resolving converging multi-drug dependencies requires a `MAX()` reduction across incoming parents. GoogleSQL / ZetaSQL (and ANSI SQL) strictly forbids `GROUP BY`, `MAX()`, and window functions inside the recursive term:
   ```
   ERROR: Recursive queries do not support GROUP BY, aggregation, or window functions in the recursive term.
   ```
2. **Premature Firing & Duplicate Explosion ($2^K$):**  
   Without an aggregation barrier, a combination claim fires prematurely for each incoming parent branch at different recursion depths. Because recursive CTEs are append-only (`UNION ALL`), flawed rows cannot be retracted, resulting in an exponential $O(2^K)$ row explosion of corrupted duplicate claims.
3. **Ghost Gaps & Lost Star Ratings:**  
   Treating overlapping claims as wasted supply erases valid coverage days, artificially depressing PDC scores (e.g., from 100% down to 79.8%) and costing health plans millions in lost CMS Quality Bonus Payments.

---

## 3. Supported BigQuery Architectures

This repository provides two complete, mathematically equivalent implementations achieving **100% exact match** with zero variance:

| Dimension | JavaScript UDF Array-Fold (`adjust_claims_dag`) | BigQuery Graph / ISO GQL (`GRAPH_TABLE`) |
| :--- | :--- | :--- |
| **Computational Complexity** | **$O(V + E)$ linear time** per member partition | **$O(\sum \text{paths})$** bounded by max hop depth $K$ |
| **Execution Engine** | In-memory V8 container inside Dremel worker | Pure declarative GoogleSQL / GQL relational execution |
| **Prerequisites** | BigQuery Standard, On-Demand, or Enterprise | BigQuery Enterprise / Enterprise Plus reservation |
| **Lineage & Auditability** | Procedural state machine | Transparent, queryable graph paths in standard SQL |
| **Best Practice Use Case** | **High-throughput nightly batch ETL (>100M rows)** | **Interactive cohort exploration & compliance audit queries** |

---

## 4. The 5 Clinical Archetypes Verified

| Patient Archetype | Clinical Scenario | Raw Naive PDC | Path Analytics Adj PDC | Star Rating Lift |
| :--- | :--- | :---: | :---: | :---: |
| **MEM_101 (Alice)** | **Early Refill Stockpiling Cascade:** Consecutive 9, 15, and 22-day early refills for Lisinopril mono-therapy. | 87.78% | **100.00%** | **+12.22%** |
| **MEM_102 (Bob)** | **Combination Pill Switch:** Lisinopril and HCTZ merge into combo pill (Zestoretic), bounded by HCTZ stop date. | 79.81% ❌ | **100.00% 🌟** | **+20.19% (Flips to 5 Stars!)** |
| **MEM_103 (Charlie)** | **Triple Diamond DAG:** 3 concurrent molecules (A+B+C) fan-in to a triple combo pill and step back to mono-therapy. | 77.06% ❌ | **100.00% 🌟** | **+22.94% (Flips to 5 Stars!)** |
| **MEM_104 (Diana)** | **True Gaps in Care:** Genuine 60-day and 90-day medication abandonment gaps. Supply is zero; 0-day shift applied. | 37.19% ❌ | **37.19% ❌** | **0.00% (Clinical integrity preserved)** |
| **MEM_105 (Evan)** | **Baseline Statin Refill Jitter:** 11-day and 20-day refill timing variations for Atorvastatin. | 77.78% ❌ | **100.00% 🌟** | **+22.22% (Flips to 5 Stars!)** |

---

## 5. Declarative BigQuery Graph Query

The following ISO GQL query evaluates cumulative supply exhaustion along ancestor paths directly in BigQuery:

```sql
SELECT
  member_id,
  claim_id,
  drug_name,
  fill_date,
  days_supply,
  GREATEST(0, MAX(path_shift)) AS time_shift,
  DATE_ADD(fill_date, INTERVAL GREATEST(0, MAX(path_shift)) DAY) AS adjusted_start_date,
  DATE_ADD(DATE_ADD(fill_date, INTERVAL GREATEST(0, MAX(path_shift)) DAY), INTERVAL days_supply DAY) AS adjusted_stop_date
FROM GRAPH_TABLE(
  `your-project.pbm_codelab.RxSupplyGraph`
  MATCH ((nodes:Claim)-[:PRECEDES_FOR_MOLECULE]->){0, 10}(v:Claim)
  COLUMNS (
    v.member_id AS member_id,
    v.claim_id AS claim_id,
    v.drug_name AS drug_name,
    v.fill_date AS fill_date,
    v.days_supply AS days_supply,
    CASE 
      WHEN ARRAY_LENGTH(nodes) = 0 THEN 0
      ELSE DATE_DIFF(
        DATE_ADD(
          (SELECT ARRAY_AGG(n.fill_date ORDER BY n.fill_date ASC LIMIT 1)[OFFSET(0)] FROM UNNEST(nodes) AS n),
          INTERVAL (SELECT SUM(n.days_supply) FROM UNNEST(nodes) AS n) DAY
        ),
        v.fill_date,
        DAY
      )
    END AS path_shift
  )
)
GROUP BY member_id, claim_id, drug_name, fill_date, days_supply
ORDER BY member_id, fill_date, claim_id;
```

---

## 6. Repository Contents

* [`rx_codelab.ipynb`](rx_codelab.ipynb): Complete, runnable Jupyter/Colab notebook containing:
  * Synthetic multi-ingredient claims dataset generation.
  * In-memory JavaScript UDF dynamic programming fold.
  * Predecessor edge sequencing table DDL (`claim_edges`).
  * BigQuery Property Graph definition (`RxSupplyGraph`).
  * Interactive GQL path queries and graph visualizations.
  * Side-by-side Raw vs. Adjusted PDC scoring and Star Rating lift analytics.

---

## 7. Regulatory References

* **CMS Quality Technical Notes:** [Medicare Part C & Part D Star Ratings Technical Notes](https://www.cms.gov/files/document/2024-star-ratings-technical-notes.pdf) — *Attachment L: Medication Adherence Measure Calculations (Pages 161–164).*
* **Pharmacy Quality Alliance (PQA):** *PQA Measure Manual: Proportion of Days Covered (PDC) for Diabetes, Hypertension (RAS Antagonists), and Cholesterol (Statins).*

---

## 8. License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
