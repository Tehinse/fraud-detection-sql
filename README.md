# fraud-detection-sql
SELECT 
    t.*,
    (CASE WHEN amount > 1000000 THEN 50 ELSE 0 END) +
    (CASE WHEN device_type = 'Web' THEN 20 ELSE 0 END) +
    (CASE 
         WHEN location <> LAG(location) OVER (PARTITION BY customer_id ORDER BY transaction_date)
         THEN 30 ELSE 0 END) +
    (CASE 
         WHEN transaction_id IN (
             SELECT t1.transaction_id
             FROM transactions t1
             JOIN transactions t2 
               ON t1.customer_id = t2.customer_id
              AND t2.transaction_date > t1.transaction_date
             GROUP BY t1.transaction_id
             HAVING COUNT(*) > 3
         ) THEN 40 ELSE 0 END) +
    (CASE WHEN fraud_flag = 1 THEN 100 ELSE 0 END)
    AS fraud_risk_score
FROM transactions t;

SELECT 
    customer_id,
    COUNT(*) AS total_transactions,
    SUM(fraud_flag) AS total_frauds,
    AVG(fraud_risk_score) AS avg_risk_score,
    MAX(fraud_risk_score) AS highest_risk_transaction,
    ROUND(100.0 * SUM(fraud_flag) / COUNT(*), 2) AS fraud_rate_percentage
FROM (
    -- Reuse the query above as a subquery
    SELECT 
        t.*,
        (CASE WHEN amount > 1000000 THEN 50 ELSE 0 END) +
        (CASE WHEN device_type = 'Web' THEN 20 ELSE 0 END) +
        (CASE 
             WHEN location <> LAG(location) OVER (PARTITION BY customer_id ORDER BY transaction_date)
             THEN 30 ELSE 0 END) +
        (CASE 
             WHEN transaction_id IN (
                 SELECT t1.transaction_id
                 FROM transactions t1
                 JOIN transactions t2 
                   ON t1.customer_id = t2.customer_id
                  AND t2.transaction_date > t1.transaction_date
                 GROUP BY t1.transaction_id
                 HAVING COUNT(*) > 3
             ) THEN 40 ELSE 0 END) +
        (CASE WHEN fraud_flag = 1 THEN 100 ELSE 0 END)
        AS fraud_risk_score
    FROM transactions t
) scored
GROUP BY customer_id
ORDER BY avg_risk_score DESC;
