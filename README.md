# SQL-code
This code was created to extract information about customers from the dataset of a financial service to answer specific questions

Database of payment transactions from 2019 to 2020 of Paytm Wallet. The database
includes 6 tables:

● fact_transaction: Store information on all types of transactions: Payments, Top-up, Transfers,
Withdrawals

● dim_scenario: Detailed description of transaction types

● dim_payment_channel: Detailed description of payment methods

● dim_platform: Detailed description of payment devices

● dim_status: Detailed description of the results of the transaction

1. What is the trend of the number of successful payment transactions with promotion
(promtion_trans) on a weekly basis and account for how much of the total number of successful
payment transactions (promotion_ratio)?

Note: promotion transactions have promotion_id <> 0

SQL-CODE:

WITH elec_trans_20 AS (

SELECT transaction_id, status_id, promotion_id, trans_20.scenario_id AS scena_id, sub_category, transaction_time

, DATEPART(week, transaction_time) AS week

, COUNT(transaction_id) OVER (PARTITION BY DATEPART(week, transaction_time)) AS total_trans

FROM fact_transaction_2020 AS trans_20

LEFT JOIN dim_scenario AS scen

ON trans_20.scenario_id = scen.scenario_id

WHERE sub_category = 'Electricity' AND status_id = 1

)
, promotion_trans_trend AS (

SELECT DISTINCT week, COUNT(transaction_id) OVER (PARTITION BY week) AS number_promotion_trans, total_trans

FROM elec_trans_20

WHERE promotion_id <> '0' 

)
SELECT *, FORMAT(number_promotion_trans*1.0/ total_trans, 'p') AS promotion_ptg

FROM promotion_trans_trend

ORDER BY week ASC

2. Out of the total number of successful paying customers enjoying the promotion, how
many % of customers have incurred any other successful payment transactions that are not
promotional transactions?

SQL-CODE

with table_cus AS (

select customer_id, transaction_id, promotion_id

, IIF(promotion_id <> '0' , 'is-promo', 'non-promo') as trans_type

, LAG (IIF(promotion_id <> '0' , 'is-promo', 'non-promo'), 1) OVER ( partition by customer_id order by transaction_id) as previous_tran

, ROW_NUMBER() over ( partition by customer_id order by transaction_id) AS rn

from fact_transaction_2020 as fact_20

join dim_scenario as scen

on fact_20.scenario_id = scen.scenario_id

where status_id = '1' and sub_category = 'electricity'

-- order by customer_id, transaction_id

)
, table_first_promo AS (

SELECT DISTINCT customer_id

FROM table_cus

WHERE rn = 1 AND trans_type = 'is-promo'

)

SELECT COUNT (distinct table_first_promo.customer_id) AS number_customer

, (SELECT COUNT (distinct customer_id ) from table_first_promo ) AS total

, COUNT (distinct table_first_promo.customer_id)*1.0 / (SELECT COUNT (distinct customer_id ) from table_first_promo )

FROM table_first_promo 

JOIN table_cus

ON table_first_promo.customer_id = table_cus.customer_id

WHERE trans_type = 'non-promo' AND previous_tran = 'is-promo'
