## SQL Interview Questions for Slice

### Question 1: Calculate Monthly Active Users (MAU) Growth Rate for Slice

**Interviewer:** We need to track our monthly active user growth. Can you calculate month-over-month MAU and the growth rate?

**You:** Great question! Let me clarify a few things. When you say "active user," do you mean a customer who made at least one transaction in that month?

**Interviewer:** Yes, exactly—anyone who completed at least one transaction.

**You:** Perfect. And for the growth rate, should I compare each month with the previous month? For example, February growth compared to January?

**Interviewer:** Correct. Calculate the percentage growth from the previous month.

**You:** Got it. Should I focus on a specific time period, or analyze all available data?

**Interviewer:** Let's look at the last 12 months of data.

**You:** Excellent. I'll use a CTE-based approach with LAG window function for this.

**Solution:**

WITH monthly_active_users AS (
  SELECT 
    DATE_TRUNC('month', transaction_date) AS month,
    COUNT(DISTINCT customer_id) AS active_users
  FROM transactions
  WHERE transaction_date >= CURRENT_DATE - INTERVAL '12 months'
    AND status = 'completed'
  GROUP BY DATE_TRUNC('month', transaction_date)
),
growth_calculation AS (
  SELECT 
    month,
    active_users,
    LAG(active_users) OVER (ORDER BY month) AS prev_month_users,
    ROUND(
      ((active_users - LAG(active_users) OVER (ORDER BY month))::DECIMAL 
       / NULLIF(LAG(active_users) OVER (ORDER BY month), 0)) * 100, 
      2
    ) AS mom_growth_rate
  FROM monthly_active_users
)
SELECT 
  TO_CHAR(month, 'YYYY-MM') AS month,
  active_users,
  prev_month_users,
  COALESCE(mom_growth_rate, 0) AS mom_growth_percentage
FROM growth_calculation
ORDER BY month;

**Explanation:**

"I'm using a two-step CTE approach. First, I aggregate distinct active customers by month using DATE_TRUNC. Then I use the LAG window function to fetch the previous month's count. The NULLIF prevents division by zero errors, and COALESCE ensures the first month shows 0% instead of NULL. This gives us clean month-over-month percentage growth, which is crucial for Slice's growth metrics tracking."

---

### Question 2: Identify Customers with Declining Transaction Patterns (Churn Risk)

**Interviewer:** We want to identify customers who might be churning. Can you find customers whose transaction activity has declined significantly in the last 30 days compared to the previous 60 days?

**You:** Excellent retention question! Just to clarify—when you say "declined significantly," what threshold should I use? For example, 50% reduction in transaction count?

**Interviewer:** Yes, let's flag anyone whose transaction count in the last 30 days is less than 50% of their average in the 31-60 day window.

**You:** Perfect. Should I consider only completed transactions, or include declined ones too?

**Interviewer:** Only completed transactions—we care about actual usage patterns.

**You:** Got it. Should I also calculate the monetary decline, or just transaction frequency?

**Interviewer:** Good thinking—include both transaction count and total spend decline.

**Solution:**

WITH customer_activity_windows AS (
  SELECT 
    customer_id,
    -- Last 30 days activity
    COUNT(CASE WHEN transaction_date >= CURRENT_DATE - INTERVAL '30 days' 
          THEN transaction_id END) AS txn_count_last_30d,
    SUM(CASE WHEN transaction_date >= CURRENT_DATE - INTERVAL '30 days' 
        THEN amount ELSE 0 END) AS spend_last_30d,
    
    -- 31-60 days activity (previous period)
    COUNT(CASE WHEN transaction_date BETWEEN CURRENT_DATE - INTERVAL '60 days' 
               AND CURRENT_DATE - INTERVAL '31 days' 
          THEN transaction_id END) AS txn_count_31_60d,
    SUM(CASE WHEN transaction_date BETWEEN CURRENT_DATE - INTERVAL '60 days' 
             AND CURRENT_DATE - INTERVAL '31 days' 
        THEN amount ELSE 0 END) AS spend_31_60d
  FROM transactions
  WHERE transaction_date >= CURRENT_DATE - INTERVAL '60 days'
    AND status = 'completed'
  GROUP BY customer_id
),
churn_risk_analysis AS (
  SELECT 
    c.customer_id,
    c.name,
    c.city,
    c.credit_score,
    caw.txn_count_last_30d,
    caw.txn_count_31_60d,
    caw.spend_last_30d,
    caw.spend_31_60d,
    ROUND((caw.txn_count_last_30d::DECIMAL / NULLIF(caw.txn_count_31_60d, 0)) * 100, 1) 
      AS txn_retention_rate,
    ROUND((caw.spend_last_30d / NULLIF(caw.spend_31_60d, 0)) * 100, 1) 
      AS spend_retention_rate,
    CASE 
      WHEN caw.txn_count_last_30d < (caw.txn_count_31_60d * 0.5) THEN 'High Risk'
      WHEN caw.txn_count_last_30d < (caw.txn_count_31_60d * 0.75) THEN 'Medium Risk'
      ELSE 'Low Risk'
    END AS churn_risk_category
  FROM customer_activity_windows caw
  JOIN customers c ON caw.customer_id = c.customer_id
  WHERE caw.txn_count_31_60d > 0  -- Had activity in previous period
)
SELECT 
  customer_id,
  name,
  city,
  credit_score,
  txn_count_last_30d,
  txn_count_31_60d,
  txn_retention_rate,
  spend_last_30d,
  spend_31_60d,
  spend_retention_rate,
  churn_risk_category
FROM churn_risk_analysis
WHERE churn_risk_category IN ('High Risk', 'Medium Risk')
ORDER BY txn_retention_rate ASC, spend_retention_rate ASC
LIMIT 100;

**Explanation:**

"I'm comparing two 30-day windows to detect activity decline. The key insight is calculating retention rates for both transaction frequency and spend amount—some customers might transact less frequently but maintain high spend per transaction. I categorize risk into High/Medium/Low based on the 50% and 75% thresholds. This helps Slice's retention team prioritize outreach to at-risk customers before they fully churn."

---

### Question 3: Merchant Category Performance - Which Categories Drive Slice Revenue?

**Interviewer:** We want to understand which merchant categories generate the most revenue for Slice. Can you analyze merchant category performance?

**You:** Absolutely! Just to confirm—by revenue for Slice, do you mean the commission we earn from merchants?

**Interviewer:** Yes, exactly. Revenue would be transaction amount multiplied by the merchant's commission rate.

**You:** Perfect. Should I calculate this for all time, or focus on a recent period?

**Interviewer:** Let's look at the last quarter (90 days) to see current trends.

**You:** Got it. Should I also include metrics like transaction count, average order value, and customer penetration by category?

**Interviewer:** Yes, that would give us a complete picture. Also show category growth trends.

**Solution:**

WITH category_metrics_current AS (
  SELECT 
    m.merchant_category,
    COUNT(DISTINCT t.transaction_id) AS total_transactions,
    COUNT(DISTINCT t.customer_id) AS unique_customers,
    SUM(t.amount) AS total_gmv,  -- Gross Merchandise Value
    SUM(t.amount * m.commission_rate / 100) AS slice_revenue,
    AVG(t.amount) AS avg_transaction_value,
    ROUND(AVG(m.commission_rate), 2) AS avg_commission_rate
  FROM transactions t
  JOIN merchants m ON t.merchant_id = m.merchant_id
  WHERE t.transaction_date >= CURRENT_DATE - INTERVAL '90 days'
    AND t.status = 'completed'
  GROUP BY m.merchant_category
),
category_metrics_previous AS (
  SELECT 
    m.merchant_category,
    SUM(t.amount * m.commission_rate / 100) AS slice_revenue_prev_quarter
  FROM transactions t
  JOIN merchants m ON t.merchant_id = m.merchant_id
  WHERE t.transaction_date BETWEEN CURRENT_DATE - INTERVAL '180 days' 
        AND CURRENT_DATE - INTERVAL '91 days'
    AND t.status = 'completed'
  GROUP BY m.merchant_category
),
category_totals AS (
  SELECT 
    SUM(total_transactions) AS overall_txn_count,
    SUM(unique_customers) AS overall_customer_count,
    SUM(slice_revenue) AS overall_revenue
  FROM category_metrics_current
)
SELECT 
  cmc.merchant_category,
  cmc.total_transactions,
  ROUND((cmc.total_transactions::DECIMAL / ct.overall_txn_count) * 100, 2) 
    AS pct_of_total_txns,
  cmc.unique_customers,
  ROUND((cmc.unique_customers::DECIMAL / ct.overall_customer_count) * 100, 2) 
    AS customer_penetration_pct,
  ROUND(cmc.total_gmv, 2) AS gmv,
  ROUND(cmc.slice_revenue, 2) AS slice_revenue,
  ROUND((cmc.slice_revenue / ct.overall_revenue) * 100, 2) AS pct_of_total_revenue,
  ROUND(cmc.avg_transaction_value, 2) AS avg_order_value,
  cmc.avg_commission_rate,
  ROUND(
    ((cmc.slice_revenue - COALESCE(cmp.slice_revenue_prev_quarter, 0)) 
     / NULLIF(cmp.slice_revenue_prev_quarter, 0)) * 100, 
    2
  ) AS qoq_revenue_growth_pct
FROM category_metrics_current cmc
CROSS JOIN category_totals ct
LEFT JOIN category_metrics_previous cmp 
  ON cmc.merchant_category = cmp.merchant_category
ORDER BY cmc.slice_revenue DESC;

**Explanation:**

"This query provides a comprehensive merchant category dashboard. I calculate Slice's actual revenue using the commission rates, not just GMV. The query includes penetration metrics showing what percentage of Slice customers use each category, and QoQ growth to spot trending categories. For example, if 'Food' has high GMV but low commission rates, while 'Electronics' has lower volume but higher margins, Slice might want to strategically partner with more electronics merchants. This data directly informs partnership strategy."

---

### Question 4: Cohort Retention Analysis - Monthly Signup Cohorts

**Interviewer:** We need a cohort retention analysis. Can you show retention rates for each monthly signup cohort over their first 6 months?

**You:** Perfect—cohort analysis is crucial for understanding user behavior! Just to clarify, by "retained," do you mean customers who made at least one transaction in that month?

**Interviewer:** Yes, any completed transaction counts as active for that month.

**You:** Got it. Should the retention be cumulative (active at any point up to that month) or specific (active in that exact month)?

**Interviewer:** Specific to each month—we want to see the retention curve month by month.

**You:** Perfect. And should I include only users who signed up in the last 12 months to have enough forward-looking data?

**Interviewer:** Yes, that makes sense. Focus on recent cohorts.

**Solution:**

WITH cohorts AS (
  SELECT 
    customer_id,
    DATE_TRUNC('month', signup_date) AS cohort_month
  FROM customers
  WHERE signup_date >= CURRENT_DATE - INTERVAL '12 months'
),
user_activities AS (
  SELECT DISTINCT
    customer_id,
    DATE_TRUNC('month', transaction_date) AS activity_month
  FROM transactions
  WHERE status = 'completed'
),
cohort_activities AS (
  SELECT 
    c.cohort_month,
    ua.customer_id,
    ua.activity_month,
    EXTRACT(MONTH FROM AGE(ua.activity_month, c.cohort_month))::INTEGER 
      AS months_since_signup
  FROM cohorts c
  JOIN user_activities ua ON c.customer_id = ua.customer_id
  WHERE EXTRACT(MONTH FROM AGE(ua.activity_month, c.cohort_month)) BETWEEN 0 AND 6
),
cohort_sizes AS (
  SELECT 
    cohort_month,
    COUNT(DISTINCT customer_id) AS cohort_size
  FROM cohorts
  GROUP BY cohort_month
),
retention_data AS (
  SELECT 
    ca.cohort_month,
    ca.months_since_signup,
    COUNT(DISTINCT ca.customer_id) AS active_users,
    cs.cohort_size
  FROM cohort_activities ca
  JOIN cohort_sizes cs ON ca.cohort_month = cs.cohort_month
  GROUP BY ca.cohort_month, ca.months_since_signup, cs.cohort_size
)
SELECT 
  TO_CHAR(cohort_month, 'YYYY-MM') AS cohort,
  cohort_size,
  MAX(CASE WHEN months_since_signup = 0 THEN ROUND(100.0 * active_users / cohort_size, 1) END) AS month_0,
  MAX(CASE WHEN months_since_signup = 1 THEN ROUND(100.0 * active_users / cohort_size, 1) END) AS month_1,
  MAX(CASE WHEN months_since_signup = 2 THEN ROUND(100.0 * active_users / cohort_size, 1) END) AS month_2,
  MAX(CASE WHEN months_since_signup = 3 THEN ROUND(100.0 * active_users / cohort_size, 1) END) AS month_3,
  MAX(CASE WHEN months_since_signup = 4 THEN ROUND(100.0 * active_users / cohort_size, 1) END) AS month_4,
  MAX(CASE WHEN months_since_signup = 5 THEN ROUND(100.0 * active_users / cohort_size, 1) END) AS month_5,
  MAX(CASE WHEN months_since_signup = 6 THEN ROUND(100.0 * active_users / cohort_size, 1) END) AS month_6
FROM retention_data
GROUP BY cohort_month, cohort_size
ORDER BY cohort_month DESC;

**Explanation:**

"This creates a classic cohort retention table. Month 0 shows what percentage of the cohort was active in their signup month (typically close to 100%), and subsequent columns show retention drop-off. This is critical for Slice to understand if product changes improve early retention. For example, if we see Month 1 retention improve from 40% to 55% after launching a new feature, that's a strong signal. The pivot format makes it easy to visualize retention curves and identify problematic cohorts."

---

### Question 5: Detect Unusual Spending Patterns - Fraud Detection Query

**Interviewer:** We need to identify potentially fraudulent transactions. Can you write a query to detect unusual spending patterns?

**You:** Important problem for fintech! Let me clarify the approach. Should I look for statistical anomalies like transactions that are 3 standard deviations above a customer's average?

**Interviewer:** Yes, that's a good start. Also flag multiple transactions within a very short time window.

**You:** Perfect. What time window should I consider suspicious? For example, 3+ transactions within 5 minutes?

**Interviewer:** Yes, 3 or more transactions within 5 minutes is unusual. Also check for international transactions that don't match the customer's usual pattern.

**You:** Got it. Should I create a risk score or just flag potential fraud cases?

**Interviewer:** Flag them with a risk level—High, Medium, Low—based on multiple signals.

**Solution:**

WITH customer_baseline AS (
  SELECT 
    customer_id,
    AVG(amount) AS avg_amount,
    STDDEV(amount) AS stddev_amount,
    COUNT(CASE WHEN is_international = TRUE THEN 1 END)::DECIMAL / 
      NULLIF(COUNT(*), 0) AS intl_transaction_ratio
  FROM transactions
  WHERE transaction_date >= CURRENT_DATE - INTERVAL '90 days'
    AND status = 'completed'
  GROUP BY customer_id
),
transaction_time_gaps AS (
  SELECT 
    transaction_id,
    customer_id,
    transaction_timestamp,
    amount,
    is_international,
    merchant_category,
    LAG(transaction_timestamp) OVER (
      PARTITION BY customer_id 
      ORDER BY transaction_timestamp
    ) AS prev_txn_timestamp,
    LEAD(transaction_timestamp) OVER (
      PARTITION BY customer_id 
      ORDER BY transaction_timestamp
    ) AS next_txn_timestamp
  FROM transactions
  WHERE transaction_date >= CURRENT_DATE - INTERVAL '7 days'
    AND status = 'completed'
),
rapid_fire_detection AS (
  SELECT 
    customer_id,
    transaction_timestamp,
    COUNT(*) OVER (
      PARTITION BY customer_id 
      ORDER BY transaction_timestamp 
      RANGE BETWEEN INTERVAL '5 minutes' PRECEDING AND CURRENT ROW
    ) AS txn_count_last_5min
  FROM transaction_time_gaps
),
fraud_signals AS (
  SELECT 
    t.transaction_id,
    t.customer_id,
    c.name AS customer_name,
    t.transaction_timestamp,
    t.amount,
    t.merchant_category,
    t.is_international,
    cb.avg_amount,
    cb.stddev_amount,
    cb.intl_transaction_ratio,
    
    -- Signal 1: Amount anomaly
    CASE 
      WHEN t.amount > (cb.avg_amount + 3 * cb.stddev_amount) THEN 1 
      ELSE 0 
    END AS amount_anomaly_flag,
    
    -- Signal 2: Rapid fire transactions
    CASE 
      WHEN rf.txn_count_last_5min >= 3 THEN 1 
      ELSE 0 
    END AS rapid_fire_flag,
    
    -- Signal 3: Unusual international transaction
    CASE 
      WHEN t.is_international = TRUE AND cb.intl_transaction_ratio < 0.1 THEN 1 
      ELSE 0 
    END AS unusual_intl_flag,
    
    -- Signal 4: Time-based anomaly (transaction at unusual hour)
    CASE 
      WHEN EXTRACT(HOUR FROM t.transaction_timestamp) BETWEEN 2 AND 5 THEN 1 
      ELSE 0 
    END AS unusual_hour_flag
  FROM transaction_time_gaps t
  JOIN customer_baseline cb ON t.customer_id = cb.customer_id
  JOIN customers c ON t.customer_id = c.customer_id
  LEFT JOIN rapid_fire_detection rf 
    ON t.customer_id = rf.customer_id 
    AND t.transaction_timestamp = rf.transaction_timestamp
),
fraud_risk_scoring AS (
  SELECT 
    *,
    (amount_anomaly_flag + rapid_fire_flag + unusual_intl_flag + unusual_hour_flag) 
      AS risk_score,
    CASE 
      WHEN (amount_anomaly_flag + rapid_fire_flag + unusual_intl_flag + unusual_hour_flag) >= 3 
        THEN 'High Risk'
      WHEN (amount_anomaly_flag + rapid_fire_flag + unusual_intl_flag + unusual_hour_flag) = 2 
        THEN 'Medium Risk'
      WHEN (amount_anomaly_flag + rapid_fire_flag + unusual_intl_flag + unusual_hour_flag) = 1 
        THEN 'Low Risk'
      ELSE 'Normal'
    END AS fraud_risk_level
  FROM fraud_signals
)
SELECT 
  transaction_id,
  customer_id,
  customer_name,
  transaction_timestamp,
  amount,
  merchant_category,
  is_international,
  ROUND(avg_amount, 2) AS customer_avg_amount,
  risk_score,
  fraud_risk_level,
  amount_anomaly_flag,
  rapid_fire_flag,
  unusual_intl_flag,
  unusual_hour_flag
FROM fraud_risk_scoring
WHERE fraud_risk_level IN ('High Risk', 'Medium Risk')
ORDER BY risk_score DESC, amount DESC
LIMIT 100;

**Explanation:**

"This multi-signal fraud detection system combines statistical analysis with behavioral patterns. I establish each customer's baseline spending over 90 days, then flag anomalies: amounts beyond 3 standard deviations, rapid-fire transactions within 5 minutes, international transactions from domestic-only users, and late-night activity. The risk score aggregates these signals—3+ flags mean high risk. This is similar to how credit card companies detect fraud. In production, Slice would feed these flags into a machine learning model, but this SQL provides immediate actionable alerts for the fraud team."

---

### Question 6: Customer Lifetime Value (CLV) Segmentation

**Interviewer:** We want to segment our customers by their lifetime value. Can you calculate CLV and create meaningful segments?

**You:** Excellent strategic question! Just to clarify—should I calculate CLV as total historical spend, or use a predictive model based on recent activity?

**Interviewer:** For now, let's use historical CLV—total revenue they've generated for Slice through commissions.

**You:** Perfect. Should I also include metrics like average transaction value, transaction frequency, and recency for richer segmentation?

**Interviewer:** Yes, use RFM (Recency, Frequency, Monetary) framework—that would be ideal.

**You:** Got it. Should I create specific segment names like "Champions," "At Risk," etc., or just numeric quartiles?

**Interviewer:** Named segments would be more actionable for the business team.

**Solution:**

WITH customer_metrics AS (
  SELECT 
    c.customer_id,
    c.name,
    c.city,
    c.credit_score,
    c.signup_date,
    
    -- Monetary: Total commission revenue generated
    SUM(t.amount * m.commission_rate / 100) AS total_clv,
    
    -- Frequency: Transaction count
    COUNT(DISTINCT t.transaction_id) AS total_transactions,
    
    -- Recency: Days since last transaction
    CURRENT_DATE - MAX(t.transaction_date) AS days_since_last_txn,
    
    -- Additional metrics
    AVG(t.amount) AS avg_transaction_value,
    COUNT(DISTINCT DATE_TRUNC('month', t.transaction_date)) AS active_months,
    MAX(t.transaction_date) AS last_transaction_date,
    MIN(t.transaction_date) AS first_transaction_date
  FROM customers c
  LEFT JOIN transactions t ON c.customer_id = t.customer_id AND t.status = 'completed'
  LEFT JOIN merchants m ON t.merchant_id = m.merchant_id
  GROUP BY c.customer_id, c.name, c.city, c.credit_score, c.signup_date
),
rfm_scoring AS (
  SELECT 
    *,
    -- Recency score (lower days = higher score)
    NTILE(5) OVER (ORDER BY days_since_last_txn ASC NULLS LAST) AS recency_score,
    
    -- Frequency score
    NTILE(5) OVER (ORDER BY total_transactions DESC) AS frequency_score,
    
    -- Monetary score
    NTILE(5) OVER (ORDER BY total_clv DESC) AS monetary_score
  FROM customer_metrics
),
customer_segments AS (
  SELECT 
    *,
    (recency_score + frequency_score + monetary_score) AS rfm_total_score,
    CASE 
      -- Champions: High R, F, M
      WHEN recency_score >= 4 AND frequency_score >= 4 AND monetary_score >= 4 
        THEN 'Champions'
      
      -- Loyal Customers: High F and M, moderate R
      WHEN frequency_score >= 4 AND monetary_score >= 4 
        THEN 'Loyal Customers'
      
      -- Potential Loyalists: Recent, moderate F and M
      WHEN recency_score >= 4 AND frequency_score BETWEEN 2 AND 4 
        THEN 'Potential Loyalists'
      
      -- At Risk: High M and F, but low R (haven't transacted recently)
      WHEN frequency_score >= 3 AND monetary_score >= 3 AND recency_score <= 2 
        THEN 'At Risk'
      
      -- Can't Lose: Very high M and F, very low R
      WHEN frequency_score >= 4 AND monetary_score >= 4 AND recency_score = 1 
        THEN 'Cannot Lose Them'
      
      -- New Customers: Recent signup, low F and M
      WHEN recency_score >= 4 AND frequency_score <= 2 
        THEN 'New Customers'
      
      -- Hibernating: Low R, F, M
      WHEN recency_score <= 2 AND frequency_score <= 2 AND monetary_score <= 2 
        THEN 'Hibernating'
      
      ELSE 'Need Attention'
    END AS customer_segment
  FROM rfm_scoring
)
SELECT 
  customer_segment,
  COUNT(*) AS customer_count,
  ROUND(AVG(total_clv), 2) AS avg_clv,
  ROUND(SUM(total_clv), 2) AS total_segment_clv,
  ROUND(AVG(total_transactions), 1) AS avg_transactions,
  ROUND(AVG(days_since_last_txn), 1) AS avg_days_since_last_txn,
  ROUND(AVG(avg_transaction_value), 2) AS avg_order_value,
  ROUND(AVG(recency_score), 2) AS avg_recency_score,
  ROUND(AVG(frequency_score), 2) AS avg_frequency_score,
  ROUND(AVG(monetary_score), 2) AS avg_monetary_score
FROM customer_segments
GROUP BY customer_segment
ORDER BY total_segment_clv DESC;

**Explanation:**

"This RFM-based segmentation gives Slice actionable customer groups. 'Champions' are your best customers—high engagement and value. 'At Risk' customers were valuable but haven't transacted recently—they need win-back campaigns. 'Cannot Lose Them' are your whales who are going dormant—urgent intervention needed. 'New Customers' need onboarding optimization. Each segment requires different marketing strategies. The beauty of this approach is it's interpretable for non-technical stakeholders and directly drives campaign targeting decisions."

---

### Question 7: Bill Payment Behavior - Identify On-Time Payers vs Defaulters

**Interviewer:** We need to analyze bill payment behavior to understand credit risk. Can you segment customers by their payment patterns?

**You:** Critical for Slice's credit business! Should I look at payment timeliness, partial payments, and default patterns?

**Interviewer:** Yes, exactly. Classify customers into Good Payers, Late Payers, Partial Payers, and Defaulters.

**You:** Perfect. What defines each category? For example, is "Late Payer" someone who paid after due date but eventually paid in full?

**Interviewer:** Yes. Good Payer = always on time and full amount. Late Payer = paid in full but after due date. Partial Payer = pays something but not full amount. Defaulter = 30+ days overdue with no payment.

**You:** Got it. Should I calculate this over all historical bills or focus on recent behavior like last 6 months?

**Interviewer:** Last 6 months to reflect current behavior patterns.

**Solution:**

WITH recent_bills AS (
  SELECT 
    b.customer_id,
    b.bill_id,
    b.bill_date,
    b.due_date,
    b.bill_amount,
    b.payment_status,
    b.payment_date,
    b.payment_amount,
    b.days_overdue,
    b.late_fee,
    b.interest_charged,
    CASE 
      WHEN b.payment_date <= b.due_date AND b.payment_amount >= b.bill_amount 
        THEN 'On Time - Full'
      WHEN b.payment_date > b.due_date AND b.payment_amount >= b.bill_amount 
        THEN 'Late - Full'
      WHEN b.payment_amount > 0 AND b.payment_amount < b.bill_amount 
        THEN 'Partial Payment'
      WHEN b.days_overdue >= 30 AND COALESCE(b.payment_amount, 0) = 0 
        THEN 'Defaulted'
      WHEN b.payment_status = 'Overdue' 
        THEN 'Overdue - Pending'
      ELSE 'Other'
    END AS bill_payment_type
  FROM bills b
  WHERE b.bill_date >= CURRENT_DATE - INTERVAL '6 months'
),
customer_payment_summary AS (
  SELECT 
    customer_id,
    COUNT(*) AS total_bills,
    
    -- Payment type breakdown
    COUNT(CASE WHEN bill_payment_type = 'On Time - Full' THEN 1 END) AS on_time_full_count,
    COUNT(CASE WHEN bill_payment_type = 'Late - Full' THEN 1 END) AS late_full_count,
    COUNT(CASE WHEN bill_payment_type = 'Partial Payment' THEN 1 END) AS partial_payment_count,
    COUNT(CASE WHEN bill_payment_type = 'Defaulted' THEN 1 END) AS defaulted_count,
    COUNT(CASE WHEN bill_payment_type = 'Overdue - Pending' THEN 1 END) AS overdue_pending_count,
    
    -- Financial metrics
    SUM(bill_amount) AS total_billed,
    SUM(COALESCE(payment_amount, 0)) AS total_paid,
    SUM(late_fee) AS total_late_fees,
    SUM(interest_charged) AS total_interest_charged,
    AVG(days_overdue) AS avg_days_overdue,
    MAX(days_overdue) AS max_days_overdue
  FROM recent_bills
  GROUP BY customer_id
),
customer_classification AS (
  SELECT 
    c.customer_id,
    c.name,
    c.city,
    c.credit_score,
    cps.total_bills,
    cps.on_time_full_count,
    cps.late_full_count,
    cps.partial_payment_count,
    cps.defaulted_count,
    cps.overdue_pending_count,
    ROUND((cps.on_time_full_count::DECIMAL / NULLIF(cps.total_bills, 0)) * 100, 1) 
      AS on_time_payment_rate,
    ROUND(cps.total_paid, 2) AS total_paid,
    ROUND(cps.total_billed, 2) AS total_billed,
    ROUND((cps.total_paid / NULLIF(cps.total_billed, 0)) * 100, 1) AS payment_completion_rate,
    ROUND(cps.total_late_fees, 2) AS total_late_fees,
    ROUND(cps.avg_days_overdue, 1) AS avg_days_overdue,
    
    -- Customer segment classification
    CASE 
      WHEN cps.on_time_full_count = cps.total_bills 
        THEN 'Excellent - Always On Time'
      
      WHEN cps.on_time_full_count::DECIMAL / cps.total_bills >= 0.8 
           AND cps.defaulted_count = 0 
        THEN 'Good Payer'
      
      WHEN cps.late_full_count::DECIMAL / cps.total_bills >= 0.5 
           AND cps.defaulted_count = 0 
        THEN 'Late Payer - Recoverable'
      
      WHEN cps.partial_payment_count >= 2 
           AND cps.defaulted_count = 0 
        THEN 'Partial Payer - Risk'
      
      WHEN cps.defaulted_count >= 1 
        THEN 'Defaulter - High Risk'
      
      WHEN cps.overdue_pending_count >= 2 
        THEN 'Multiple Overdue - Watch List'
      
      ELSE 'Needs Review'
    END AS payment_behavior_segment
  FROM customer_payment_summary cps
  JOIN customers c ON cps.customer_id = c.customer_id
)
SELECT 
  payment_behavior_segment,
  COUNT(*) AS customer_count,
  ROUND(AVG(on_time_payment_rate), 1) AS avg_on_time_rate,
  ROUND(AVG(payment_completion_rate), 1) AS avg_payment_completion_rate,
  ROUND(AVG(credit_score), 0) AS avg_credit_score,
  ROUND(SUM(total_late_fees), 2) AS total_late_fees_collected,
  ROUND(AVG(avg_days_overdue), 1) AS avg_days_overdue,
  ROUND(SUM(total_billed - total_paid), 2) AS total_outstanding_amount
FROM customer_classification
GROUP BY payment_behavior_segment
ORDER BY 
  CASE payment_behavior_segment
    WHEN 'Excellent - Always On Time' THEN 1
    WHEN 'Good Payer' THEN 2
    WHEN 'Late Payer - Recoverable' THEN 3
    WHEN 'Partial Payer - Risk' THEN 4
    WHEN 'Multiple Overdue - Watch List' THEN 5
    WHEN 'Defaulter - High Risk' THEN 6
    ELSE 7
  END;

**Explanation:**

"This payment behavior segmentation is critical for Slice's credit risk management. I classify each bill payment, then aggregate to customer level. 'Excellent' customers get credit limit increases and lower interest rates. 'Defaulters' need collections and credit limit reduction. The interesting insight is comparing credit scores across segments—if 'Defaulters' have high credit scores from other bureaus, it might indicate Slice-specific issues rather than general creditworthiness. The outstanding amount by segment helps prioritize collections efforts."

---

### Question 8: Referral Program Effectiveness Analysis

**Interviewer:** We have a referral program. Can you analyze its effectiveness—which customers bring valuable referrals?

**You:** Great growth lever to analyze! Should I look at both the referrer's quality and the referred customer's quality?

**Interviewer:** Yes, exactly. We want to know which types of customers refer high-value customers.

**You:** Perfect. Should I measure the referred customer's quality by their transaction volume in the first 90 days?

**Interviewer:** Yes, and also their retention rate—do they stick around or churn quickly?

**You:** Got it. Should I also analyze if certain acquisition channels produce better referrers?

**Interviewer:** Good thinking—yes, include that dimension.

**Solution:**

WITH referral_pairs AS (
  SELECT 
    r.referral_id,
    r.referrer_customer_id,
    r.referred_customer_id,
    r.referral_date,
    r.referral_status,
    r.reward_amount,
    
    -- Referrer details
    c1.name AS referrer_name,
    c1.acquisition_channel AS referrer_acquisition_channel,
    c1.signup_date AS referrer_signup_date,
    
    -- Referred customer details
    c2.name AS referred_name,
    c2.signup_date AS referred_signup_date
  FROM referrals r
  JOIN customers c1 ON r.referrer_customer_id = c1.customer_id
  JOIN customers c2 ON r.referred_customer_id = c2.customer_id
  WHERE r.referral_status = 'Completed'
),
referred_customer_performance AS (
  SELECT 
    t.customer_id AS referred_customer_id,
    COUNT(DISTINCT t.transaction_id) AS txn_count_first_90d,
    SUM(t.amount) AS total_spend_first_90d,
    COUNT(DISTINCT DATE_TRUNC('month', t.transaction_date)) AS active_months_first_90d,
    MAX(t.transaction_date) AS last_transaction_date
  FROM transactions t
  JOIN referral_pairs rp ON t.customer_id = rp.referred_customer_id
  WHERE t.transaction_date BETWEEN rp.referred_signup_date 
        AND rp.referred_signup_date + INTERVAL '90 days'
    AND t.status = 'completed'
  GROUP BY t.customer_id
),
referrer_performance AS (
  SELECT 
    rp.referrer_customer_id,
    rp.referrer_name,
    rp.referrer_acquisition_channel,
    COUNT(DISTINCT rp.referred_customer_id) AS total_referrals,
    
    -- Quality of referred customers
    AVG(COALESCE(rcp.txn_count_first_90d, 0)) AS avg_referred_txn_count,
    AVG(COALESCE(rcp.total_spend_first_90d, 0)) AS avg_referred_spend,
    
    -- Retention of referred customers
    COUNT(CASE WHEN COALESCE(rcp.active_months_first_90d, 0) >= 2 THEN 1 END)::DECIMAL 
      / NULLIF(COUNT(DISTINCT rp.referred_customer_id), 0) * 100 AS referred_retention_rate,
    
    -- Rewards earned
    SUM(rp.reward_amount) AS total_rewards_earned,
    
    -- Referrer's own performance
    COUNT(DISTINCT t.transaction_id) AS referrer_own_txn_count,
    SUM(COALESCE(t.amount, 0)) AS referrer_own_total_spend
  FROM referral_pairs rp
  LEFT JOIN referred_customer_performance rcp 
    ON rp.referred_customer_id = rcp.referred_customer_id
  LEFT JOIN transactions t 
    ON rp.referrer_customer_id = t.customer_id 
    AND t.status = 'completed'
  GROUP BY rp.referrer_customer_id, rp.referrer_name, rp.referrer_acquisition_channel
),
referrer_segments AS (
  SELECT 
    *,
    CASE 
      WHEN total_referrals >= 5 AND avg_referred_spend > 5000 
        THEN 'Super Referrer - High Value'
      WHEN total_referrals >= 5 AND avg_referred_spend BETWEEN 2000 AND 5000 
        THEN 'Super Referrer - Medium Value'
      WHEN total_referrals >= 3 AND referred_retention_rate > 60 
        THEN 'Quality Referrer - High Retention'
      WHEN total_referrals >= 3 
        THEN 'Active Referrer'
      WHEN total_referrals BETWEEN 1 AND 2 
        THEN 'Occasional Referrer'
      ELSE 'Single Referrer'
    END AS referrer_segment
  FROM referrer_performance
)
SELECT 
  referrer_segment,
  COUNT(*) AS referrer_count,
  SUM(total_referrals) AS total_referrals,
  ROUND(AVG(avg_referred_txn_count), 1) AS avg_txn_per_referred_customer,
  ROUND(AVG(avg_referred_spend), 2) AS avg_spend_per_referred_customer,
  ROUND(AVG(referred_retention_rate), 1) AS avg_retention_rate,
  ROUND(SUM(total_rewards_earned), 2) AS total_rewards_paid,
  ROUND(AVG(referrer_own_txn_count), 1) AS avg_referrer_own_txn_count,
  ROUND(AVG(referrer_own_total_spend), 2) AS avg_referrer_own_spend
FROM referrer_segments
GROUP BY referrer_segment
ORDER BY total_referrals DESC;

**Explanation:**

"This analysis reveals that not all referrals are equal. 'Super Referrers' who bring 5+ customers generating high spend are gold—Slice should give them VIP treatment and higher rewards. The retention rate metric is crucial: if referred customers churn quickly, the referral program isn't building lasting value. I also compare the referrer's own behavior—typically, engaged customers refer engaged customers. Slice can use this to identify potential super referrers early and incentivize them proactively."

---

### Question 9: Transaction Decline Analysis - Why Are Transactions Failing?

**Interviewer:** We're seeing increased transaction declines. Can you analyze decline patterns to identify the root causes?

**You:** Important operational question! Should I break down declines by decline reason, merchant category, and customer segments?

**Interviewer:** Yes, exactly. We need to understand if it's insufficient limit, technical issues, or fraud blocks.

**You:** Perfect. Should I also look at time-based patterns—are declines higher during certain hours or days?

**Interviewer:** Good thinking—yes, include temporal patterns and compare decline rates across customer credit score bands.

**You:** Got it. Should I also analyze if customers successfully transact elsewhere after a decline, or do they stop trying?

**Interviewer:** Excellent point—yes, include retry behavior and customer loss from declines.

**Solution:**

WITH declined_transactions AS (
  SELECT 
    t.transaction_id,
    t.customer_id,
    t.transaction_date,
    t.transaction_timestamp,
    t.amount,
    t.merchant_category,
    t.decline_reason,
    c.credit_score,
    c.credit_limit,
    c.current_outstanding,
    
    -- Time dimensions
    EXTRACT(DOW FROM t.transaction_timestamp) AS day_of_week,
    EXTRACT(HOUR FROM t.transaction_timestamp) AS hour_of_day,
    
    -- Available credit at decline time
    (c.credit_limit - c.current_outstanding) AS available_credit
  FROM transactions t
  JOIN customers c ON t.customer_id = c.customer_id
  WHERE t.status = 'declined'
    AND t.transaction_date >= CURRENT_DATE - INTERVAL '30 days'
),
successful_transactions AS (
  SELECT 
    customer_id,
    COUNT(*) AS successful_txn_count
  FROM transactions
  WHERE status = 'completed'
    AND transaction_date >= CURRENT_DATE - INTERVAL '30 days'
  GROUP BY customer_id
),
customer_retry_behavior AS (
  SELECT 
    dt.customer_id,
    COUNT(DISTINCT dt.transaction_id) AS decline_count,
    COALESCE(st.successful_txn_count, 0) AS successful_after_decline_count,
    CASE 
      WHEN COALESCE(st.successful_txn_count, 0) = 0 THEN 'Lost - No Retry'
      WHEN COALESCE(st.successful_txn_count, 0) > 0 THEN 'Recovered - Retried'
      ELSE 'Unknown'
    END AS customer_outcome
  FROM declined_transactions dt
  LEFT JOIN successful_transactions st ON dt.customer_id = st.customer_id
  GROUP BY dt.customer_id, st.successful_txn_count
),
decline_analysis AS (
  SELECT 
    dt.decline_reason,
    dt.merchant_category,
    CASE 
      WHEN dt.credit_score >= 750 THEN 'Excellent (750+)'
      WHEN dt.credit_score BETWEEN 700 AND 749 THEN 'Good (700-749)'
      WHEN dt.credit_score BETWEEN 650 AND 699 THEN 'Fair (650-699)'
      ELSE 'Poor (<650)'
    END AS credit_score_band,
    CASE 
      WHEN dt.day_of_week IN (0, 6) THEN 'Weekend'
      ELSE 'Weekday'
    END AS day_type,
    CASE 
      WHEN dt.hour_of_day BETWEEN 9 AND 17 THEN 'Business Hours'
      WHEN dt.hour_of_day BETWEEN 18 AND 22 THEN 'Evening'
      ELSE 'Night/Early Morning'
    END AS time_period,
    
    COUNT(*) AS decline_count,
    AVG(dt.amount) AS avg_declined_amount,
    AVG(dt.available_credit) AS avg_available_credit,
    
    -- Was decline justified?
    COUNT(CASE WHEN dt.amount > dt.available_credit THEN 1 END) AS justified_insufficient_limit,
    COUNT(CASE WHEN dt.amount <= dt.available_credit AND dt.decline_reason = 'insufficient_limit' 
          THEN 1 END) AS unjustified_insufficient_limit
  FROM declined_transactions dt
  GROUP BY 
    dt.decline_reason,
    dt.merchant_category,
    credit_score_band,
    day_type,
    time_period
)
SELECT 
  decline_reason,
  COUNT(*) AS total_declines,
  ROUND((COUNT(*)::DECIMAL / SUM(COUNT(*)) OVER ()) * 100, 2) AS pct_of_total_declines,
  STRING_AGG(DISTINCT merchant_category, ', ') AS top_affected_categories,
  ROUND(AVG(avg_declined_amount), 2) AS avg_amount,
  SUM(justified_insufficient_limit) AS justified_count,
  SUM(unjustified_insufficient_limit) AS unjustified_count
FROM decline_analysis
GROUP BY decline_reason
ORDER BY total_declines DESC;

**Explanation:**

"This decline analysis helps Slice pinpoint operational issues. If 'insufficient_limit' is the top reason but many are unjustified (amount < available credit), there's a system bug. If 'suspected_fraud' spikes for excellent credit score customers, the fraud model is over-blocking. The time-based patterns might reveal infrastructure issues—if declines spike during business hours, payment gateway capacity might be insufficient. The retry behavior metric is critical: customers who stop trying after declines represent lost revenue. This analysis directly informs product and engineering priorities."

---

### Question 10: Credit Limit Optimization - Who Deserves Limit Increases?

**Interviewer:** We want to proactively increase credit limits for deserving customers. Can you identify customers who should get limit increases?

**You:** Excellent growth opportunity! Should I look at payment history, spending patterns, and credit utilization to identify candidates?

**Interviewer:** Yes, exactly. We want customers who are creditworthy and would actually use the additional limit.

**You:** Perfect. What criteria define "creditworthy"? For example, no late payments in last 6 months and credit utilization above 70%?

**Interviewer:** Yes, clean payment history is essential. Also check if they're bumping against their current limit frequently—that indicates demand for higher limits.

**You:** Got it. Should I suggest specific increase amounts, or just flag candidates?

**Interviewer:** Suggest specific amounts based on their usage patterns and risk profile.

**Solution:**

WITH customer_credit_behavior AS (
  SELECT 
    c.customer_id,
    c.name,
    c.credit_score,
    c.credit_limit,
    c.current_outstanding,
    c.signup_date,
    
    -- Credit utilization
    ROUND((c.current_outstanding / NULLIF(c.credit_limit, 0)) * 100, 2) 
      AS current_utilization_pct,
    
    -- Transaction metrics (last 90 days)
    COUNT(DISTINCT CASE WHEN t.transaction_date >= CURRENT_DATE - INTERVAL '90 days' 
          THEN t.transaction_id END) AS txn_count_90d,
    SUM(CASE WHEN t.transaction_date >= CURRENT_DATE - INTERVAL '90 days' 
        THEN t.amount ELSE 0 END) AS total_spend_90d,
    AVG(CASE WHEN t.transaction_date >= CURRENT_DATE - INTERVAL '90 days' 
        THEN t.amount END) AS avg_txn_amount_90d,
    
    -- How often do they max out?
    COUNT(CASE WHEN t.transaction_date >= CURRENT_DATE - INTERVAL '90 days' 
               AND t.status = 'declined' 
               AND t.decline_reason = 'insufficient_limit' 
          THEN 1 END) AS limit_decline_count_90d
  FROM customers c
  LEFT JOIN transactions t ON c.customer_id = t.customer_id
  GROUP BY c.customer_id, c.name, c.credit_score, c.credit_limit, 
           c.current_outstanding, c.signup_date
),
payment_history_check AS (
  SELECT 
    b.customer_id,
    COUNT(*) AS total_bills_6m,
    COUNT(CASE WHEN b.payment_date <= b.due_date THEN 1 END) AS on_time_payments,
    COUNT(CASE WHEN b.days_overdue >= 30 THEN 1 END) AS severe_late_payments,
    SUM(b.late_fee) AS total_late_fees_6m
  FROM bills b
  WHERE b.bill_date >= CURRENT_DATE - INTERVAL '6 months'
  GROUP BY b.customer_id
),
credit_limit_candidates AS (
  SELECT 
    ccb.*,
    COALESCE(phc.on_time_payments, 0) AS on_time_payments,
    COALESCE(phc.total_bills_6m, 0) AS total_bills_6m,
    COALESCE(phc.severe_late_payments, 0) AS severe_late_payments,
    COALESCE(phc.total_late_fees_6m, 0) AS total_late_fees_6m,
    
    -- Clean payment rate
    ROUND(
      (COALESCE(phc.on_time_payments, 0)::DECIMAL / NULLIF(phc.total_bills_6m, 0)) * 100, 
      1
    ) AS on_time_payment_rate,
    
    -- Risk score (custom)
    CASE 
      WHEN ccb.credit_score >= 750 
           AND COALESCE(phc.severe_late_payments, 0) = 0 
           AND ccb.current_utilization_pct < 90 
        THEN 'Low Risk'
      
      WHEN ccb.credit_score BETWEEN 700 AND 749 
           AND COALESCE(phc.severe_late_payments, 0) = 0 
        THEN 'Medium Risk'
      
      ELSE 'High Risk'
    END AS risk_category,
    
    -- Suggested limit increase
    CASE 
      WHEN ccb.credit_score >= 750 
           AND ccb.current_utilization_pct >= 70 
           AND COALESCE(phc.on_time_payments, 0) = COALESCE(phc.total_bills_6m, 0)
           AND ccb.limit_decline_count_90d >= 2
        THEN ROUND(ccb.credit_limit * 1.5, -3)  -- 50% increase
      
      WHEN ccb.credit_score >= 720 
           AND ccb.current_utilization_pct >= 60 
           AND COALESCE(phc.on_time_payment_rate, 0) >= 90
        THEN ROUND(ccb.credit_limit * 1.3, -3)  -- 30% increase
      
      WHEN ccb.credit_score >= 700 
           AND ccb.current_utilization_pct >= 50 
           AND COALESCE(phc.on_time_payment_rate, 0) >= 80
        THEN ROUND(ccb.credit_limit * 1.2, -3)  -- 20% increase
      
      ELSE ccb.credit_limit  -- No increase
    END AS suggested_new_limit
  FROM customer_credit_behavior ccb
  LEFT JOIN payment_history_check phc ON ccb.customer_id = phc.customer_id
)
SELECT 
  customer_id,
  name,
  credit_score,
  credit_limit AS current_limit,
  suggested_new_limit,
  (suggested_new_limit - credit_limit) AS limit_increase_amount,
  ROUND(((suggested_new_limit - credit_limit)::DECIMAL / credit_limit) * 100, 1) 
    AS increase_percentage,
  current_utilization_pct,
  txn_count_90d,
  ROUND(total_spend_90d, 2) AS total_spend_90d,
  limit_decline_count_90d,
  on_time_payment_rate,
  severe_late_payments,
  risk_category
FROM credit_limit_candidates
WHERE suggested_new_limit > credit_limit  -- Only show increase candidates
  AND risk_category IN ('Low Risk', 'Medium Risk')  -- Exclude high risk
ORDER BY limit_increase_amount DESC
LIMIT 100;

**Explanation:**

"This query identifies customers who deserve credit limit increases based on proven creditworthiness and demonstrated need. The scoring considers multiple dimensions: credit score (external validation), payment history (Slice-specific behavior), utilization (need for higher limit), and decline frequency (blocked demand). The suggested increases are tiered: 50% for excellent customers with perfect payment history and high utilization, down to 20% for good customers. This proactive approach increases revenue (more spending capacity) while maintaining credit quality. Slice can automate these increases or use them for manual review."