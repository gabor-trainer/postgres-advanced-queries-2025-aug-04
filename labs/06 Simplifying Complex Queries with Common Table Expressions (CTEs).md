### **Simplifying Complex Queries with Common Table Expressions (CTEs)**

**Objective:**
This lab will teach you how to use non-recursive Common Table Expressions (CTEs) to break down complex analytical queries into simple, logical, and readable steps. You will solve a multi-part business question by building a query where each CTE prepares data for the next, demonstrating how CTEs can replace complicated nested subqueries and improve query maintainability.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line).
4.  **Basic SQL Knowledge:** Familiarity with `SELECT`, `JOIN`, `GROUP BY`, and aggregate functions.

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run `SELECT count(*) FROM employees;`. The result should be `9`.

---

### **Lab Exercises**

**Introduction to the Scenario:**
The Northwind management team wants to analyze employee performance for the year 1997. They have a multi-part question: "For each sales employee, what was their total sales revenue for 1997, how many distinct customers did they serve, and how does their total revenue compare to the average revenue of all sales employees for that year?"

Answering this requires several layers of aggregation:
1.  Calculate revenue per order.
2.  Aggregate that data up to the employee level.
3.  Calculate the average revenue across all employees.
4.  Combine these results.

Attempting this with nested subqueries would be complex and difficult to read. CTEs provide a clean, step-by-step solution.

---

#### **Task 1: The Foundation - Calculating Revenue per Order**

**Description:**
First, create a query that calculates the total revenue for each individual order. This involves joining `order_details` and `orders`, applying the discount formula, and grouping by `order_id`. This will form the basis of our first CTE.

**SQL Query:**
```sql
-- Calculate the total revenue for each order placed in 1997
SELECT
  o.order_id,
  o.employee_id,
  o.customer_id,
  SUM(od.unit_price * od.quantity * (1 - od.discount)) AS order_revenue
FROM order_details AS od
JOIN orders AS o ON od.order_id = o.order_id
WHERE EXTRACT(YEAR FROM o.order_date) = 1997
GROUP BY o.order_id, o.employee_id, o.customer_id;
```

**Expected Output/Explanation:**
The query returns a list of orders from 1997, showing the `employee_id` and `customer_id` associated with each, along with its calculated total revenue. This result set is our foundational data.

**Learning Point:**
This is a standard aggregation query that prepares the detailed data needed for higher-level analysis. It will become the first logical block in our multi-step CTE query.

---

#### **Task 2: First CTE - Aggregating Sales per Employee**

**Description:**
Now, convert the query from Task 1 into a CTE named `OrderRevenue`. Then, write a main query that selects from this CTE to aggregate the data further. Calculate the total sales revenue and the count of distinct customers for each employee in 1997.

**SQL Query:**
```sql
-- Step 1: Define a CTE to pre-calculate revenue for each order in 1997
WITH OrderRevenue AS (
  SELECT
    o.employee_id,
    o.customer_id,
    SUM(od.unit_price * od.quantity * (1 - od.discount)) AS order_revenue
  FROM order_details AS od
  JOIN orders AS o ON od.order_id = o.order_id
  WHERE EXTRACT(YEAR FROM o.order_date) = 1997
  GROUP BY o.order_id, o.employee_id, o.customer_id
)
-- Step 2: Query the CTE to aggregate results for each employee
SELECT
  e.first_name || ' ' || e.last_name AS employee_name,
  SUM(orv.order_revenue) AS total_revenue,
  COUNT(DISTINCT orv.customer_id) AS distinct_customers
FROM OrderRevenue AS orv
JOIN employees AS e ON orv.employee_id = e.employee_id
GROUP BY e.employee_id, employee_name
ORDER BY total_revenue DESC;
```

**Expected Output/Explanation:**
The result shows a summary for each employee for 1997, listing their full name, total revenue, and the number of unique customers they managed. Margaret Peacock should appear at the top with the highest revenue.

**Learning Point:**
This task demonstrates the primary purpose of a CTE: to create a temporary, named result set (`OrderRevenue`) that can be referenced like a regular table in the main query. This simplifies the logic by separating the per-order calculation from the per-employee aggregation.

---

#### **Task 3: Chaining CTEs - Calculating the Overall Average**

**Description:**
Now, let's add the final piece of the analysis: comparing each employee to the average. To do this, we will chain two CTEs.
1.  The first CTE, `EmployeeSales`, will be the result of your query from Task 2 (calculating total sales per employee).
2.  The second CTE, `AverageSales`, will query the `EmployeeSales` CTE to calculate the single average revenue across all employees.
3.  The final `SELECT` statement will join `EmployeeSales` and `AverageSales` to show each employee's performance against the average.

**SQL Query:**
```sql
-- CTE 1: Aggregate sales and customer counts for each employee in 1997
WITH EmployeeSales AS (
  SELECT
    o.employee_id,
    SUM(od.unit_price * od.quantity * (1 - od.discount)) AS total_revenue,
    COUNT(DISTINCT o.customer_id) AS distinct_customers
  FROM order_details AS od
  JOIN orders AS o ON od.order_id = o.order_id
  WHERE EXTRACT(YEAR FROM o.order_date) = 1997
  GROUP BY o.employee_id
),
-- CTE 2: Calculate the overall average revenue from the first CTE
AverageSales AS (
  SELECT AVG(total_revenue) AS overall_average_revenue
  FROM EmployeeSales
)
-- Final Query: Combine the results to show individual vs. average performance
SELECT
  e.first_name || ' ' || e.last_name AS employee_name,
  es.total_revenue,
  es.distinct_customers,
  avg.overall_average_revenue,
  (es.total_revenue - avg.overall_average_revenue) AS difference_from_average
FROM EmployeeSales AS es
JOIN employees AS e ON es.employee_id = e.employee_id
CROSS JOIN AverageSales AS avg -- CROSS JOIN is appropriate as AverageSales has only one row
ORDER BY es.total_revenue DESC;
```

**Expected Output/Explanation:**
The final report now includes each employee's total revenue, customer count, the overall average revenue for 1997, and a calculated column showing how far above or below the average each employee performed.

**Learning Point:**
This task demonstrates the power of chaining CTEs. The first CTE (`EmployeeSales`) acts as a source table for the second CTE (`AverageSales`). The final `SELECT` statement can then easily combine results from both named result sets. This creates a highly readable and self-documenting query that is easy to debug and modify.

---

### **Post-Lab Activities**

**Optional Challenges:**

1.  **Add Ranking:** Modify the final query to include a ranking of employees based on their `total_revenue`. You will need to research window functions like `RANK()` or `DENSE_RANK()`. Can you add this rank column using a third CTE?
2.  **Filter by Performance:** Use the final query as a subquery (or another CTE) to show only the employees who performed *above* the average.

**Further Exploration:**

*   CTEs are not just for `SELECT` statements. You can use them with `INSERT`, `UPDATE`, and `DELETE` statements. Research "Data-Modifying Statements in WITH".
*   The next logical topic is recursive CTEs, which are used for querying hierarchical data, such as organizational charts or parts explosions.

---

### **Conclusion**

In this lab, you have learned how to structure complex queries using Common Table Expressions. You practiced creating a foundational CTE, aggregating data from it, and then chaining multiple CTEs to build a sophisticated, multi-step analysis. This approach significantly improves the readability and maintainability of your SQL code compared to using deeply nested subqueries, making it an essential skill for any data professional.