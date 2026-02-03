# SQL Interview Questions for Slice

## About Slice
Slice is a fintech company focused on payments, credit cards, and merchant solutions. These SQL questions are designed around common business scenarios in the payments/fintech domain.

---

## Question 1: Monthly Active Users (MAU) Growth Rate

**Interviewer:** "We want to calculate the Month-over-Month growth rate of Monthly Active Users (MAU). Can you write a query for this?"

**Candidate:** "Sure! Before I begin, I have a few clarifying questions:
1. How do we define an 'active user'? Is it any user who made at least one transaction, or logged in, or something else?
2. What time period should I consider?
3. Should I handle cases where the previous month had zero users (to avoid division by zero)?"

**Interviewer:** "Good questions. Let's define active user as someone who made at least one transaction. Consider the last 12 months. Yes, please handle edge cases."

**Candidate:** "Perfect. Here's my approach:

### Step-wise Approach:
1. First, I'll aggregate unique users per month from the transactions table
2. Use LAG() window function to get previous month's MAU
3. Calculate growth rate as ((current - previous) / previous) * 100
4. Handle NULL and zero cases

### Code:

```sql
WITH monthly_active_users AS (
    SELECT 
        DATE_TRUNC('month', transaction_date) AS month,
        COUNT(DISTINCT user_id) AS mau
    FROM transactions
    WHERE transaction_date >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '12 months'
    GROUP BY DATE_TRUNC('month', transaction_date)
),
mau_with_previous AS (
    SELECT 
        month,
        mau,
        LAG(mau) OVER (ORDER BY month) AS previous_mau
    FROM monthly_active_users
)
SELECT 
    month,
    mau,
    previous_mau,
    CASE 
        WHEN previous_mau IS NULL OR previous_mau = 0 THEN NULL
        ELSE ROUND(((mau - previous_mau) * 100.0 / previous_mau), 2)
    END AS growth_rate_percentage
FROM mau_with_previous
ORDER BY month;
```

### Key Points:
- Used CTEs for readability
- `DATE_TRUNC` normalizes dates to month level
- `LAG()` window function efficiently gets previous row
- Handled division by zero with CASE statement"

---

## Question 2: Identify Churned Users

**Interviewer:** "Can you write a query to identify users who churned? These are users who were active in the previous month but not in the current month."

**Candidate:** "Got it. Let me clarify:
1. By 'current month', do you mean the calendar current month or should it be parameterized?
2. What constitutes 'active' - any transaction?
3. Should I return just user IDs or additional user details?"

**Interviewer:** "Use current calendar month. Active means at least one transaction. Return user_id, name, and their last transaction date."

**Candidate:** "Great, here's my approach:

### Step-wise Approach:
1. Find users active in previous month
2. Find users active in current month
3. Use LEFT JOIN or EXCEPT to find users in previous but not in current
4. Join with users table for additional details

### Code:

```sql
WITH previous_month_users AS (
    SELECT DISTINCT user_id
    FROM transactions
    WHERE transaction_date >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
      AND transaction_date < DATE_TRUNC('month', CURRENT_DATE)
),
current_month_users AS (
    SELECT DISTINCT user_id
    FROM transactions
    WHERE transaction_date >= DATE_TRUNC('month', CURRENT_DATE)
      AND transaction_date < DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '1 month'
),
churned_users AS (
    SELECT user_id 
    FROM previous_month_users
    WHERE user_id NOT IN (SELECT user_id FROM current_month_users)
)
SELECT 
    u.user_id,
    u.name,
    u.email,
    MAX(t.transaction_date) AS last_transaction_date
FROM churned_users c
JOIN users u ON c.user_id = u.user_id
JOIN transactions t ON c.user_id = t.user_id
GROUP BY u.user_id, u.name, u.email
ORDER BY last_transaction_date DESC;
```

### Alternative using LEFT JOIN:

```sql
SELECT 
    pm.user_id,
    u.name,
    MAX(t.transaction_date) AS last_transaction_date
FROM previous_month_users pm
LEFT JOIN current_month_users cm ON pm.user_id = cm.user_id
JOIN users u ON pm.user_id = u.user_id
JOIN transactions t ON pm.user_id = t.user_id
WHERE cm.user_id IS NULL
GROUP BY pm.user_id, u.name;
```

### Key Points:
- LEFT JOIN with NULL check is generally more performant than NOT IN for large datasets
- Used date ranges to ensure accurate month boundaries"

---

## Question 3: Top Merchants by Transaction Volume

**Interviewer:** "Write a query to find the top 10 merchants by transaction volume for each category."

**Candidate:** "Sure! A few questions:
1. By 'volume', do you mean total count of transactions or total monetary value?
2. Should I handle ties? (e.g., if 10th and 11th have same volume)
3. What if a category has fewer than 10 merchants?"

**Interviewer:** "Volume means total monetary value. Yes, include ties. Include all merchants if less than 10."

**Candidate:** "Perfect, I'll use DENSE_RANK() for ties.

### Step-wise Approach:
1. Aggregate transaction amounts by merchant and category
2. Use DENSE_RANK() window function partitioned by category
3. Filter for rank <= 10

### Code:

```sql
WITH merchant_volume AS (
    SELECT 
        m.category,
        m.merchant_id,
        m.merchant_name,
        SUM(t.amount) AS total_volume,
        COUNT(*) AS transaction_count
    FROM transactions t
    JOIN merchants m ON t.merchant_id = m.merchant_id
    WHERE t.transaction_date >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month'
    GROUP BY m.category, m.merchant_id, m.merchant_name
),
ranked_merchants AS (
    SELECT 
        category,
        merchant_id,
        merchant_name,
        total_volume,
        transaction_count,
        DENSE_RANK() OVER (
            PARTITION BY category 
            ORDER BY total_volume DESC
        ) AS rank
    FROM merchant_volume
)
SELECT 
    category,
    rank,
    merchant_id,
    merchant_name,
    total_volume,
    transaction_count
FROM ranked_merchants
WHERE rank <= 10
ORDER BY category, rank;
```

### Key Points:
- `DENSE_RANK()` ensures ties get same rank and no gaps
- `ROW_NUMBER()` would skip ties, `RANK()` would leave gaps
- Partitioned by category to get top 10 per category"

---

## Question 4: Detecting Fraudulent Transactions

**Interviewer:** "Can you write a query to flag potentially fraudulent transactions? Consider transactions that are 3x the user's average transaction amount or multiple transactions within 5 minutes."

**Candidate:** "Great question! Let me clarify:
1. Should I calculate the average from all historical transactions or a rolling window?
2. For the time-based rule, is it any 5-minute window or specifically from the same merchant?
3. Should I flag if either condition is true or both?"

**Interviewer:** "Use 90-day rolling average. Time-based applies to any merchant. Flag if either condition is true."

**Candidate:** "Got it.

### Step-wise Approach:
1. Calculate 90-day rolling average per user
2. Identify transactions 3x above average
3. Use LAG/LEAD to find transactions within 5 minutes
4. Combine both conditions with UNION or CASE

### Code:

```sql
WITH user_avg AS (
    SELECT 
        user_id,
        AVG(amount) AS avg_amount
    FROM transactions
    WHERE transaction_date >= CURRENT_DATE - INTERVAL '90 days'
    GROUP BY user_id
),
transactions_with_context AS (
    SELECT 
        t.transaction_id,
        t.user_id,
        t.amount,
        t.transaction_date,
        t.merchant_id,
        ua.avg_amount,
        LAG(t.transaction_date) OVER (
            PARTITION BY t.user_id 
            ORDER BY t.transaction_date
        ) AS prev_transaction_time,
        LEAD(t.transaction_date) OVER (
            PARTITION BY t.user_id 
            ORDER BY t.transaction_date
        ) AS next_transaction_time
    FROM transactions t
    LEFT JOIN user_avg ua ON t.user_id = ua.user_id
)
SELECT 
    transaction_id,
    user_id,
    amount,
    transaction_date,
    merchant_id,
    avg_amount,
    CASE 
        WHEN amount > 3 * COALESCE(avg_amount, 0) THEN 'HIGH_AMOUNT'
        ELSE NULL 
    END AS amount_flag,
    CASE 
        WHEN transaction_date - prev_transaction_time < INTERVAL '5 minutes'
          OR next_transaction_time - transaction_date < INTERVAL '5 minutes'
        THEN 'RAPID_SUCCESSION'
        ELSE NULL 
    END AS time_flag,
    CASE 
        WHEN amount > 3 * COALESCE(avg_amount, 0) 
          OR transaction_date - prev_transaction_time < INTERVAL '5 minutes'
          OR next_transaction_time - transaction_date < INTERVAL '5 minutes'
        THEN TRUE 
        ELSE FALSE 
    END AS is_potentially_fraudulent
FROM transactions_with_context
WHERE amount > 3 * COALESCE(avg_amount, 0)
   OR transaction_date - prev_transaction_time < INTERVAL '5 minutes'
   OR next_transaction_time - transaction_date < INTERVAL '5 minutes'
ORDER BY transaction_date DESC;
```

### Key Points:
- Used window functions for time-based detection
- COALESCE handles new users with no history
- Separate flags help understand why transaction was flagged"

---

## Question 5: Calculate User Retention Cohorts

**Interviewer:** "We want to analyze user retention by signup cohort. Can you build a cohort retention table?"

**Candidate:** "Absolutely! Clarifying questions:
1. How do we define retention - any activity or specific action (transaction)?
2. What granularity - weekly or monthly cohorts?
3. How many periods should I track?"

**Interviewer:** "Retention = made a transaction. Monthly cohorts. Track for 6 months."

**Candidate:** "Perfect, I'll build a cohort analysis.

### Step-wise Approach:
1. Identify signup month for each user (cohort)
2. For each user, identify which months they were active
3. Calculate months since signup
4. Aggregate by cohort and period
5. Calculate retention percentage

### Code:

```sql
WITH user_cohorts AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', signup_date) AS cohort_month
    FROM users
),
user_activities AS (
    SELECT 
        t.user_id,
        uc.cohort_month,
        DATE_TRUNC('month', t.transaction_date) AS activity_month
    FROM transactions t
    JOIN user_cohorts uc ON t.user_id = uc.user_id
),
cohort_activity AS (
    SELECT DISTINCT
        cohort_month,
        user_id,
        -- Calculate months since signup
        (EXTRACT(YEAR FROM activity_month) - EXTRACT(YEAR FROM cohort_month)) * 12 +
        (EXTRACT(MONTH FROM activity_month) - EXTRACT(MONTH FROM cohort_month)) AS months_since_signup
    FROM user_activities
    WHERE activity_month >= cohort_month
),
cohort_sizes AS (
    SELECT 
        cohort_month,
        COUNT(DISTINCT user_id) AS cohort_size
    FROM user_cohorts
    GROUP BY cohort_month
),
retention_data AS (
    SELECT 
        ca.cohort_month,
        ca.months_since_signup,
        COUNT(DISTINCT ca.user_id) AS retained_users
    FROM cohort_activity ca
    WHERE ca.months_since_signup <= 6
    GROUP BY ca.cohort_month, ca.months_since_signup
)
SELECT 
    rd.cohort_month,
    cs.cohort_size,
    rd.months_since_signup,
    rd.retained_users,
    ROUND(100.0 * rd.retained_users / cs.cohort_size, 2) AS retention_rate
FROM retention_data rd
JOIN cohort_sizes cs ON rd.cohort_month = cs.cohort_month
ORDER BY rd.cohort_month, rd.months_since_signup;
```

### Pivot Version (for visualization):

```sql
SELECT 
    cohort_month,
    cohort_size,
    MAX(CASE WHEN months_since_signup = 0 THEN retention_rate END) AS month_0,
    MAX(CASE WHEN months_since_signup = 1 THEN retention_rate END) AS month_1,
    MAX(CASE WHEN months_since_signup = 2 THEN retention_rate END) AS month_2,
    MAX(CASE WHEN months_since_signup = 3 THEN retention_rate END) AS month_3,
    MAX(CASE WHEN months_since_signup = 4 THEN retention_rate END) AS month_4,
    MAX(CASE WHEN months_since_signup = 5 THEN retention_rate END) AS month_5,
    MAX(CASE WHEN months_since_signup = 6 THEN retention_rate END) AS month_6
FROM (
    -- previous query as subquery
) retention_base
GROUP BY cohort_month, cohort_size
ORDER BY cohort_month;
```

### Key Points:
- Cohort analysis is crucial for understanding user behavior over time
- Month 0 should always be ~100% (signup month)
- Declining retention rates are normal; goal is to minimize drop-off"

---

## Question 6: Running Total of Credit Limit Utilization

**Interviewer:** "For our credit card users, calculate the running credit utilization percentage throughout the month."

**Candidate:** "Interesting! Let me clarify:
1. Is utilization = (total spent / credit limit)?
2. Should I consider payments made that reduce the balance?
3. Do I need daily granularity or transaction-level?"

**Interviewer:** "Yes, utilization = spent/limit. Yes, include payments. Transaction-level granularity."

**Candidate:** "Got it.

### Step-wise Approach:
1. Get each user's credit limit
2. Calculate running sum of transactions (positive for purchases, negative for payments)
3. Compute utilization at each point

### Code:

```sql
WITH credit_events AS (
    SELECT 
        user_id,
        transaction_id,
        transaction_date,
        transaction_type,
        CASE 
            WHEN transaction_type = 'PURCHASE' THEN amount
            WHEN transaction_type = 'PAYMENT' THEN -amount
            ELSE 0
        END AS balance_change
    FROM transactions
    WHERE DATE_TRUNC('month', transaction_date) = DATE_TRUNC('month', CURRENT_DATE)
),
running_balance AS (
    SELECT 
        ce.user_id,
        ce.transaction_id,
        ce.transaction_date,
        ce.transaction_type,
        ce.balance_change,
        SUM(ce.balance_change) OVER (
            PARTITION BY ce.user_id 
            ORDER BY ce.transaction_date, ce.transaction_id
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS cumulative_balance
    FROM credit_events ce
)
SELECT 
    rb.user_id,
    rb.transaction_id,
    rb.transaction_date,
    rb.transaction_type,
    rb.balance_change,
    rb.cumulative_balance,
    u.credit_limit,
    ROUND(100.0 * rb.cumulative_balance / NULLIF(u.credit_limit, 0), 2) AS utilization_percentage,
    CASE 
        WHEN rb.cumulative_balance / NULLIF(u.credit_limit, 0) > 0.9 THEN 'CRITICAL'
        WHEN rb.cumulative_balance / NULLIF(u.credit_limit, 0) > 0.7 THEN 'HIGH'
        WHEN rb.cumulative_balance / NULLIF(u.credit_limit, 0) > 0.3 THEN 'MODERATE'
        ELSE 'LOW'
    END AS utilization_tier
FROM running_balance rb
JOIN users u ON rb.user_id = u.user_id
ORDER BY rb.user_id, rb.transaction_date, rb.transaction_id;
```

### Key Points:
- Used `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` for clarity (default for ORDER BY)
- NULLIF prevents division by zero
- Added utilization tiers for business context
- Transaction_id in ORDER BY handles same-timestamp transactions"

---

## Question 7: Find Users with Declining Transaction Frequency

**Interviewer:** "Identify users whose transaction frequency has been declining for 3 consecutive months."

**Candidate:** "Good one! Questions:
1. By frequency, do you mean count of transactions?
2. Does 'declining' mean strictly less than previous month, or could it include same?
3. Should I look at the most recent 3 months or any 3 consecutive months?"

**Interviewer:** "Yes, count of transactions. Strictly less than. Most recent 3 months."

**Candidate:** "Perfect.

### Step-wise Approach:
1. Calculate monthly transaction counts per user
2. Use LAG to get previous months' counts
3. Check if each month is less than the previous
4. Filter for 3 consecutive declines

### Code:

```sql
WITH monthly_transactions AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', transaction_date) AS month,
        COUNT(*) AS transaction_count
    FROM transactions
    WHERE transaction_date >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '4 months'
    GROUP BY user_id, DATE_TRUNC('month', transaction_date)
),
with_previous AS (
    SELECT 
        user_id,
        month,
        transaction_count,
        LAG(transaction_count, 1) OVER (PARTITION BY user_id ORDER BY month) AS prev_month_1,
        LAG(transaction_count, 2) OVER (PARTITION BY user_id ORDER BY month) AS prev_month_2,
        LAG(transaction_count, 3) OVER (PARTITION BY user_id ORDER BY month) AS prev_month_3
    FROM monthly_transactions
),
declining_users AS (
    SELECT 
        user_id,
        month AS current_month,
        transaction_count AS current_count,
        prev_month_1,
        prev_month_2,
        prev_month_3
    FROM with_previous
    WHERE month = DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '1 month' -- Most recent complete month
      AND transaction_count < prev_month_1
      AND prev_month_1 < prev_month_2
      AND prev_month_2 < prev_month_3
      -- Ensure all months exist
      AND prev_month_1 IS NOT NULL
      AND prev_month_2 IS NOT NULL
      AND prev_month_3 IS NOT NULL
)
SELECT 
    d.user_id,
    u.name,
    u.email,
    d.prev_month_3 AS month_3_ago_count,
    d.prev_month_2 AS month_2_ago_count,
    d.prev_month_1 AS last_month_count,
    d.current_count,
    ROUND(100.0 * (d.prev_month_3 - d.current_count) / d.prev_month_3, 2) AS total_decline_percentage
FROM declining_users d
JOIN users u ON d.user_id = u.user_id
ORDER BY total_decline_percentage DESC;
```

### Key Points:
- Multiple LAG calls efficiently compare multiple periods
- NULL checks ensure user was active all 4 months
- Added total decline percentage for prioritization
- These users are at risk of churning - good targets for re-engagement"

---

## Question 8: Merchant Category Spending Analysis

**Interviewer:** "Write a query to show each user's spending distribution across merchant categories, and identify their primary spending category."

**Candidate:** "Sure! Clarifications:
1. Should I show percentage distribution or absolute amounts?
2. By 'primary', do you mean highest spend?
3. Any time constraint?"

**Interviewer:** "Both percentage and absolute. Yes, highest spend. Last 6 months."

**Candidate:** "Great.

### Step-wise Approach:
1. Aggregate spending by user and category
2. Calculate total spending per user
3. Compute percentages
4. Identify primary category using FIRST_VALUE or ranking

### Code:

```sql
WITH category_spending AS (
    SELECT 
        t.user_id,
        m.category,
        SUM(t.amount) AS category_amount,
        COUNT(*) AS transaction_count
    FROM transactions t
    JOIN merchants m ON t.merchant_id = m.merchant_id
    WHERE t.transaction_date >= CURRENT_DATE - INTERVAL '6 months'
      AND t.transaction_type = 'PURCHASE'
    GROUP BY t.user_id, m.category
),
user_totals AS (
    SELECT 
        user_id,
        SUM(category_amount) AS total_spending
    FROM category_spending
    GROUP BY user_id
),
spending_with_percentage AS (
    SELECT 
        cs.user_id,
        cs.category,
        cs.category_amount,
        cs.transaction_count,
        ut.total_spending,
        ROUND(100.0 * cs.category_amount / NULLIF(ut.total_spending, 0), 2) AS percentage_of_total,
        ROW_NUMBER() OVER (
            PARTITION BY cs.user_id 
            ORDER BY cs.category_amount DESC
        ) AS category_rank
    FROM category_spending cs
    JOIN user_totals ut ON cs.user_id = ut.user_id
)
SELECT 
    user_id,
    total_spending,
    MAX(CASE WHEN category_rank = 1 THEN category END) AS primary_category,
    MAX(CASE WHEN category_rank = 1 THEN percentage_of_total END) AS primary_category_pct,
    -- Pivot categories
    SUM(CASE WHEN category = 'Food & Dining' THEN percentage_of_total ELSE 0 END) AS food_dining_pct,
    SUM(CASE WHEN category = 'Shopping' THEN percentage_of_total ELSE 0 END) AS shopping_pct,
    SUM(CASE WHEN category = 'Travel' THEN percentage_of_total ELSE 0 END) AS travel_pct,
    SUM(CASE WHEN category = 'Entertainment' THEN percentage_of_total ELSE 0 END) AS entertainment_pct,
    SUM(CASE WHEN category = 'Bills & Utilities' THEN percentage_of_total ELSE 0 END) AS bills_pct,
    SUM(CASE WHEN category = 'Other' THEN percentage_of_total ELSE 0 END) AS other_pct
FROM spending_with_percentage
GROUP BY user_id, total_spending
ORDER BY total_spending DESC;
```

### Detail View (one row per user-category):

```sql
SELECT 
    sp.user_id,
    u.name,
    sp.category,
    sp.category_amount,
    sp.transaction_count,
    sp.percentage_of_total,
    CASE WHEN sp.category_rank = 1 THEN 'PRIMARY' ELSE '' END AS is_primary
FROM spending_with_percentage sp
JOIN users u ON sp.user_id = u.user_id
ORDER BY sp.user_id, sp.category_rank;
```

### Key Points:
- ROW_NUMBER() efficiently identifies the primary category
- Pivot syntax allows horizontal view of categories
- This data is valuable for personalized offers and recommendations"

---

## Question 9: Calculate 7-Day Rolling Average Transaction Amount

**Interviewer:** "Calculate the 7-day rolling average transaction amount for the platform, along with daily comparison to the rolling average."

**Candidate:** "Got it! Questions:
1. Should it be a trailing 7-day average (current day + 6 previous)?
2. Do you want all transactions or average per user first?
3. Should I handle days with no transactions?"

**Interviewer:** "Trailing 7-day. All transactions aggregated at platform level. Yes, handle missing days."

**Candidate:** "Perfect.

### Step-wise Approach:
1. Generate a date spine for continuous dates
2. Aggregate daily transaction amounts
3. Use window function for 7-day rolling average
4. Compare daily amount to rolling average

### Code:

```sql
WITH date_spine AS (
    SELECT generate_series(
        CURRENT_DATE - INTERVAL '90 days',
        CURRENT_DATE,
        '1 day'::interval
    )::date AS date
),
daily_stats AS (
    SELECT 
        DATE(transaction_date) AS date,
        SUM(amount) AS daily_amount,
        COUNT(*) AS transaction_count,
        COUNT(DISTINCT user_id) AS unique_users,
        AVG(amount) AS avg_transaction_amount
    FROM transactions
    WHERE transaction_date >= CURRENT_DATE - INTERVAL '90 days'
      AND transaction_type = 'PURCHASE'
    GROUP BY DATE(transaction_date)
),
daily_with_gaps AS (
    SELECT 
        ds.date,
        COALESCE(d.daily_amount, 0) AS daily_amount,
        COALESCE(d.transaction_count, 0) AS transaction_count,
        COALESCE(d.unique_users, 0) AS unique_users,
        d.avg_transaction_amount
    FROM date_spine ds
    LEFT JOIN daily_stats d ON ds.date = d.date
),
rolling_metrics AS (
    SELECT 
        date,
        daily_amount,
        transaction_count,
        unique_users,
        avg_transaction_amount,
        AVG(daily_amount) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS rolling_7day_avg_amount,
        SUM(transaction_count) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS rolling_7day_total_transactions,
        COUNT(*) OVER (
            ORDER BY date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS days_in_window
    FROM daily_with_gaps
)
SELECT 
    date,
    daily_amount,
    transaction_count,
    unique_users,
    ROUND(rolling_7day_avg_amount, 2) AS rolling_7day_avg,
    rolling_7day_total_transactions,
    ROUND(daily_amount - rolling_7day_avg_amount, 2) AS variance_from_avg,
    CASE 
        WHEN rolling_7day_avg_amount = 0 THEN NULL
        ELSE ROUND(100.0 * (daily_amount - rolling_7day_avg_amount) / rolling_7day_avg_amount, 2)
    END AS pct_variance,
    CASE 
        WHEN daily_amount > rolling_7day_avg_amount * 1.2 THEN 'ABOVE_TREND'
        WHEN daily_amount < rolling_7day_avg_amount * 0.8 THEN 'BELOW_TREND'
        ELSE 'NORMAL'
    END AS trend_status
FROM rolling_metrics
WHERE days_in_window = 7  -- Ensure full 7-day window
ORDER BY date DESC;
```

### Key Points:
- Date spine ensures no gaps in time series
- `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` = 7-day window
- Filter `days_in_window = 7` avoids partial calculations at start
- Variance calculations help identify anomalies"

---

## Question 10: Find Users Eligible for Credit Limit Increase

**Interviewer:** "Write a query to identify users eligible for a credit limit increase based on: good payment history, account age, and spending patterns."

**Candidate:** "Great business-relevant question! Let me clarify the criteria:
1. What defines 'good payment history'? No missed payments?
2. Minimum account age requirement?
3. What spending pattern - consistent usage or high utilization?
4. Are there any other disqualifying factors?"

**Interviewer:** "Good payment history = no late payments in last 6 months. Account age >= 6 months. Spending pattern = average utilization between 30-80%. Also exclude users who received an increase in last 6 months."

**Candidate:** "Perfect, clear criteria.

### Step-wise Approach:
1. Calculate payment history score
2. Check account age
3. Calculate average utilization
4. Check for recent limit increases
5. Combine all criteria

### Code:

```sql
WITH payment_history AS (
    SELECT 
        user_id,
        COUNT(*) AS total_payments,
        SUM(CASE WHEN payment_status = 'LATE' THEN 1 ELSE 0 END) AS late_payments,
        SUM(CASE WHEN payment_status = 'MISSED' THEN 1 ELSE 0 END) AS missed_payments
    FROM payment_records
    WHERE payment_date >= CURRENT_DATE - INTERVAL '6 months'
    GROUP BY user_id
),
account_age AS (
    SELECT 
        user_id,
        signup_date,
        CURRENT_DATE - signup_date AS days_since_signup,
        (CURRENT_DATE - signup_date) / 30 AS months_since_signup
    FROM users
),
monthly_utilization AS (
    SELECT 
        t.user_id,
        DATE_TRUNC('month', t.transaction_date) AS month,
        SUM(CASE WHEN t.transaction_type = 'PURCHASE' THEN t.amount ELSE 0 END) AS monthly_spend
    FROM transactions t
    WHERE t.transaction_date >= CURRENT_DATE - INTERVAL '6 months'
    GROUP BY t.user_id, DATE_TRUNC('month', t.transaction_date)
),
avg_utilization AS (
    SELECT 
        mu.user_id,
        AVG(mu.monthly_spend) AS avg_monthly_spend,
        u.credit_limit,
        AVG(mu.monthly_spend) / NULLIF(u.credit_limit, 0) AS avg_utilization_ratio
    FROM monthly_utilization mu
    JOIN users u ON mu.user_id = u.user_id
    GROUP BY mu.user_id, u.credit_limit
),
recent_increases AS (
    SELECT DISTINCT user_id
    FROM credit_limit_changes
    WHERE change_date >= CURRENT_DATE - INTERVAL '6 months'
      AND change_type = 'INCREASE'
),
eligibility_check AS (
    SELECT 
        u.user_id,
        u.name,
        u.email,
        u.credit_limit AS current_limit,
        aa.months_since_signup,
        COALESCE(ph.late_payments, 0) AS late_payments,
        COALESCE(ph.missed_payments, 0) AS missed_payments,
        ROUND(COALESCE(au.avg_utilization_ratio, 0) * 100, 2) AS avg_utilization_pct,
        CASE WHEN ri.user_id IS NOT NULL THEN TRUE ELSE FALSE END AS had_recent_increase,
        -- Eligibility flags
        CASE WHEN aa.months_since_signup >= 6 THEN TRUE ELSE FALSE END AS meets_age_requirement,
        CASE WHEN COALESCE(ph.late_payments, 0) = 0 
              AND COALESCE(ph.missed_payments, 0) = 0 THEN TRUE ELSE FALSE END AS good_payment_history,
        CASE WHEN COALESCE(au.avg_utilization_ratio, 0) BETWEEN 0.3 AND 0.8 
             THEN TRUE ELSE FALSE END AS healthy_utilization
    FROM users u
    LEFT JOIN account_age aa ON u.user_id = aa.user_id
    LEFT JOIN payment_history ph ON u.user_id = ph.user_id
    LEFT JOIN avg_utilization au ON u.user_id = au.user_id
    LEFT JOIN recent_increases ri ON u.user_id = ri.user_id
    WHERE u.account_status = 'ACTIVE'
)
SELECT 
    user_id,
    name,
    email,
    current_limit,
    months_since_signup,
    late_payments,
    missed_payments,
    avg_utilization_pct,
    -- Recommended increase (example: 20% for good users, 30% for excellent)
    CASE 
        WHEN avg_utilization_pct >= 60 THEN ROUND(current_limit * 0.30, -2) -- 30% increase, rounded to nearest 100
        ELSE ROUND(current_limit * 0.20, -2) -- 20% increase
    END AS recommended_increase,
    current_limit + CASE 
        WHEN avg_utilization_pct >= 60 THEN ROUND(current_limit * 0.30, -2)
        ELSE ROUND(current_limit * 0.20, -2)
    END AS proposed_new_limit
FROM eligibility_check
WHERE meets_age_requirement = TRUE
  AND good_payment_history = TRUE
  AND healthy_utilization = TRUE
  AND had_recent_increase = FALSE
ORDER BY avg_utilization_pct DESC, months_since_signup DESC;
```

### Summary Statistics:

```sql
SELECT 
    COUNT(*) AS total_eligible_users,
    SUM(current_limit) AS total_current_exposure,
    SUM(recommended_increase) AS total_proposed_increase,
    AVG(avg_utilization_pct) AS avg_utilization_of_eligible,
    AVG(months_since_signup) AS avg_account_age_months
FROM (
    -- Previous query as subquery
) eligible_users;
```

### Key Points:
- Multiple CTEs organize complex business logic clearly
- LEFT JOINs ensure we don't exclude users missing from some tables
- COALESCE handles NULLs appropriately
- Recommended increase logic can be adjusted based on business rules
- This query would be valuable for credit risk team and could be scheduled monthly"

---

## Bonus Tips for SQL Interviews

### General Framework for Answering SQL Questions:

1. **Clarify Requirements** (30 seconds)
   - Define key terms
   - Understand edge cases
   - Confirm output format

2. **State Your Approach** (1 minute)
   - Break down the problem
   - Mention which SQL features you'll use
   - Discuss trade-offs if any

3. **Write the Query** (3-5 minutes)
   - Start with CTEs for complex queries
   - Write incrementally, testing logic
   - Add comments for complex sections

4. **Validate & Optimize** (1 minute)
   - Check for edge cases
   - Discuss potential indexes
   - Mention performance considerations

### Key SQL Concepts for Fintech:
- Window functions (LAG, LEAD, ROW_NUMBER, DENSE_RANK, running totals)
- Date/time manipulation
- CTEs for readability
- Handling NULLs and edge cases
- Cohort analysis
- Rolling calculations
- Fraud detection patterns
- Financial calculations (interest, utilization, etc.)

### Common Pitfalls to Avoid:
- Division by zero
- Not handling NULL values
- Incorrect date boundaries
- Not considering ties in ranking
- Forgetting to deduplicate when needed
- Using GROUP BY with wrong columns
