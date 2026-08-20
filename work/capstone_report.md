# Capstone Report — Refresh / Content Opportunity Scoring

- **Author:** Aman
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** https://github.com/aman-data-search/flyrank-internship-ml
- **Date:** August 2026

---

## 0. Abstract
Maintaining organic search visibility across enterprise content inventories requires timely identification of decaying assets before traffic loss becomes irreversible. In this work, we formulate search traffic decay as a supervised classification and ranking task evaluated across 101,441 content items spanning 44 independent client domains. We extract 60-day historical search exposure, CTR efficiency, and query portfolio diversity metrics using DuckDB against a 78.8-million-row warehouse release. Using a strict out-of-client validation split (GroupShuffleSplit on client ID), our gradient boosted tree ensemble achieves a 0.869 ROC-AUC and 96.0% Precision@50, substantially outperforming heuristic baseline rules (0.418 ROC-AUC and 38.0% Precision@50 against a 52.2% class prior). We operationalize these calibrated probabilities into a four-tier editorial action playbook that ranks decay risk by expected search exposure loss.

---

## 1. Problem framing
Digital publishing and SEO teams face an operational bottleneck: inventories span thousands of published URLs, but editorial bandwidth is strictly constrained. Content teams cannot manually audit every page every month.

* **Decision Supported:** Editorial bandwidth triage — moving from reactive ad-hoc rewrites to a proactive priority queue.
* **Unit of Analysis:** A single pseudonymized content item (`content_hash_id`) belonging to a single client (`client_hash_id`) evaluated over a 60-day historical and outcome window.
* **Output:** Continuous calibrated decay probability $P(\text{decline})$, an Expected Exposure Loss priority score, and four deterministic action reason codes.
* **Human Action:** Editorial teams execute targeted interventions: defending Page 1 rankings, rewriting underperforming title/meta snippets, expanding thin topic clusters, or pruning stale assets.
* **Cost of Errors:**
  * *False Positive:* Consumes 4–8 hours of editorial time rewriting content that was healthy or experiencing brief seasonality, risking disruption to stable rankings.
  * *False Negative:* Leaves high-traffic URLs to decay unnoticed, resulting in compounding losses in search impressions, organic visibility, and conversions.
* **Why ML Helps:** Traffic decay is driven by non-linear interactions across ranking positions, query portfolio diversification, and click-through efficiency that cannot be captured by static boolean filters.

---

## 2. Data safety
* **Data Used:** The official gated pseudonymized warehouse release (`FlyRank/internship-warehouse`, Build ID: `flyrank_pseudonymized_warehouse_release_v20260703`), querying `dim_clients` (104 rows), `dim_content` (519k rows), `fact_query_90d` (2.41M rows), and mid-panel partitions of `fact_content_daily_performance` (March–April 2026).
* **Quarantined Columns (Leakage Prevention):**
  * Target-derived columns (`trend_direction`, `trend_pct`) were strictly excluded.
  * Outcome-window metrics (`imp_last30`, `clk_last30`) were quarantined exclusively for label calculation.
  * Proprietary internal scoring flags (`priority_score`, `health_score`, `action_type`) were excluded to prevent circular learning.
* **Public Safety:** All URLs, domain names, client names, and raw query strings have been permanently pseudonymized into cryptographic hashes. No client-identifying details appear anywhere in `work/`.

---

## 3. Baseline
To establish a rigorous comparison, we implemented a transparent multi-factor heuristic rule reflecting standard industry practice:
$$\text{Baseline Score} = 0.45 \times \text{Visibility} + 0.35 \times \text{Freshness Risk} + 0.20 \times \text{Position Opportunity}$$

* **Why it's a fair comparison:** It incorporates the same historical signals available to the model (impressions, days since update, striking-distance average positions) without future-window knowledge.
* **Baseline Performance (Out-of-Client Test Set, $N=7{,}038$):**
  * **ROC-AUC:** 0.4183 (worse than random due to heavy volume bias on non-declining pages)
  * **Precision@20:** 40.0% (vs. 52.2% class prior)
  * **Precision@50:** 38.0% (vs. 52.2% class prior)
  * **Precision@100:** 37.0% (vs. 52.2% class prior)

---

## 4. Model / analysis
* **Method:** Histogram-based Gradient Boosted Classification Trees (`HistGradientBoostingClassifier`) alongside a linear benchmark (`LogisticRegression`). Gradient boosted trees natively handle non-linear interactions and skewed exposure distributions.
* **Exact Feature List (11 Features):**
  * *Volume & Efficiency:* `imp_prev30`, `clk_prev30`, `ctr_prev30`, `pos_prev30`
  * *Query Portfolio Diversity:* `visible_queries`, `rare_share`, `anon_share`, `top_query_share`
  * *Freshness & Metadata:* `content_age_days`, `days_since_last_update`, `content_type`
* **Target Definition:** Binary classification where `is_declining = 1` indicates an empirical impression contraction greater than 20% in the subsequent 30-day outcome window ($\text{imp\_last30} < 0.80 \times \text{imp\_prev30}$).

---

## 5. Evaluation
* **Validation Design:** Strict 80/20 `GroupShuffleSplit` partitioned on `client_hash_id` (35 training clients, $N=94{,}403$; 9 test clients, $N=7{,}038$; 0 overlapping clients). This evaluates out-of-domain generalization on entirely unseen websites.

### Model vs. Baseline Comparison Table (Held-Out Test Clients)
| Model / Approach | ROC-AUC | Brier Score | Precision@20 | Precision@50 | Precision@100 |
| :--- | :---: | :---: | :---: | :---: | :---: |
| Naive Class Prior (Base Rate) | 0.5000 | 0.2495 | 52.2% | 52.2% | 52.2% |
| Heuristic Action Rule Baseline | 0.4183 | — | 40.0% | 38.0% | 37.0% |
| Logistic Regression (Linear) | 0.7949 | 0.1869 | 80.0% | 84.0% | 84.0% |
| **Gradient Boosted Trees (HistGB)** | **0.8688** | **0.1541** | **100.0%** | **96.0%** | **97.0%** |

* **Error Analysis:** The heuristic rule failed because raw volume and age do not indicate decay; high-traffic evergreen pages were systematically misclassified as urgent declines (false positives). The HistGB model leveraged query portfolio diversification (`visible_queries`) to accurately separate stable high-volume pages from vulnerable single-query assets.

---

## 6. Interpretation
* **Permutation Feature Importance (ROC-AUC Drop):**
  1. `visible_queries` (+0.3115): The single strongest predictor of resilience. Content ranking across multiple distinct queries is far less likely to suffer abrupt monthly traffic collapse.
  2. `anon_share` (+0.1205): High reliance on aggregated anonymized long-tail queries indicates ranking instability.
  3. `imp_prev30` (+0.1033): High historical impressions provide scale context.
  4. `content_age_days` (+0.0441): Aging assets carry moderate baseline decay probability.
* **Negative Findings:** Categorical `content_type` and raw `word_count` exhibited near-zero standalone feature importance (+0.0000 to +0.0010), demonstrating that content structure alone is uninformative without search visibility context.

---

## 7. Recommendation
Pages are prioritized by **Expected Exposure Loss**:
$$\text{Priority Score} = P(\text{decline} \mid \mathbf{x}) \times \log_{10}(\text{imp\_prev30} + 1)$$

### Operational Action Playbook
1. **`high_volume_page_one_decay` $\to$ Defend Core Asset:** ($P \ge 0.60$, $\text{pos} \le 10$, $\text{imp} \ge 1{,}000$). Deep factual refresh, schema validation, and competitor content gap audit.
2. **`striking_distance_ctr_decay` $\to$ Optimize Title & SERP Snippet:** ($P \ge 0.60$, $\text{pos} \le 20$, $\text{CTR} < 1.0\%$). Rewrite meta titles and headers to improve click-through conversion.
3. **`concentrated_query_risk` $\to$ Topic Diversification:** ($P \ge 0.60$, $\text{top\_query\_share} \ge 70\%$). Add subheadings and expand adjacent subtopics to broaden search footprint.
4. **`stale_inventory_decay` $\to$ Standard Content Refresh:** ($P \ge 0.60$, $\text{days\_since\_update} \ge 180$). Routine copy updates and link cleanup.

* **Limitations:** The model captures observational patterns and decision-support signals. It does not measure Google algorithm causality or guarantee traffic recovery.

---

## 8. Reproducibility
* **Environment:** Python 3.10+, `duckdb>=1.0.0`, `scikit-learn>=1.3.0`, `pandas>=2.0.0`, `matplotlib>=3.7.0`.
* **Random Seeds:** `random_state=42` across all data splits and model initializations.
* **Verification Artifacts:**
  * Notebook: `work/notebooks/capstone.ipynb`
  * Metrics Receipt: `work/outputs/capstone_metrics.json`
  * Action Queue CSV: `work/outputs/capstone_recommendations.csv`
  * Evaluation Chart: `work/outputs/capstone_model_evaluation.png`

---

## 9. Acknowledgments & data credit
Built on the FlyRank ML Internship dataset (https://flyrank.ai).