# Ikea-sales-SQLproject

![Ikea-logo](https://github.com/user-attachments/assets/26794655-c1c0-4cdf-a4a2-dd9d062cfbf0)

Welcome to the **IKEA Sales SQL Project**! This project leverages a detailed dataset of millions of sales records, product inventory, and store information across IKEA's global operations. The analysis focuses on uncovering sales trends, product performance, and inventory management insights to assist in data-driven decision-making.

---

## Table of Contents
- [Introduction](#introduction)
- [Project Structure](#project-structure)
- [Database Schema](#database-schema)
- [Business Problems](#business-problems)
- [SQL Queries & Analysis](#sql-queries--analysis)
- [Getting Started](#getting-started)
- [Questions & Feedback](#questions--feedback)
- [Contact Me](#contact-me)
- [ERD (Entity-Relationship Diagram)](#erd-entity-relationship-diagram)

---

## Introduction

The IKEA Sales SQL Project demonstrates the use of SQL to analyze retail data, including **sales records**, **store performance**, **product trends**, and **inventory status**. Using a robust schema, this project answers critical business questions and provides actionable insights to optimize IKEA's operational efficiency and profitability.

---

## Project Structure

1. **SQL Scripts**: Contains SQL queries to create the database schema, populate tables, and perform analyses.
2. **Dataset**: Includes sales data, product information, store details, and inventory records.
3. **Analysis**: SQL queries solve key business problems, leveraging advanced SQL techniques like joins, aggregations, and subqueries.

---

## Database Schema

### 1. **Products Table**
- **product_id**: Unique identifier for each product (Primary Key).
- **product_name**: Name of the product.
- **category**: Category to which the product belongs.
- **subcategory**: Subcategory of the product.
- **unit_price**: Price per unit of the product.

### 2. **Stores Table**
- **store_id**: Unique identifier for each store (Primary Key).
- **store_name**: Name of the store.
- **city**: City where the store is located.
- **country**: Country where the store operates.

### 3. **Sales Table**
- **order_id**: Unique identifier for each sales order (Primary Key).
- **order_date**: Date when the order was placed.
- **product_id**: Foreign key referencing the `products` table.
- **qty**: Quantity of the product sold.
- **discount_percentage**: Discount applied to the order.
- **unit_price**: Price per unit of the product at the time of sale.
- **store_id**: Foreign key referencing the `stores` table.

### 4. **Inventory Table**
- **inventory_id**: Unique identifier for each inventory record (Primary Key).
- **product_id**: Foreign key referencing the `products` table.
- **current_stock**: Current stock level of the product.
- **reorder_level**: Minimum stock level to trigger a reorder.

### 5. **Global Sales Table**
- **CTA Table**: this table was created by joining `sales`, `products`, and `stores` tables.
  ```sql
  CREATE TABLE global_sales
  AS
    SELECT 
	      s.*,
	      p.product_name,
	      p.category,
	      p.subcategory,
	      st.store_name,
	      st.city,
	      st.country	
    FROM sales as s
    JOIN 
      products as p
    ON s.product_id = p.product_id
    JOIN
      stores as st
    ON st.store_id = s.store_id;
  ```
---

## Business Problems

This project tackles the following business problems:

### Easy-Level Queries
1. Find the average discount and total revenue generated for each subcategory across all stores.
   
   ```sql
    SELECT 
          subcategory,
          AVG(discount_percentage) AS avg_discount,
    SUM(net_sales) AS total_rev
    FROM global_sales
    GROUP BY subcategory;
    ```
2. List all products that are low in stock (below the reorder level).
   ``` sql
     SELECT 
	        product_id
     FROM inventory
     WHERE reorder_level <= current_stock
   ```
4. Calculate total sales revenue for each store.
   ```sql
   SELECT
	      store_id,
	      SUM(net_sales) as total_revenue
   FROM global_sales
   GROUP BY 1;
    ```
   
5. Find the top 3 stores with the highest sales in a specific country.
  ```sql
  WITH rank_stores
  AS (
	      SELECT
		        country,
		        store_id,
		        dense_rank() OVER(Partition by store_id ORDER by net_sales) as rank_,
		        sum(net_sales) as total_sales
	      FROM global_sales 
	      WHERE country = 'Vietnam' 
	      GROUP BY 1, 2, net_sales
	)
SELECT
	    country,
	    store_id,
	    rank_,
	    total_sales
FROM rank_stores
WHERE rank_ <= 3
ORDER BY total_sales;
```
  
5. Retrieve sales data for the last 6 months.
   ``` sql
   WITH latest_date AS (
    SELECT MAX(order_date) AS last_data_date 
    FROM global_sales
    )
    SELECT 
	      store_id,
	      TO_CHAR(order_date,'Month') as _month_,
        category, 
        SUM(net_sales) AS total_rev
    FROM global_sales
    WHERE order_date >= (
    SELECT DATE_TRUNC('month', last_data_date) - INTERVAL '6 months' 
    FROM latest_date
    )
    GROUP BY category, _month_, store_id
    ORDER BY _month_ DESC;
   ```

### Medium to Hard-Level Queries
1. Retrieve the top three products by total sales revenue in each store.
 ```sql
    WITH tsr
    AS (
         SELECT 
	          store_id,
            product_id,
	          round(sum(net_sales)::numeric, 2) as total_sales,
	          DENSE_RANK() OVER (PARTITION BY store_id ORDER BY sum(net_sales) DESC) AS RANK1
        FROM global_sales
        GROUP BY 1, 2)
   SELECT 
  	    store_id,
	      product_id, 
	      total_sales, 
	      RANK1 
  FROM tsr 
  WHERE RANK1 <= 3;
```
2. Determine the average sales revenue generated by each product in stores where it sold above the average sales quantity.
``` sql
WITH average_products_qty
AS (
	SELECT
	   product_name,
	   AVG(qty) as sales_qty
FROM global_sales
GROUP BY product_name),
 filter_st_results
 AS (
	 SELECT
	 	g.product_name,
		g.store_id,
		g.net_sales
	FROM global_sales as g
	JOIN average_products_qty as a
	ON a.product_name = g.product_name
	WHERE g.qty > a.sales_qty)
SELECT 
	store_id,
    product_name,
    AVG(net_sales) AS average_sales_rev
FROM 
    filter_st_results
GROUP BY 
	store_id, product_name;
```
3. identify the latest sale for each product in each store.
``` sql
SELECT 
    product_id,
    store_id,
    MAX(EXTRACT(YEAR FROM order_date)) AS l_s
FROM (SELECT 
            product_id,
	          store_id,
	          order_date,
            COUNT(DISTINCT store_id) AS d_s_c
      FROM global_sales
      GROUP BY product_id, store_id, order_date
      ORDER BY product_id
) AS count_1
GROUP BY product_id, store_id
ORDER BY product_id, store_id;
```
4. Retrieve stores with a total discount percentage above the average discount for all stores.
``` sql
WITH discount_summary
AS 
(
	SELECT 
			COUNT(DISTINCT discount_percentage)  AS total_disc
  FROM global_sales
		)
SELECT 
	store_id,
	AVG(discount_percentage) as avg_disc
FROM global_sales
GROUP BY store_id
HAVING (SELECT total_disc FROM discount_summary) > AVG(discount_percentage)
ORDER BY avg_disc DESC;
```

5. Find products whose sales exceed the highest sales of any other product in the same category.
```sql
  SELECT
	  product_id,
    category,
    DENSE_RANK() OVER (PARTITION BY category order by total_sales DESC) as highest_sales,
    total_sales AS sales
  FROM (
	      SELECT 
		        product_id,
		        category,
		        sum(net_sales) as total_sales
	      FROM global_sales
	      GROUP BY 1,2
      )
ORDER BY category, highest_sales;
```
---

## SQL Queries & Analysis

All SQL queries developed for this project are available in the `queries.sql` file. The queries demonstrate advanced SQL skills, including:

- Aggregations with `GROUP BY`.
- Filtering data using `WHERE` and `HAVING`.
- Joining multiple tables to uncover insights.
- Using subqueries and window functions for complex analyses.

---

## Getting Started

### Prerequisites
- PostgreSQL (or any SQL-compatible database).
- Basic knowledge of SQL.

### Steps to Run
1. **Clone the Repository**:
   ```bash
   git clone https://github.com/jkusi6/ikea-sales-sql-project.git
   ```
2. **Set Up the Database**:
   - Run `schema.sql` to create the database schema.
   - Populate tables with sample data using `data.sql`.

3. **Execute Queries**:
   - Open `queries.sql` and execute the queries for analysis.

---

## Questions & Feedback

Feel free to reach out with questions or suggestions. Here's an example query for reference:

### Example Query
**Question**: Retrieve the total sales revenue for each store in a specific country.
```sql
SELECT 
    s.store_name, 
    SUM(sales.qty * sales.unit_price) AS total_revenue
FROM 
    sales
JOIN 
    stores s ON sales.store_id = s.store_id
WHERE 
    s.country = 'USA'
GROUP BY 
    s.store_name
ORDER BY 
    total_revenue DESC;
```

---

## Contact Me

📧 **[Email](mailto:jorekusi@gmail.com)** 
💼 **[LinkedIn](https://linkedin.com/in/jore-kusi)**  

---

## ERD (Entity-Relationship Diagram)

Here’s the ERD for the IKEA Retail Sales SQL Project:

![IKEA ERD](https://github.com/user-attachments/assets/31b876ff-e398-4f98-80d7-9188b827c624)


---

