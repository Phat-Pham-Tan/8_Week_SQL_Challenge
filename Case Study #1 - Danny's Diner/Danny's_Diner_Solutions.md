# 🍜 Danny's Diner Solutions

<p align="right"> Using Microsoft SQL Server </p>

## Questions

### 1. What is the total amount each customer spent at the restaurant?

```sql
SELECT customer_id,sum(menu.price) as Total_Spent
FROM sales 
Inner join menu 
ON sales.product_id = menu.product_id 
GROUP BY customer_id;
```
#### Result 
||customer_id | Total_Spent |
|---|---|---|
| 1 | A | 76 |
| 2 | B | 74 |
| 3 | C | 36 |



#

### 2. How many days has each customer visited the restaurant?

```sql
SELECT  sales.customer_id, 
        count(distinct sales.order_date) as Total_days
FROM sales 
GROUP BY sales.customer_id;
```
#### Result
||customer_id | Total_Spent |
|---|---|---|
| 1 | A | 4 |
| 2 | B | 6 |
| 3 | C | 2 |

#

### 3. What was the first item from the menu purchased by each customer?


- Create a  Common Table Expressions (CTE), I named it as PURCHASED_RANK: 
  - In it, create a new column to show a `ranking of the items purchased by each customer (customer_id) based on the date of purchase (order_date):
    - Rank 1 will be the first item purchased with the earliest date
    - For this you can use `RANK` or `DENSE_RANK`(I use DENSE_RANK  in this case)


```sql
WITH PURCHASED_RANK AS (
    			SELECT customer_id, 
			       order_date,
			       product_name, 
    			       DENSE_RANK() OVER(ORDER BY order_date ) AS RANK
			FROM sales
			LEFT JOIN menu
			ON sales.product_id = menu.product_id 
			)
SELECT distinct customer_id,product_name
FROM PURCHASED_RANK 
WHERE RANK =1 
```
#### PURCHASED_RANK table output:

  | customer_id | order_date | product_name | RANK |
|---|---|---|---|
| 1 | 2021-01-01 | sushi | 1 |
| 2 | 2021-01-01 | curry | 1 |
| 3 | 2021-01-01 | curry | 1 |
| 4 | 2021-01-01 | ramen | 1 |
| 5 | 2021-01-01 | ramen | 1 |
| 6 | 2021-01-02 | curry | 2 |
| 7 | 2021-01-04 | sushi | 3 |
| 8 | 2021-01-07 | curry | 4 |
| 9 | 2021-01-07 | ramen | 4 |
| 10 | 2021-01-10 | ramen | 5 |
| 11 | 2021-01-11 | ramen | 6 |
| 12 | 2021-01-11 | ramen | 6 |
| 13 | 2021-01-11 | sushi | 6 |
| 14 | 2021-01-16 | ramen | 7 |
| 15 | 2021-02-01 | ramen | 8 |


  
#### Question Result: 

|| customer_id | product_name |
|---|---|---|
| 1 | A | curry |
| 2 | A | sushi |
| 3 | B | curry |
| 4 | C | ramen |


  - Customer A's first orders were Curry & Sushi
  - Customer B's was Curry
  - Customer C's was Ramen 

#

### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?
 
 ```sql
SELECT TOP 1 (COUNT(s.product_id)) AS most_purchased, product_name
FROM dbo.sales AS s
JOIN dbo.menu AS m
 ON s.product_id = m.product_id
GROUP BY s.product_id, product_name
ORDER BY most_purchased DESC
 ```
 #### Result
|   | most_purchased | product_name |
|---|----------------|--------------|
| 1 | 8              | ramen        |


#

### 5. Which item was the most popular for each customer?

- Create a `CTE` names as rank_favorite_item 
- In it, create a new column to show a `ranking of the items purchased by each customer (customer_id) based on the times purchased (COUNT of product_id).
  - Rank 1 will be the most purchased item, 2 the second...
  - For this you can use RANK or DENSE_RANK.
- From that CTE we then want to `select the most purchased item by each customer`. 
  - This is `WHERE rank = 1`

```sql
WITH rank_favorite_item AS (
				SELECT customer_id,
				       product_name, 
				       COUNT(sales.product_id) AS NUMBER_ORDER,
				       rank() OVER(PARTITION BY customer_id 
				ORDER BY COUNT(sales.product_id) DESC) AS RANK
				FROM sales
				INNER JOIN menu
				ON sales.product_id = menu.product_id
				GROUP BY customer_id,product_name
)
SELECT customer_id,product_name,NUMBER_ORDER
FROM rank_favorite_item
WHERE RANK =1;
```

#### rank_favorite_item table output:
|  | customer_id | product_name | NUMBER_ORDER | RANK |
|--|-------------|--------------|--------------|------|
| 1| A           | ramen        | 3            | 1    |
| 2| A           | curry        | 2            | 2    |
| 3| A           | sushi        | 1            | 3    |
| 4| B           | sushi        | 2            | 1    |
| 5| B           | curry        | 2            | 1    |
| 6| B           | ramen        | 2            | 1    |
| 7| C           | ramen        | 3            | 1    |

#### Question Result:
 |  | customer_id | product_name | order_count |
|--|-------------|--------------|-------------|
| 1| A           | ramen        | 3           |
| 2| B           | sushi        | 2           |
| 3| B           | curry        | 2           |
| 4| B           | ramen        | 2           |
| 5| C           | ramen        | 3           |


- Customer A's favourite item is Ramen
- Customer B likes all items 
- Customer C likes Ramen

#

### 6. Which item was purchased first by the customer after they became a member?

- Create a `CTE` named as 'rank_purchase_item'
  - In it, create a new column to show a `ranking of the items purchased by each customer (customer_id) based on the date of purchase (order_date)`. 
    - Rank 1 will be the first item purchased
    - For this you can use RANK or DENSE_RANK.
  - We need to include a `WHERE clause` in the CTE as we `only want items purchased after they became a member`.
    - WHERE order_date >= join_date
- From that CTE we then want to `select the first item purchased by each customer`. 
  - This is `WHERE rank = 1`

```sql
WITH rank_purchase_item AS(
			     SELECT sales.customer_id, 
				    product_name, 
				    order_date, 
				    join_date,
				    DENSE_RANK() OVER(Partition by sales.customer_id ORDER BY order_date) AS RANK 
			     FROM sales 
			     INNER JOIN menu
			     ON sales.product_id = menu.product_id
			     INNER JOIN members
			     ON sales.customer_id = members.customer_id
			     WHERE sales.order_date >= members.join_date
)
SELECT customer_id, product_name
FROM rank_purchase_item
WHERE RANK =1;
```
#### rank_purchase_item table output:

|  | customer_id | product_name | order_date | join_date | RANK |
|--|-------------|--------------|------------|-----------|-----|
| 1| A           | curry        | 2021-01-07 | 2021-01-07| 1   |
| 2| A           | ramen        | 2021-01-10 | 2021-01-07| 2   |
| 3| A           | ramen        | 2021-01-11 | 2021-01-07| 3   |
| 4| A           | ramen        | 2021-01-11 | 2021-01-07| 3   |
| 5| B           | sushi        | 2021-01-11 | 2021-01-09| 1   |
| 6| B           | ramen        | 2021-01-16 | 2021-01-09| 2   |
| 7| B           | ramen        | 2021-02-01 | 2021-01-09| 3   |

#### Result
|  | customer_id | product_name |
|--|-------------|--------------|
| 1| A           | ramen        |
| 2| B           | sushi        |

- Customer A purchased Curry firstly 
- Customer B puchased Sushi firstly

#

### 7. Which item was purchased just before the customer became a member?

- Create a `CTE` as rank_purchase_order
  - In it, create a new column to show a `ranking of the items purchased by each customer (customer_id) based on the date of purchase (order_date) in descending order`. 
    - Rank 1 will be the last item purchased (the item purchased on latest date)
    - For this you can use RANK or DENSE_RANK.
  - We need to include a `WHERE clause` in the CTE as we `only want items purchased before they became a member`.
    - WHERE order_date < join_date
- From that CTE we then want to `select the first item purchased by each customer`. 
  - This is `WHERE rank = 1`

```sql
WITH rank_purchase_order AS (
                            SELECT sales.customer_id, 
			     	    product_name, 
				    order_date, 
				    join_date,
    				    DENSE_RANK() OVER(Partition by sales.customer_id ORDER BY order_date DESC) AS RANK 
   			    FROM sales 
			    INNER JOIN menu
			    ON sales.product_id = menu.product_id
			    INNER JOIN members
			    ON sales.customer_id = members.customer_id
			    WHERE sales.order_date < members.join_date
                             )
SELECT customer_id, product_name
FROM rank_purchase_order
WHERE RANK =1
```
#### Result
|  | customer_id | product_name |
|--|-------------|--------------|
| 1| A           | sushi        |
| 2| A           | curry        |
| 3| B           | sushi        |

- Customer A's last item purchased before becoming a member was Sushi & Curry
- Customer B's was Sushi

#

### 8. What is the total items and amount spent for each member before they became a member?

```sql
SELECT  sales.customer_id,
        count(product_name) as total_item, 
        SUM(price) as Total_amount_spent
FROM sales 
INNER JOIN menu
ON sales.product_id = menu.product_id
INNER JOIN members
ON sales.customer_id = members.customer_id
WHERE sales.order_date < members.join_date
GROUP BY sales.customer_id;
```
#### Result
|  | customer_id | total_item | Total_amount_spent |
|--|-------------|------------|---------------------|
| 1| A           | 2          | 25                  |
| 2| B           | 3          | 40                  |

#

### 9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

- Every $1 = 10 points.
- For sushi $1 = 20 points (2 x 10). 
- We want to create total point column of each type product through SUM of CASE WHEN:
	- When the product name is sushi then multiply the price by 20, when not then multiply it by 10.


 

```sql
SELECT customer_id,
        SUM(CASE WHEN product_name = 'sushi' THEN price *20 
        ELSE price * 10 END ) AS total_point
FROM sales S
JOIN menu M ON S.product_id = M.product_id
GROUP BY  customer_id;
```
#### Result
|  | customer_id | total_point |
|--|-------------|-------------|
| 1| A           | 860         |
| 2| B           | 940         |
| 3| C           | 360         |

# 

### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

- I create a CTE table with valid_date that include 1 week after joining and end of January column
- For sushi earn x2 point, and items between join_date and valid_date earn x2 point (first week joining)
- I also including condition within January order_date <= last_date

```sql
WITH dates_cte AS(
		SELECT *, 
			DATEADD(DAY, 6, join_date) AS valid_date, 
			EOMONTH('2021-01-1') AS last_date
		FROM members
)

SELECT s.customer_id,
	sum(CASE WHEN s.product_id = 1 THEN price*20
		WHEN s.order_date between d.join_date and d.valid_date THEN price*20
		ELSE price*10 
	END) as total_points
FROM dates_cte d,
     sales s,
     menu m
WHERE d.customer_id = s.customer_id 
	AND m.product_id = s.product_id
	AND s.order_date <= d.last_date
GROUP BY s.customer_id;
```
#### Result
|  | customer_id | total_points |
|--|-------------|--------------|
| 1| A           | 1370         |
| 2| B           | 820          |

---

## Bonus Questions


###  Join all Thing & Rank All Things 

- Danny also requires further information about the ranking of customer products, but he purposely does not need the ranking for non-member purchases so he expects null ranking values for the records when customers are not yet part of the loyalty program.
- Join all Thing :For this create a CTE (named as member_table) to have column 'member' by order_date >= join_date.
- Rank All Things :Then SELECT everything from that table and add a new column for the ranking.
   - For the RANK we need to `PARTITION by both customer_id and member`.
   - You can use RANK or DENSE_RANK.
 
 ```sql
WITH member_table as (  SELECT  s.customer_id,
                                s.order_date,m.product_name,m.price,
                                CASE WHEN s.order_date >= me.join_date THEN 'Y'
                                ELSE 'N'  END AS member
                        FROM sales s
                        INNER JOIN menu m ON s.product_id = m.product_id
                        LEFT JOIN members me ON s.customer_id = me.customer_id
		)
SELECT  customer_id,
	order_date,
	product_name,
	price,
	member,
    CASE WHEN member ='N' THEN NULL 
    ELSE RANK() OVER(PARTITION BY customer_id, member ORDER BY order_date) END AS ranking
FROM member_table
 ```
#### Result Join all Thing 
|  | customer_id | order_date | product_name | price | member |
|--|-------------|------------|--------------|-------|--------|
| 1| A           | 2021-01-01 | sushi        | 10    | N      |
| 2| A           | 2021-01-01 | curry        | 15    | N      |
| 3| A           | 2021-01-07 | curry        | 15    | Y      |
| 4| A           | 2021-01-10 | ramen        | 12    | Y      |
| 5| A           | 2021-01-11 | ramen        | 12    | Y      |
| 6| A           | 2021-01-11 | ramen        | 12    | Y      |
| 7| B           | 2021-01-01 | curry        | 15    | N      |
| 8| B           | 2021-01-02 | curry        | 15    | N      |
| 9| B           | 2021-01-04 | sushi        | 10    | N      |
|10| B           | 2021-01-11 | sushi        | 10    | Y      |
|11| B           | 2021-01-16 | ramen        | 12    | Y      |
|12| B           | 2021-02-01 | ramen        | 12    | Y      |
|13| C           | 2021-01-01 | ramen        | 12    | N      |
|14| C           | 2021-01-01 | ramen        | 12    | N      |
|15| C           | 2021-01-07 | ramen        | 12    | N      |

#### Result Rank All Things 
|  | customer_id | order_date | product_name | price | member | ranking |
|--|-------------|------------|--------------|-------|--------|---------|
| 1| A           | 2021-01-01 | sushi        | 10    | N      | Null    |
| 2| A           | 2021-01-01 | curry        | 15    | N      | Null    |
| 3| A           | 2021-01-07 | curry        | 15    | Y      | 1       |
| 4| A           | 2021-01-10 | ramen        | 12    | Y      | 2       |
| 5| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
| 6| A           | 2021-01-11 | ramen        | 12    | Y      | 3       |
| 7| B           | 2021-01-01 | curry        | 15    | N      | Null    |
| 8| B           | 2021-01-02 | curry        | 15    | N      | Null    |
| 9| B           | 2021-01-04 | sushi        | 10    | N      | Null    |
|10| B           | 2021-01-11 | sushi        | 10    | Y      | 1       |
|11| B           | 2021-01-16 | ramen        | 12    | Y      | 2       |
|12| B           | 2021-02-01 | ramen        | 12    | Y      | 3       |
|13| C           | 2021-01-01 | ramen        | 12    | N      | Null    |
|14| C           | 2021-01-01 | ramen        | 12    | N      | Null    |
|15| C           | 2021-01-07 | ramen        | 12    | N      | Null    |
