# Capstone Report — CTR / Engagement Opportunity Scoring

- **Author:** Jothsna Sri Kathyayani
- **Lane:** Lane 4 — CTR / Engagement Opportunity Scoring
- **Repo:** `jsk-flyrank-ml-internship`
- **Date:** 2026-10-18

## 0. Abstract

This project investigates whether machine learning can help prioritize pages whose click-through rates (CTR) fall below a reference benchmark for pages in similar search-position tiers. The analysis uses FlyRank internship warehouse data, with March 2026 features, April outcomes for development, May features for scoring, and June outcomes for a held-out temporal evaluation. A Random Forest classifier was compared with a transparent CTR-gap baseline using a client-level development split and a frozen later-period evaluation. On the June evaluation set of 21,548 page pairs across eight held-out clients, the model achieved 82.55% Precision@10%, compared with 80.65% for the baseline, against a 55.35% positive-label rate. The output is intended to support human review and content-prioritization decisions; it does not establish that making a recommended change will cause CTR to improve.

## 1. Problem framing

### Decision supported

The project supports the decision of which pages a FlyRank editor or analyst should investigate first when looking for potential search-engagement opportunities.

- **Unit of analysis:** A client-content page pair observed across monthly feature and outcome windows.
- **Output:** A ranked list of candidate pages for human review.
- **Human action:** Investigate a page's search intent, content relevance, snippet quality, competition, and measurement coverage before deciding whether an update is warranted.
- **Cost of a wrong call:** A high-ranked page may not benefit from an edit, while a genuinely underperforming page may be missed. Unnecessary changes can waste editorial time and may not improve performance.

A transparent rule can rank pages based on their CTR gap from a reference. Machine learning may help combine multiple available signals to prioritize the review queue. The model is therefore evaluated against a simple baseline rather than assumed to be useful merely because it is more complex.

The task is **decision support**, not a claim to predict Google's ranking algorithm or estimate the causal effect of content changes.

## 2. Data safety

### Data and time windows

The analysis uses FlyRank internship warehouse data queried through DuckDB and remote Parquet sources. The experiment uses monthly page-level features and outcomes:

- **March 2026:** Development feature snapshot.
- **April 2026:** Development outcome period and reference-median calculation.
- **May 2026:** Frozen-model scoring features for temporal evaluation.
- **June 2026:** Held-out outcome period used for the final evaluation.

The development cohort required at least 300 March impressions, valid March CTR and weighted position, and a measurable April CTR outcome. Page pairs without measurable outcomes were excluded from evaluation rather than treated as zero-performing pages.

The June evaluation was performed after the evaluation protocol was fixed. The June outcome was not used to tune the model or select features.

### Features used

The nine March feature inputs were:

1. `log_impressions_march`
2. `ctr_march_pct`
3. `weighted_position_march`
4. `gsc_days_available`
5. `search_volume`
6. `word_count`
7. `march_position_tier`
8. `content_type`
9. `main_intent`

The content intent field was joined using client and content keys. Pseudonymous identifiers were used for joins, grouping, and client-level splitting, not as model features.

### Exclusions and leakage precautions

Client-identifying details, domains, page URLs, private search queries, and identifiable page-level predictions must not be included in public artifacts or Git commits. Pseudonymous IDs are not model features.

The label-derived fields `trend_direction` and `trend_pct` must be excluded from the feature matrix because they may expose outcome information. The model uses features from the defined feature snapshot rather than future outcomes.

CTR is undefined when impressions are zero. Missing or unavailable measurements are not equivalent to a true zero. Observations without measurable outcome CTR are excluded from the relevant evaluation.

**Data-safety verification required before submission:** Audit every file under `work/` and the staged Git changes for client-identifying details, raw warehouse extracts, private queries, URLs, IDs, and page-level predictions. Do not claim the audit is complete until it has actually been performed.

## 3. Baseline

The baseline is a transparent CTR-gap ranking rule. Within each March position tier, it compares a page's March CTR with the training clients' median March CTR for that tier. Pages with larger positive shortfalls receive higher priority.

The baseline is appropriate because it is simple, interpretable, and uses information available at the feature snapshot. It is evaluated on the same held-out observations and with the same ranking metrics as the Random Forest model.

### Development evaluation

The development evaluation used 11,338 page pairs from eight held-out clients. The positive-label rate was 51.52%.

| Metric | CTR-gap baseline | Random Forest |
|---|---:|---:|
| Precision@10% | 80.69% | 85.19% |
| Lift over positive-label rate | 1.566× | 1.654× |
| Recall@10% | 15.67% | 16.54% |
| Mean positive shortfall among top 10% | 0.1195 pp | 0.0887 pp |

The Random Forest improved Precision@10% by 4.50 percentage points in development. However, the baseline's selected pages had a larger mean shortfall from the reference, indicating that identifying more below-reference pages and identifying pages with the largest shortfalls are not identical objectives.

## 4. Model / analysis

### Method

A Random Forest classifier was trained to rank pages by their likelihood of falling below the outcome-period reference CTR for their position tier. The method supports nonlinear relationships and interactions among numeric and categorical features without requiring a linear relationship between each feature and the target.

The preprocessing pipeline used median imputation for numeric features and most-frequent imputation plus one-hot encoding for categorical features. The model configuration was:

- `n_estimators=200`
- `min_samples_leaf=10`
- `class_weight="balanced"`
- `random_state=42`
- `n_jobs=-1`

The development split was grouped by client: 29 clients for training and eight held-out clients for testing. The split contained 61,874 training page pairs and 11,338 test page pairs.

### Target definition

A page is labelled positive when its outcome-period CTR is below the frozen April reference median for the page's feature-period search-position tier.

The training-client April medians used as the reference were:

| Position tier | April reference median CTR |
|---|---:|
| Positions 11–20 | 0.0920% |
| Positions 21–50 | 0.0471% |
| Positions 4–10 | 0.1524% |
| Top 3 | 0.1700% |

The positions 51+ tier was excluded from the primary model comparison because the frozen reference median was zero, making the positive-shortfall comparison unsuitable for that tier.

### June temporal evaluation

The trained model was applied to May features without retraining. The June outcome was compared against the frozen April reference medians for the corresponding May position tiers. The evaluated June set contained 21,548 page pairs across eight held-out clients; the positive-label rate was 55.35%.

The June model configuration was not tuned using June outcomes.

## 5. Evaluation

### Split strategy

The development split was grouped by client, using seed 42, to reduce the risk of pages from the same client appearing in both training and test sets. A later temporal evaluation used May features and June outcomes without retraining the model.

This provides evidence about performance on held-out clients and a later outcome period, although it does not guarantee generalization to every client or future search environment.

### June held-out results

| Metric | CTR-gap baseline | Random Forest |
|---|---:|---:|
| Evaluated page pairs | 21,548 | 21,548 |
| Held-out clients | 8 | 8 |
| Positive-label rate | 55.35% | 55.35% |
| Precision@10% | 80.65% | 82.55% |
| Lift over positive-label rate | 1.457× | 1.491× |
| Recall@10% | 14.57% | 14.92% |
| Mean positive shortfall among top 10% | 0.1236 pp | 0.0582 pp |
| ROC-AUC | — | 0.7908 |
| Average precision | — | 0.7784 |

The model improved Precision@10% by 1.90 percentage points over the baseline in June. Its lift was also higher. The baseline nevertheless selected a top-ranked group with a larger mean positive shortfall, so the model should not be interpreted as uniformly superior for every possible prioritization objective.

### Position-tier error analysis

June results showed meaningful variation across tiers:

- **Positions 11–20:** Model Precision@10% was 76.53%, compared with 76.24% for the baseline.
- **Positions 21–50:** Model Precision@10% was 86.88%, compared with 86.05% for the baseline.
- **Positions 4–10:** The baseline performed better, at 81.53%, compared with 74.29% for the model.
- **Top 3:** The baseline achieved 100.00%, compared with 80.77% for the model, but this tier contained only 255 evaluated page pairs and its top 10% group was very small. The result is therefore uncertain.

These findings suggest that a single ranking policy may not be optimal for all position tiers. They support further evaluation of a tier-aware or hybrid policy rather than an immediate universal replacement of the baseline.

## 6. Interpretation

The Random Forest improved overall ranking precision in both the development evaluation and the June held-out evaluation. The June improvement was smaller than the development improvement, which is consistent with the possibility that performance changes across time periods and client populations.

The largest reported feature importance was March CTR (approximately 0.575), followed by log impressions (approximately 0.157), weighted position (approximately 0.096), and word count (approximately 0.074). Search volume, GSC availability, and categorical features contributed smaller reported importance values.

These importances describe the fitted model's use of the available inputs; they do not demonstrate that changing any one feature will improve CTR. In particular, CTR is closely related to the label definition, so its high importance is not evidence of a causal discovery.

The tier analysis is an important negative result: the model did not outperform the baseline in every segment. The smaller June precision gain also indicates that the larger development gain should not be assumed to persist unchanged.

## 7. Recommendation

1. **Pilot a human-reviewed ranking queue.** Use the model as a supplementary signal for identifying pages to investigate, not as an automatic content-editing system.
2. **Test a hybrid policy.** Compare the model and baseline in positions 11–50, where the model performed better in June, against the baseline in positions 4–10 and top 3, where the baseline performed better. This is a candidate policy, not a proven improvement.
3. **Require editorial investigation.** Review intent match, content relevance, snippet quality, competition, and measurement coverage before deciding whether to make a change.
4. **Evaluate prospectively.** Freeze the proposed policy before the next outcome period and compare it against the baseline on new data without tuning on that evaluation period.
5. **Monitor by tier and data coverage.** Track Precision@10%, recall, lift, measurement availability, and segment-level performance. Reassess the model if performance or data distributions change.

**Confidence:** Moderate that the model improved overall June ranking precision on the evaluated cohort; lower confidence that it will generalize to other clients or future periods; insufficient evidence to claim that model-guided content changes cause CTR improvements.

## 8. Reproducibility

### Recorded experiment configuration

- **Development split seed:** 42
- **Training clients:** 29
- **Held-out development clients:** 8
- **Model:** Random Forest classifier
- **Model parameters:** 200 trees, minimum leaf size 10, balanced class weights
- **Preprocessing:** Numeric median imputation; categorical most-frequent imputation and one-hot encoding
- **Development record:** `work/notebooks/development_record.json`
- **Notebook:** `work/notebooks/capstone.ipynb`

The experiment used DuckDB to query remote Parquet data and avoid loading the full warehouse table into memory. Access credentials should be configured through the Colab Secrets or an equivalent environment-secret mechanism; credentials must not be committed.

### Re-run instructions

The following is the intended workflow, **not yet a verified end-to-end reproduction command sequence**:

1. Clone the repository and install the documented dependencies.
2. Configure the required warehouse access secret in the execution environment.
3. Run the notebook from a fresh runtime, in order, to rebuild the development cohort, client-level split, preprocessing pipeline, model, baseline, and evaluation metrics.
4. Build the sealed temporal evaluation frame using the committed evaluation script or notebook cell.
5. Produce the aggregate metrics file from that run and compare the output with the reported results.

Before submission, add or verify the exact dependency installation command and exact notebook/script execution commands. Confirm that the notebook can rebuild its data frames and metrics without relying on variables from a previous Colab session.

### Sealed evaluation artifacts required

To make the June evaluation independently checkable, commit both:

- The code cell or script that constructs the sealed evaluation frame and applies the frozen evaluation protocol.
- The aggregate metrics JSON produced by that evaluation.

The existing `work/notebooks/development_record.json` records aggregate experiment results, but its presence alone does not prove that the sealed evaluation can be reconstructed. Do not claim full reproducibility until the build code and generated metrics file are committed and verified.

Do not commit raw data, credentials, client-identifying information, page URLs, private queries, or identifiable predictions.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).
