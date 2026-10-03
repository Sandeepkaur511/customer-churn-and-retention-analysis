# Customer Churn & Retention Analysis (Power BI)

A four-page Power BI report and the data analysis behind it, built on a star-schema data model over **5,000 e-commerce customers, 12,065 orders and 7,203 support tickets**. The project has two parts: **how the report is built** (data model, DAX, visual choices) and **what it shows** (where churn concentrates and what goes with it).

![Dashboard preview](screenshots/churn-executive-summary.png)

> **Data note:** the dataset is **synthetic**. Patterns are cleaner and segments more evenly sized than in real data, so the findings demonstrate the *method*, not real business results. Monetary values in the source files are in **₹ (INR)**.

## Business question

An e-commerce business loses roughly one in three customers. Where is churn concentrated, how quickly do customers leave, does a poor support experience go with it, and which customers should a retention team look at first?

## Questions the report answers

- What is the overall churn rate, and how does it trend month by month?
- Which acquisition channels produce the most loyal customers?
- How do support tickets and ratings relate to churn?
- Which region and age segments carry the highest churn?
- How does retention decay across signup cohorts?

## Report pages

| Page | What it shows |
|---|---|
| Executive Summary | Active vs total customers, churn rate, monthly churn volume vs rate, churn by acquisition channel, ticket and rating gaps |
| Cohort & Segment Analysis | Cohort retention heatmap, region × age churn matrix, decomposition tree |
| Predictive / Risk View | Tickets vs risk score (threshold 60), at-risk customer table |
| Customer Profile | Drill-through: one customer's orders and support tickets |

Screenshots are in [`screenshots/`](screenshots/) and a PDF export is in [`report/`](report/).

## How the report is built

### Data model

A star schema with two fact tables. `orders` and `support_tickets` are the facts; `customers` and `Date` are the dimensions. Each fact joins to `customers` on `customer_id` (many-to-one, single direction), and a generated `Date` table joins to orders and tickets. All 23 measures live in a `Key Measures` holder table.

| Table | Type | Role |
|---|---|---|
| customers | Import | Hub table, one row per customer (5,000). Measures also reference added columns: `Cohort Month`, `Cohort Month Sort`, `Tenure Months`, `Risk Score`, `Risk Tier`. |
| orders | Import | One row per order (12,065). |
| support_tickets | Import | One row per ticket (7,203). |
| Date | Calculated | `CALENDAR(2024-01-01, 2026-12-31)` plus Day, Week Number, Month Year, Year, Weekday, Quarter, Month Number, Year Month Sort. |
| Month Offset | Calculated, disconnected | `GENERATESERIES(0, 24, 1)`. Columns of the cohort matrix. |
| Key Measures | Measure holder | Placeholder table that keeps all measures together. |

| From (many) | To (one) | Cardinality | Filter direction | State |
|---|---|---|---|---|
| orders[customer_id] | customers[customer_id] | Many-to-one | Single | Active |
| support_tickets[customer_id] | customers[customer_id] | Many-to-one | Single | Active |
| orders[order_date] | Date[Date] | Many-to-one | Single | Active |
| support_tickets[ticket_date] | Date[Date] | Many-to-one | Single | Active |
| customers[signup_date] | Date[Date] | Many-to-one | Single | Inactive |
| customers[last_active_date] | Power BI automatic date table | Many-to-one | Single | Active |

**Design choices and what they made easy**

- **One customer dimension.** Channel, region, age group and churn status all sit on `customers` and filter orders and tickets downstream, so "spend by churn status" or "tickets by channel" needs no extra joins.
- **Two customer dates, one date table.** A customer has a signup date and a last-active date. The signup relationship to `Date` is inactive, and the monthly measures (`New Customers`, `Churned Customers (Monthly)`) filter by date range explicitly inside `CALCULATE`. That lets a single month axis show both signups and churn.
- **A disconnected `Month Offset` table (0–24).** It supplies the columns of the cohort heatmap, so one measure (`Retention %`) answers "how many of this cohort remain after N months" for every cell.
- **Cohort and risk logic as added customer columns** (`Cohort Month`, `Tenure Months`, `Risk Score`, `Risk Tier`), so measures such as `At-Risk Customers` can count by tier without repeating the scoring logic.

### DAX measures

23 measures in 6 groups. Full list with code: [`powerbi/dax_measures.md`](powerbi/dax_measures.md). Model notes: [`powerbi/data_model.md`](powerbi/data_model.md).

- **Base counts and value** (8): Total Customers, Active Customers, Churned Customers, Total Orders, Total Spend, Average Order Value, Average Order per Customer, New Customers
- **Churn and revenue at stake** (3): Customer Churn Rate, Revenue Retained, Revenue Churn Rate
- **Monthly trend** (2): Churned Customers (Monthly), Monthly Churn Rate
- **Support and satisfaction** (4): Avg Support Tickets, Avg Rating Given, Churned vs Active Ticket Ratio, Churned vs Active Rating Gap
- **Cohort retention** (3): Cohort Size, Retained Customers, Retention %
- **Risk** (3): At-Risk Customers, Avg Risk Score, At-Risk Share of Active

**Design choices and what they made easy**

- **Small base measures, everything else built on them.** Rates use `DIVIDE` over base counts (`Customer Churn Rate` = churned ÷ total), so tiles, charts and the matrix share one definition.
- **Churned-vs-active comparisons in one measure.** `Churned vs Active Ticket Ratio` overrides `churn_flag` twice inside one measure, giving a single number (2.6×) that still responds to slicers.
- **`ALLEXCEPT` for cohorts.** It keeps each cohort row fixed, so `Retained Customers` ÷ `Cohort Size` gives retention for that cohort at the chosen month offset.

### Visual choices

Each visual answers one question.

| Question | Visual | Measures | Why it makes the analysis easier |
|---|---|---|---|
| How big is churn, and is it changing? | KPI cards + combo chart (churned customers as columns, churn rate as a line) | `Customer Churn Rate`, `Churned Customers (Monthly)`, `Monthly Churn Rate` | Volume and rate sit side by side, so a bigger customer base is not mistaken for a rising churn rate. |
| Which acquisition channel retains best? | Bar chart of churn rate by channel, sorted high to low | `Customer Churn Rate` | One measure, four bars: the ranking is visible at a glance. |
| Do unhappy customers churn? | Two KPI tiles: ticket ratio and rating gap | `Churned vs Active Ticket Ratio`, `Churned vs Active Rating Gap` | The churned-versus-active comparison collapses into one number each, with no side-by-side reading needed. |
| How fast do cohorts decay? | Matrix as a heatmap: cohort month × month offset, colour-scaled | `Cohort Size`, `Retained Customers`, `Retention %` | Decay appears as colour fading left to right, and an unusual cohort stands out as a band. |
| Which segment is worst? | Region → age-group matrix, sorted by churn rate, plus a decomposition tree | `Total Customers`, `Churned Customers`, `Customer Churn Rate` | The matrix gives exact, sortable numbers; the tree lets a viewer follow channel → region → age to the biggest pocket. |
| Who should we look at first? | Scatter of tickets vs risk score (60-point line), at-risk table, drill-through customer profile | `Avg Risk Score`, `At-Risk Customers`, `At-Risk Share of Active` | The chart shows where the threshold sits, the table is the worklist, and drill-through opens one customer’s orders and tickets. |

### Key measure excerpts

```dax
-- Retained Customers
VAR SelectedOffset = SELECTEDVALUE('Month Offset'[Offset])
RETURN CALCULATE(
    DISTINCTCOUNT(customers[customer_id]),
    ALLEXCEPT(customers, customers[Cohort Month], customers[Cohort Month Sort]),
    customers[Tenure Months] >= SelectedOffset
)

-- Retention %
DIVIDE([Retained Customers], [Cohort Size], 0)

-- Monthly Churn Rate
DIVIDE(
    [Churned Customers (Monthly)],
    CALCULATE(DISTINCTCOUNT(customers[customer_id]), customers[signup_date] <= MAX('Date'[Date])),
    0
)
```

## What the analysis found (synthetic data)

- **Acquisition channel is the biggest lever.** Churn is 46.2% for Social, 35.9% for Paid, 25.0% for Organic and 15.7% for Referral.
- **Churn is front-loaded.** Most cohorts lose roughly a quarter to a third of their customers within four months, then the retention curve flattens.
- **Support friction moves with churn.** Churned customers filed 2.6× as many tickets on average (4.1 vs 1.6). Churn is about 2% for customers with 0–1 tickets and about 70% at 3 or more.
- **Risk is concentrated in specific segments.** Asia-Pacific 18–24 churns at 58.0%, Europe 25–34 at 44.8% and Europe 45–54 at 43.6%, while Asia-Pacific 35–44 sits at 15.6%.

## Recommendations to test

These are hypotheses, not proven interventions.

- Shift acquisition effort toward referral and review targeting quality on Social and Paid.
- Design an onboarding and re-engagement programme for months 1–4, where most customers are lost.
- Flag customers at 3+ tickets for proactive outreach, and test whether faster resolution lowers churn.
- Pilot a targeted retention offer for Asia-Pacific 18–24.

## Definitions

- **Churned customer:** `churn_flag = 1` (measure `Churned Customers`).
- **Churn rate:** distinct churned customers ÷ distinct customers = 1,608 ÷ 5,000 = 32.16%.
- **Revenue retained:** `Total Spend` for customers with `churn_flag = 0` (₹32.19M in the source data). **Revenue churn rate:** spend of churned customers ÷ total spend (34.8% on this data).
- **Monthly churn rate:** customers churned in the month (by `last_active_date`) ÷ customers who signed up by the end of that month.
- **Cohort retention:** for a signup-month cohort, customers with `Tenure Months` ≥ N ÷ cohort size. TODO: state how `Tenure Months` is calculated.
- **At-risk customers:** active customers (`churn_flag = 0`) whose `Risk Tier` is "High". TODO: explain in 3–5 lines how `Risk Score` (0–100) and `Risk Tier` are built (inputs, weights, rule-based or fitted, why 60 is the threshold). Interviewers will ask.

## Data

| File | Rows | Description |
|---|---|---|
| `customers.csv` | 5,000 | One row per customer: region, channel, age group, status, spend, ticket count, rating, churn flag |
| `orders.csv` | 12,065 | One row per order: date, value, category, discount, payment method, status |
| `support_tickets.csv` | 7,203 | One row per ticket: date, issue category, resolution status, rating |

**Where the data came from:** TODO: say how the data was produced (script, LLM, public source) and link it.

**Preparation:** `total_spend`, `order_value` (text such as "₹3,513.97") and `discount_applied` ("5.0%") are text in the source and were converted to numbers; `signup_date` and `last_active_date` are day-first text dates converted to dates. TODO: confirm these steps match your Power Query.

### Data observations

- **1,750 customers (35%) have zero orders**, yet have a churn flag and ticket counts.
- **Ticket counts disagree between tables:** `customers.csv` sums to 12,053 tickets but `support_tickets.csv` has 7,203 rows. The ticket measures use `customers[customer_service_tickets_count]`.
- **The churn flag is not a simple inactivity cut-off.** A 90-day "no order" rule reproduces only about 86% of flags.
- **Flags and dates do not fully agree:** 506 customers flagged active have last-active dates before 2026, and `days_since_last_order` does not match `last_active_date`. The at-risk table's top rows (all flagged active) show last-active dates from Dec 2024 to May 2025.
- Every region has exactly 1,250 customers and every region × age cell exactly 250, a sign the data is synthetic.
- **Currency:** `Total Spend` and `Revenue Retained` are formatted with a $ symbol while `Average Order Value` uses ₹. TODO: use ₹ throughout.

## Limitations

- **Synthetic data.** Regions and region-by-age segments are evenly sized and patterns are cleaner than real data, so results demonstrate method, not real business outcomes.
- **Correlation, not causation.** Tickets and churn rise together; this analysis cannot show that tickets cause churn.
- **Monthly churn rate definition.** It divides churn in a month by all customers who signed up by the end of that month, including those who already left, so it reads lower than an "active at start of month" rate would.
- **Data consistency.** The dataset’s flags and dates do not fully agree (for example, 506 customers flagged active have last-active dates before 2026), so the report follows the churn flag.
- Recommendations are hypotheses to test, not proven interventions.

## Repository structure

```
customer-churn-retention-powerbi/
├── README.md
├── data/                  # synthetic CSVs + data_dictionary.md
├── powerbi/               # .pbix file + dax_measures.md
├── screenshots/           # one PNG per report page
└── report/                # PDF export
```

## Tools

Power BI Desktop · DAX · Data modelling · Cohort analysis · Segmentation

## Author

Sandeep Kaur · [Portfolio](TODO_your_portfolio_url) · [LinkedIn](https://www.linkedin.com/in/sandeep-kaur-4b960219a/)
