### **Advanced Data Manipulation with PostgreSQL**

**Objective:**
This lab provides hands-on experience with advanced PostgreSQL DML operations. You will learn how to perform an "UPSERT" using `INSERT ... ON CONFLICT`, leverage the `RETURNING` clause to get immediate feedback from data modifications, and update tables by joining them to other data sources. These are critical, high-performance techniques for modern application development and data integration tasks.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line).
4.  **Basic SQL Knowledge:** Familiarity with `INSERT`, `UPDATE`, and `DELETE` statements.

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run `SELECT count(*) FROM products;`. The result should be `77`.

---

### **Lab Exercises**

**Introduction to the Scenario:**
In many business applications, you need to synchronize data. For example, you might receive a daily file of product price updates. Some products in the file may be new and need to be inserted, while others are existing products that need their prices updated. Performing this efficiently without errors is key. This lab simulates such scenarios.

---

#### **Task 1: Creating a Staging Table for Updates**

**Description:**
First, we need a place to hold our incoming data before applying it to the main `products` table. Create a temporary table named `product_updates` that has the same structure as `products`. Then, insert a few records into it: one for a brand new product and two for existing products with updated prices.

**SQL Query:**
```sql
-- Create a temporary table to stage our changes
CREATE TEMP TABLE product_updates (
    product_id smallint NOT NULL,
    product_name character varying(40) NOT NULL,
    unit_price real,
    units_in_stock smallint
);

-- Insert records: one new product and two updates for existing products
INSERT INTO product_updates (product_id, product_name, unit_price, units_in_stock)
VALUES
    (1, 'Chai', 19.50, 45),       -- Existing Product ID 1 (Chai) with a new price
    (2, 'Chang', 21.00, 20),      -- Existing Product ID 2 (Chang) with a new price
    (78, 'Peruvian Coffee', 25.00, 50); -- A completely new product
```

**Expected Output/Explanation:**
The query will execute without returning a result set, but it creates and populates the `product_updates` table for use in the following tasks. You can verify its contents with `SELECT * FROM product_updates;`.

**Learning Point:**
This task demonstrates a common ETL (Extract, Transform, Load) pattern: creating a temporary staging table to hold incoming data before merging it into a production table.

---

#### **Task 2: Performing an UPSERT with `ON CONFLICT DO UPDATE`**

**Description:**
Now, apply the changes from `product_updates` to the main `products` table. For any product that already exists (based on a conflict on `product_id`), you must update its `unit_price` and `units_in_stock`. For any new product, you must insert it. Use the `INSERT ... ON CONFLICT` statement to perform this "UPSERT" operation in a single, atomic command.

**SQL Query:**
```sql
-- Wrap in a transaction to observe changes and then revert
BEGIN;

-- Perform the UPSERT operation
INSERT INTO products (product_id, product_name, unit_price, units_in_stock, discontinued)
SELECT product_id, product_name, unit_price, units_in_stock, 0 FROM product_updates
ON CONFLICT (product_id) DO UPDATE SET
  unit_price = EXCLUDED.unit_price,
  units_in_stock = EXCLUDED.units_in_stock;

-- Verify the changes before rolling back
SELECT product_id, product_name, unit_price, units_in_stock
FROM products
WHERE product_id IN (1, 2, 78);

ROLLBACK;
```

**Expected Output/Explanation:**
The `SELECT` statement within the transaction will show that "Chai" now has a price of `19.50`, "Chang" has a price of `21.00`, and the new "Peruvian Coffee" (product 78) has been added to the table. The `ROLLBACK` ensures your original data remains unchanged after the lab.

**Learning Point:**
This task demonstrates the `INSERT ... ON CONFLICT DO UPDATE` statement. This is the standard PostgreSQL way to perform an UPSERT. The `EXCLUDED` pseudo-table is used to reference the values from the row that was proposed for insertion but caused the conflict.

---

#### **Task 3: Ignoring Duplicates with `ON CONFLICT DO NOTHING`**

**Description:**
Imagine a different business rule: you only want to insert new products from the update list and completely ignore any products that already exist. Modify the query from Task 2 to use the `DO NOTHING` action upon conflict.

**SQL Query:**
```sql
BEGIN;

-- Attempt to insert, but do nothing if the product already exists
INSERT INTO products (product_id, product_name, unit_price, units_in_stock, discontinued)
SELECT product_id, product_name, unit_price, units_in_stock, 0 FROM product_updates
ON CONFLICT (product_id) DO NOTHING;

-- Verify the changes
-- Original prices for Chai and Chang should be unchanged
SELECT product_id, product_name, unit_price FROM products WHERE product_id IN (1, 2);

-- The new product should exist
SELECT product_id, product_name, unit_price FROM products WHERE product_id = 78;

ROLLBACK;
```

**Expected Output/Explanation:**
Within the transaction, the query for products 1 and 2 will show their original prices (18.00 and 19.00). The query for product 78 will show that it was successfully inserted. This confirms that the existing records were ignored and only the new record was added.

**Learning Point:**
This task introduces the `DO NOTHING` clause for `ON CONFLICT`, a useful tool for preventing duplicate key errors when you only care about inserting new records and can safely discard conflicting ones.

---

#### **Task 4: Capturing Results with `UPDATE ... RETURNING`**

**Description:**
A supplier has announced a 10% price increase on all products in the "Confections" category (category_id = 3). Write an `UPDATE` statement to apply this price increase. Crucially, use the `RETURNING` clause to immediately get a report of the `product_id`, `product_name`, the old price, and the newly calculated price for all affected products.

**SQL Query:**
```sql
BEGIN;

-- Update prices and return a before-and-after report
UPDATE products
SET unit_price = unit_price * 1.10
WHERE category_id = 3
RETURNING
    product_id,
    product_name,
    (unit_price / 1.10) AS original_price, -- Calculate the old price for the report
    unit_price AS new_price;

ROLLBACK;
```

**Expected Output/Explanation:**
The query will directly return a table with four columns for all products in the "Confections" category. For example, for "Teatime Chocolate Biscuits", you will see the `original_price` (around 9.20) and the `new_price` (around 10.12). This immediate feedback is invaluable for logging and auditing.

**Learning Point:**
This task demonstrates the power of the `RETURNING` clause on an `UPDATE` statement. It allows you to retrieve data from the rows that were just modified without needing a separate `SELECT` query, which is both efficient and guarantees you are seeing the result of that specific transaction.

---

#### **Task 5: Updating from a Staging Table (Join-style `UPDATE`)**

**Description:**
The `product_updates` staging table from Task 1 contains new `units_in_stock` counts. Write an `UPDATE` statement that joins the `products` table to `product_updates` on `product_id` and updates the `units_in_stock` in the main table with the values from the staging table.

**SQL Query:**
```sql
BEGIN;

-- Update the products table using values from the product_updates table
UPDATE products p
SET
  units_in_stock = pu.units_in_stock
FROM
  product_updates pu
WHERE
  p.product_id = pu.product_id;

-- Verify the stock levels for Chai and Chang have been updated
SELECT product_id, product_name, units_in_stock FROM products WHERE product_id IN (1, 2);

ROLLBACK;
```

**Expected Output/Explanation:**
The verification query will show that "Chai" now has 45 units in stock and "Chang" has 20, reflecting the data from the `product_updates` table.

**Learning Point:**
This task showcases the PostgreSQL-specific `UPDATE ... FROM` syntax, which allows you to perform an update based on a join to another table. This is a highly efficient pattern for applying batch updates from a staging table or another data source.

---

### **Post-Lab Activities**

**Optional Challenges:**

1.  **DELETE with RETURNING:** A product, "Guaraná Fantástica" (product_id 24), has been recalled. Write a `DELETE` statement to remove it from the `products` table and use `RETURNING *` to get a complete record of the product that was just deleted. (Remember to use a transaction!).
2.  **Conditional UPSERT:** Modify the `ON CONFLICT` query from Task 2. Add a `WHERE` clause to the `DO UPDATE` part so that it only updates the price if the new price is actually higher than the old price (`EXCLUDED.unit_price > products.unit_price`).

---

### **Conclusion**

In this lab, you have practiced several advanced DML patterns that are essential for building robust data management applications. You learned how to handle inserts and updates simultaneously with `ON CONFLICT` (UPSERT), how to efficiently get data back from `INSERT`, `UPDATE`, and `DELETE` operations using `RETURNING`, and how to perform bulk updates by joining tables. These techniques are fundamental for tasks involving data synchronization, auditing, and high-performance batch processing.