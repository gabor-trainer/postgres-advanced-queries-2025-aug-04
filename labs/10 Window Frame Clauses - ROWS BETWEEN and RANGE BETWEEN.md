### **Window Frame Clauses: ROWS BETWEEN and RANGE BETWEEN**

**Objective:**
This lab will clarify the distinctions between `ROWS BETWEEN` and `RANGE BETWEEN` frame clauses in PostgreSQL window functions. You will gain practical experience in defining window frames based on physical row counts versus logical value offsets, particularly observing their behavior when ties are present in the ordering column.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line).
4.  **Basic SQL Knowledge:** Familiarity with `SELECT`, `FROM`, `JOIN`, `WHERE`, and the basic structure of window functions with `OVER()` and `ORDER BY`.

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run `SELECT count(*) FROM products;`. The result should be `77`.

---

### **Lab Exercises**

**Introduction to Window Frames:**
Window functions operate on a defined "window" of rows. The `OVER()` clause specifies this window. Beyond `PARTITION BY` and `ORDER BY`, you can precisely control which rows are included in the calculation for the current row using a *frame clause*. The frame clause is specified after `ORDER BY` within the `OVER()` clause.

There are two primary types of frame clauses: `ROWS` and `RANGE`. Both define a subset of rows within the current partition, but they interpret the frame boundaries differently.

---

#### **Task 1: Understanding `ROWS BETWEEN` vs. `RANGE BETWEEN` (Theoretical Explanation)**

**Description:**
This task provides a theoretical overview of the `ROWS BETWEEN` and `RANGE BETWEEN` frame clauses and their critical differences.

*   **`ROWS BETWEEN` Clause:**
    *   Defines the window frame based on a fixed number of *physical rows* relative to the current row.
    *   It's concerned with the count of rows (e.g., "the previous 2 rows," "the current row and the next 10 rows").
    *   When ordering columns have tied values, `ROWS BETWEEN` still treats each physical row distinctly for determining the frame boundaries.
    *   Syntax: `ROWS BETWEEN <start_bound> AND <end_bound>`
        *   `<start_bound>` / `<end_bound>` can be `UNBOUNDED PRECEDING`, `N PRECEDING`, `CURRENT ROW`, `N FOLLOWING`, `UNBOUNDED FOLLOWING`.

*   **`RANGE BETWEEN` Clause:**
    *   Defines the window frame based on a *logical value offset* from the current row's ordering column.
    *   It's concerned with a value range in the `ORDER BY` column (e.g., "all rows where the value is within 5 of the current row's value").
    *   **Crucially, when ordering columns have tied values, `RANGE BETWEEN` includes *all* rows that have the same `ORDER BY` value as the current row in the frame, regardless of their physical position within the partition.** This can make the effective "count" of rows in the frame larger than expected.
    *   Requires exactly one `ORDER BY` column in the window definition, and it must be of a type that allows addition/subtraction (numeric or date/time).
    *   Syntax: `RANGE BETWEEN <start_bound> AND <end_bound>`
        *   `<start_bound>` / `<end_bound>` can be `UNBOUNDED PRECEDING`, `VALUE PRECEDING`, `CURRENT ROW`, `VALUE FOLLOWING`, `UNBOUNDED FOLLOWING`. `VALUE` refers to a specific offset relative to the `ORDER BY` column's value.

**Key Difference Summary:**
The core distinction lies in how they handle rows with identical values in the `ORDER BY` column. `ROWS` operates on a strict *physical count* of rows, even if those rows are tied. `RANGE` operates on a *logical value range*, ensuring all tied values within that range are included, effectively expanding the frame to cover all logically equivalent rows.

**Learning Point:**
This task establishes the foundational understanding of the two primary window frame clauses. It is crucial to internalize the "physical rows" versus "logical value offset" distinction, particularly concerning how tied values are treated.

---

#### **Task 2: Synthetic Example - Illustrating the Difference with Ties**

**Description:**
Let's create a simple dataset with tied values in an ordering column to clearly illustrate how `ROWS BETWEEN` and `RANGE BETWEEN` define their frames differently. We'll use a running sum as the window function.

**SQL Query:**
```sql
-- Create a synthetic dataset with tied values
WITH SampleData AS (
    SELECT 1 AS id, 10 AS value, 100 AS order_val UNION ALL
    SELECT 2, 20, 110 UNION ALL
    SELECT 3, 30, 110 UNION ALL -- Tied with row 2 on order_val
    SELECT 4, 40, 120 UNION ALL
    SELECT 5, 50, 120   -- Tied with row 4 on order_val
)
SELECT
    id,
    value,
    order_val,
    -- Using ROWS BETWEEN: Sum of current row and 1 physically preceding row
    SUM(value) OVER (ORDER BY order_val, id ROWS BETWEEN 1 PRECEDING AND CURRENT ROW) AS sum_rows_frame,
    -- Using RANGE BETWEEN: Sum of current row and values within 0 of current order_val (i.e., all tied rows with current order_val)
    SUM(value) OVER (ORDER BY order_val RANGE BETWEEN 0 PRECEDING AND CURRENT ROW) AS sum_range_frame
FROM
    SampleData
ORDER BY
    order_val, id;
```

**Expected Output/Explanation:**
Observe the `sum_rows_frame` and `sum_range_frame` columns, especially for `order_val = 110` and `120`.

*   **`sum_rows_frame`:** For `id=3` (order_val=110), `sum_rows_frame` will be `(30 + 20) = 50`. It explicitly looks at the physical row 2 and row 3.
*   **`sum_range_frame`:** For `id=3` (order_val=110), `sum_range_frame` will also be `(30 + 20) = 50`. Here, `RANGE BETWEEN 0 PRECEDING AND CURRENT ROW` means "include all rows whose `order_val` is within the range [current_order_val - 0, current_order_val + 0]". Since both rows 2 and 3 have `order_val = 110`, they are both included in the frame for each other when their `order_val` is the current row's reference point.

If the `RANGE` window frame in this example was `RANGE BETWEEN 10 PRECEDING AND CURRENT ROW`, and we had a row with `order_val = 100` and `value = 10`, and another with `order_val = 110` and `value = 20`, then for the row with `order_val = 110`, the `RANGE` frame would include both the `110` and `100` `order_val` rows because `110-10 = 100`. The sum would be `10+20=30`. With `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW`, it would only include the current and one previous row, regardless of its `order_val`. This example uses `0 PRECEDING` to simplify and focus on the 'all tied values' aspect.

**Learning Point:**
This task visually demonstrates how `ROWS BETWEEN` strictly counts physical rows while `RANGE BETWEEN` expands the frame to include all logically tied rows within the specified offset of the `ORDER BY` column's value.

---

#### **Task 3: Real-Life Examples: `ROWS BETWEEN`**

**Description:**
These tasks demonstrate practical applications of `ROWS BETWEEN` where the window frame is defined by a count of physical rows.

1.  **Task 3.1: 3-Order Moving Average of Order Freight per Customer:** For each customer, calculate the average `freight` cost of the current order and the two *physically preceding* orders, based on `order_id`. This creates a moving average based on a fixed count of past transactions.
2.  **Task 3.2: Count of Products Ordered in the Next 5 Sales Details:** For each `order_detail` line item, count `product_id`s in the current line and the *next five physically appearing* line items within the same `order_id`, ordered by `product_id`.

**SQL Query:**
```sql
-- Task 3.1: 3-Order Moving Average of Order Freight per Customer
SELECT
    customer_id,
    order_id,
    order_date,
    freight,
    AVG(freight) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS three_order_moving_avg_freight
FROM
    orders
ORDER BY
    customer_id, order_date;

-- Task 3.2: Count of Products Ordered in the Next 5 Sales Details
SELECT
    order_id,
    product_id,
    quantity,
    COUNT(product_id) OVER (
        PARTITION BY order_id
        ORDER BY product_id
        ROWS BETWEEN CURRENT ROW AND 5 FOLLOWING
    ) AS distinct_products_in_next_5_lines
FROM
    order_details
ORDER BY
    order_id, product_id;
```

**Expected Output/Explanation:**
*   **Task 3.1:** For each customer, the `three_order_moving_avg_freight` will display the average freight of the current order and its two immediate predecessors. The first order for a customer will only average itself, the second will average itself and the first, and so on.
*   **Task 3.2:** For each line item, `distinct_products_in_next_5_lines` will count unique products from the current line and the next five physically distinct line items within the same order.

**Learning Point:**
`ROWS BETWEEN` is ideal for "moving window" calculations based on a fixed count of prior, current, or subsequent events/records, irrespective of their exact value in the ordering column.

---

#### **Task 4: Real-Life Examples: `RANGE BETWEEN`**

**Description:**
These tasks demonstrate practical applications of `RANGE BETWEEN` where the window frame is defined by a logical offset based on the `ORDER BY` column's value.

1.  **Task 4.1: Average Price of Logically "Similar" Products per Category:** For each product, calculate the average `unit_price` of all other products within the same `category` that are priced within a $2.00 range (from current price - $2.00 to current price + $2.00).
2.  **Task 4.2: Total Freight for Orders within a 5-Day Window per Customer:** For each order, calculate the sum of `freight` costs for all orders placed by the *same customer* within a 5-day window centered on the current order's date (2 days preceding, 2 days following).

**SQL Query:**
```sql
-- Task 4.1: Average Price of Logically "Similar" Products per Category
SELECT
    product_name,
    category_id,
    unit_price,
    AVG(unit_price) OVER (
        PARTITION BY category_id
        ORDER BY unit_price
        RANGE BETWEEN 2.00 PRECEDING AND 2.00 FOLLOWING
    ) AS avg_price_similar_products
FROM
    products
ORDER BY
    category_id, unit_price;

-- Task 4.2: Total Freight for Orders within a 5-Day Window per Customer
SELECT
    customer_id,
    order_id,
    order_date,
    freight,
    SUM(freight) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
        RANGE BETWEEN INTERVAL '2 DAY' PRECEDING AND INTERVAL '2 DAY' FOLLOWING
    ) AS total_freight_5_day_window
FROM
    orders
ORDER BY
    customer_id, order_date;
```

**Expected Output/Explanation:**
*   **Task 4.1:** `avg_price_similar_products` will provide an average that includes the current product and any other products in the same category whose price falls within the specified $2.00 range of the current product's price. Notice how rows with the same `unit_price` (ties) will have the same `avg_price_similar_products` because `RANGE` includes all tied values.
*   **Task 4.2:** `total_freight_5_day_window` will sum the freight costs for all orders placed by that customer within a 5-day time frame (2 days before, current day, 2 days after).

**Learning Point:**
`RANGE BETWEEN` is highly valuable for calculations based on value proximity (e.g., "products within X price range," "events within Y time period"). It automatically includes all tied values, making it robust for such logical grouping.

---

### **Conclusion:**

In this lab, you have gained a clear understanding of the distinct behaviors of `ROWS BETWEEN` and `RANGE BETWEEN` frame clauses. You observed how `ROWS` operates on physical row counts, while `RANGE` operates on logical value offsets, especially when handling tied values in the `ORDER BY` column. Mastery of these frame clauses is crucial for performing precise and context-aware analytical calculations with window functions.