# Valuelabs-Data-analyst-workflow
**end-to-end Pricing & Collections Analytics workflow** you can understand and explain in an interview.

## 1. Overall architecture

```text
MySQL Finance Database
        │
        ├── Customers
        ├── Invoices
        ├── Payments
        ├── Collections
        └── Pricing
                │
                ▼
        Python Data Layer
        ┌───────────────────────┐
        │ DataLoader            │
        │ DataValidator         │
        │ DataPreprocessor      │
        │ FinanceAnalyzer       │
        │ ReportGenerator       │
        └───────────────────────┘
                │
                ▼
        Clean Analytical Dataset
                │
        ┌───────┴────────┐
        ▼                ▼
     Python/EDA       Excel Reports
        │
        ▼
   Business KPIs
        │
        ▼
    Power BI Dashboard
        │
        ▼
Business / Collections Team
        │
        ▼
Collections Prioritization
& Operational Decisions
```

---

# 2. Business problem

The business has thousands of invoices and customers.

The collections team needs answers to questions such as:

```text
Which customers have overdue payments?

How much money is currently outstanding?

Which invoices are severely overdue?

Which customers regularly delay payments?

Which business units have poor collection performance?

Are pricing changes affecting revenue?

How much has been collected this month?

Which accounts should the collections team prioritize?
```

Your role is to convert raw finance data into these actionable insights.

---

# 3. Source data

Assume the company has these MySQL tables.

### Customers

```text
customer_id
customer_name
country
industry
customer_segment
```

### Invoices

```text
invoice_id
customer_id
invoice_date
due_date
invoice_amount
currency
service_type
location
```

### Payments

```text
payment_id
invoice_id
payment_date
payment_amount
payment_method
```

### Collections

```text
collection_id
customer_id
collection_date
amount_collected
collector
collection_status
```

### Pricing

```text
pricing_id
customer_id
service_type
location
currency
price
effective_date
```

---

# 4. Step 1 — Extract data from MySQL

You first connect Python to MySQL.

```python
import pandas as pd
import mysql.connector


class MySQLLoader:

    def __init__(self, host, user, password, database):
        self.connection = mysql.connector.connect(
            host=host,
            user=user,
            password=password,
            database=database
        )

    def load_data(self, query):
        return pd.read_sql(query, self.connection)
```

Then load the required tables.

```python
loader = MySQLLoader(
    host="localhost",
    user="root",
    password="password",
    database="finance_db"
)

invoices = loader.load_data("""
    SELECT *
    FROM invoices
""")

payments = loader.load_data("""
    SELECT *
    FROM payments
""")

customers = loader.load_data("""
    SELECT *
    FROM customers
""")
```

### Why OOP here?

Instead of writing database logic repeatedly, you create reusable classes.

```text
MySQLLoader
      ↓
load_data()
      ↓
Any finance table can be loaded
```

This demonstrates the **OOP requirement from your resume**.

---

# 5. Step 2 — Data validation

Before analysis, validate the datasets.

```python
class DataValidator:

    def check_missing_values(self, df):
        return df.isnull().sum()

    def check_duplicates(self, df):
        return df.duplicated().sum()

    def check_negative_values(self, df, columns):
        return {
            col: (df[col] < 0).sum()
            for col in columns
        }

    def validate(self, df, numeric_columns):
        return {
            "missing_values": self.check_missing_values(df),
            "duplicates": self.check_duplicates(df),
            "negative_values": self.check_negative_values(
                df, numeric_columns
            )
        }
```

Example:

```python
validator = DataValidator()

validation_report = validator.validate(
    invoices,
    ["invoice_amount"]
)

print(validation_report)
```

You check:

```text
Missing values
Duplicate invoices
Invalid dates
Negative invoice amounts
Invalid customer IDs
Invalid payment amounts
```

---

# 6. Step 3 — Data preprocessing

Finance data normally needs cleaning.

```python
class DataPreprocessor:

    def clean_invoices(self, df):

        df = df.copy()

        df["invoice_date"] = pd.to_datetime(
            df["invoice_date"]
        )

        df["due_date"] = pd.to_datetime(
            df["due_date"]
        )

        df["invoice_amount"] = (
            pd.to_numeric(
                df["invoice_amount"],
                errors="coerce"
            )
        )

        df["invoice_amount"] = (
            df["invoice_amount"]
            .fillna(0)
        )

        df = df.drop_duplicates(
            subset=["invoice_id"]
        )

        return df

    def clean_payments(self, df):

        df = df.copy()

        df["payment_date"] = pd.to_datetime(
            df["payment_date"]
        )

        df["payment_amount"] = (
            pd.to_numeric(
                df["payment_amount"],
                errors="coerce"
            )
        )

        df["payment_amount"] = (
            df["payment_amount"]
            .fillna(0)
        )

        return df
```

Usage:

```python
processor = DataPreprocessor()

invoices = processor.clean_invoices(invoices)
payments = processor.clean_payments(payments)
```

---

# 7. Step 4 — Combine invoice and payment data

Now we need to determine how much has been paid against each invoice.

```python
payment_summary = (
    payments
    .groupby("invoice_id")
    .agg(
        total_paid=("payment_amount", "sum"),
        last_payment_date=("payment_date", "max")
    )
    .reset_index()
)
```

Join it with invoices.

```python
df = invoices.merge(
    payment_summary,
    on="invoice_id",
    how="left"
)
```

Missing payments mean that no payment has been received.

```python
df["total_paid"] = (
    df["total_paid"]
    .fillna(0)
)
```

---

# 8. Step 5 — Create business features

This is the most important analytical part.

### Outstanding amount

```python
df["outstanding_amount"] = (
    df["invoice_amount"] -
    df["total_paid"]
)
```

### Overdue days

```python
today = pd.Timestamp.today()

df["overdue_days"] = (
    today - df["due_date"]
).dt.days

df["overdue_days"] = (
    df["overdue_days"].clip(lower=0)
)
```

### Aging bucket

```python
def aging_bucket(days):

    if days <= 0:
        return "Current"

    elif days <= 30:
        return "1-30 Days"

    elif days <= 60:
        return "31-60 Days"

    elif days <= 90:
        return "61-90 Days"

    else:
        return "90+ Days"


df["aging_bucket"] = (
    df["overdue_days"]
    .apply(aging_bucket)
)
```

Now each invoice contains useful business information.

```text
invoice_id
customer_id
invoice_amount
total_paid
outstanding_amount
overdue_days
aging_bucket
```

---

# 9. Step 6 — Customer-level aggregation

The business usually wants customer-level insights rather than only invoice-level data.

```python
customer_summary = (
    df.groupby("customer_id")
      .agg(
          total_invoiced=("invoice_amount", "sum"),
          total_paid=("total_paid", "sum"),
          total_outstanding=("outstanding_amount", "sum"),
          avg_overdue_days=("overdue_days", "mean"),
          invoice_count=("invoice_id", "count")
      )
      .reset_index()
)
```

Calculate collection rate.

```python
customer_summary["collection_rate"] = (
    customer_summary["total_paid"] /
    customer_summary["total_invoiced"]
)
```

---

# 10. Step 7 — Identify high-risk customers

Now you turn analysis into business action.

For example:

```python
customer_summary["high_risk"] = (
    (
        customer_summary["avg_overdue_days"] > 60
    )
    &
    (
        customer_summary["total_outstanding"] > 50000
    )
)
```

Now you can identify:

```text
Customer A → ₹1,20,000 outstanding → 92 days overdue → HIGH RISK

Customer B → ₹15,000 outstanding → 10 days overdue → LOW RISK

Customer C → ₹85,000 outstanding → 74 days overdue → HIGH RISK
```

This is where your analysis directly supports **collections prioritization**.

---

# 11. Step 8 — EDA

Now analyze the dataset.

### Outstanding by aging bucket

```python
aging_analysis = (
    df.groupby("aging_bucket")
      ["outstanding_amount"]
      .sum()
      .reset_index()
)
```

### Collection performance by customer

```python
collection_analysis = (
    customer_summary
    .sort_values(
        "total_outstanding",
        ascending=False
    )
)
```

### Monthly collections

```python
payments["month"] = (
    payments["payment_date"]
    .dt.to_period("M")
)

monthly_collections = (
    payments.groupby("month")
            ["payment_amount"]
            .sum()
            .reset_index()
)
```

You could identify trends such as:

```text
January   ₹45L collected
February  ₹52L collected
March     ₹48L collected
April     ₹61L collected
```

This supports **trend analysis and operational monitoring**.

---

# 12. Step 9 — Pricing analysis

Now bring the pricing dataset into the workflow.

For example:

```python
pricing_analysis = (
    pricing
    .groupby(
        ["service_type", "location"]
    )
    .agg(
        avg_price=("price", "mean"),
        min_price=("price", "min"),
        max_price=("price", "max")
    )
    .reset_index()
)
```

You can analyze:

```text
Service Type
      ↓
Location
      ↓
Currency
      ↓
Average Price
      ↓
Price Trend
      ↓
Revenue Impact
```

For example:

```text
India → Service A → ₹1,200 average
US    → Service A → $40 average
UK    → Service A → £32 average
```

---

# 13. Step 10 — Power BI

Your cleaned analytical dataset is then loaded into Power BI.

A realistic dashboard could contain:

```text
┌─────────────────────────────────────────────┐
│          COLLECTIONS OVERVIEW                │
├────────────┬─────────────┬──────────────────┤
│ Outstanding│ Collection  │ Overdue          │
│ ₹12.4 Cr   │ Rate 82.3%  │ ₹4.1 Cr          │
├────────────┴─────────────┴──────────────────┤
│                                             │
│ Collection Trend                            │
│ ███████████████████████                     │
│                                             │
├─────────────────────────────────────────────┤
│ Invoice Aging                               │
│                                             │
│ Current     █████████                       │
│ 1-30 Days   ███████                         │
│ 31-60 Days  █████                           │
│ 61-90 Days  ███                             │
│ 90+ Days    ██████                          │
│                                             │
├─────────────────────────────────────────────┤
│ Top High-Risk Customers                     │
│                                             │
│ Customer | Outstanding | Overdue | Risk     │
│ A        | ₹1.2L       | 92     | HIGH      │
│ B        | ₹85K        | 74     | HIGH      │
│ C        | ₹70K        | 68     | HIGH      │
└─────────────────────────────────────────────┘
```

---

# 14. Step 11 — Excel reporting

You can generate an Excel report from Python.

```python
with pd.ExcelWriter(
    "collections_report.xlsx",
    engine="openpyxl"
) as writer:

    customer_summary.to_excel(
        writer,
        sheet_name="Customer Summary",
        index=False
    )

    aging_analysis.to_excel(
        writer,
        sheet_name="Aging Analysis",
        index=False
    )

    monthly_collections.to_excel(
        writer,
        sheet_name="Monthly Collections",
        index=False
    )

    pricing_analysis.to_excel(
        writer,
        sheet_name="Pricing Analysis",
        index=False
    )
```

So your reporting workflow becomes:

```text
MySQL
 ↓
Python
 ↓
Clean Data
 ↓
Business Metrics
 ↓
Excel + Power BI
```

---

# 15. Step 12 — Reusable OOP architecture

To make the project more production-like, structure your Python code like this:

```text
finance_analytics/
│
├── config/
│   └── config.py
│
├── data/
│   └── sample_data/
│
├── src/
│   ├── database/
│   │   └── mysql_loader.py
│   │
│   ├── preprocessing/
│   │   └── data_preprocessor.py
│   │
│   ├── validation/
│   │   └── data_validator.py
│   │
│   ├── analysis/
│   │   ├── collections_analyzer.py
│   │   └── pricing_analyzer.py
│   │
│   └── reporting/
│       └── report_generator.py
│
├── notebooks/
│   └── eda.ipynb
│
├── reports/
│
├── main.py
└── requirements.txt
```

The classes connect like this:

```text
MySQLLoader
     ↓
DataValidator
     ↓
DataPreprocessor
     ↓
CollectionsAnalyzer
     ↓
PricingAnalyzer
     ↓
ReportGenerator
     ↓
Excel / Power BI
```

---

# 16. Complete execution flow

Your actual project flow can be represented as:

```text
                         BUSINESS REQUIREMENT
                                │
                                ▼
                "Identify collection risks
                 and monitor performance"
                                │
                                ▼
                         MySQL DATABASE
                                │
                ┌───────────────┼──────────────┐
                ▼               ▼              ▼
             Invoice          Payment       Pricing
              Data             Data           Data
                │               │              │
                └───────────────┼──────────────┘
                                ▼
                         PYTHON / PANDAS
                                │
                                ▼
                      DATA VALIDATION
                                │
                                ▼
                       DATA CLEANING
                                │
                                ▼
                         DATA JOINING
                                │
                                ▼
                       FEATURE ENGINEERING
                                │
             ┌──────────────────┼──────────────────┐
             ▼                  ▼                  ▼
        Overdue Days       Aging Bucket       Collection Rate
             │                  │                  │
             └──────────────────┼──────────────────┘
                                ▼
                             EDA
                                │
             ┌──────────────────┼───────────────────┐
             ▼                  ▼                   ▼
       Customer Trends     Aging Analysis      Pricing Trends
             │                  │                   │
             └──────────────────┼───────────────────┘
                                ▼
                        BUSINESS SEGMENTATION
                                │
                                ▼
                     HIGH-RISK CUSTOMERS
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
               POWER BI                   EXCEL
               DASHBOARD                 REPORT
                    │                       │
                    └───────────┬───────────┘
                                ▼
                      BUSINESS / COLLECTIONS
                              TEAM
                                │
                                ▼
                    COLLECTION PRIORITIZATION
                                │
                                ▼
                         BUSINESS ACTION
```

# 17. What you should say in the interview

A strong explanation is:

> “I worked on an end-to-end Pricing and Collections analytics workflow. The data came primarily from MySQL tables containing customers, invoices, payments, collections, and pricing information. I used reusable Python OOP components for data loading, validation, preprocessing, analysis, and report generation.
>
> After extracting the data, I performed data quality checks, handled missing and duplicate records, standardized dates and numeric fields, and joined invoice and payment datasets. I engineered business features such as outstanding amount, overdue days, aging buckets, collection rate, and customer-level payment behavior.
>
> I then performed EDA and trend analysis to identify overdue patterns, collection performance, customer-level risks, and pricing trends. Based on outstanding amounts and overdue behavior, I helped identify high-priority customers for collections follow-up.
>
> Finally, I prepared Excel reports and Power BI dashboards containing KPIs such as total outstanding, collection rate, aging distribution, monthly collections, and high-risk customers. I worked with business and operations teams to validate the requirements and ensure the analysis supported actual operational decisions.”

That gives you a **real end-to-end story rather than saying only “I did data analysis.”**

 **ValueLabs finance/IT-services context**  your pricing work can be explained as **pricing analytics**, not as “I decided the price for customers.” Your role was to analyze pricing data and identify trends, anomalies, and business impacts.

### What pricing work could realistically look like

Imagine the company provides IT services to clients across different countries.

The pricing data could contain:

```text
Customer
Service
Employee/Resource Type
Location
Currency
Billing Rate
Hours
Invoice Amount
Contract
Effective Date
```

Your workflow would be:

```text
Raw Pricing Data
      ↓
MySQL Extraction
      ↓
Python / Pandas Cleaning
      ↓
Pricing Analysis
      ↓
Rate & Trend Analysis
      ↓
Customer / Location Comparison
      ↓
Power BI Dashboard
      ↓
Business Decision
```

### 1. Analyze billing rates

You could compare the average billing rate by service, location, or customer.

For example:

```python
pricing_summary = (
    pricing_df
    .groupby(["location", "service_type"])
    .agg(
        avg_rate=("billing_rate", "mean"),
        min_rate=("billing_rate", "min"),
        max_rate=("billing_rate", "max")
    )
    .reset_index()
)
```

This answers:

> “What is the average billing rate for each service across different locations?”

For example:

```text
Location     Service       Avg Rate
India        Development   $25/hr
India        Testing       $20/hr
US           Development   $75/hr
US           Testing       $60/hr
UK           Development   $65/hr
```

---

### 2. Pricing trend analysis

You can analyze how rates changed over time.

```python
pricing_df["effective_date"] = pd.to_datetime(
    pricing_df["effective_date"]
)

monthly_rate = (
    pricing_df
    .groupby(
        pricing_df["effective_date"].dt.to_period("M")
    )["billing_rate"]
    .mean()
)
```

Then you could identify:

```text
Month       Average Rate
Jan         $42
Feb         $44
Mar         $43
Apr         $47
May         $49
```

So your business insight would be:

> “I analyzed pricing trends over time to identify increases or decreases in average billing rates across services and locations.”

---

### 3. Customer-level pricing analysis

You can compare the rates being charged to different customers for similar services.

```python
customer_pricing = (
    pricing_df
    .groupby(["customer_id", "service_type"])
    .agg(
        avg_rate=("billing_rate", "mean")
    )
    .reset_index()
)
```

Then compare customers:

```text
Customer A → Development → $70/hr
Customer B → Development → $62/hr
Customer C → Development → $48/hr
```

This could trigger a business question:

> “Why are similar services being billed at significantly different rates?”

That is a useful pricing-analysis insight.

---

### 4. Identify pricing anomalies

This is another strong example to mention.

Suppose the average rate for a particular service is ₹2,000/hour, but one customer is being billed ₹1,200.

You can flag it:

```python
avg_rate = pricing_df["billing_rate"].mean()

pricing_df["rate_difference_pct"] = (
    (pricing_df["billing_rate"] - avg_rate)
    / avg_rate
) * 100
```

Then:

```python
pricing_df["pricing_flag"] = np.where(
    pricing_df["rate_difference_pct"] < -20,
    "Below Expected Rate",
    "Normal"
)
```

Now the business team can investigate unusually low or high rates.

---

### 5. Currency/location analysis

Because your pricing context involved **client location/currency**, this is especially relevant.

For example:

```text
Customer    Location    Currency    Rate
A           US          USD         75
B           India       INR         2200
C           UK          GBP         60
```

You would standardize currencies when a common comparison was required and make sure the analysis didn't incorrectly compare USD, INR, and GBP as if they were the same unit.

The important interview point is:

> “I worked with pricing data across different client locations and currencies, so part of the preprocessing involved validating currency and location information before performing comparisons.”

---

### 6. Pricing vs revenue analysis

Pricing becomes more useful when connected to actual billing.

For example:

```python
pricing_df["revenue"] = (
    pricing_df["billing_rate"] *
    pricing_df["billable_hours"]
)
```

Now you can analyze:

```text
Billing Rate
     ×
Billable Hours
     ↓
Revenue
```

For example:

```text
Customer A
Rate = $70/hour
Hours = 1,000
Revenue = $70,000
```

You can then determine whether revenue changes were driven by:

* higher billing rates
* higher billable hours
* changes in customer/service mix
* location changes

---

## 7. Pricing dashboard

A Power BI dashboard could have:

```text
┌──────────────────────────────────────────┐
│          PRICING ANALYTICS               │
├────────────┬────────────┬────────────────┤
│ Avg Rate   │ Revenue    │ Rate Change    │
│ $52/hr     │ $4.8M      │ +8.4%          │
├────────────┴────────────┴────────────────┤
│                                          │
│ Average Rate by Location                 │
│ US       █████████████                   │
│ UK       ███████████                     │
│ India    █████                           │
│                                          │
├──────────────────────────────────────────┤
│ Pricing Trend                            │
│                                          │
│ Jan ─ Feb ─ Mar ─ Apr ─ May ─ Jun       │
│                                          │
├──────────────────────────────────────────┤
│ Customer Pricing                         │
│ Customer | Service | Rate | Variance     │
│ A        | Dev     | $70  | +12%         │
│ B        | Dev     | $62  | -1%          │
│ C        | Dev     | $48  | -23% ⚠       │
└──────────────────────────────────────────┘
```

---

# How I'd recommend you explain it in your interview

Don't say:

> “I was responsible for setting customer prices.”

Unless you actually did that.

Instead say:

> **“I worked on pricing analytics rather than directly setting prices. We had pricing and billing data across different customers, services, locations and currencies. I used MySQL to extract the data and Python/Pandas to clean and validate it. I analyzed billing rates by customer, service and location, looked at pricing trends over time, and identified unusual rate variations. I also connected pricing information with billable hours and revenue to understand the business impact. Finally, I created Power BI dashboards and Excel reports so the business teams could monitor pricing performance and investigate significant rate variations.”**

### If they ask: “Give me one concrete example.”

Say:

> **“For example, I analyzed billing rates for the same service across different customers and locations. After standardizing the data, I calculated average rates and the variance from the benchmark rate. If a customer's rate was significantly lower or higher than the expected range, I flagged it for further business review. I presented these variations through Power BI along with revenue and volume trends. This helped the business team investigate pricing inconsistencies and make better pricing-related decisions.”**

That is a **credible Data Analyst-level pricing project** and fits your stated Python + MySQL + Power BI + Excel experience without overstating your responsibilities.

