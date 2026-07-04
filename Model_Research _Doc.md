# Model Research Document
## AI-Assisted Threat Detection Dashboard

**Purpose of this document:** Before implementation, this document researches
and compares the candidate algorithms, models, data sources, and tools
available for each module of the project, and justifies the choices carried
forward into the build. It is organized to mirror the four modules defined
in the project statement.

---

## 1. Module 1 — Security Data Aggregation & Threat Intelligence Layer

### 1.1 Data Source Research

| Source Type | Examples | Notes |
|---|---|---|
| Network/host logs | Firewall logs, IDS/IPS logs, VPN logs, endpoint logs | Primary raw input; usually semi-structured (syslog, CEF, JSON) |
| Public benchmark datasets | CICIDS2017, NSL-KDD, UNSW-NB15, KDD Cup 99 | Used for training/testing anomaly and classification models before real data is available |
| Threat intelligence feeds | AlienVault OTX, abuse.ch, MISP feeds | Provide known-malicious IPs, domains, and file hashes to enrich events |
| Vulnerability data | CVE/NVD databases, CVSS scores | Used to weight asset risk in Module 3 |
| Attack framework mapping | MITRE ATT&CK matrix | Standardizes attack technique labels (e.g., T1110 Brute Force) across all detected events |

**Recommended approach:** Ingest logs into a normalized event schema
(timestamp, source IP, destination IP/port, protocol, event type, severity,
raw payload) so that every downstream module — anomaly detection, scoring,
dashboarding — works against one consistent structure regardless of the
original log format. Each normalized event is optionally enriched with a
MITRE ATT&CK technique ID and a CVE reference where applicable.

### 1.2 Normalization & Enrichment Techniques Considered

- **Rule-based field mapping** — straightforward, transparent, easy to
  extend; chosen for initial implementation.
- **Schema-on-read with pandas** — flexible for prototyping in Colab/Jupyter
  before a production pipeline exists.
- **ETL frameworks (Apache NiFi, Logstash)** — considered for production
  scale, out of scope for the prototype stage of this project.

---

## 2. Module 2 — AI-Based Threat Detection & Anomaly Analysis Engine

### 2.1 Anomaly Detection Algorithms Compared

| Algorithm | How it works (brief) | Strengths | Limitations |
|---|---|---|---|
| Z-score / statistical thresholding | Flags points beyond N standard deviations from the mean | Simple, fast, explainable | Assumes roughly normal distribution; misses complex patterns |
| Isolation Forest | Isolates points via random recursive splits; anomalies isolate faster | Handles high-dimensional data well, no distribution assumption, scikit-learn support | Less interpretable than rule-based methods |
| One-Class SVM | Learns a boundary around "normal" data | Effective in high-dimensional feature spaces | Sensitive to parameter tuning, slower on large datasets |
| Local Outlier Factor (LOF) | Compares local density of a point to its neighbors | Good for local, density-based anomalies | Computationally heavier; less scalable |
| Autoencoders (Neural Network) | Learns to reconstruct normal traffic; high reconstruction error flags anomalies | Captures complex non-linear patterns | Requires more data and tuning; less interpretable |

**Recommendation:** Start with **Isolation Forest** as the primary anomaly
detector (good balance of accuracy, interpretability, and scikit-learn
support for rapid prototyping), with z-score thresholding used as a
lightweight secondary/explainable check for simple metrics (e.g., packet
size, login frequency).

### 2.2 Classification Models Compared (for labeling attack type once flagged)

| Model | Strengths | Limitations | Fit for this project |
|---|---|---|---|
| Logistic Regression | Simple, fast, interpretable coefficients | Struggles with non-linear boundaries | Good baseline model |
| Random Forest | Handles non-linear data, robust to noise, feature importance built in | Larger model size, less interpretable than single trees | Strong candidate for production classifier |
| Gradient Boosting (XGBoost/LightGBM) | Typically highest accuracy on tabular security data | More tuning required, higher training cost | Candidate for a later accuracy upgrade |
| Support Vector Machine (SVM) | Effective in high-dimensional spaces | Slower on large datasets, less scalable | Considered, not primary choice |

**Recommendation:** Use **Logistic Regression** for the initial working
prototype (fast to train, easy to explain to non-technical stakeholders on
the dashboard), with **Random Forest** as the upgrade path once more labeled
data is available, since it materially improves accuracy while remaining
reasonably interpretable via feature importances.

### 2.3 Confidence Scoring

Threat confidence scores are derived by combining:
1. Model output probability (e.g., `predict_proba` from scikit-learn),
2. Anomaly score from the anomaly detector (e.g., Isolation Forest's
   anomaly score),
3. A rule-based adjustment for known-malicious indicators (IP/domain/hash
   matches from threat intel feeds).

---

## 3. Module 3 — Risk Prioritization & Security Intelligence Module

### 3.1 Risk Scoring Approaches Compared

| Approach | Description | Pros | Cons |
|---|---|---|---|
| Rule-based weighted scoring | Score = weighted sum of severity, asset criticality, exposure, confidence | Transparent, easy to tune and explain | Requires manual weight calibration |
| CVSS-based scoring | Uses the industry-standard CVSS formula for vulnerability severity | Standardized, widely recognized | Designed for vulnerabilities, not full behavioral events |
| Bayesian risk models | Probabilistic combination of multiple risk factors | Handles uncertainty formally | More complex to implement and explain |
| Event correlation (SIEM-style rules) | Combines multiple related low-severity events into one high-severity incident | Reduces alert fatigue, surfaces multi-stage attacks | Requires well-defined correlation rules up front |

**Recommendation:** A **hybrid model** — a rule-based weighted score
(severity level, asset criticality, presence in threat intel feeds, and
anomaly confidence) — as the core scoring engine, supplemented by simple
event correlation logic (e.g., grouping repeated alerts from the same source
IP within a time window) to catch multi-stage attacks such as brute-force
followed by lateral movement.

### 3.2 Asset Criticality Weighting

Assets (devices) are assigned a criticality tier (e.g., DMZ-facing servers
> internal servers > employee laptops), which multiplies into the final risk
score so that identical attack types are prioritized differently depending
on what they're targeting.

---

## 4. Module 4 — Threat Monitoring Dashboard & Security Reporting Platform

### 4.1 Visualization Technology Comparison

| Tool | Strengths | Limitations | Fit for this project |
|---|---|---|---|
| Plotly/Dash | Python-native, highly interactive, integrates directly with the ML pipeline (pandas/scikit-learn) | Requires more manual layout work than BI tools | **Primary choice** — keeps the whole stack in Python |
| Power BI | Strong enterprise BI features, easy drag-and-drop dashboards | Less flexible for custom ML-driven visuals, Windows/enterprise-oriented | Considered for executive reporting exports |
| Tableau | Excellent for exploratory data visualization, strong storytelling features | Commercial licensing, separate from the Python ML pipeline | Considered conceptually, not used directly in the prototype |

**Recommendation:** Build the interactive dashboard with **Plotly/Dash**
since it keeps data science and visualization in a single Python codebase,
which simplifies connecting live model outputs (risk scores, anomaly flags)
directly to dashboard components. Power BI/Tableau concepts are referenced
for exporting periodic executive summary reports.

### 4.2 Dashboard Feature Set (derived from project outcomes)

- Real-time/near-real-time alert feed with severity color-coding
- Attack-type distribution charts (bar/pie)
- Time-series trend of alert volume
- Top offending source IPs / most-targeted devices
- Drill-down from summary view to individual alert detail
- Filter by severity, device, date range, attack type

---

## 5. Recommended Technology Stack Summary

| Module | Primary Choice | Alternative / Upgrade Path |
|---|---|---|
| 1. Data Aggregation & Threat Intel | Rule-based normalization + MITRE ATT&CK mapping | ETL frameworks (NiFi/Logstash) for production scale |
| 2. Anomaly & Classification Engine | Isolation Forest + Logistic Regression | Random Forest / XGBoost for higher accuracy |
| 3. Risk Prioritization | Rule-based weighted scoring + basic event correlation | Bayesian risk modeling |
| 4. Dashboard & Reporting | Plotly/Dash | Power BI/Tableau for executive exports |

---

## 6. Candidate Public Datasets for Prototyping

| Dataset | Description | Typical Use |
|---|---|---|
| CICIDS2017 | Modern labeled network traffic with multiple attack types (DoS, brute force, infiltration, botnet) | Anomaly detection + classification training/testing |
| NSL-KDD | Refined version of the older KDD Cup 99 dataset | Baseline intrusion detection benchmarking |
| UNSW-NB15 | Contemporary mixed normal/attack traffic dataset | Additional classification benchmarking |

These datasets are recommended as substitutes for real organizational log
data during development and testing, before the pipeline is validated
against live or simulated production logs (as referenced in the project's
Milestone 3 evaluation criteria).

---

## 7. References & Further Reading

- MITRE ATT&CK Framework — attack technique taxonomy used for event mapping
- NVD / CVE Database — vulnerability severity and metadata source
- scikit-learn documentation — algorithm implementations (Isolation Forest,
  Logistic Regression, Random Forest, One-Class SVM)
- Plotly/Dash documentation — dashboard framework reference
- CICIDS2017, NSL-KDD, UNSW-NB15 dataset documentation — benchmark dataset
  references for model training and evaluation

