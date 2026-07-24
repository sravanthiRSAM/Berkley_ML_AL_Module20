### Forecasting Customer Provisioning Ticket Volume & Resolution Effort

**Author:** Sravanthi Gandu

#### Executive summary

Provisioning teams currently react to spikes in Customer Provisioning Tickets (CPT)
*after* they happen, which leads to missed SLAs and last-minute staffing scrambles. This
project uses one year of internal ticket data (2026 year-to-date, ~44K order line items
across ~17K tickets) to (1) understand what drives how long a ticket takes to resolve and
(2) lay the groundwork for forecasting incoming ticket volume.

This initial report covers the exploratory data analysis (EDA) and a **baseline regression
model** that predicts a ticket's resolution time from information available **at intake**.
The baseline Ridge model explains ~24% of the variance in (log) resolution time and
produces a ranked list of the factors that drive delay — a first, interpretable version of
the "which tickets will be slow?" signal the operations team needs.

#### Rationale

Why should anyone care about this question? Because reacting late is expensive.
Overstaffing in quiet periods wastes money; understaffing in busy periods breaks SLAs and
erodes customer trust. If managers can *see the wave coming* and know *which tickets will
be heavy*, they can plan headcount ahead of time, shift work toward bottleneck areas, and
warn customers early instead of after a deadline slips. The goal is a simpler, calmer
planning process that operations staff can act on without needing to understand the model
behind it.

#### Research Question

Can we forecast the volume of incoming Customer Provisioning Tickets (CPT) and predict the
resolution effort of each ticket at intake, in order to anticipate workload and flag
at-risk tickets before they breach SLA?

#### Data Sources

- **Internal `cpt_active_tickets` table** from the provisioning system
  (`dpaas_uccatalog_prd.pdbia.cpt_active_tickets` in Databricks Unity Catalog).
- This report uses the **2026 year-to-date** slice: **44,299** order line items spanning
  **16,932** distinct tickets, from 2026-01-01 to 2026-07-24.
- The extract keeps analytical fields only — ticket dates (`create_date`, `due_date`,
  `start_date`, `end_date`, `implementation_date`), status fields (`status`,
  `status_reason`, `type`, `order_type`, `order_reason`), business dimensions (`geo`,
  `country`, `market_segment`, `cloud`, `offer_family`), and workload keys.
- **Privacy:** the `assignee` field (employee email) is one-way hashed to `assignee_hash`
  at extraction, and no customer/employee PII (names, emails, org names) is included in
  this repository. The extraction query lives in [`extract_data.py`](extract_data.py).

#### Methodology

- **Data cleaning:** robust ISO-8601 date parsing, removal of exact-duplicate rows,
  correction of negative (back-dated) resolution durations, IQR-based flagging of
  long-running outliers (kept, not deleted, since they are the "at-risk" cases of
  interest), and explicit `"UNKNOWN"` imputation of missing categoricals.
- **Feature engineering:** derived the target `resolution_days`
  (`implementation_date − create_date`), intake-time calendar features
  (month, ISO week, day-of-week, hour), and `line_items_per_ticket` as a ticket-size proxy.
- **EDA & visualization:** pandas, Seaborn/Matplotlib, and Plotly to explore categorical
  distributions, the (heavily right-skewed) resolution-time distribution, resolution time
  across business dimensions, a numeric correlation matrix, and the weekly/monthly ticket
  arrival pattern.
- **Baseline model:** a **Ridge regression** (with cross-validated regularization) on
  `log1p(resolution_days)`, using one-hot-encoded categoricals plus scaled numeric
  features, benchmarked against a naive mean predictor. Only intake-time features are used,
  so no future information leaks into the model.

#### Results

- **Resolution time is extremely right-skewed** — median **15.2 days** but a mean of
  **~121 days** and a tail past 1,700 days — which is why the target is modeled on a log
  scale.
- **Volume arrives unevenly** week to week with clear spikes, and the mix is dominated by
  the **COM (commercial)** segment, while **GOV/EDU** and certain offer families carry
  longer resolution tails — motivating a forward-looking volume forecast.
- **Baseline model performance (held-out test set, n = 1,115):**

  | Model | MAE (days) | RMSE (days) | R² (log scale) |
  |---|---|---|---|
  | Baseline (mean) | 115.9 | 269.6 | 0.00 |
  | **Ridge regression** | **111.7** | **258.8** | **0.24** |

- The Ridge model beats the naive baseline on every metric and explains **~24%** of the
  variance in log resolution time — real signal for a first baseline. The largest drivers
  of *longer* resolution are certain offer families (e.g. data-services / analytics
  offerings) and larger multi-line tickets; several other offer families and clouds are
  associated with *faster* resolution.
- **Evaluation metric:** **MAE (days)** is the primary metric — it is in interpretable
  units and robust to the heavy skew; **RMSE (days)** is tracked because it penalizes the
  large misses that correspond to SLA breaches; **R² (log scale)** gives a scale-free
  goodness-of-fit for comparing future models.

#### Next steps

- Build the companion **time-series volume forecast** (weekly/monthly ticket counts via
  classical decomposition and ARMA) to complete the "how many are coming" half.
- Add **non-linear regressors** (Random Forest, Gradient Boosting) to capture interactions
  the linear baseline misses, and address the collinearity between the calendar features.
- Reframe the effort estimate as an explicit **SLA-breach classification** flag at intake.
- Enrich features (requestor system behavior, historical assignee throughput) and validate
  on a full multi-year window rather than 2026 YTD.

#### Outline of project

- [Notebook 1 — Data Cleaning & Feature Engineering](notebooks/01_data_cleaning_and_feature_engineering.ipynb)
- [Notebook 2 — Exploratory Data Analysis](notebooks/02_exploratory_data_analysis.ipynb)
- [Notebook 3 — Baseline Model](notebooks/03_baseline_model.ipynb)

Data extraction script: [`extract_data.py`](extract_data.py) · Cleaned dataset:
`data/cpt_clean.csv`

##### Contact and Further Information

Sravanthi Gandu — sravanthireddy.gandu@gmail.com
