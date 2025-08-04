### **The Danger of NULLs in Subqueries**

**Objective:**
This lab will demonstrate a critical, non-intuitive behavior in SQL where a `NULL` value returned by a subquery used with a `NOT IN` predicate can lead to an incorrect, empty result set. You will learn why this occurs and master the correct, robust patterns (`IS NOT NULL` and `NOT EXISTS`) to prevent this common error.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line).
4.  **Basic SQL Knowledge:** Familiarity with `SELECT`, `FROM`, `WHERE`, and basic subqueries.

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run `SELECT count(*) FROM shippers;`. The result should be `6`.

---

### **Lab Exercises**

**Introduction to the Scenario:**
A common business request is to find entities that are *not* associated with a certain activity. For example, "find customers who have never ordered" or "find products never sold." A frequent, but potentially flawed, approach is to use a `NOT IN` clause with a subquery. This lab exposes the danger of that pattern.

We will investigate which shippers have *not* handled a specific set of recent orders.

---

#### **Task 1: The Baseline - A `NOT IN` Query with Clean Data**

**Description:**
First, let's establish a baseline. We want to identify any shippers who were **not** used for the first three orders in the system (`order_id` 10248, 10249, 10250). The subquery will return a list of `shipper_id` values that were used, and the outer query will find the shippers not in that list.

**SQL Query:**
```sql
-- Find shippers NOT used for the first three orders
SELECT company_name
FROM shippers
WHERE shipper_id NOT IN (
    SELECT ship_via
    FROM orders
    WHERE order_id IN (10248, 10249, 10250)
);
```

**Expected Output/Explanation:**
The subquery correctly identifies that shippers `1`, `2`, and `3` were used. The query then correctly returns the shippers who were not in that list. The output should be:
```
company_name
------------------
Alliance Shippers
UPS
DHL
```
This result is correct and intuitive.

**Learning Point:**
This task demonstrates a standard `NOT IN` subquery that works as expected when the data returned by the subquery is clean (i.e., contains no `NULL` values).

---

#### **Task 2: The Anomaly - Introducing a `NULL`**

**Description:**
Now, let's simulate a data-entry error. Imagine an order was created in the system, but its shipper was never assigned, leaving `ship_via` as `NULL`. We will temporarily add such an order and run the *exact same query* from Task 1. To avoid permanently changing our data, we will wrap this exercise in a transaction and then roll it back.

**SQL Query:**
```sql
-- Start a transaction to keep our changes temporary
BEGIN;

-- Simulate a bad record: an order with a NULL shipper
INSERT INTO orders (order_id, customer_id, employee_id, order_date, ship_via)
VALUES (99999, 'VINET', 1, CURRENT_DATE, NULL);

-- !! RERUN THE EXACT SAME QUERY FROM TASK 1 !!
SELECT company_name
FROM shippers
WHERE shipper_id NOT IN (
    -- The subquery now includes the initial orders AND our bad record
    SELECT ship_via
    FROM orders
    WHERE order_id IN (10248, 10249, 10250, 99999) -- Added our new order
);

-- Clean up by rolling back the transaction
ROLLBACK;
```

**Expected Output/Explanation:**
The query now returns **zero rows**. This is incorrect. We know from Task 1 that three shippers should be listed. The presence of a single `NULL` in the subquery's result set (`{3, 1, 2, NULL}`) caused the entire query to fail silently by returning an empty set.

This happens because `shipper_id NOT IN (3, 1, 2, NULL)` is evaluated as `shipper_id <> 3 AND shipper_id <> 1 AND shipper_id <> 2 AND shipper_id <> NULL`. The comparison of any value to `NULL` results in `UNKNOWN`, not `TRUE` or `FALSE`. Since the `WHERE` clause only includes rows that evaluate to `TRUE`, no rows are returned.

**Learning Point:**
This task critically demonstrates that if a subquery in a `NOT IN` clause returns even one `NULL` value, the entire outer query will return an empty set. This is the "danger of `NULL`."

---

#### **Task 3: The Solution - Filtering `NULL`s from the Subquery**

**Description:**
The most direct way to fix the flawed query is to ensure the subquery can never return a `NULL` value. Modify the subquery from Task 2 to explicitly exclude rows where the `ship_via` column `IS NOT NULL`.

**SQL Query:**
```sql
-- We will use the same transaction block to simulate the bad data
BEGIN;

INSERT INTO orders (order_id, customer_id, employee_id, order_date, ship_via)
VALUES (99999, 'VINET', 1, CURRENT_DATE, NULL);

-- Rerun the query, but this time, fix the subquery
SELECT company_name
FROM shippers
WHERE shipper_id NOT IN (
    SELECT ship_via
    FROM orders
    WHERE order_id IN (10248, 10249, 10250, 99999)
    AND ship_via IS NOT NULL -- This is the fix
);

ROLLBACK;
```

**Expected Output/Explanation:**
The query now returns the correct list of three shippers again. By adding `AND ship_via IS NOT NULL`, we guarantee that the list passed to `NOT IN` does not contain `NULL`, preventing the logical failure.

**Learning Point:**
This task provides a direct solution to the `NOT IN` problem: always filter out potential `NULL` values from the subquery result set.

---

#### **Task 4: The Professional Standard - Using `NOT EXISTS`**

**Description:**
While filtering for `NULL` works, it requires you to remember to do it every time. A more robust and often more performant pattern is to use `NOT EXISTS`. Rewrite the query using a `NOT EXISTS` clause to achieve the same goal. `NOT EXISTS` is not susceptible to the `NULL` problem.

**SQL Query:**
```sql
-- We will use the same transaction block one last time
BEGIN;

INSERT INTO orders (order_id, customer_id, employee_id, order_date, ship_via)
VALUES (99999, 'VINET', 1, CURRENT_DATE, NULL);

-- Rewrite the query using the robust NOT EXISTS pattern
SELECT s.company_name
FROM shippers s
WHERE NOT EXISTS (
    SELECT 1 -- The value selected here does not matter
    FROM orders o
    WHERE o.ship_via = s.shipper_id
      AND o.order_id IN (10248, 10249, 10250, 99999)
);

ROLLBACK;
```

**Expected Output/Explanation:**
This query also returns the correct list of three shippers, even with the `NULL` value present in the underlying data. The `NOT EXISTS` clause checks for the existence of rows satisfying the correlated condition (`o.ship_via = s.shipper_id`). When `o.ship_via` is `NULL`, the equality check fails (evaluates to `UNKNOWN`), so no row is found for that `NULL`, and it correctly does not interfere with the logic for other rows.

**Learning Point:**
This task demonstrates the `NOT EXISTS` pattern, which is the preferred professional standard for performing "anti-joins" (finding records in one table with no corresponding records in another). It is safer because it is immune to the `NULL` issue and often performs better.

---

### **Post-Lab Activities**

**Optional Challenges:**

1.  Think about the `employees` table. The `reports_to` column is nullable (the CEO reports to no one). How would you write a query to find all employees who do *not* manage anyone? Test both a `NOT IN` and a `NOT EXISTS` approach to see the results.
2.  The `orders` table's `customer_id` is also nullable. Can you construct a scenario similar to this lab using that column?

---

### **Conclusion**

In this lab, you have witnessed firsthand how a `NULL` value can cause a `NOT IN` subquery to fail silently and produce incorrect results. You learned that this is due to SQL's three-valued logic. Most importantly, you have practiced two methods to mitigate this risk: explicitly filtering `NULL`s from the subquery and, more robustly, using the `NOT EXISTS` pattern as the professional standard for this type of query.