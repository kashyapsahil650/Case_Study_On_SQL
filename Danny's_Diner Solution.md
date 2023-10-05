# :ramen: :curry: :sushi: Case Study #1: Danny's Diner

## Case Study Questions

1. What is the total amount each customer spent at the restaurant?
2. How many days has each customer visited the restaurant?
3. What was the first item from the menu purchased by each customer?
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
5. Which item was the most popular for each customer?
6. Which item was purchased first by the customer after they became a member?
7. Which item was purchased just before the customer became a member?
8. What is the total items and amount spent for each member before they became a member?

***

###  1. What is the total amount each customer spent at the restaurant?
<details>
  <summary>Click here for the solution</summary>
  
```sql
SELECT s.customer_id AS customer, SUM(m.price) AS spentamount_cust 
FROM sales s 
JOIN menu m 
ON s.product_id = m.product_id 
GROUP BY s.customer_id;
```
</details>

#### Output:
|customer|spentamount_cust|
|--------|----------------|
|A|	76
|B	|74
|C	|36

***

###  2. How many days has each customer visited the restaurant?
<details>
  <summary>Click here for the solution</summary>
  
```sql
SELECT	customer_id, COUNT(DISTINCT order_date) AS Visit_frequency
FROM	sales
GROUP BY customer_id;
```
</details>

#### Output:
|customer_id|	Visit_frequency|
|-----------|----------------|
|A|	4
|B	|6
|C	|2

***

###  3. What was the first item from the menu purchased by each customer?
-- Asssumption: Since the timestamp is missing, all items bought on the first day is considered as the first item(provided multiple items were purchased on the first day)
<details>
  <summary>Click here for the solution</summary>
  
```sql
SELECT	s.customer_id AS customer, m.product_name AS food_item
FROM	sales s
JOIN	menu m
ON		s.product_id = m.product_id
WHERE	s.order_date = (SELECT MIN(order_date) FROM sales WHERE customer_id = s.customer_id)
ORDER BY s.product_id;
```
</details>
	
#### Output:
|customer	|food_item|
|--------|----------|
|A|	sushi
|B	|curry
|C	|ramen

***

###  4. What is the most purchased item on the menu and how many times was it purchased by all customers?
<details>
  <summary>Click here for the solution</summary>
  
```sql
select product_name,count(order_date) as count from sales s join menu m on s.product_id=m.product_id group by product_name order by count desc limit 1;
```
</details>

#### Output:
|product_name|	order_count|
|------------|-------------|
|ramen	|8|
***

###  5. Which item was the most popular for each customer?
-- Asssumption: Products with the highest purchase counts are all considered to be popular for each customer
<details>
  <summary>Click here for the solution</summary>
  
```sql
with t as(select customer_id,product_id,count(order_date) count_order from sales group by  customer_id,product_id),
t2 as(
select t.customer_id,
product_name,count_order,rank() over(partition by t.customer_id order by t.count_order desc) as r from
t join menu m on t.product_id=m.product_id)
select * from t2 where r=1;
```
</details>

#### Output:

| customer_id  | product_name  | order_count  | rank  |
|--------------|---------------|--------------|-------|
| A            | ramen         | 3            | 1     |
| B            | ramen         | 2            | 1     |
| B            | curry         | 2            | 1     |
| B            | sushi         | 2            | 1     |
| C            | ramen         | 3            | 1     |

***

###  6. Which item was purchased first by the customer after they became a member?
-- Before answering question 6, I created a membership_validation table to validate only those customers joining in the membership program:
<details>
  <summary>Click here for the solution</summary>
  
```sql
DROP TABLE #Membership_validation;
CREATE TABLE #Membership_validation
(
   customer_id varchar(1),
   order_date date,
   product_name varchar(5),
   price int,
   join_date date,
   membership varchar(5)
)

INSERT INTO #Membership_validation
SELECT s.customer_id, s.order_date, m.product_name, m.price, mem.join_date,
       CASE WHEN s.order_date >= mem.join_date THEN 'X' ELSE '' END AS membership
FROM sales s
INNER JOIN menu m ON s.product_id = m.product_id
LEFT JOIN members mem ON mem.customer_id = s.customer_id
WHERE join_date IS NOT NULL
ORDER BY customer_id, order_date;

select * from #Membership_validation;
```
</details>

#### Temporary Table {#Membership_validation} Output:

|	customer_id| order_date	   | product_name |	price |	join_date    |membership |
|:------------:|:---------------:|:--------------:|:-------:|:--------------:|:-----------:|
|		A		 |2021-01-01|			sushi|		 10|		2021-01-07|	|
|		A		 |2021-01-01		    |curry		| 15		|2021-01-07	|  |
|		A		 |2021-01-07			|curry		| 15		|2021-01-07	| 	  X |
|	 A		 |2021-01-10			|ramen		| 12		|2021-01-07	|   	X
|		A		 |2021-01-11		|	ramen	    | 12		|2021-01-07	|   	X
|	A		  |2021-01-11		  |	ramen		 |12	|	2021-01-07		|       X
|		B		|2021-01-01		|	curry		| 15	|	2021-01-09	|           |
|		B	  |	2021-01-02		|	curry		| 15	|	2021-01-09	|           |
|		B		|2021-01-04		|	sushi		 |10	|	2021-01-09	|           |
|	B		|2021-01-11		|	sushi		 |10	|	2021-01-09	|	        X  |   
|	B		|2021-01-16		|	ramen		| 12	|	2021-01-09|		        X|
|	B		|2021-02-01		|	ramen		 |12		|2021-01-09	|     	  X|

<details>
  <summary>Click here for the solution</summary>
  
```sql
WITH cte_first_after_mem AS (
  SELECT 
    customer_id,
    product_name,
    order_date,
    RANK() OVER(PARTITION BY customer_id ORDER BY order_date) AS purchase_order
  FROM #Membership_validation
  WHERE membership = 'X'
)
SELECT * FROM cte_first_after_mem
WHERE purchase_order = 1;
```
</details>

#### Output:
|customer_id|	product_name|	order_date|	purchase_order|
|:---------:|:-----------:|:--------:|:-------------:|
|A|	curry|	2021-01-07|	1|
|B	|sushi|	2021-01-11	|1|

***

###  7. Which item was purchased just before the customer became a member?
<details>
  <summary>Click here for the solution</summary>
  
```sql
WITH cte_last_before_mem AS (
  SELECT 
    customer_id,
    product_name,
    order_date,
    RANK() OVER( PARTITION BY customer_id ORDER BY order_date DESC) AS purchase_order
  FROM #Membership_validation
  WHERE membership = ''
)
SELECT * FROM cte_last_before_mem 
WHERE purchase_order = 1;
```
</details>

#### Output:
|customer_id	|product_name|	order_date	|purchase_order|
|:---------:|:------------:|:----------:|:-------------:|
|A	|sushi|	2021-01-01|	1|
|A	|curry|	2021-01-01|	1|
|B|	sushi|	2021-01-04|	1|

***

###  8. What is the total items and amount spent for each member before they became a member?
<details>
  <summary>Click here for the solution</summary>
  
```sql
WITH cte_spent_before_mem AS (
  SELECT 
    customer_id,
    product_name,
    price
  FROM #Membership_validation
  WHERE membership = ''
)
SELECT 
	customer_id,
  SUM(price) AS total_spent,
  COUNT(*) AS total_items
FROM cte_spent_before_mem
GROUP BY customer_id
ORDER BY customer_id;
```
</details>

#### Output:
|customer_id	|total_spent	|total_items|
|:---------:|:------------:|:---------:|
|A|	25|	2|
|B	|40|	3|

***



Click [here](https://github.com/kashyapsahil650/Case_Study_On_SQL#challenge-case-studies) to move back to the 8-Week-SQL-Challenge repository!


