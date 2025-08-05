### **Solutions to Optional Challenges**

#### **Optional Challenge 1: Direct Path Display**

**Description:**
Modify the final `SELECT` in Task 2 to display the full path including the employee themselves at the beginning of the string. This means removing the `SUBSTRING` logic that was previously used to exclude the employee's own name from the chain.

**Solution SQL Query:**
```sql
-- Use a Recursive CTE to find the full management chain for each employee
WITH RECURSIVE ManagementChain AS (
    -- Anchor Member: Start with each employee and their immediate manager
    SELECT
        E.employee_id AS original_employee_id,
        E.first_name || ' ' || E.last_name AS employee_name,
        E.reports_to AS current_manager_id,
        CAST(E.first_name || ' ' || E.last_name AS TEXT) AS path_so_far -- Start path with the employee themselves
    FROM
        employees AS E

    UNION ALL

    -- Recursive Member: Climb up the hierarchy
    SELECT
        MC.original_employee_id,
        MGR.first_name || ' ' || MGR.last_name AS employee_name,
        MGR.reports_to AS current_manager_id,
        MC.path_so_far || ', ' || CAST(MGR.first_name || ' ' || MGR.last_name AS TEXT) -- Append manager to path
    FROM
        employees AS MGR
    JOIN
        ManagementChain AS MC ON MGR.employee_id = MC.current_manager_id
    WHERE
        MC.current_manager_id IS NOT NULL
)
-- Final Query: Select the complete path for each original employee, including themselves
SELECT
    Emp.first_name || ' ' || Emp.last_name AS employee_name,
    FinalPath.path_so_far AS full_management_path -- Directly use path_so_far
FROM
    employees AS Emp
JOIN
    ManagementChain AS FinalPath ON Emp.employee_id = FinalPath.original_employee_id
WHERE
    FinalPath.current_manager_id IS NULL -- Select only the rows where the path is complete (reached the top)
ORDER BY
    Emp.employee_id;
```

**Expected Output/Explanation:**
The `managers_chain` column will now be named `full_management_path` and will include the employee's name at the beginning of their respective chain. For "Nancy Davolio," the output would be "Nancy Davolio, Andrew Fuller". For "Andrew Fuller", it would simply be "Andrew Fuller" (as he has no manager).

**Learning Point:**
This solution highlights how the `path_so_far` column was designed from the start to build the full path. By simply removing the `SUBSTRING` logic, we retrieve the complete concatenated string, demonstrating flexibility in output formatting based on the initial path construction within the CTE.

---

#### **Optional Challenge 2: Level Tracking Upwards**

**Description:**
Add a `level` column to the `ManagementChain` CTE, starting from `0` for the employee themselves and incrementing by `1` for each step up the hierarchy.

**Solution SQL Query:**
```sql
-- Use a Recursive CTE to find the full management chain and their level for each employee
WITH RECURSIVE ManagementChain AS (
    -- Anchor Member: Start with each employee and their immediate manager, level 0 for self
    SELECT
        E.employee_id AS original_employee_id,
        E.first_name || ' ' || E.last_name AS employee_name_at_level,
        E.reports_to AS current_manager_id, -- The ID of the manager at this step
        0 AS level -- Level 0 for the employee themselves
    FROM
        employees AS E

    UNION ALL

    -- Recursive Member: Climb up the hierarchy, incrementing level
    SELECT
        MC.original_employee_id,
        MGR.first_name || ' ' || MGR.last_name AS employee_name_at_level,
        MGR.reports_to AS current_manager_id,
        MC.level + 1 AS level -- Increment level for each step up
    FROM
        employees AS MGR
    JOIN
        ManagementChain AS MC ON MGR.employee_id = MC.current_manager_id
    WHERE
        MC.current_manager_id IS NOT NULL
)
-- Final Query: Select each original employee and their entire management chain with levels
SELECT
    Emp.first_name || ' ' || Emp.last_name AS employee_name,
    STRING_AGG(
        MC.employee_name_at_level || ' (Level ' || MC.level || ')',
        ', ' ORDER BY MC.level DESC
    ) AS management_chain_with_levels
FROM
    employees AS Emp
JOIN
    ManagementChain AS MC ON Emp.employee_id = MC.original_employee_id
GROUP BY
    Emp.employee_id,
    employee_name
ORDER BY
    Emp.employee_id;
```

**Expected Output/Explanation:**
The `management_chain_with_levels` column will now show each manager's name followed by their level in parentheses. For example, for "Nancy Davolio," you might see "Andrew Fuller (Level 1), Nancy Davolio (Level 0)". For "Michael Suyama," you might see "Andrew Fuller (Level 2), Steven Buchanan (Level 1), Michael Suyama (Level 0)". The `STRING_AGG` is ordered by `level DESC` to ensure the highest-level manager appears first in the string, mirroring a top-down reporting chain.

**Learning Point:**
This solution demonstrates how to incorporate and propagate a numerical counter within a recursive CTE. The `level` column is initialized in the anchor member and then incremented in the recursive member, providing valuable context about the depth within the hierarchy for each traversed node.