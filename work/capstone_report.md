# Capstone Report — Refresh / Content Opportunity Scoring
- **Author:** Aiman Shahid
- **Lane:** Refresh / Content Opportunity Scoring
- **Repo:** github.com/aimanshahid800/flyrank-ml-internship
- **Date:** August 2026

## 0. Abstract
This study asks which pages in a content portfolio most need a refresh, and whether a simple age/position rule is enough to answer that question. A rule-based baseline scored AUC 0.494 (chance-level) against a measured decline label, so a Random Forest model was trained instead, reaching AUC 0.760 under a grouped, leakage-audited validation split. The model surfaced a non-obvious, actionable pattern: many top-priority pages are not stale — they already rank well but suffer from low click-through, pointing to snippet/title fixes rather than content rewrites. The resulting ranked queue is 3.5x more reliable than the rule baseline (11.3% vs 41.1% low-confidence picks) and is intended as decision-support for editorial review, not an automated or causal claim about search performance. Full limitations, human-review requirements, and reproducibility details are documented throughout this report.

## 1. Problem framing
Content editors managing large page portfolios need a repeatable way to prioritize which pages deserve a refresh this cycle — manual review of thousands of pages doesn't scale. **Unit of analysis:** individual content page (content_id). **Output:** a decline-priority score and ranked action label (refresh_now / monitor / no_action). **Action taken:** an editor reviews and refreshes the highest-priority pages first. **Cost of a wrong call:** a false refresh_now wastes editor time on a page that wasn't declining; a false no_action lets a genuinely declining high-traffic page keep losing visibility unnoticed. ML helps here because the baseline rule (age + position gap) tested at chance-level (AUC 0.494) — a learned model was needed to find real signal.

## 2. Data safety
**Source:** `data/raw/content_refresh_anonymized.csv` (30,000 rows, 44 columns), derived from FlyRank/internship-warehouse (Hugging Face, gated) — `fact_content_daily_performance` (month=2026-03) + `dim_content`. **Features used:** content_age_days, impressions_last_30d, avg_position, ctr. **Excluded:** GA4 session data (only 4.2% availability), all client-identifying fields, raw domain/URL/query strings. **Leakage check (w06 audit):** trend_pct and trend_direction's raw components were checked; no hard leakage found. One caution flagged — impressions_last_30d partially overlaps with how trend_direction (and therefore the label) is itself defined, since trend is calculated from 30-day impression change. This is a definitional overlap, not future-data leakage. content_id is used only for grouping in the split, never as a feature. Confirmed: no client names, domains, or private queries appear anywhere in `work/`.

## 3. Baseline
A rule-based baseline was built first (ML-07): score = 0.6×age_score + 0.4×position_gap_score, producing reason codes (stale_content / position_opportunity / low_priority) and action labels. Tested against is_declining on the same data, this rule scored **AUC 0.494** — effectively chance. This is a fair, honest comparison since it uses the identical label and data as the model, and it's a genuine negative finding: age and position-gap alone don't predict decline in this dataset.

## 4. Model / analysis
**Method:** Random Forest (n_estimators=200, max_depth=8, class_weight="balanced") — chosen for its ability to handle non-linear feature interactions and provide feature importances. **Features:** content_age_days, impressions_last_30d, avg_position, ctr (left out: GA4-derived fields, due to low coverage). **Target:** is_declining = 1 if trend_direction == "down" (>10% impression decline, 30d vs previous 30d), else 0 — a proxy for real-world decline, not a verified ground truth.

## 5. Evaluation
**Split:** GroupShuffleSplit by content_id, 80/20 — not time-aware, since only one month of data is available. **Split sensitivity check (w06):** naive random split AUC = 0.747 vs grouped split AUC = 0.760 — minimal difference, since each row is a distinct content_id with no natural duplicates in this snapshot; grouping remains the correct default. **Metrics on same split:** Baseline AUC 0.494, Random Forest AUC 0.760, base rate 0.542. **Error analysis:** 725 false negatives — model under-flags high-traffic declining pages (avg 2,889 impressions), trusting popularity as a stability signal. 1,050 false positives — model over-flags newer (avg 206 days), lower-traffic (avg 485 impressions) pages.

## 6. Interpretation
**Feature importances:** avg_position (0.375) > content_age_days (0.258) > impressions_last_30d (0.256) > ctr (0.111). **Key finding:** top-priority pages in the ranked queue are not primarily old/stale — they are high-ranking pages (position #1-3) with low CTR (reason_code low_ctr is the largest single category: 1,663 of 3,108 refresh_now picks). This is a surprise relative to the naive assumption that refresh = rewrite old content. **Negative result:** the baseline rule's core assumption (age + position gap predicts decline) does not hold in this dataset — a well-understood "no effect," not a failure.

## 7. Recommendation
Full ranked queue: `work/outputs/w07_ranked_action_queue.csv` (30,000 pages scored — 3,108 refresh_now, 19,646 monitor, 7,246 no_action). **How an editor uses this tomorrow:** start with refresh_now picks tagged low_ctr — these are fast wins (title/snippet fix, not full rewrite) on pages that already rank well. **Reliability:** only 11.3% of refresh_now picks have <10 impressions, vs 41.1% for the rule baseline — meaningfully more trustworthy. **Human review required:** top 20 refresh_now picks each cycle, any pick with <10 impressions, any low_ctr pick on an already top-3 page (may need manual snippet diagnosis). **Never automate:** auto-publishing changes, deprioritizing no_action pages without manual check (false negatives exist), using model output as sole justification without human-verified traffic data. **Confidence:** directional and decision-support only — not causal, not validated beyond this single-month sample.

## 8. Reproducibility
**Repo:** github.com/aimanshahid800/flyrank-ml-internship. **Notebooks (run in order):** `work/notebooks/w04_baseline_score.ipynb` (ML-07 baseline) → `work/notebooks/w05_model.ipynb` (model + grouped split) → `work/notebooks/w06_validation_audit.ipynb` (split sensitivity + leakage audit) → `work/notebooks/w07_action_playbook.ipynb` (ranked queue + exports) → `work/notebooks/capstone.ipynb` (full write-up). **To re-run:** clone repo, open each notebook in Colab, Runtime → Run All (random_state=42 throughout). **Exports:** `work/outputs/w07_ranked_action_queue.csv`, `w07_top10_preview.csv`, `w07_action_summary.csv`, chart PNGs in `work/outputs/charts/`. All notebooks executed top-to-bottom with no errors and committed under `work/notebooks/`.

## 9. Acknowledgments & data credit
Built on the FlyRank ML Internship dataset. Data source: https://flyrank.ai

---
> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · no causal claims without an experiment or causal design · no
> "predicted Google's algorithm" · no client-identifying details · numbers in this report
> match a fresh re-run.
