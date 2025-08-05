### **Advanced Analytics with PostgreSQL Window Functions**

**Objective:**
This lab will provide an in-depth understanding of PostgreSQL's Window Functions. You will learn to use `ROW_NUMBER()`, `FIRST_VALUE()`, `LAST_VALUE()`, `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()`, `CUME_DIST()`, and `NTILE()` to perform complex analytical tasks such as sequential numbering, ranking, calculating running totals, accessing preceding/following rows, and percentile analysis. You will also solidify your understanding of the `OVER()` clause and its `PARTITION BY` and `ORDER BY` components.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line).
4.  **Basic SQL Knowledge:** Familiarity with `SELECT`, `FROM`, `JOIN`, `WHERE`, `GROUP BY`, and aggregate functions.

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run `SELECT count(*) FROM orders;`. The result should be `830`.

---

### **Lab Exercises**

**Introduction to Window Functions (`OVER()` Clause):**
Window functions perform calculations across a set of table rows that are related to the current row. Unlike aggregate functions, window functions do not group rows together; instead, they return a result for each row. The `OVER()` clause defines the "window" or set of rows over which the function operates.

*   `OVER ()`: Defines the entire result set as the window.
*   `OVER (PARTITION BY column)`: Divides the rows into groups or partitions, and the function operates independently within each partition.
*   `OVER (ORDER BY column)`: Orders the rows within the window (or partition) and determines the order in which the function processes the rows. This is crucial for functions that depend on row order (e.g., `ROW_NUMBER()`, `LAG()`, `LEAD()`, running totals).

---

#### **Task 1: `ROW_NUMBER()` - Assigning Sequential Row Numbers**

**Description:**
`ROW_NUMBER()` assigns a unique, sequential integer to each row within its partition, starting from 1.

1.  **Task 1.1: Order History for a Specific Customer:** Retrieve all orders for customer 'ALFKI' and assign a row number to each order based on its `order_date` in ascending order.
2.  **Task 1.2: Nth Order per Employee:** For each employee, find their 3rd order (if it exists) by `order_date`. This requires partitioning by `employee_id`.

**SQL Query:**
```sql
-- Task 1.1: Order History for a Specific Customer
SELECT
    order_id,
    order_date,
    ROW_NUMBER() OVER (ORDER BY order_date) AS row_num
FROM
    orders
WHERE
    customer_id = 'ALFKI'
ORDER BY
    order_date;

-- Task 1.2: Nth Order per Employee
WITH EmployeeOrders AS (
    SELECT
        employee_id,
        order_id,
        order_date,
        ROW_NUMBER() OVER (PARTITION BY employee_id ORDER BY order_date) AS rn
    FROM
        orders
)
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    eo.order_id,
    eo.order_date
FROM
    EmployeeOrders AS eo
JOIN
    employees AS e ON eo.employee_id = e.employee_id
WHERE
    eo.rn = 3
ORDER BY
    employee_name;
```

**Expected Output/Explanation:**
*   **Task 1.1:** You will see 'ALFKI''s orders numbered sequentially from 1.
*   **Task 1.2:** You will get a list of orders, where each order is the 3rd order placed by that specific employee. Some employees might not have a 3rd order, so they won't appear.

**Learning Point:**
`ROW_NUMBER()` is used for unique sequencing. Task 1.1 demonstrates `OVER (ORDER BY ...)`. Task 1.2 introduces `OVER (PARTITION BY ... ORDER BY ...)` which creates separate numbering sequences for each `employee_id`. The `ORDER BY` clause within `OVER()` is critical for defining the sequence.

---

#### **Task 2: Running Totals / Aggregates within a Window (Implicit `ORDER BY` Impact)**

**Description:**
While not a function itself, the `ORDER BY` clause within `OVER()` is fundamental to running calculations like sums, averages, min/max over a "growing" window.

1.  **Task 2.1: Running Total Freight per Customer:** For each customer, calculate the running total of `freight` costs, ordered by `order_date`.
2.  **Task 2.2: Daily Average Order Value:** For each order, show its `order_revenue` and the average `order_revenue` for all orders on that specific `order_date`.

**SQL Query:**
```sql
-- Task 2.1: Running Total Freight per Customer
SELECT
    customer_id,
    order_id,
    order_date,
    freight,
    SUM(freight) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_freight_total
FROM
    orders
ORDER BY
    customer_id, order_date;

-- Task 2.2: Daily Average Order Value
WITH OrderRevenue AS (
    SELECT
        order_id,
        order_date,
        SUM(unit_price * quantity * (1 - discount))::NUMERIC(10, 2) AS order_revenue
    FROM
        order_details
    GROUP BY
        order_id, order_date
)
SELECT
    orv.order_id,
    orv.order_date,
    orv.order_revenue,
    AVG(orv.order_revenue) OVER (PARTITION BY orv.order_date) AS daily_average_order_value
FROM
    OrderRevenue AS orv
ORDER BY
    orv.order_date, orv.order_id;
```

**Expected Output/Explanation:**
*   **Task 2.1:** For each customer, the `running_freight_total` column will accumulate the `freight` costs as you move down their orders, ordered by date.
*   **Task 2.2:** For each order, you'll see its individual revenue and then the average revenue of *all* orders that occurred on the same day.

**Learning Point:**
This task demonstrates how `ORDER BY` within `OVER()` defines a "frame" for aggregations. For `SUM()`, it creates a running total (summing from the start of the partition up to the current row). For `AVG()`, when `ORDER BY` is present, it computes the average of rows up to the current row (a running average). When `ORDER BY` is omitted from `AVG() OVER (PARTITION BY ...)`, it averages *all* rows in the partition.

---

#### **Task 3: `FIRST_VALUE()` and `LAST_VALUE()` - Retrieving Boundary Values**

**Description:**
`FIRST_VALUE()` returns the value of the expression from the first row in the window frame. `LAST_VALUE()` returns the value from the last row.

1.  **Task 3.1: First and Last Order Price per Customer:** For each order, show the `freight` cost of the *first* order and the *last* order placed by that customer (ordered by `order_date`).
2.  **Task 3.2: Earliest and Latest Hire Date per Region:** For each employee, display their `hire_date`, and the `hire_date` of the earliest and latest hired employee within their `region`.

**SQL Query:**
```sql
-- Task 3.1: First and Last Order Price per Customer
SELECT
    customer_id,
    order_id,
    order_date,
    freight,
    FIRST_VALUE(freight) OVER (PARTITION BY customer_id ORDER BY order_date) AS first_order_freight,
    LAST_VALUE(freight) OVER (PARTITION BY customer_id ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_order_freight -- Frame required for LAST_VALUE
FROM
    orders
ORDER BY
    customer_id, order_date;

-- Task 3.2: Earliest and Latest Hire Date per Region
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    e.region,
    e.hire_date,
    FIRST_VALUE(e.hire_date) OVER (PARTITION BY e.region ORDER BY e.hire_date) AS earliest_hire_in_region,
    LAST_VALUE(e.hire_date) OVER (PARTITION BY e.region ORDER BY e.hire_date ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS latest_hire_in_region -- Frame required for LAST_VALUE
FROM
    employees AS e
WHERE e.region IS NOT NULL -- Exclude Andrew Fuller for this regional analysis, as his region is NULL
ORDER BY
    e.region, e.hire_date;
```

**Expected Output/Explanation:**
*   **Task 3.1:** For each customer, `first_order_freight` will consistently show the freight of their very first order. `last_order_freight` will show the freight of their very last order.
*   **Task 3.2:** For employees in 'WA' or 'UK' regions, you'll see their individual hire date, alongside the earliest and latest hire date among colleagues in their respective regions.

**Learning Point:**
`FIRST_VALUE()` and `LAST_VALUE()` retrieve values from the boundaries of the window. For `LAST_VALUE()`, it's crucial to understand the default window frame. By default, the frame for ordered windows is `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. To make `LAST_VALUE()` truly get the last value of the *entire partition*, you often need to explicitly define the frame as `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

---

#### **Task 4: `RANK()` and `DENSE_RANK()` - Assigning Ranks with Ties**

**Description:**
`RANK()` assigns a rank to each row within its partition, with gaps in rank for ties. `DENSE_RANK()` also assigns a rank but does not leave gaps in the ranking sequence when there are ties.

1.  **Task 4.1: Product Price Rank within Category:** For each product, show its `unit_price` and its `RANK()` and `DENSE_RANK()` based on price within its `category`.
2.  **Task 4.2: Customer Order Count Rank (Overall):** Rank all customers by their total number of orders in 1997. Show both `RANK()` and `DENSE_RANK()` to observe the difference.

**SQL Query:**
```sql
-- Task 4.1: Product Price Rank within Category
SELECT
    product_name,
    category_id,
    unit_price,
    RANK() OVER (PARTITION BY category_id ORDER BY unit_price DESC) AS price_rank,
    DENSE_RANK() OVER (PARTITION BY category_id ORDER BY unit_price DESC) AS price_dense_rank
FROM
    products
ORDER BY
    category_id, unit_price DESC;

-- Task 4.2: Customer Order Count Rank (Overall)
WITH CustomerOrderCounts AS (
    SELECT
        customer_id,
        COUNT(order_id) AS total_orders
    FROM
        orders
    WHERE EXTRACT(YEAR FROM order_date) = 1997
    GROUP BY
        customer_id
)
SELECT
    c.company_name,
    coc.total_orders,
    RANK() OVER (ORDER BY coc.total_orders DESC) AS order_rank,
    DENSE_RANK() OVER (ORDER BY coc.total_orders DESC) AS order_dense_rank
FROM
    CustomerOrderCounts AS coc
JOIN
    customers AS c ON coc.customer_id = c.customer_id
ORDER BY
    coc.total_orders DESC, c.company_name;
```

**Expected Output/Explanation:**
*   **Task 4.1:** You'll observe how `RANK()` might skip numbers (e.g., if two products tie for rank 1, the next rank is 3), while `DENSE_RANK()` will continue the sequence (1, 2, 3...).
*   **Task 4.2:** Similar to above, `RANK()` will show gaps if customers have the same number of orders, `DENSE_RANK()` will not.

**Learning Point:**
`RANK()` and `DENSE_RANK()` are used for assigning ranks. The choice between them depends on whether you want gaps in the ranking sequence due to ties. `RANK()` is useful when ties should consume rank numbers, `DENSE_RANK()` when you want a continuous sequence.

---

#### **Task 5: `LAG()` and `LEAD()` - Accessing Previous and Next Rows**

**Description:**
`LAG(expression, offset, default)` accesses data from a row at a specified physical offset *before* the current row in the window. `LEAD(expression, offset, default)` accesses data from a row at a specified physical offset *after* the current row.

1.  **Task 5.1: Previous and Next Order Date for Same Customer:** For each order, show the `order_date`, the `order_date` of the previous order, and the `order_date` of the next order placed by the *same customer*.
2.  **Task 5.2: Employee Hire Date Comparison:** For each employee, display their `hire_date` and the `hire_date` of the employee hired immediately after them (overall, by `hire_date`).

**SQL Query:**
```sql
-- Task 5.1: Previous and Next Order Date for Same Customer
SELECT
    customer_id,
    order_id,
    order_date,
    LAG(order_date, 1, '1900-01-01'::DATE) OVER (PARTITION BY customer_id ORDER BY order_date) AS previous_order_date,
    LEAD(order_date, 1, '2999-12-31'::DATE) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order_date
FROM
    orders
ORDER BY
    customer_id, order_date;

-- Task 5.2: Employee Hire Date Comparison
SELECT
    first_name || ' ' || last_name AS employee_name,
    hire_date,
    LAG(hire_date, 1, '1900-01-01'::DATE) OVER (ORDER BY hire_date) AS previous_hire_date_overall,
    LEAD(hire_date, 1, '2999-12-31'::DATE) OVER (ORDER BY hire_date) AS next_hire_date_overall
FROM
    employees
ORDER BY
    hire_date;
```

**Expected Output/Explanation:**
*   **Task 5.1:** For a given order, `previous_order_date` will show when the same customer placed their prior order, and `next_order_date` will show their subsequent order. The default values handle the first/last orders.
*   **Task 5.2:** For each employee, you'll see the hire date of the person hired just before them and just after them across the entire company.

**Learning Point:**
`LAG()` and `LEAD()` are essential for time-series analysis or any sequential data comparison. The `offset` parameter specifies how many rows away to look, and the `default` parameter provides a value if the offset goes beyond the window boundaries.

---

#### **Task 6: `CUME_DIST()` and `NTILE()` - Percentiles and Buckets**

**Description:**
`CUME_DIST()` calculates the cumulative distribution of a value within its partition. It returns a value between 0 and 1, representing the relative position of a row within a group. `NTILE(n)` divides the rows in an ordered partition into a specified number (`n`) of groups, assigning an integer rank from 1 to `n`.

1.  **Task 6.1: Cumulative Distribution of Freight Cost:** Calculate the cumulative distribution of `freight` costs for all orders.
2.  **Task 6.2: Product Price Quartiles per Category:** Divide products into 4 price quartiles (`NTILE(4)`) within each `category`.
3.  **Task 6.3: Top 3 Employee Sales Groups:** Divide all employees into 3 performance groups (`NTILE(3)`) based on their total sales revenue for 1997.

**SQL Query:**
```sql
-- Task 6.1: Cumulative Distribution of Freight Cost
SELECT
    order_id,
    freight,
    CUME_DIST() OVER (ORDER BY freight) AS freight_cume_dist
FROM
    orders
ORDER BY
    freight;

-- Task 6.2: Product Price Quartiles per Category
SELECT
    product_name,
    category_id,
    unit_price,
    NTILE(4) OVER (PARTITION BY category_id ORDER BY unit_price DESC) AS price_quartile
FROM
    products
ORDER BY
    category_id, unit_price DESC;

-- Task 6.3: Top 3 Employee Sales Groups
WITH EmployeeSales AS (
    SELECT
        employee_id,
        SUM(od.unit_price * od.quantity * (1 - od.discount))::NUMERIC(10, 2) AS total_revenue
    FROM
        orders AS o
    JOIN
        order_details AS od ON o.order_id = od.order_id
    WHERE EXTRACT(YEAR FROM o.order_date) = 1997
    GROUP BY
        employee_id
)
SELECT
    e.first_name || ' ' || e.last_name AS employee_name,
    es.total_revenue,
    NTILE(3) OVER (ORDER BY es.total_revenue DESC) AS performance_group
FROM
    EmployeeSales AS es
JOIN
    employees AS e ON es.employee_id = e.employee_id
ORDER BY
    performance_group, es.total_revenue DESC;
```

**Expected Output/Explanation:**
*   **Task 6.1:** `freight_cume_dist` will show the proportion of orders with `freight` less than or equal to the current row's freight value.
*   **Task 6.2:** `price_quartile` will assign products within each category into a 1, 2, 3, or 4 group based on their price relative to others in that category.
*   **Task 6.3:** `performance_group` will divide employees into three groups (1 being the top-performing group, 3 the lowest) based on their 1997 sales revenue.

**Learning Point:**
`CUME_DIST()` is useful for percentile analysis, identifying where a value falls within a sorted distribution. `NTILE()` is excellent for creating balanced groups or buckets (e.g., quartiles, deciles, custom groups) without manual calculations or complex `CASE` statements.

---

### **Post-Lab Activities:**

**Optional Challenges:**

1.  **Running Average Freight:** Modify Task 2.1 to calculate a 3-order *moving average* of `freight` for each customer, instead of a running total. (Hint: Research `ROWS BETWEEN` window frames).
2.  **Top N per Group (without `ROW_NUMBER`):** Find the top 2 most expensive products in each category using `RANK()` or `DENSE_RANK()`.
3.  **Difference from Previous Order Revenue:** For each customer, calculate the difference in `order_revenue` between the current order and their previous order.

**Further Exploration:**

*   **Window Frames:** Dive deeper into `ROWS BETWEEN`, `RANGE BETWEEN`, and custom window frames (`WINDOW name AS (...)`).
*   **Other Window Functions:** Explore `NTH_VALUE()`, `PERCENT_RANK()`, and `APPROX_PERCENTILE()`.
*   **Performance:** Understand how `PARTITION BY` and `ORDER BY` affect the performance of window functions, especially on large datasets.

---

### **Conclusion:**

In this lab, you have gained practical expertise in a wide array of PostgreSQL Window Functions. You successfully applied `ROW_NUMBER()`, `FIRST_VALUE()`, `LAST_VALUE()`, `RANK()`, `DENSE_RANK()`, `LAG()`, `LEAD()`, `CUME_DIST()`, and `NTILE()` to solve common analytical problems. Mastery of these functions, along with a solid understanding of the `OVER()` clause and its components, is crucial for advanced data analysis and business intelligence reporting in PostgreSQL.