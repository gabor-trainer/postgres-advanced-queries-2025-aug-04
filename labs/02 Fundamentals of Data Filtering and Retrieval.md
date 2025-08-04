### Fundamentals of Data Filtering and Retrieval

**Objective:**
This lab will guide you through the fundamental clauses and functions of the `SELECT` statement to filter, format, and limit data from the Northwind database. By the end, you will be proficient in using comparison operators, pattern matching, handling `NULL` values, retrieving unique records, and implementing pagination.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line).
4.  **Basic SQL Knowledge:** Familiarity with the basic structure of a `SELECT` statement.

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run a simple query to confirm connectivity and data presence (`SELECT count(*) FROM customers;`). The result should be `91`.

---

### **Lab Exercises:**

**Introduction to Tables Used:**
We will primarily be working with the `products`, `customers`, and `orders` tables. These tables contain information about company products, client details, and sales transactions, respectively.

---

#### **Task 1: Filtering Data with Comparison Operators**

**Description:**
The first step in data analysis is often to isolate a subset of data. Your task is to query the `products` table to find all products with a `unit_price` greater than $50. Order the results from the most expensive to the least expensive to make the data easy to review.

**SQL Query:**

```sql
-- Retrieve expensive products, ordered by price
SELECT
  product_name,
  unit_price
FROM products
WHERE
  unit_price > 50
ORDER BY
  unit_price DESC;
```

**Expected Output/Explanation:**
The result set will contain two columns: `product_name` and `unit_price`. You will see a list of high-value products, with "Côte de Blaye" at a price of 263.50 at the top of the list.

**Learning Point:**
This task introduces the `WHERE` clause for numerical filtering using comparison operators (`>`). It also incorporates `ORDER BY ... DESC` to sort the result set in descending order.

---

#### **Task 2: Pattern Matching with `LIKE` and `ILIKE`**

**Description:**
You often need to find records based on partial text. First, perform a case-sensitive search to find all products whose name begins with 'Ch'. Second, perform a case-insensitive search on the `customers` table to find any `company_name` that contains the word 'market', regardless of its casing (e.g., 'market', 'Market', 'MARKET').

**SQL Query:**

```sql
-- Part 1: Case-sensitive search for products starting with 'Ch'
SELECT product_name
FROM products
WHERE product_name LIKE 'Ch%';

-- Part 2: Case-insensitive search for customers with 'market' in their name
SELECT company_name
FROM customers
WHERE company_name ILIKE '%market%';
```

**Expected Output/Explanation:**
The first query will return products like "Chai" and "Chang". The second query will return "Bottom-Dollar Markets", "Great Lakes Food Market", "Lehmanns Marktstand", and "White Clover Markets", demonstrating the utility of `ILIKE` for finding text without enforcing case rules.

**Learning Point:**
This task demonstrates pattern matching using `LIKE` (case-sensitive) and its PostgreSQL-specific, case-insensitive variant `ILIKE`. The `%` wildcard is used to match any sequence of characters.

---

#### **Task 3: Handling NULL Values and Aliasing Columns**

**Description:**
The `customers` table has a `region` column that contains `NULL` values for some records. Write a query that lists the `company_name` and `region`. If the `region` is `NULL`, display the string 'N/A' instead. To improve readability, rename the resulting column to `display_region`.

**SQL Query:**

```sql
-- Replace NULL region values and provide a column alias
SELECT
  company_name,
  COALESCE(region, 'N/A') AS display_region
FROM customers;
```

**Expected Output/Explanation:**
The result set will show two columns. For customers in countries like Germany or Mexico, where the `region` is not specified, you will see 'N/A' in the `display_region` column instead of a blank or `NULL` value.

**Learning Point:**
This task demonstrates how to handle `NULL` values using the `COALESCE` function and how to assign a custom, readable name to a result column using the `AS` keyword (an alias).

---

#### **Task 4: Retrieving Unique Values with `DISTINCT`**

**Description:**
To generate a report of all countries where Northwind has customers, you need a unique list of countries from the `customers` table. Write a query to produce this list, sorted alphabetically.

**SQL Query:**

```sql
-- Get a unique, sorted list of countries
SELECT DISTINCT
  country
FROM customers
ORDER BY
  country;
```

**Expected Output/Explanation:**
The query will return a single column containing a list of 21 unique countries, starting with "Argentina" and ending with "Venezuela". Each country will appear only once, no matter how many customers are located there.

**Learning Point:**
This task introduces the `DISTINCT` keyword to eliminate duplicate rows from a result set, providing a clean list of unique values.

---

#### **Task 5: Paginating Results with `LIMIT` and `OFFSET`**

**Description:**
Imagine you are building an application interface that displays products on pages. First, write a query to show the top 5 most expensive products. Next, write a second query to show the *next* 5 most expensive products (i.e., the products that would appear on the second page).

**SQL Query:**

```sql
-- Query for Page 1: Top 5 most expensive products
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC
LIMIT 5;

-- Query for Page 2: Products 6 through 10
SELECT product_name, unit_price
FROM products
ORDER BY unit_price DESC
OFFSET 5
LIMIT 5;
```

**Expected Output/Explanation:**
The first query will list the top 5 products, starting with "Côte de Blaye". The second query will start with "Sir Rodney's Marmalade" (the 6th most expensive product) and list the next five from there.

**Learning Point:**
This task demonstrates how to control the number of rows returned using `LIMIT` and how to skip rows using `OFFSET` to implement data pagination, a common requirement in application development.

---

#### **Task 6: Introduction to Subqueries with `NOT IN`**

**Description:**
A key business question is, "Which of our customers have never placed an order?" To answer this, you need to find all `customer_id`s that exist in the `customers` table but do *not* exist in the `orders` table. Use a subquery to accomplish this.

**SQL Query:**

```sql
-- Find customers who have never placed an order
SELECT company_name
FROM customers
WHERE
  customer_id NOT IN (
    SELECT DISTINCT customer_id
    FROM orders
  );
```

**Expected Output/Explanation:**
The query will return a small list of customers who have no corresponding entries in the `orders` table. You should see two results: "Paris spécialités" and "FISSA Fabrica Inter. Salchichas S.A.".

**Learning Point:**
This task introduces subqueries. The inner query (`SELECT DISTINCT customer_id FROM orders`) runs first, generating a complete list of ordering customers. The outer query then uses this list as a filter condition with the `NOT IN` operator to find the customers who are not in that list.

---

### **Post-Lab Activities:**

**Optional Challenges:**

1.  Find all unique cities in 'Germany' that have customers.
2.  List the `company_name` and `contact_name` for the first 5 customers located in the 'UK', ordered by `company_name`.
3.  Find all employees whose `title` contains the word 'Sales' but is not 'Sales Manager'. (Hint: Use `ILIKE` and the `<>` not-equal operator).

**Further Exploration:**

*   The logical next step is to combine data from multiple tables. Research the `JOIN` clause (`INNER JOIN`, `LEFT JOIN`).
*   Explore aggregate functions (`COUNT`, `SUM`, `AVG`) in combination with the `GROUP BY` clause to summarize data. For example, try to count the number of customers in each country.

---

### **Conclusion:**

In this lab, you have practiced the essential techniques for querying and filtering data in PostgreSQL. You have used the `WHERE` clause with various conditions, applied pattern matching with `LIKE` and `ILIKE`, managed `NULL` values with `COALESCE`, created readable column aliases, isolated unique records with `DISTINCT`, implemented pagination with `LIMIT` and `OFFSET`, and utilized a subquery for advanced filtering. These skills are the foundation for all subsequent data analysis and manipulation tasks.