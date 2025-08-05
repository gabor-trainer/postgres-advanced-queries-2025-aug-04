### **Synchronizing Data with the MERGE Statement**

**Objective:**
This lab will guide you through using the `MERGE` statement to handle complex data synchronization tasks efficiently. You will simulate a common data warehousing scenario: synchronizing a daily sales transaction log with a monthly summary table. You will learn to use `WHEN MATCHED` to update existing records, `WHEN NOT MATCHED` to insert new ones, and even `WHEN MATCHED ... THEN DELETE` to remove records based on specific conditions.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 15 or higher is required for the `MERGE` statement).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line).
4.  **Basic SQL Knowledge:** Familiarity with `SELECT`, `INSERT`, and `UPDATE` statements.

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run `SELECT count(*) FROM order_details;`. The result should be `2155`.

---

### **Lab Exercises**

**Introduction to the Scenario:**
Imagine you are a data engineer for Northwind. Each day, you receive a summary of product sales. Your task is to update a master summary table, `monthly_product_sales`, with this new data.

*   If a product is sold for the first time in a given month, a new record must be **inserted** into the summary.
*   If a product that already has a summary record for the month is sold again, its existing record must be **updated** (e.g., by adding to the total quantity sold).
*   If a data correction indicates returns have made a product's monthly sales zero or negative, the record should be **deleted**.

The `MERGE` statement is the ideal tool for handling all these conditions in a single, atomic operation.

---

#### **Task 1: Create the Master Summary Table**

**Description:**
First, you need a target table for the `MERGE` operation. Create a new table named `monthly_product_sales`. This table will store the aggregated sales quantity and revenue for each product on a monthly basis. The primary key must enforce that there is only one summary record per product per month.

**SQL Query:**
```sql
-- Create the target table for our MERGE operations
CREATE TABLE monthly_product_sales (
    sales_month         DATE NOT NULL,
    product_id          SMALLINT NOT NULL,
    total_quantity_sold INT,
    total_revenue       NUMERIC(10, 2),
    CONSTRAINT pk_monthly_product_sales PRIMARY KEY (sales_month, product_id)
);
```

**Expected Output/Explanation:**
The query will create the `monthly_product_sales` table. It will be empty initially.

**Learning Point:**
The `MERGE` statement requires a unique constraint or primary key on the target table to determine if a source row `MATCHED` an existing target row. The composite primary key on `(sales_month, product_id)` serves this critical purpose.

---

#### **Task 2: Create and Populate the Daily Sales Staging Table**

**Description:**
Next, simulate the arrival of a daily sales file. Create a temporary staging table, `daily_sales_log`, and populate it with aggregated sales data from the `orders` and `order_details` tables for a specific day, '1996-07-08'.

**SQL Query:**```sql
-- Create a temporary staging table for one day's sales data
CREATE TEMP TABLE daily_sales_log (
    sales_date      DATE,
    product_id      SMALLINT,
    quantity_sold   INT,
    day_revenue     NUMERIC(10, 2)
);

-- Populate it with aggregated data for a specific day
INSERT INTO daily_sales_log (sales_date, product_id, quantity_sold, day_revenue)
SELECT
    o.order_date,
    od.product_id,
    SUM(od.quantity),
    SUM(od.unit_price * od.quantity * (1 - od.discount))
FROM orders o
JOIN order_details od ON o.order_id = od.order_id
WHERE o.order_date = '1996-07-08'
GROUP BY o.order_date, od.product_id;
```

**Expected Output/Explanation:**
This creates and populates the `daily_sales_log` table with three rows representing the products sold on that day. You can verify with `SELECT * FROM daily_sales_log;`.

**Learning Point:**
This establishes the `source` table for our `MERGE` statement. In a real-world scenario, this data might come from an external file or a different system that is loaded into a staging table like this one.

---

#### **Task 3: The Initial `MERGE` - Inserting New Records**

**Description:**
Perform the first synchronization. Since `monthly_product_sales` is empty, every row from `daily_sales_log` will be a new record for the month of July 1996. Use `MERGE` to insert these new records. We'll use a transaction to roll back the changes after inspection.

**SQL Query:**
```sql
BEGIN;

-- MERGE the daily log into the monthly summary
MERGE INTO monthly_product_sales M -- M is an alias for the target table
USING daily_sales_log S -- S is an alias for the source table
ON M.product_id = S.product_id AND M.sales_month = DATE_TRUNC('month', S.sales_date)
WHEN NOT MATCHED THEN
    INSERT (sales_month, product_id, total_quantity_sold, total_revenue)
    VALUES (DATE_TRUNC('month', S.sales_date), S.product_id, S.quantity_sold, S.day_revenue);

-- Verify that the new rows were inserted
SELECT * FROM monthly_product_sales;

ROLLBACK;
```

**Expected Output/Explanation:**
The verification `SELECT` will show three new rows in `monthly_product_sales`, one for each product from the daily log, with a `sales_month` of `1996-07-01`.

**Learning Point:**
This task demonstrates the `WHEN NOT MATCHED THEN INSERT` clause. The `ON` condition is key; it defines how to match rows between the source and target. Because no rows in the target table met the condition, the `INSERT` action was taken for all source rows.

---

#### **Task 4: A Second `MERGE` - Updating and Inserting**

**Description:**
Now, simulate the next day's sales data. This new data includes sales for a product that was sold yesterday (triggering an `UPDATE`) and sales for a product not sold yesterday (triggering an `INSERT`).

**SQL Query:**
```sql
BEGIN;
-- First, run the MERGE from Task 3 to have some initial data
MERGE INTO monthly_product_sales M USING daily_sales_log S ON M.product_id = S.product_id AND M.sales_month = DATE_TRUNC('month', S.sales_date) WHEN NOT MATCHED THEN INSERT (sales_month, product_id, total_quantity_sold, total_revenue) VALUES (DATE_TRUNC('month', S.sales_date), S.product_id, S.quantity_sold, S.day_revenue);

-- Clear the daily log and load the next day's data with explicit casting
TRUNCATE daily_sales_log;
INSERT INTO daily_sales_log (sales_date, product_id, quantity_sold, day_revenue)
SELECT '1996-07-09'::DATE, 41, 9, 87.75  -- CORRECTED: Explicit cast to DATE
UNION ALL
SELECT '1996-07-09'::DATE, 20, 10, 810; -- CORRECTED: Explicit cast to DATE

-- Now, perform the MERGE again with the new daily data
MERGE INTO monthly_product_sales M
USING daily_sales_log S
ON M.product_id = S.product_id AND M.sales_month = DATE_TRUNC('month', S.sales_date)
WHEN MATCHED THEN
    UPDATE SET
        total_quantity_sold = M.total_quantity_sold + S.quantity_sold,
        total_revenue = M.total_revenue + S.day_revenue
WHEN NOT MATCHED THEN
    INSERT (sales_month, product_id, total_quantity_sold, total_revenue)
    VALUES (DATE_TRUNC('month', S.sales_date), S.product_id, S.quantity_sold, S.day_revenue);

-- Verify the results: one row updated, one new row inserted
SELECT * FROM monthly_product_sales ORDER BY product_id;

ROLLBACK;
```

**Expected Output/Explanation:**
The verification `SELECT` will show four rows. The record for product `41` will have an updated `total_quantity_sold` (the original quantity from July 8th plus 9). A new record for product `20` will now exist.

**Learning Point:**
This task demonstrates the `WHEN MATCHED THEN UPDATE` clause. The `MERGE` statement correctly identified that product `41` already existed for the month and applied the update logic, while product `20` did not exist and triggered the insert logic.

---

#### **Task 5: A Final `MERGE` - Using a Conditional `DELETE`**

**Description:**
A data correction file arrives. It turns out all 10 units of Product 20 that were sold were returned due to a defect. The file represents this as a negative sale. The business rule is to remove any product from the monthly summary if its total sales become zero or less.

**SQL Query:**
```sql
BEGIN;
-- Perform the setup and MERGE from Task 4 to get a full data set (this uses the corrected cast)
MERGE INTO monthly_product_sales M USING (SELECT o.order_date, od.product_id, SUM(od.quantity) AS quantity_sold, SUM(od.unit_price * od.quantity * (1 - od.discount)) AS day_revenue FROM orders o JOIN order_details od ON o.order_id = od.order_id WHERE o.order_date = '1996-07-08' GROUP BY o.order_date, od.product_id) S ON M.product_id = S.product_id AND M.sales_month = DATE_TRUNC('month', S.sales_date) WHEN NOT MATCHED THEN INSERT VALUES (DATE_TRUNC('month', S.sales_date), S.product_id, S.quantity_sold, S.day_revenue);
MERGE INTO monthly_product_sales M USING (SELECT '1996-07-09'::DATE AS sales_date, 41 AS product_id, 9 AS quantity_sold, 87.75 AS day_revenue UNION ALL SELECT '1996-07-09'::DATE, 20, 10, 810) S ON M.product_id = S.product_id AND M.sales_month = DATE_TRUNC('month', S.sales_date) WHEN MATCHED THEN UPDATE SET total_quantity_sold = M.total_quantity_sold + S.quantity_sold, total_revenue = M.total_revenue + S.day_revenue WHEN NOT MATCHED THEN INSERT VALUES (DATE_TRUNC('month', S.sales_date), S.product_id, S.quantity_sold, S.day_revenue);

-- Simulate the data correction file for the returns
TRUNCATE daily_sales_log;
INSERT INTO daily_sales_log VALUES ('1996-07-10'::DATE, 20, -10, -810.00); -- CORRECTED: Explicit cast to DATE

-- MERGE with a conditional DELETE
MERGE INTO monthly_product_sales M
USING daily_sales_log S
ON M.product_id = S.product_id AND M.sales_month = DATE_TRUNC('month', S.sales_date)
WHEN MATCHED AND (M.total_quantity_sold + S.quantity_sold <= 0) THEN
    DELETE
WHEN MATCHED THEN
    UPDATE SET
        total_quantity_sold = M.total_quantity_sold + S.quantity_sold,
        total_revenue = M.total_revenue + S.day_revenue;

-- Verify product 20 has been deleted
SELECT * FROM monthly_product_sales WHERE product_id = 20; -- Should return 0 rows

ROLLBACK;
```

**Expected Output/Explanation:**
The final `SELECT` query will return zero rows, confirming that the record for product 20 was successfully deleted by the `MERGE` statement because its total quantity sold became zero.

**Learning Point:**
This task demonstrates the `WHEN MATCHED AND <condition> THEN DELETE` clause. This powerful feature allows for complex conditional logic within a single `MERGE` statement, enabling you to insert, update, or delete rows in one pass over the data.

---

### **Conclusion**

In this lab, you mastered the `MERGE` statement to solve a realistic data synchronization problem. You learned how to use the `USING` and `ON` clauses to define the source and matching logic, and how to implement different actions for matched and unmatched rows with `WHEN MATCHED` and `WHEN NOT MATCHED`. This single statement is a highly efficient and readable alternative to writing complex procedural code with separate `INSERT`, `UPDATE`, and `DELETE` commands.