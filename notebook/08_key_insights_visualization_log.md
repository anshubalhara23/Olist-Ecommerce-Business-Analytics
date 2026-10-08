# 08 — Key Insights Visualization Log
Olist E-Commerce Python Analytics

Documents what `08_key_insights_visualization.ipynb` shows. Unlike 04-07, this notebook doesn't derive new findings — every chart visualizes a result already established and numerically confirmed in an earlier notebook's log. Built with matplotlib + seaborn only, as one combined notebook rather than four separate per-notebook plotting notebooks, so all 8 key charts live in a single place a reader can scan quickly.

Charts 3, 4, 6, 7, 8 read small pre-aggregated summary tables exported as CSVs from 05/06/07 (`../data/summary/*.csv`) instead of re-running full merge pipelines. Charts 1 and 2 are cheap enough (≤1 merge) to recompute fresh from the cleaned base tables.

---

## Foundational decision
Each chart is paired with the exact finding it visualizes, so the chart and the number behind it never drift apart — the chart's purpose is to make an already-confirmed finding legible at a glance, not to re-derive it.

---

## Chart 1 — Monthly sales trend (04)
**Type**: Line chart.
Visualizes the gross/net monthly revenue series from 04, Section 2 (Sales Over Time). Makes the ramp-up (2016-09/10/12) and tail-off (2018-09/10) months visually obvious as the near-empty edges they are, rather than requiring the reader to cross-check volume separately.

## Chart 2 — Category revenue Pareto (04)
**Type**: Pareto chart (bar + cumulative-% line, dual axis via `ax.twinx()`).
Visualizes 04, Q6's concentration figures: top 3 categories = 25.2% of revenue, top 20 = 83.9%. The cumulative line makes the steepness of the curve immediately visible — a single number like "83.9%" doesn't convey how front-loaded the curve is the way the line does.

## Chart 3 — RFM heatmap (05)
**Type**: `sns.heatmap` on the Recency × Monetary tertile cross-tab.
Visualizes 05, Section B Q2's near-uniform 3×3 cell distribution — the heatmap's flat coloring across all 9 cells is itself the finding: Recency and Monetary behave as independent variables in this customer base, with no "loyal high-spender" cluster to highlight.

## Chart 4 — Seller revenue concentration curve (05)
**Type**: Lorenz-curve-style cumulative concentration plot.
Visualizes 05, Section A Q2's multi-cutoff concentration figures (top 5/10/30/50/100 sellers), shown as a continuous curve instead of discrete cutoff points — makes clear that seller concentration is steeper than category concentration (Chart 2), supporting Summary takeaway #1 in the 05 log.

## Chart 5 — Monthly late-delivery rate, annotated (06)
**Type**: Annotated line chart (`ax.annotate` with arrows).
Visualizes 06, Q7's monthly late-rate series, with the two statistically real spikes called out directly on the chart: Nov 2017 (14.3%) and Feb/Mar 2018 (16.0%/21.4%). The near-empty 2016 ramp-up months are shown but not annotated as spikes, consistent with the log's noise-vs-signal distinction.

## Chart 6 — Same-state vs. cross-state delivery (06)
**Type**: Grouped bar chart, mean + median side by side.
Visualizes 06, Q9: same-state delivery (7.46 days) vs. cross-state (14.59 days) — mean and median shown together (rather than a boxplot) specifically to avoid outlier distortion, consistent with the right-skew theme used throughout the project.

## Chart 7 — Repeat rate by first-order delivery status (07)
**Type**: Bar chart.
Visualizes 07, A.1: 2.62% (Late) vs. 3.13% (On Time) vs. 4.23% (Unknown) repeat-customer rate. The Unknown bar is shown with its "retry order, not genuine loyalty" caveat carried over from the 07 log, so the chart doesn't imply Unknown orders are the best outcome.

## Chart 8 — Seller scorecard scatter plot (07)
**Type**: Scatter plot, review score vs. delay, with `axhline`/`axvline` threshold reference lines at review=4 and delay=median.
Visualizes 07, D.4's tightened-threshold scorecard: top-50 revenue sellers plotted by review and delay, with the 16-pass/34-fail split visible as a quadrant pattern — most of the 34 "weak" sellers cluster just past the delay line rather than the review line, visually reinforcing the log's point that delay speed (not satisfaction) drives most of the weak flag.

---

## Summary takeaways
1. Every chart in this notebook ties back to a specific, already-confirmed number in an earlier log — nothing here is a new finding, only a visualization of one.
2. Three visualization choices were made specifically to avoid misleading a quick-glance reader: mean+median bars instead of boxplots (Chart 6, avoids outlier distortion), a Lorenz-style curve instead of discrete bars (Chart 4, shows steepness continuously), and explicit annotation of only the statistically real spikes rather than every local peak (Chart 5, avoids drawing attention to near-zero-volume noise).
3. The export/recompute split (CSV summary tables for 05/06/07-derived charts, fresh recompute for 04-derived charts) keeps this notebook fast to run without requiring it to re-run the full cleaning/merge pipeline from scratch.
