# Data Cleaning and Initial Checks

Before performing any analysis, I validated and cleaned the CRM sales data to ensure accuracy and reliability. The steps I followed were:

---

## Step 1: Check for Duplicate Opportunities

**Goal:**  
Ensure each sales opportunity has a unique `opportunity_id`. Duplicates can inflate revenue and distort KPIs.

**SQL Query:**
```sql
SELECT opportunity_id, COUNT(*) AS count_duplicates
FROM sales_pipeline
GROUP BY opportunity_id
HAVING COUNT(*) > 1; 
 ```


**Result:**  
No duplicate `opportunity_id` values were found.This means every opportunity is unique.


## Step 2: Check for Missing Critical Fields

**Goal:**  
Check that all critical fields (`sales_agent`, `product`, `account`, `deal_stage`) are not missing.

**SQL Query:**
```sql
SELECT *
FROM sales_pipeline
WHERE sales_agent IS NULL
   OR product IS NULL
   OR account IS NULL
   OR deal_stage IS NULL;
 ```
 **Result:**

| Field   | Total Missing |
|---------|---------------|
| account | 1,203         |
| sales_agent | 0         |
| product | 0             |
| deal_stage | 0          |



The data has 8,003 total rows, of which 1,203 rows have missing account values. All other critical fields (`sales_agent`, `product`, `account`, `deal_stage`) are complete. The missing accounts will need to be addressed before analysis.


## Solution for Missing Accounts

After investigation, the account information for the 1,203 missing rows cannot be recovered immediately. To preserve revenue in reporting, these records have been assigned "Unknown Account" and will be updated once the correct information is obtained from the sales team.

**SQL Query to assign Unknown Account:**
```sql
UPDATE sales_pipeline
SET account = 'Unknown Account'
WHERE account IS NULL;
```

## Step 3: Deal Stage Consistency

**Goal:**  
Ensure that Won and Lost deals have both `close_date` and `close_value`, and that open deals (Prospecting and Engaging) do not have these fields filled. This prevents errors in revenue reporting and pipeline analysis.

**SQL Query to check deal stage consistency:**
```sql
SELECT *
FROM sales_pipeline
WHERE (deal_stage IN ('Won','Lost') AND (close_date IS NULL OR close_value IS NULL))
   OR (deal_stage IN ('Prospecting','Engaging') AND (close_date IS NOT NULL OR close_value IS NOT NULL));
```

**Result:**

| Validation Check                          | Status  |
|-------------------------------------------|---------|
| Won/Lost deals missing close_date         | 0 rows  |
| Won/Lost deals missing close_value        | 0 rows  |
| Open deals with close_date filled         | 0 rows  |
| Open deals with close_value filled        | 0 rows  |

All deal stages are consistent. Closed deals contain valid closing information, and open deals do not contain premature closing data.

## 3b. Validate Date Format

Ensure `engage_date` and `close_date` are stored as DATE data types.

SQL Query:

```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'sales_pipeline'
  AND column_name IN ('engage_date','close_date');
 ```

 **Query Result**

| column_name | data_type |
|-------------|----------|
| close_date  | date     |
| engage_date | date     |


Both `close_date` and `engage_date` are stored as DATE data types. 
This confirms that no format correction or type conversion is required.

## Step 4: Revenue Validation

**Goal:**  
Ensure all revenue values are valid. Checks include:  
- No NULL revenue  
- No negative revenue  
- No zero-value revenue for closed deals  
- Revenue is stored as numeric

---

### SQL Queries

**1. Check for NULL revenue**
```sql
SELECT COUNT(*) AS null_revenue_count
FROM sales_pipeline
WHERE close_value IS NULL;
 ```

**2. Check for negative revenue**
```sql
 SELECT COUNT(*) AS negative_revenue_count
FROM sales_pipeline
WHERE close_value < 0;
 ```

 **3. Check for zero revenue in closed dealse**
```sql
 SELECT COUNT(*) AS zero_revenue_closed_deals
FROM sales_pipeline
WHERE deal_stage = 'Won'
AND close_value = 0;
 ```

 **4. Confirm revenue data type**
```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'sales_pipeline'
AND column_name = 'close_value';
 ```

 ### Step 4: Revenue Validation Result

| Check Type                     | Result |
|--------------------------------|--------|
| NULL Revenue                    | 0      |
| Negative Revenue                | 0      |
| Zero Revenue (Closed Deals)     | 0      |
| Revenue Data Type               | numeric |

All revenue values are valid. No NULL, negative or inconsistent revenue records were found.

## Step 5: Revenue Outlier Detection

**Goal:**  
Identify unusually high or low `close_value` entries that could distort reporting or analysis.

---

### SQL Query

```sql
SELECT MIN(close_value) AS min_revenue,
       MAX(close_value) AS max_revenue,
       PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY close_value) AS q1,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY close_value) AS median,
       PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY close_value) AS q3
FROM sales_pipeline;
```

### Result

| Statistic           | close_value |
|--------------------|------------|
| Minimum             | 0          |
| 1st Quartile (Q1)   | 0          |
| Median              | 0          |
| 3rd Quartile (Q3)   | 647.25     |
| Maximum             | 30288      |


Most deals have low or zero revenue, which is expected for smaller or early-stage opportunities. The  flagged high-value deal of 30,288 USD has been confirmed by the sales team as legitimate. Overall, all revenue values are within reasonable ranges, and no further adjustments are needed.


### Summary

After completing the data cleaning and validation steps:

- **Dataset integrity is confirmed:** All sales opportunities have unique IDs, revenue values are complete and numeric and dates are standardized.  
- **Missing accounts:** 1,203 records were assigned `"Unknown Account"`; these will be updated once the sales team provides the missing information.  
- **Revenue review:** Outliers were identified and reviewed with the sales team. The high-value deal of 30,288 was confirmed as legitimate.  
- **Data readiness:** The dataset is now clean and reliable, suitable for accurate performance reporting, sales pipeline analysis and revenue forecasting.  
- **Next steps:** Business teams can confidently generate KPIs, dashboards and analytics without data quality concerns and missing account updates can be incorporated as they become available.