### **Navigating Hierarchical Data with Recursive Common Table Expressions (CTEs)**

**Objective:**
This lab will teach you how to effectively query hierarchical data structures in PostgreSQL using Recursive Common Table Expressions (CTEs). You will learn to identify the limitations of non-recursive SQL for such problems and then construct a recursive CTE to flatten an organizational hierarchy, listing all direct and indirect subordinates for each employee.

**Prerequisites:**
Before starting this lab, please ensure you have the following:

1.  **PostgreSQL Server:** A running PostgreSQL instance (version 12 or higher recommended).
2.  **Northwind Database:** The `northwind` database schema and data must be already installed and accessible on your PostgreSQL server.
3.  **SQL Client:** Access to a PostgreSQL client tool (e.g., pgAdmin Query Tool, psql command line).
4.  **Basic SQL Knowledge:** Familiarity with `SELECT`, `JOIN`, `GROUP BY`, and standard (non-recursive) CTEs.

**Environment Setup:**

1.  **Connect to Database:** Open your preferred SQL client and connect to the `northwind` database.
2.  **Verify Connection:** Run `SELECT count(*) FROM employees;`. The result should be `9`.

---

### **Lab Exercises**

**Introduction to the Scenario:**
The `employees` table in the Northwind database is structured hierarchically using the `reports_to` column, which references the `employee_id` of the employee's manager. This creates a classic organizational tree. A common business requirement is to generate a list for each manager, detailing all the employees who report to them, whether directly or indirectly.

---

#### **Task 1: Listing Direct Reports (Non-Recursive Approach and Its Limitation)**

**Description:**
First, let's try to solve this problem using only non-recursive SQL. Write a query that lists each manager's name and a comma-separated list of their *direct* subordinates. This involves a self-join on the `employees` table. Observe that this approach cannot capture indirect (sub-level) reports.

**SQL Query:**

```sql
-- List each employee and their direct reports
SELECT
    M.first_name || ' ' || M.last_name AS manager_name,
    STRING_AGG(E.first_name || ' ' || E.last_name, ', ')
        AS direct_subordinates
FROM
    employees AS M -- Manager
JOIN
    employees AS E ON E.reports_to = M.employee_id -- Employee reports to Manager
GROUP BY
    M.employee_id,
    manager_name
ORDER BY
    M.employee_id;
```

**Expected Output/Explanation:**
You will see "Andrew Fuller" listed with his direct reports (Nancy Davolio, Janet Leverling, Margaret Peacock, Laura Callahan). "Steven Buchanan" will be listed with his direct reports (Michael Suyama, Robert King, Anne Dodsworth). Other employees who do not directly manage anyone (like Nancy Davolio) will not appear in this result set. This demonstrates that while direct relationships are easy to find, this query does not show *all* employees under Andrew Fuller (e.g., it does not list those reporting to Steven Buchanan).

**Learning Point:**
This task demonstrates how to query immediate hierarchical relationships using a self-join and `STRING_AGG` for list aggregation. It simultaneously highlights the limitation of non-recursive SQL: it cannot effectively traverse a hierarchy of unknown depth to find *all* descendants in a single query.

---

#### **Task 2: Flattening the Entire Subordinate Hierarchy (Recursive CTE)**

**Description:**
Now, let's address the full requirement: for each employee, generate a comma-separated list of *all* their direct and indirect reports. This requires a Recursive CTE. Your CTE will start with direct reports (the anchor member) and then iteratively find reports of those reports (the recursive member) until no more subordinates are found. The final query will then aggregate these results.

**SQL Query:**
```sql
-- Use a Recursive CTE to find all subordinates for each manager
WITH RECURSIVE SubordinatesHierarchy AS (
    -- Anchor Member: Find all direct reports
    SELECT
        E.employee_id AS manager_id,
        S.employee_id AS subordinate_id,
        S.first_name || ' ' || S.last_name AS subordinate_name
    FROM
        employees AS E -- Potential Manager
    JOIN
        employees AS S ON S.reports_to = E.employee_id -- Direct Subordinate

    UNION ALL

    -- Recursive Member: Find indirect reports
    SELECT
        SH.manager_id,
        E.employee_id AS subordinate_id,
        E.first_name || ' ' || E.last_name AS subordinate_name
    FROM
        employees AS E -- Current employee
    JOIN
        SubordinatesHierarchy AS SH ON E.reports_to = SH.subordinate_id
)
-- Final Query: Aggregate the results for each manager
SELECT
    M.first_name || ' ' || M.last_name AS manager_name,
    STRING_AGG(SH.subordinate_name, ', ') AS all_subordinates
FROM
    employees AS M -- All employees (potential managers)
LEFT JOIN
    SubordinatesHierarchy AS SH ON M.employee_id = SH.manager_id
GROUP BY
    M.employee_id,
    manager_name
ORDER BY
    M.employee_id;
```

**Expected Output/Explanation:**
"Andrew Fuller" will now have a comprehensive list of all employees under him, including Steven Buchanan and all of Steven's reports. Employees who manage no one (e.g., Nancy Davolio, Michael Suyama) will appear with `NULL` or an empty string in the `all_subordinates` column, as they have no one reporting under them.

**Learning Point:**
This task introduces the core structure of a Recursive CTE:
*   **Anchor Member:** The initial query that establishes the base set of rows (direct reports).
*   **`UNION ALL`:** Combines the results of the anchor and recursive members.
*   **Recursive Member:** References the CTE itself (`SubordinatesHierarchy`) to find the next level of the hierarchy (reports of reports).
The recursion continues until no new rows are returned by the recursive member.

---

#### **Task 3: Including the Manager in Their Own Subordinate List (Recursive CTE Variation)**

**Description:**
Sometimes, a business requirement might dictate that a manager should also be included in their own list of reports (as the "top" of their own chain). Modify the Recursive CTE from Task 2 so that each `manager_name` column lists not only their subordinates but also themselves, at the beginning of the list.

**SQL Query:**
```sql
-- Use a Recursive CTE to find all subordinates for each manager, including the manager themselves
WITH RECURSIVE SubordinatesHierarchyPlusSelf AS (
    -- Anchor Member: Start with each employee as their own initial "subordinate" in their own chain
    SELECT
        E.employee_id AS manager_id,
        E.employee_id AS current_employee_id,
        E.first_name || ' ' || E.last_name AS current_employee_name
    FROM
        employees AS E

    UNION ALL

    -- Recursive Member: Find employees reporting to anyone found in the previous step (including managers themselves)
    SELECT
        SH.manager_id,
        E.employee_id AS current_employee_id,
        E.first_name || ' ' || E.last_name AS current_employee_name
    FROM
        employees AS E -- The employee whose reports_to matches the current SH.current_employee_id
    JOIN
        SubordinatesHierarchyPlusSelf AS SH ON E.reports_to = SH.current_employee_id
)
-- Final Query: Aggregate the results for each manager, including themselves
SELECT
    M.first_name || ' ' || M.last_name AS employee_name,
    STRING_AGG(SH.current_employee_name, ', ' ORDER BY SH.manager_id, SH.current_employee_name)
        AS all_employees_under_or_self -- Added ORDER BY to make output consistent
FROM
    employees AS M
LEFT JOIN
    SubordinatesHierarchyPlusSelf AS SH ON M.employee_id = SH.manager_id
GROUP BY
    M.employee_id,
    employee_name
ORDER BY
    M.employee_id;
```

**Expected Output/Explanation:**
The list for "Andrew Fuller" will now start with "Andrew Fuller", followed by all his direct and indirect reports. Similarly, "Steven Buchanan" will start his list with "Steven Buchanan" and then his reports. Employees who manage no one will appear with only their own name in the list.

**Learning Point:**
This task demonstrates how to adjust the anchor member of a Recursive CTE to include the starting element (the manager) in the traversal path. It illustrates the flexibility in defining the base case for your recursive query. The `ORDER BY` clause within `STRING_AGG` is also important for ensuring consistent output ordering of the comma-separated list.

---

### **Post-Lab Activities**

**Optional Challenges:**

1.  **Reverse Hierarchy (Paths Upwards):** Can you modify a recursive CTE to list, for each employee, their entire management chain (who they report to, who that person reports to, and so on, up to the CEO)?
2.  **Level Tracking:** Modify the Recursive CTE in Task 2 to include a `level` column, indicating the depth of each subordinate relative to their top-level manager (e.g., direct reports are level 1, their reports are level 2, etc.).

**Further Exploration:**

*   Explore `CYCLE` clause for recursive CTEs, which helps detect and manage cycles in graphs (though not typically present in strict `reports_to` hierarchies).
*   Investigate scenarios where recursive CTEs are used for bill of materials (parts explosion) or network pathfinding.

---

### **Conclusion**

In this lab, you gained practical experience with Recursive CTEs, a powerful feature for handling hierarchical data. You understood why simple joins are insufficient for deep hierarchies and successfully constructed recursive queries to flatten an organizational structure. This fundamental technique is indispensable for analyzing complex relationships within your database.