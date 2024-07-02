# 💸 Case Study #4 - Data Bank
## Task :
This Case study of an initiative that combines banking and cloud data storage. Focuses on exploring how this unique model works, how the business is doing, and using data to forecast for future data storage requirements and developments. View the Case Study [Link](https://8weeksqlchallenge.com/case-study-4/)


## 📝 Solution A. Customer Nodes Exploration

### 1. How many unique nodes are there on the Data Bank system?

````sql
SELECT 
  COUNT(DISTINCT node_id) AS unique_node_count
FROM customer_nodes;
````
**Answer:**

![1 CS 4](https://user-images.githubusercontent.com/96012488/200128877-cb0f5d02-95c1-4611-bb43-075b388541f9.png)

- There are 5 unique nodes.



### 2. What is the number of nodes per region?

````sql
SELECT 
  region_id,COUNT(DISTINCT node_id) node_count
FROM customer_nodes
GROUP BY region_id;
 ````
 
 **Answer:**
 
 ![2 CS 4](https://user-images.githubusercontent.com/96012488/200128911-7a6d64d1-cf52-4b92-97ed-dc652d189f46.png)

- Each region has 5 unique nodes.
 
 ### 3. How many customers are allocated to each region?
 
 ````sql
 SELECT 
  n.region_id,r.region_name,COUNT(DISTINCT customer_id) AS customer_count
 FROM customer_nodes n
 JOIN regions r
 ON n.region_id=r.region_id
 GROUP BY n.region_id,r.region_name
 ORDER BY n.region_id;
 ````
 
 **Answer:**
 
 ![3 CS 4](https://user-images.githubusercontent.com/96012488/200128988-2bd1b62c-f8c5-4302-bf50-eb6920814dc7.png)


 
 ### 4. How many days on average are customers reallocated to a different node?
 
 ````sql
 SELECT 
  AVG(DATEDIFF(day,start_date,end_date)) avg_reallocation_days
 FROM customer_nodes
 WHERE end_date NOT LIKE '9999-%';
 ````
 
 **Answer:**
 
 ![4 CS 4](https://user-images.githubusercontent.com/96012488/200129014-67030e60-2216-4fad-ade5-224bc8e6450d.png)


 ### 5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
 
 ````sql
 WITH diff_cte AS
 (
 SELECT 
  n.region_id,r.region_name,customer_id,DATEDIFF(day,start_date,end_date) AS reallocation_days
 FROM customer_nodes n
 JOIN regions r
 ON n.region_id=r.region_id
 WHERE end_date NOT LIKE '9999-%'
 )
 SELECT DISTINCT 
      region_id,region_name,
	  PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY reallocation_days) OVER( PARTITION BY region_id) AS median,
	  PERCENTILE_DISC(0.80) WITHIN GROUP (ORDER BY reallocation_days) OVER( PARTITION BY region_id) AS percentile_80th,
	  PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY reallocation_days) OVER( PARTITION BY region_id) AS percentile_95th
 FROM diff_cte;
 ````
 
 **Answer:**
 
 ![5 CS 4](https://user-images.githubusercontent.com/96012488/200129032-9cfdd683-166e-4d20-93b5-2a693a72fff3.png)
 
 - The median and the 90th percentile of reallocation days is 15 and 28 respectively for each region.
 - The 80th percentile of reallocation days is 23 for Africa and Europe while 24 for Asia, America and Australia.

---------------------------------------------------------------------------------------------------------------------------------------------

## 📝 Solution B. Customer Transactions

### 1. What is the unique count and total amount for each transaction type?

````sql
SELECT 
  txn_type,COUNT(txn_type) AS unique_count,SUM(txn_amount) AS total_amount
FROM customer_transactions
GROUP BY txn_type;
````

**Answer:**

![cs 4 1](https://user-images.githubusercontent.com/96012488/200153699-11ce1fd0-51e6-4ced-9244-46b891e209c0.png)



### 2. What is the average total historical deposit counts and amounts for all customers?

````sql
WITH deposit_cte AS
 (
  SELECT customer_id,COUNT(txn_type) AS deposit_count,SUM(txn_amount) AS deposit_amount
  FROM customer_transactions
  WHERE txn_type = 'deposit'
  GROUP BY customer_id
 )
 SELECT
  AVG(deposit_count) AS avg_deposit_count,AVG(deposit_amount) AS avg_total_amount_deposited
 FROM deposit_cte;
 ````
 
 **Answer:**
 
 ![Screenshot 2022-11-06 092833](https://user-images.githubusercontent.com/96012488/200153705-1d7d1b89-6ce0-4b67-9117-2257b230c03c.png)

 
 ### 3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
 
 ````sql
 WITH months_cte AS
 (
  SELECT customer_id, DATENAME(month,txn_date) AS txn_month,DATEPART(month,txn_date) AS txn_month_id
  FROM customer_transactions
 )
 ,counts_cte AS
 (
  SELECT t.customer_id,txn_month,txn_month_id,SUM(CASE WHEN txn_type='deposit' THEN 1 ELSE 0 END) AS d,
         SUM(CASE WHEN txn_type='purchase' THEN 1 ELSE 0 END) AS p,
		SUM(CASE WHEN txn_type='withdrawal' THEN 1 ELSE 0 END) AS w
  FROM customer_transactions t
  JOIN months_cte
  ON months_cte.customer_id=t.customer_id
  GROUP BY t.customer_id,txn_month,txn_month_id
 )
    SELECT 
       txn_month,COUNT(customer_id) AS customer_count
    FROM counts_cte
    WHERE d>1 AND (p=1 OR w=1)
    GROUP BY txn_month,txn_month_id
    ORDER BY txn_month_id;
 ````
   
   **Answer:**
   
   ![Screenshot 2022-11-06 093022](https://user-images.githubusercontent.com/96012488/200153709-ff4ae689-2206-47f6-b759-295e2fba584d.png)

   
   ### 4. What is the closing balance for each customer at the end of the month?
   
   ````sql
   WITH amounts_cte AS
  (
    SELECT t.customer_id, DATENAME(month,txn_date) AS txn_month,DATEPART(month,txn_date) AS txn_month_id,
		SUM(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END )AS transaction_amount
    FROM customer_transactions t
    GROUP BY customer_id,DATENAME(month,txn_date),DATEPART(month,txn_date)
   )
      SELECT 
        customer_id,txn_month,
        SUM(transaction_amount) 
	    OVER (PARTITION BY customer_id ORDER BY txn_month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW ) AS closing_balance
      FROM amounts_cte
      ORDER  BY customer_id,txn_month_id;
 ````
 
 **Answer:**
 
 ![Screenshot 2022-11-06 093249](https://user-images.githubusercontent.com/96012488/200153717-3fbfb8d3-839d-4e3f-b8a3-2abbb296df5f.png)

-------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Solution C. Data Allocation Challenge

Data bank has 3 different options to allocate data storage space to its customers.

- **Option 1:** data is allocated based off the amount of money at the end of the previous month

- **Option 2:** data is allocated on the average amount of money kept in the account in the previous 30 days

- **Option 3:** data is updated real-time

### :dart: Goal: To estimate Data Storage Space that will need to be provisioned per month in each option

### Requirements

Now, since the Data Bank team wants to estimate how much data storage space will need to be provisioned per month for each option, it requires the following data elements to help it with the estimation.

- running customer balance column that includes the impact each transaction

````sql
WITH cte AS
 (
   SELECT t.customer_id,txn_date, DATENAME(month,txn_date) AS txn_month,txn_type,
	  (CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END )AS transaction_amount
   FROM customer_transactions t
 )
    SELECT *,
      SUM(transaction_amount) 
      OVER (PARTITION BY customer_id ORDER BY txn_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
    FROM cte
    ORDER BY customer_id;
 ````
 **Output:**
 
 ![Screenshot 2022-11-06 140401](https://user-images.githubusercontent.com/96012488/200161529-b75eeaf0-a771-40e1-90f0-acbb132bd0e7.png)
 
 - customer balance at the end of each month

````sql
WITH amounts_cte AS
(
  SELECT t.customer_id, DATENAME(month,txn_date) AS txn_month,DATEPART(month,txn_date) AS txn_month_id,
         SUM(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END )AS transaction_amount
  FROM customer_transactions t
  GROUP BY customer_id,DATENAME(month,txn_date),DATEPART(month,txn_date)
)
  SELECT customer_id,txn_month,
	 SUM(transaction_amount) 
         OVER (PARTITION BY customer_id ORDER BY txn_month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW ) AS closing_balance
  FROM amounts_cte
  ORDER BY customer_id,txn_month_id;
 ````
 
 **Output:**
 
 I have shown the output for a few customers to save space here.
 
 ![Screenshot 2022-11-06 141610](https://user-images.githubusercontent.com/96012488/200161957-aa6d938f-fa44-4c0e-8e4f-897dd3633c03.png)
 

- minimum, average and maximum values of the running balance for each customer

````sql
WITH txnamount_cte AS
 (
   SELECT t.customer_id,txn_date, DATENAME(month,txn_date) AS txn_month,txn_type,
          CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END )AS transaction_amount
   FROM customer_transactions t
 )
 , running_balance_cte AS
 (
   SELECT *, 
          SUM(transaction_amount) 
          OVER (PARTITION BY customer_id ORDER BY txn_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
   FROM txnamount_cte
 )
    SELECT 
       customer_id, MIN(running_balance) AS min_balance, AVG(running_balance) AS avg_balance, MAX(running_balance) AS max_balance
    FROM running_balance_cte
    GROUP BY customer_id
    ORDER BY customer_id;
 ````
 
 **Output:**
 
 Again, I have shown output for just 20 customers to save up the space.
 
 ![Screenshot 2022-11-06 142223](https://user-images.githubusercontent.com/96012488/200162144-538cfd76-35d4-49e3-a91e-fe378888b667.png)
 
 
 ### Data Storage Space Per Month
 
 Now, the amount of Data Storage Space required per month in each of the 3 options are as follows:  
 
 - **Option 1** 
 
 **Data_Allocated per customer per month based on the closing balance of previous month**
 
 
 ````sql
 WITH monthlyamounts_cte AS
 (
  SELECT t.customer_id, DATENAME(month,txn_date) AS txn_month, DATEPART(month,txn_date) AS txn_month_id,
	 	SUM(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END )AS transaction_amount
  FROM customer_transactions t
  GROUP BY customer_id,DATENAME(month,txn_date),DATEPART(month,txn_date)
 )
 ,closing_balance_cte AS
 (
  SELECT customer_id,txn_month,txn_month_id,
	 	SUM(transaction_amount) 
	 	OVER (PARTITION BY customer_id ORDER BY txn_month_id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW ) AS closing_balance
  FROM monthlyamounts_cte
 )
 ,prev_closing_balance_cte AS
 (
  SELECT *, 
 	LAG(closing_balance) OVER(PARTITION BY customer_id ORDER BY txn_month_id) AS prev_closing_balance
  FROM closing_balance_cte
 )
   SELECT *, 
	   (CASE WHEN prev_closing_balance >= 0 THEN prev_closing_balance ELSE 0 END) AS data_allocated
   FROM prev_closing_balance_cte;
   ````
   
   ![Screenshot 2022-11-06 161202](https://user-images.githubusercontent.com/96012488/200166205-dfbd4917-1249-4724-8a58-bcf6cc81ede6.png)


:arrow_right: **Total Data required per month**

````sql
WITH monthlyamounts_cte AS
(
  SELECT t.customer_id, DATENAME(month,txn_date) AS txn_month,DATEPART(month,txn_date) AS txn_month_id,
	SUM(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END )AS transaction_amount
  FROM customer_transactions t
  GROUP BY customer_id,DATENAME(month,txn_date),DATEPART(month,txn_date)
 )
 , closing_balance_cte AS
 (
  SELECT customer_id,txn_month,txn_month_id,
	SUM(transaction_amount) 
	OVER (PARTITION BY customer_id ORDER BY txn_month_id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW ) AS closing_balance
  FROM monthlyamounts_cte
 )
 , prev_closing_balance_cte AS
 (
  SELECT *, 
  	LAG(closing_balance) OVER(PARTITION BY customer_id ORDER BY txn_month_id) AS prev_closing_balance
  FROM closing_balance_cte
 )
 ,data_cte AS
 (
  SELECT *,
  	CASE WHEN prev_closing_balance >= 0 THEN prev_closing_balance ELSE 0 END AS data_allocated
  FROM prev_closing_balance_cte
 )
  SELECT 
  	txn_month,txn_month_id,SUM(data_allocated) AS total_data_required
  FROM data_cte
  GROUP BY txn_month,txn_month_id
  ORDER BY txn_month_id;
````
![Screenshot 2022-11-06 161439](https://user-images.githubusercontent.com/96012488/200166326-6bc9a7c7-9766-4074-a246-4b376d2ff4fc.png)

**Note:** Since there is no previous closing balance for January, data required for this month is 0.

- **Option 2**

**Data_Allocated per customer per month based on average running balance of the previous 30 days**

````sql
WITH txnamount_cte AS
 (
  SELECT t.customer_id,txn_date, DATENAME(month,txn_date) AS txn_month,txn_type,DATEPART(month,txn_date) AS txn_month_id,
	(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END ) AS transaction_amount
  FROM customer_transactions t
 )
 , running_balance_cte AS
 (
  SELECT *, 
  	SUM(transaction_amount) 
	OVER (PARTITION BY customer_id ORDER BY txn_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
  FROM txnamount_cte
 )
 , avg_running_balance_cte AS
 (
  SELECT DISTINCT 
  	customer_id,txn_month,txn_month_id,
	AVG(running_balance) 
	OVER(PARTITION BY customer_id,txn_month ORDER BY txn_month) AS avg_running_balance
  FROM running_balance_cte
 )
 , prev_avg_rb_cte AS
 (
  SELECT *, 
 	LAG(avg_running_balance) OVER(PARTITION BY customer_id ORDER BY txn_month_id) AS prev_avg_rb
  FROM avg_running_balance_cte
 )
  SELECT *, 
 	CASE WHEN prev_avg_rb >= 0 THEN prev_avg_rb ELSE 0 END AS data_allocated
  FROM prev_avg_rb_cte
  ORDER BY customer_id;
 ````
![Screenshot 2022-11-06 162551](https://user-images.githubusercontent.com/96012488/200166715-f885ebf9-5d92-435c-8500-c4ccb1bbfe7e.png)

➡️ **Total Data required per month**

````sql
WITH txnamount_cte AS
 (
  SELECT t.customer_id,txn_date, DATENAME(month,txn_date) AS txn_month,txn_type, DATEPART(month,txn_date) AS txn_month_id,
	(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END )AS transaction_amount
  FROM customer_transactions t
 )
 , running_balance_cte AS
 (
  SELECT *, 
   	SUM(transaction_amount) 
	OVER (PARTITION BY customer_id ORDER BY txn_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
  FROM txnamount_cte
 )
 , avg_running_balance_cte AS
 (
  SELECT DISTINCT 
  	customer_id, txn_month, txn_month_id,
  	AVG(running_balance) 
	OVER(PARTITION BY customer_id,txn_month ORDER BY txn_month) AS avg_running_balance
  FROM running_balance_cte
 )
 , prev_avg_rb_cte AS
 (
  SELECT *, 
 	LAG(avg_running_balance) 
	OVER(PARTITION BY customer_id ORDER BY txn_month_id) AS prev_avg_rb
  FROM avg_running_balance_cte
 )
 , data_cte AS
 (
  SELECT *, 
  	CASE WHEN prev_avg_rb >= 0 THEN prev_avg_rb ELSE 0 END AS data_allocated
  FROM prev_avg_rb_cte
 )
  SELECT 
  	txn_month,SUM(data_allocated) AS total_data_required
  FROM data_cte
  GROUP BY txn_month,txn_month_id
  ORDER BY txn_month_id;
````

![Screenshot 2022-11-06 162950](https://user-images.githubusercontent.com/96012488/200166894-6ec4c7d6-a472-496a-b4ad-690b97bb5a1e.png)

**Note:** Since there are no transactions before January, data required for this month is 0.

- **Option 3**

**Data_Allocated per customer per transaction based on the running balance**

````sql
WITH txnamount_cte AS
 (
  SELECT t.customer_id,txn_date, DATENAME(month,txn_date) AS txn_month,txn_type,
	(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END )AS transaction_amount
  FROM customer_transactions t
 )
 , running_balance_cte AS
 (
  SELECT *, 
  	SUM(transaction_amount) 
  	OVER (PARTITION BY customer_id ORDER BY txn_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
  FROM txnamount_cte
 )
  SELECT *, 
  	(CASE WHEN running_balance >= 0 THEN running_balance ELSE 0 END) AS data_allocated
  FROM running_balance_cte
  ORDER BY customer_id;
````

![Screenshot 2022-11-06 164254](https://user-images.githubusercontent.com/96012488/200167363-248a1894-3c9f-4285-9dde-496276e2bcff.png)


➡️ **Total Data required per month**

````sql
 WITH txnamount_cte AS
 (
  SELECT t.customer_id,txn_date,DATEPART(month,txn_date) AS txn_month_id, DATENAME(month,txn_date) AS txn_month,txn_type,
	(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END )AS transaction_amount
  FROM customer_transactions t
 )
 , running_balance_cte AS
 (
  SELECT *, 
  	SUM(transaction_amount) 
	OVER (PARTITION BY customer_id ORDER BY txn_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
  FROM txnamount_cte
 )
 , data_cte AS
 (
  SELECT *, 
  	(CASE WHEN running_balance >= 0 THEN running_balance ELSE 0 END) AS data_allocated
  FROM running_balance_cte
 )
  SELECT 
  	txn_month, SUM(data_allocated) AS total_data_required 
  FROM data_cte
  GROUP BY txn_month,txn_month_id
  ORDER BY txn_month_id;
````

![Screenshot 2022-11-06 164131](https://user-images.githubusercontent.com/96012488/200167319-ef54654b-dfd8-4f61-b19a-91ce1d9d04f2.png)


### Conclusion

➡️ Option 1 and 2 have almost the same data requirements, whereas Option 3 requires much more data to be provisioned in all the months.
 

  
