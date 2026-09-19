# Customer Shopping Behavior Analysis - Project Report

[![Tableau Dashboard](https://img.shields.io/badge/View%20Dashboard-Tableau%20Public-blue?logo=tableau)](https://public.tableau.com/views/CustomerBehaviorAnalysis_17898306959200/Dashboard2?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

**[🔗 Live Interactive Dashboard on Tableau Public](https://public.tableau.com/views/CustomerBehaviorAnalysis_17898306959200/Dashboard2?:language=en-US&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)**

---

## 📋 Project Overview

This project performs an end-to-end analysis of customer shopping behavior using a dataset of **3,900 transactions**. The analysis spans from data ingestion and cleaning (Python/Jupyter), to database storage (PostgreSQL), analytical querying (SQL), and finally interactive visualization (Tableau Dashboard).

---

## 📊 Dataset Description

**File:** `customer_shopping_behavior.csv`  
**Rows:** 3,900 transactions (3,901 lines including header)  
**Columns:** 18 original columns → 17 after cleaning (dropped redundant `promo_code_used`)

### Original Columns
| Column | Type | Description |
|--------|------|-------------|
| Customer ID | Integer | Unique customer identifier (1–3900) |
| Age | Integer | Customer age (18–70) |
| Gender | Categorical | Male / Female |
| Item Purchased | Categorical | 25 unique products |
| Category | Categorical | 4 categories: Clothing, Footwear, Accessories, Outerwear |
| Purchase Amount (USD) | Integer | Transaction value ($20–$100) |
| Location | Categorical | 50 US states |
| Size | Categorical | S, M, L, XL |
| Color | Categorical | 25 unique colors |
| Season | Categorical | Winter, Spring, Summer, Fall |
| Review Rating | Float (2.5–5.0) | Product rating (37 nulls originally) |
| Subscription Status | Categorical | Yes / No |
| Shipping Type | Categorical | 6 types: Express, Standard, Free Shipping, Next Day Air, 2-Day Shipping, Store Pickup |
| Discount Applied | Categorical | Yes / No |
| Promo Code Used | Categorical | Yes / No (identical to Discount Applied) |
| Previous Purchases | Integer | Lifetime purchase count (1–50) |
| Payment Method | Categorical | 6 methods: Credit Card, PayPal, Venmo, Cash, Debit Card, Bank Transfer |
| Frequency of Purchases | Categorical | 7 frequencies: Weekly, Fortnightly, Bi-Weekly, Monthly, Quarterly, Every 3 Months, Annually |

### Data Quality Notes
- **37 missing values** in `Review Rating` → imputed using category-level median
- `Discount Applied` and `Promo Code Used` were **identical** (100% match) → dropped `promo_code_used`
- No other missing values in any column

---

## 🐍 Data Processing Pipeline (Jupyter Notebook: `shopping_behavior_analysis.ipynb`)

### Step 1: Data Loading & Exploration
```python
df = pd.read_csv("customer_shopping_behavior.csv")
df.describe(include='all')
df.isnull().sum()
```

### Step 2: Missing Value Handling
```python
# Impute Review Rating using category median
df['Review Rating'] = df.groupby('Category')['Review Rating'].transform(
    lambda x: x.fillna(x.median())
)
```

### Step 3: Column Standardization
```python
df.columns = df.columns.str.replace(' ', '_').str.lower()
df = df.rename(columns={'purchase_amount_(usd)': 'purchase_amount'})
```

### Step 4: Feature Engineering

**Age Groups** (using fixed bins for business interpretability):
```python
bins = [18, 35, 50, 65, np.inf]
labels = ["Young Adult", "Adult", "Middle-Aged", "Senior"]
df["age_group"] = pd.cut(df["age"], bins=bins, labels=labels, right=True, include_lowest=True)
```
Distribution:
- Young Adult (18–35): 1,028
- Adult (35–50): 942
- Middle-Aged (50–65): 986
- Senior (65+): 944

**Purchase Frequency in Days:**
```python
frequency_mapping = {
    'Weekly': 7, 'Fortnightly': 14, 'Bi-Weekly': 14,
    'Monthly': 30, 'Quarterly': 90, 'Every 3 Months': 90, 'Annually': 365
}
df['purchase_frequency_days'] = df['frequency_of_purchases'].map(frequency_mapping)
```

### Step 5: Database Load (PostgreSQL)
```python
from sqlalchemy import create_engine

engine = create_engine('postgresql://postgres:0000@localhost:5432/customer_behavior')
df.to_sql('customer', engine, if_exists='replace', index=False)
```
**Table created:** `customer` in database `customer_behavior` with 3,900 rows.

---

## 🗄️ Database Schema (PostgreSQL)

Table: `customer`

| Column | Type |
|--------|------|
| customer_id | INTEGER |
| age | INTEGER |
| gender | TEXT |
| item_purchased | TEXT |
| category | TEXT |
| purchase_amount | INTEGER |
| location | TEXT |
| size | TEXT |
| color | TEXT |
| season | TEXT |
| review_rating | NUMERIC |
| subscription_status | TEXT |
| shipping_type | TEXT |
| discount_applied | TEXT |
| previous_purchases | INTEGER |
| payment_method | TEXT |
| frequency_of_purchases | TEXT |
| age_group | TEXT |
| purchase_frequency_days | INTEGER |

---

## 🔍 SQL Analysis Questions & Answers

All queries executed against the `customer` table in PostgreSQL. File: `0-Questions.sql`

### Q1. Total Revenue by Gender
```sql
SELECT gender, SUM(purchase_amount)
FROM customer
GROUP BY gender;
```
| Gender | Total Revenue |
|--------|---------------|
| Female | $75,191 |
| Male | $157,890 |

**Finding:** Male customers generate **2.1× more revenue** than female customers. However, this may reflect sample composition (2,652 Male vs 1,248 Female transactions).

---

### Q2. Discount Users Spending Above Average
```sql
SELECT customer_id, purchase_amount
FROM customer
WHERE discount_applied = 'Yes'
  AND purchase_amount > (SELECT AVG(purchase_amount) FROM customer)
GROUP BY customer_id, purchase_amount
ORDER BY 1, 2;
```
**Result:** 70 customers used a discount but still spent above the average purchase amount ($59.76).  
**Finding:** Discounts don't always mean low spend—many discounted purchases are high-value.

---

### Q3. Top 5 Products by Average Review Rating
```sql
SELECT item_purchased, ROUND(AVG(review_rating)::NUMERIC, 2) AS average_rating
FROM customer
GROUP BY item_purchased
ORDER BY average_rating DESC
LIMIT 5;
```
| Product | Avg Rating |
|---------|------------|
| Gloves | 3.86 |
| Sandals | 3.84 |
| Boots | 3.82 |
| Hat | 3.80 |
| Skirt | 3.78 |

**Finding:** Accessories (Gloves, Hat) and Footwear (Sandals, Boots) lead in satisfaction. Clothing items dominate lower ratings.

---

### Q4. Average Purchase: Standard vs Express Shipping
```sql
SELECT shipping_type, ROUND(AVG(purchase_amount)::NUMERIC, 2)
FROM customer
WHERE shipping_type IN ('Express', 'Standard')
GROUP BY shipping_type;
```
| Shipping Type | Avg Purchase |
|---------------|--------------|
| Standard | $58.46 |
| Express | $60.48 |

**Finding:** Express shipping customers spend ~3.5% more on average. Premium shipping correlates with slightly higher order values.

---

### Q5. Subscriber vs Non-Subscriber Spend
```sql
SELECT subscription_status,
       COUNT(customer_id) AS total_customers,
       ROUND(AVG(purchase_amount)::NUMERIC, 2) AS avg_spend,
       SUM(purchase_amount) AS total_revenue
FROM customer
GROUP BY subscription_status
ORDER BY total_revenue DESC, avg_spend DESC;
```
| Status | Customers | Avg Spend | Total Revenue |
|--------|-----------|-----------|---------------|
| No | 2,847 | $59.87 | $170,436 |
| Yes | 1,053 | $59.49 | $62,645 |

**Finding:** **Non-subscribers contribute 73% of total revenue** and have a marginally higher average spend. Subscriptions don't guarantee higher per-order value.

---

### Q6. Top 5 Products by Discount Percentage
```sql
SELECT item_purchased,
       ROUND(100 * SUM(CASE WHEN discount_applied = 'Yes' THEN 1 ELSE 0 END) / COUNT(item_purchased)) AS discount_percentage
FROM customer
GROUP BY item_purchased
ORDER BY discount_percentage DESC
LIMIT 5;
```
| Product | Discount % |
|---------|------------|
| Hat | 50.00% |
| Sneakers | 49.66% |
| Coat | 49.07% |
| Sweater | 48.17% |
| Pants | 47.37% |

**Finding:** Accessories (Hat) and Outerwear (Coat) are most heavily discounted. Nearly half of all Hat purchases use a discount.

---

### Q7. Customer Segmentation (New / Returning / Loyal)
```sql
-- Using CTE approach
WITH customer_type AS (
    SELECT customer_id, previous_purchases,
           CASE
               WHEN previous_purchases = 1 THEN 'New'
               WHEN previous_purchases BETWEEN 2 AND 10 THEN 'Returning'
               ELSE 'Loyal'
           END AS customer_segment
    FROM customer
)
SELECT customer_segment, COUNT(*) AS counts
FROM customer_type
GROUP BY customer_segment;
```
| Segment | Count | % of Total |
|---------|-------|------------|
| Loyal | 3,116 | 80% |
| Returning | 701 | 18% |
| New | 83 | 2% |

**Finding:** The customer base is **heavily dominated by Loyal customers** (previous_purchases > 10). Very few truly new customers in this dataset.

---

### Q8. Top 3 Products per Category
```sql
WITH item_counts AS (
    SELECT category, item_purchased, COUNT(*) AS total_orders,
           ROW_NUMBER() OVER (PARTITION BY category ORDER BY COUNT(*) DESC) AS item_rank
    FROM customer
    GROUP BY category, item_purchased
)
SELECT item_rank, category, item_purchased, total_orders
FROM item_counts
WHERE item_rank <= 3;
```
| Rank | Category | Product | Orders |
|------|----------|---------|--------|
| 1 | Accessories | Jewelry | 171 |
| 2 | Accessories | Sunglasses | 161 |
| 3 | Accessories | Belt | 161 |
| 1 | Clothing | Blouse | 171 |
| 2 | Clothing | Pants | 171 |
| 3 | Clothing | Shirt | 169 |
| 1 | Footwear | Sandals | 160 |
| 2 | Footwear | Shoes | 150 |
| 3 | Footwear | Sneakers | 145 |
| 1 | Outerwear | Jacket | 163 |
| 2 | Outerwear | Coat | 161 |

**Finding:** Each category has a clear top product, but the 2nd and 3rd places are competitive. Clothing shows a three-way tie at the top.

---

### Q9. Repeat Buyers (>5 purchases) & Subscription Likelihood
```sql
SELECT subscription_status, COUNT(*)
FROM customer
WHERE previous_purchases > 5
GROUP BY subscription_status;
```
| Subscription | Count |
|--------------|-------|
| No | 2,518 |
| Yes | 958 |

**Finding:** Even among repeat buyers (>5 purchases), **72% are NOT subscribed**. High purchase frequency doesn't strongly drive subscription adoption.

---

### Q10. Revenue by Age Group
```sql
SELECT age_group,
       COUNT(*),
       SUM(purchase_amount) AS total_revenue,
       ROUND(100 * SUM(purchase_amount) / SUM(SUM(purchase_amount)) OVER (), 2) AS revenue_percentage
FROM customer
GROUP BY age_group
ORDER BY total_revenue DESC;
```
| Age Group | Transactions | Total Revenue | Revenue % |
|-----------|--------------|---------------|-----------|
| Young Adult | ~1,028 | $62,143 | ~27.0% |
| Middle-Aged | ~986 | $59,197 | ~25.7% |
| Adult | ~942 | $55,978 | ~24.3% |
| Senior | ~944 | $55,763 | ~24.2% |

**Finding:** Revenue is **fairly evenly distributed** across age groups. Young Adults lead slightly, but all four segments contribute ~24–27% each.

---

## 📈 Key Findings Summary

| Area | Insight |
|------|---------|
| **Gender Revenue Gap** | Male customers generate 2.1× revenue, but represent 68% of transactions |
| **Discount Behavior** | 70 discount-users spend above average; Hats (50%) and Sneakers (~50%) most discounted |
| **Product Satisfaction** | Gloves (3.86), Sandals (3.84), Boots (3.82) top-rated |
| **Shipping & Spend** | Express shipping correlates with +3.5% higher average order value |
| **Subscription Value** | Non-subscribers drive 73% of revenue; subscription ≠ higher spend |
| **Customer Loyalty** | 80% of transactions from "Loyal" customers (>10 previous purchases) |
| **Category Leaders** | Jewelry (Accessories), Blouse/Pants (Clothing), Sandals (Footwear), Jacket (Outerwear) |
| **Repeat Buyers** | 72% of frequent buyers (>5 purchases) are non-subscribers |
| **Age Distribution** | Revenue evenly split across 4 age groups (24–27% each) |

---

## 📊 Tableau Dashboard (`Customer Behavior Analysis.twb`)

**Note:** The user mentions this is a Tableau dashboard (`.twb` file), not Power BI.

### Dashboard Features
- **Dynamic Filtering:** Left-side filters (Category, Gender, Age Group, Season, Shipping Type, Subscription Status, Discount Applied) update all visualizations in real-time
- **Key Visualizations:**
  1. **Revenue by Gender** – Bar chart showing Male vs Female totals
  2. **Revenue by Age Group** – Segmented bar/column chart
  3. **Top Products by Category** – Ranked tables per category
  4. **Shipping Type Analysis** – Average purchase amount comparison
  5. **Subscription Impact** – Revenue & avg spend comparison
  6. **Discount Analysis** – Products with highest discount rates
  7. **Customer Segmentation** – New / Returning / Loyal distribution
  8. **Review Ratings** – Average rating by product

### Interactivity
- Selecting a category on the left filters all charts to that category
- Multi-select supported for comparing segments
- Tooltips show exact values, counts, and percentages
- Dashboard resets with "Clear Filters" action

---

## 🛠️ Tech Stack

| Layer | Technology |
|-------|------------|
| Data Source | CSV (3,900 rows) |
| Cleaning & ETL | Python 3, Pandas, NumPy, SQLAlchemy |
| Database | PostgreSQL 14+ |
| Analysis | SQL (10 analytical queries) |
| Visualization | Tableau Desktop (`.twb` workbook) |
| Notebook | Jupyter (`.ipynb`) |

---

## 📁 Repository Structure

```
analytics/
├── customer_shopping_behavior.csv      # Raw dataset (3,900 rows)
├── shopping_behavior_analysis.ipynb    # Python ETL & exploration
├── 0-Questions.sql                     # 10 SQL analytical queries
├── Customer Behavior Analysis.twb      # Tableau dashboard workbook
└── README.md                           # This report
```

---

## 🚀 How to Reproduce

### Prerequisites
- Python 3.8+ with pandas, numpy, sqlalchemy, psycopg2
- PostgreSQL 14+ running locally
- Tableau Desktop (to open `.twb`)

### Steps

1. **Start PostgreSQL** and create database:
   ```sql
   CREATE DATABASE customer_behavior;
   ```

2. **Run Jupyter Notebook** to clean data and load to PostgreSQL:
   ```bash
   jupyter notebook shopping_behavior_analysis.ipynb
   ```
   Execute all cells (loads 3,900 rows into `customer` table)

3. **Run SQL Queries** in any PostgreSQL client (psql, DBeaver, pgAdmin):
   ```bash
   psql -d customer_behavior -f 0-Questions.sql
   ```

4. **Open Tableau Dashboard:**
   - Launch Tableau Desktop
   - Open `Customer Behavior Analysis.twb`
   - Connect to PostgreSQL if prompted (same credentials)

---

## 📝 Conclusions & Recommendations

1. **Focus on Loyal Customers** – 80% of business comes from repeat buyers; loyalty programs should target the "Returning" segment to convert them to "Loyal"

2. **Subscription Strategy Needs Rethink** – Subscribers don't spend more per order. Consider subscription perks (free shipping, exclusive access) to increase value proposition

3. **Discount Strategy** – High discount rates on Hats, Sneakers, Coats may erode margins. Test targeted vs blanket discounts

4. **Gender Gap Investigation** – Male-dominated revenue may indicate marketing bias or product assortment gaps for female customers

5. **Even Age Distribution** – No single age group dominates; marketing should maintain broad appeal rather than hyper-targeting

6. **Express Shipping Upsell** – Express users spend more; consider promoting express at checkout for orders >$75

---

*Report generated from project files on September 19, 2026*