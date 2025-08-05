### **Solution to Optional Challenge**

#### **Optional Challenge 2: Level Tracking (Subordinates Hierarchy)**

**Description:**
Modify the Recursive CTE in Task 2 to include a `level` column, indicating the depth of each subordinate relative to their top-level manager (e.g., direct reports are level 1, their reports are level 2, etc.).

**Solution SQL Query:**
```sql
-- Use a Recursive CTE to find all subordinates for each manager, now with a level column
WITH RECURSIVE SubordinatesHierarchy AS (
    -- Anchor Member: Find all direct reports, assign them level 1
    SELECT
        E.employee_id AS manager_id,
        S.employee_id AS subordinate_id,
        S.first_name || ' ' || S.last_name AS subordinate_name,
        1 AS level -- Direct reports are level 1
    FROM
        employees AS E -- Potential Manager
    JOIN
        employees AS S ON S.reports_to = E.employee_id -- Direct Subordinate

    UNION ALL

    -- Recursive Member: Find indirect reports, incrementing the level
    SELECT
        SH.manager_id,
        E.employee_id AS subordinate_id,
        E.first_name || ' ' || E.last_name AS subordinate_name,
        SH.level + 1 AS level -- Increment level for each step deeper in the hierarchy
    FROM
        employees AS E -- Current employee
    JOIN
        SubordinatesHierarchy AS SH ON E.reports_to = SH.subordinate_id
)
-- Final Query: Aggregate the results for each manager, including subordinate names and their levels
SELECT
    M.first_name || ' ' || M.last_name AS manager_name,
    STRING_AGG(
        SH.subordinate_name || ' (Level ' || SH.level || ')',
        ', ' ORDER BY SH.level, SH.subordinate_name -- Order by level then name for consistent output
    ) AS all_subordinates_with_levels
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
The output will now include the `level` for each subordinate in the comma-separated list. For "Andrew Fuller," his `all_subordinates_with_levels` list will show "Nancy Davolio (Level 1), Janet Leverling (Level 1), Laura Callahan (Level 1), Margaret Peacock (Level 1), Steven Buchanan (Level 1), Michael Suyama (Level 2), Robert King (Level 2), Anne Dodsworth (Level 2)". Notice how Steven's direct reports (Michael, Robert, Anne) are now correctly identified as Level 2 relative to Andrew.

**Learning Point:**
This solution demonstrates how to incorporate a level counter into a recursive CTE. The `level` is initialized in the anchor member and then incremented in the recursive member for each step down the hierarchy. This is crucial for analyzing the depth of relationships within tree-like data structures. The `ORDER BY` clause within `STRING_AGG` ensures that subordinates are listed by their level, providing a logical flow in the output string.