# SqlCode
This  is an example of my sql coding structure

WITH 
--Calculate Recency, Frequency and Monetary score
t1 AS (
    SELECT 
        CustomerID, 
        Country,
        DATE_DIFF('2011-12-01', CAST(MAX(InvoiceDate) AS DATE), DAY) AS recency,
        COUNT(DISTINCT InvoiceNo) AS frequency,
        ROUND(SUM(UnitPrice*Quantity),0) AS monetary
    FROM 
        turing_data_analytics.rfm
    WHERE  CustomerID IS NOT NULL 
      AND UnitPrice>0 
      AND Quantity>0 
      AND CAST(InvoiceDate AS DATE ) BETWEEN '2010-12-01' AND '2011-12-01'

    GROUP BY 
        CustomerID, Country

),
--Calculate Recency, Frequency and Monetary percentiles
t2 AS (
    SELECT *,
       PERCENTILE_CONT(recency, 0.25) OVER() AS r25,
        PERCENTILE_CONT(recency, 0.5) OVER() AS r50,
        PERCENTILE_CONT(recency, 0.75) OVER() AS r75,
        PERCENTILE_CONT(recency, 1) OVER() AS r100,
        PERCENTILE_CONT(frequency, 0.25) OVER() AS f25,
        PERCENTILE_CONT(frequency, 0.5) OVER() AS f50,
        PERCENTILE_CONT(frequency, 0.75) OVER() AS f75,
        PERCENTILE_CONT(frequency, 1) OVER() AS f100,
        PERCENTILE_CONT(monetary, 0.25) OVER() AS m25,
        PERCENTILE_CONT(monetary, 0.5) OVER() AS m50,
        PERCENTILE_CONT(monetary, 0.75) OVER() AS m75,
        PERCENTILE_CONT(monetary, 1) OVER() AS m100
    FROM 
        t1
),
--Calculate RFM scores
t3 AS (
    SELECT 
        *,
        CASE 
            WHEN recency <= r25 THEN 4 
            WHEN recency <= r50 AND recency > r25 THEN 3 
            WHEN recency <= r75 AND recency > r50 THEN 2 
            WHEN recency <= r100 AND recency > r75 THEN 1 
        END AS r_score,
        CASE 
            WHEN monetary <= m25 THEN 1
            WHEN monetary <= m50 AND monetary > m25 THEN 2 
            WHEN monetary <= m75 AND monetary > m50 THEN 3 
            WHEN monetary <= m100 AND monetary > m75 THEN 4 
        END AS m_score,
        CASE 
            WHEN frequency <= f25 THEN 1
            WHEN frequency <= f50 AND frequency > f25 THEN 2 
            WHEN frequency <= f75 AND frequency > f50 THEN 3 
            WHEN frequency <= f100 AND frequency > f75 THEN 4 
        END AS f_score
    FROM 
        t2
),
t4 as (
  SELECT *,
  CAST (ROUND ((f_score +m_score)/2 ) AS INT64) as fm_score
  FROM t3
),
--Define RFM segments
t5 AS (
    SELECT 
        CustomerID, 
        Country,
        recency,
        frequency, 
        monetary,
        r_score,
        f_score,
        m_score,
        fm_score,
CASE WHEN (r_score = 4 AND fm_score =4 ) 
OR (r_score = 4 AND fm_score = 3) 
THEN 'Champions'
WHEN (r_score = 4 AND fm_score = 2) 
OR (r_score = 4 AND fm_score = 2)
OR (r_score = 3 AND fm_score = 3)
OR (r_score = 4 AND fm_score = 3)
THEN 'Potential Loyalists'
WHEN r_score = 4 AND fm_score = 1 THEN 'Recent Customers'
WHEN (r_score = 4 AND fm_score = 1) 
OR (r_score = 3 AND fm_score = 1)
THEN 'Promising'
WHEN (r_score = 3 AND fm_score = 2) 
OR (r_score = 2 AND fm_score = 3)
OR (r_score = 2 AND fm_score = 2)
THEN 'Customers Needing Attention'
WHEN r_score = 2 AND fm_score = 1 THEN 'About to Sleep'
WHEN (r_score = 2 AND fm_score = 4) 
OR (r_score = 2 AND fm_score = 4)
OR (r_score = 1 AND fm_score = 3)
THEN 'Cant lose them'
WHEN (r_score = 1 AND fm_score = 4)
OR (r_score = 1 AND fm_score = 4) 
THEN 'At Risk'
WHEN r_score = 1 AND fm_score = 2 THEN 'Hibernating'
WHEN r_score = 1 AND fm_score = 1 THEN 'Lost'
END AS rfm_segment 

    FROM t4
)

SELECT *
 FROM t5

