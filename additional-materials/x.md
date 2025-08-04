## **PostgreSQL Core Functions, Variables, and Idioms: A Professional Guide**

### **Introduction**


All examples are written for a standard PostgreSQL environment and utilize the `northwind` database schema. Queries can be executed directly in pgAdmin's Query Tool or via `psql`.

---

### **1. Essential Built-in Functions**

PostgreSQL offers a comprehensive library of functions. The following are among the most critical for daily operations.

#### **A. String Functions**

These functions are fundamental for data cleaning, formatting for reports, and dynamic query generation.

| Function                       | Description                                                                        | Northwind Example                                                                              |
| :----------------------------- | :--------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------- |
| `CONCAT()` / `                 |                                                                                    | `                                                                                              | Concatenates two or more strings. The ` |  | ` operator is the standard SQL equivalent. | `SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM employees;` |
| `LOWER()`, `UPPER()`           | Converts a string to all lowercase or all uppercase.                               | `SELECT company_name, UPPER(country) FROM customers WHERE country = 'Germany';`                |
| `LENGTH()`                     | Returns the number of characters in a string.                                      | `SELECT product_name, LENGTH(product_name) AS name_length FROM products;`                      |
| `SUBSTRING()`                  | Extracts a substring based on a starting position and length.                      | `SELECT postal_code, SUBSTRING(postal_code FROM 1 FOR 3) FROM customers WHERE country = 'UK';` |
| `POSITION()`                   | Finds the position of a substring within another string. Returns 0 if not found.   | `SELECT address, POSITION(' ' IN address) AS first_space FROM employees;`                      |
| `TRIM()`, `LTRIM()`, `RTRIM()` | Removes leading, trailing, or both-sided whitespace or specified characters.       | `SELECT TRIM(both ' ' from territory_description) FROM territories;`                           |
| `SPLIT_PART()`                 | Splits a string on a specified delimiter and returns the nth part (1-based index). | `SELECT photo_path, SPLIT_PART(photo_path, '/', 4) AS filename FROM employees;`                |
| `REPLACE()`                    | Replaces all occurrences of a specified substring with another substring.          | `SELECT address, REPLACE(address, 'Ave.', 'Avenue') FROM employees;`                           |

#### **B. Numeric Functions**

These are critical for calculations in financial analysis, statistical reporting, and data transformation.

| Function  | Description                                                     | Northwind Example                                                      |
| :-------- | :-------------------------------------------------------------- | :--------------------------------------------------------------------- |
| `ROUND()` | Rounds a numeric value to a specified number of decimal places. | `SELECT unit_price, ROUND(unit_price) AS rounded_price FROM products;` |
| `CEIL()`  | Rounds a number up to the nearest integer.                      | `SELECT freight, CEIL(freight) AS freight_rounded_up FROM orders;`     |
| `FLOOR()` | Rounds a number down to the nearest integer.                    | `SELECT freight, FLOOR(freight) AS freight_rounded_down FROM orders;`  |
| `ABS()`   | Returns the absolute (non-negative) value of a number.          | `SELECT ABS(-15.5);`                                                   |

#### **C. Date/Time Functions**

Essential for time-series analysis, business intelligence reporting, and managing temporal data.

| Function            | Description                                                                     | Northwind Example                                                                       |
| :------------------ | :------------------------------------------------------------------------------ | :-------------------------------------------------------------------------------------- |
| `NOW()`             | Returns the current timestamp with time zone.                                   | `SELECT NOW();`                                                                         |
| `CURRENT_DATE`      | Returns the current date.                                                       | `SELECT CURRENT_DATE;`                                                                  |
| `AGE()`             | Calculates the interval between two timestamps. Often used to determine age.    | `SELECT first_name, last_name, AGE(birth_date) AS current_age FROM employees;`          |
| `EXTRACT()`         | Extracts a specific field (e.g., year, month, day, dow) from a date/timestamp.  | `SELECT order_date, EXTRACT(YEAR FROM order_date) AS order_year FROM orders;`           |
| `DATE_TRUNC()`      | Truncates a timestamp to a specified precision (e.g., month, week).             | `SELECT order_date, DATE_TRUNC('month', order_date) AS month FROM orders;`              |
| `TO_CHAR()`         | Formats a date or numeric value into a string based on a specified format mask. | `SELECT order_date, TO_CHAR(order_date, 'YYYY-Mon-DD') AS formatted_date FROM orders;`  |
| Interval Arithmetic | Add or subtract intervals from timestamps.                                      | `SELECT hire_date, hire_date + INTERVAL '1 month' AS first_review_date FROM employees;` |

#### **D. Aggregate Functions**

The foundation of all analytical queries. Used with the `GROUP BY` clause.

| Function         | Description                                                                                    | Northwind Example                                                                         |
| :--------------- | :--------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------------------------- |
| `COUNT()`        | Counts the number of rows. `COUNT(*)` counts all rows; `COUNT(column)` counts non-null values. | `SELECT country, COUNT(*) FROM customers GROUP BY country;`                               |
| `SUM()`          | Calculates the sum of a set of numeric values.                                                 | `SELECT SUM(unit_price * quantity * (1 - discount)) AS total_revenue FROM order_details;` |
| `AVG()`          | Calculates the average of a set of numeric values.                                             | `SELECT category_id, AVG(unit_price) FROM products GROUP BY category_id;`                 |
| `MIN()`, `MAX()` | Returns the minimum or maximum value in a set.                                                 | `SELECT MIN(order_date) AS first_order, MAX(order_date) AS last_order FROM orders;`       |

#### **E. Conditional Expressions**

Used to implement logic within queries, essential for data transformation and custom reporting columns.

| Expression   | Description                                                                      | Northwind Example                                                                                                                                                          |
| :----------- | :------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CASE`       | The standard SQL way to create conditional logic (if/then/else).                 | `SELECT product_name, unit_price, CASE WHEN unit_price < 20 THEN 'Inexpensive' WHEN unit_price < 50 THEN 'Moderate' ELSE 'Expensive' END AS price_category FROM products;` |
| `COALESCE()` | Returns the first non-null argument from a list of expressions.                  | `SELECT company_name, region, COALESCE(region, 'N/A') AS display_region FROM customers;`                                                                                   |
| `NULLIF()`   | Returns `NULL` if two arguments are equal, otherwise returns the first argument. | `SELECT product_name, NULLIF(units_on_order, 0) AS on_order FROM products;`                                                                                                |

---

### **2. System Information and Session Variables**

PostgreSQL provides numerous functions and runtime parameters to inspect and control the current session's state. These are often referred to informally as "variables."

| Item                 | Type              | Description                                                                      | Example Usage                                              |
| :------------------- | :---------------- | :------------------------------------------------------------------------------- | :--------------------------------------------------------- |
| `current_user`       | Special Register  | Returns the username of the current execution context.                           | `SELECT current_user;`                                     |
| `session_user`       | Special Register  | Returns the username of the user who initiated the session.                      | `SELECT session_user;`                                     |
| `current_database()` | Function          | Returns the name of the current database.                                        | `SELECT current_database();`                               |
| `current_schema()`   | Function          | Returns the name of the current schema (first in the `search_path`).             | `SELECT current_schema();`                                 |
| `search_path`        | Runtime Parameter | Shows or sets the schema search path for the current session.                    | `SHOW search_path; SET search_path TO public, extensions;` |
| `work_mem`           | Runtime Parameter | Shows or sets the memory available for internal sort operations and hash tables. | `SHOW work_mem; SET work_mem = '128MB';`                   |

---

### **3. Common PostgreSQL Idioms and Patterns**

These are powerful, often PostgreSQL-specific, constructs that enable more efficient and readable queries.

#### **A. Common Table Expressions (`WITH` Clause)**

**Idiom:** Use a CTE to break down complex queries into logical, readable steps. It improves maintainability by replacing nested subqueries or temporary tables.

**Example:** Find the top 2 product categories by total sales revenue.

```sql
WITH CategorySales AS (
  SELECT
    p.category_id,
    c.category_name,
    SUM(od.unit_price * od.quantity * (1 - od.discount)) AS total_revenue
  FROM order_details od
  JOIN products p ON od.product_id = p.product_id
  JOIN categories c ON p.category_id = c.category_id
  GROUP BY p.category_id, c.category_name
)
SELECT
  category_name,
  total_revenue
FROM CategorySales
ORDER BY total_revenue DESC
LIMIT 2;
```

#### **B. Returning Data from DML Statements (`RETURNING`)**

**Idiom:** Append a `RETURNING` clause to `INSERT`, `UPDATE`, or `DELETE` statements to retrieve values from the modified rows without needing a subsequent `SELECT` query. This is highly efficient.

**Example:** Insert a new shipper and immediately return its system-generated `shipper_id`. (Use a transaction to avoid permanent changes during practice).

```sql
BEGIN;

INSERT INTO shippers (company_name, phone)
VALUES ('Training Logistics', '(555) 555-1234')
RETURNING shipper_id, company_name;

-- This will revert the INSERT statement.
ROLLBACK;
```

#### **C. Getting the Top N per Group (`DISTINCT ON`)**

**Idiom:** Use PostgreSQL's `DISTINCT ON (group_column)` combined with `ORDER BY` to efficiently select the first row for each distinct group. This is often more concise and performant than window function alternatives for this specific task.

**Example:** Find the most recent order for each customer.

```sql
SELECT DISTINCT ON (customer_id)
  customer_id,
  order_id,
  order_date
FROM orders
ORDER BY customer_id, order_date DESC;
```

#### **D. Generating Series (`generate_series`)**

**Idiom:** Use `generate_series()` to create a series of numbers or timestamps. This is invaluable for creating report templates, finding gaps in data, or generating sample data.

**Example:** Create a list of all dates in January 1997.

```sql
SELECT day::date
FROM generate_series(
  '1997-01-01'::timestamp,
  '1997-01-31'::timestamp,
  '1 day'::interval
) AS day;
```

#### **E. Conditional Aggregation (`FILTER` Clause)**

**Idiom:** Use the `FILTER (WHERE ...)` clause with aggregate functions to perform calculations over a subset of rows. It is SQL-standard and often more readable than using a `CASE` statement inside the aggregate.

**Example:** Count total orders and just the orders shipped to Germany for each customer.

```sql
SELECT
  customer_id,
  COUNT(*) AS total_orders,
  COUNT(*) FILTER (WHERE ship_country = 'Germany') AS german_orders
FROM orders
GROUP BY customer_id
ORDER BY german_orders DESC, total_orders DESC;
```

