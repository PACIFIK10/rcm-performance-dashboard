# Revenue Cycle Performance Analytics Dashboard

**Tool:** Tableau Desktop | **Backend:** PostgreSQL (Supabase) | **Data volume:** ~119,000 simulated claims across 36 months

## Project Overview

Healthcare revenue cycle management (RCM) teams live and die by a handful of operational metrics: how fast claims get paid, how often they get denied, which payers and specialties are dragging down performance, and whether actual collections are tracking against forecast. This dashboard was built to answer those questions at a glance, using a simulated but realistically-structured claims dataset modeled on patterns from my own experience working in healthcare revenue cycle operations.

Rather than pulling from a static CSV, I deployed a 7-table relational schema (claims, denials, denial reasons, payers, providers, AR snapshots, and monthly forecasts) to a live PostgreSQL database on Supabase, then connected Tableau directly to it — treating the project as an end-to-end pipeline exercise, not just a visualization exercise.

## Data Generation (Python / Google Colab)

Before any dashboard work began, the underlying dataset had to exist. Rather than using an off-the-shelf CSV, I wrote a Python data-generation pipeline in Google Colab (`psycopg2`, `faker`, `pandas`) that:

- **Deployed the 7-table schema** directly to Supabase via `psycopg2`
- **Seeded reference data** — payers (commercial, Medicare, Medicaid, etc.) and providers across multiple specialties, plus a denial-reason lookup table
- **Simulated ~119,677 claims across 36 months**, with every parameter grounded in real RCM domain knowledge rather than arbitrary randomness — for example, Medicaid was built in as the slowest-paying payer, Medicare as the highest-denial-rate payer, specialty-level denial rates ranging from ~32% (Chiropractic) down to ~17% (Plastic Surgery), and a 45% resubmission rate reflecting realistic claim write-off behavior
- **Validated data at every stage** before inserting it into Supabase, so downstream analysis started from clean, referentially-consistent tables
- **Loaded everything into Supabase** through the session pooler connection, the same endpoint later connected to Tableau

This step is what makes the dashboard's numbers tell a coherent, realistic story rather than looking like noise — the denial-rate and AR-aging patterns visible in the dashboard are a direct result of decisions made at the data-generation stage.

## Business Questions the Dashboard Answers

1. **Where are we losing revenue to denials, and with which payers?**
2. **How long is cash sitting in accounts receivable, and which payers are the slowest to pay?**
3. **Which provider specialties are getting claims right on the first submission — and which need process support?**
4. **Are we collecting what we forecasted, month over month?**

## What Each Chart Shows

**Denial Rate by Payer** — A ranked bar chart of denial rate (denied claims ÷ total claims) per payer. Medicare Part B and Medicare Advantage came out highest in the simulation (~34–38%), consistent with the more rigid documentation and medical-necessity requirements those payers typically enforce. Commercial payers (Aetna/UHC) sat lowest (~20%), reflecting comparatively streamlined adjudication rules.

**Days in AR** — Average days between claim submission and payment, by payer. State Medicaid and Managed Medicaid were the slowest payers (~50–55 days), which tracks with real-world Medicaid reimbursement timelines. Medicare Part B was fastest (~30 days).

**First-Pass Resolution Rate by Specialty** — The share of claims resolved correctly on the first submission, with no denial or resubmission, broken out by provider specialty. Chiropractic Care and Optometry led at ~75–80%, benefiting from simpler, more standardized claim structures. Plastic & Reconstructive Surgery trailed at ~55–58%, reflecting the heavier documentation and prior-authorization burden those claims typically carry.

**Monthly Forecast vs. Actual Collections** — A 36-month dual-line trend comparing forecasted to actual collections, letting a revenue cycle leader see at a glance whether the department is tracking ahead of or behind projection in any given month.

## Technical Build Notes

- **Data modeling:** Built the primary data source as a relationship model (not a traditional join) in Tableau, linking claim → denial → denial_reason and claim → payer / provider, to avoid row duplication from fan-out. The `monthly_forecast` table was intentionally kept as a separate data source, since it's a pre-aggregated summary table with no shared grain with claim-level data.
- **Extract vs. Live connection:** Switched from a live PostgreSQL connection to an Extract after running into repeated connection instability — a good practical lesson in when a live connection is the wrong choice for a Supabase free-tier backend.
- **Debugging a real infrastructure issue:** Mid-project, Supabase auto-paused the database after a period of inactivity (a free-tier behavior), which broke the live connection and wiped the in-progress workbook. Diagnosed the failure from the Postgres error message, resumed the project from the Supabase dashboard, and rebuilt the data model and all four sheets from scratch — a useful, unplanned exercise in troubleshooting a production-like connection failure rather than just building charts against a dataset that always works.
- **Interactivity:** Added a dashboard filter action so selecting a payer in the Denial Rate chart cross-filters the related Days in AR view, letting a user drill from "which payer denies the most" straight into "how slow is that same payer to pay."
- **Packaging:** Saved as a Tableau Packaged Workbook (.twbx) so the extract travels with the file — the dashboard renders correctly even if the live Supabase connection is unavailable.

## Skills Demonstrated

- Synthetic data generation grounded in domain expertise (Python, Faker)
- Relational data modeling and schema design for a healthcare claims domain
- SQL/PostgreSQL backend management (Supabase)
- Tableau calculated fields, relationship modeling, dual-axis charts, and dashboard interactivity
- Applying domain expertise from RCM operations to validate that the data and results were realistic, not just technically correct
- Troubleshooting a live data connection failure under real infrastructure constraints

---

*This project is paired with a Medicare Claims Denial Prediction Model as part of a combined portfolio demonstrating both descriptive analytics (this dashboard) and predictive modeling for healthcare revenue cycle use cases.*
