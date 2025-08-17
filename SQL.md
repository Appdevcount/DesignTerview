Thanks for waiting! Here's an expanded and detailed list of 50 top SQL Server interview questions, complete with explanations and SQL query examples to help you understand and implement each concept effectively.

---

🧠 Core SQL Concepts (1–10)

1. What is SQL?
   - Structured Query Language used to manage relational databases.

2. What is RDBMS?
   - Relational Database Management System stores data in tables with relationships.

3. What is the difference between DELETE, TRUNCATE, and DROP?
`sql
DELETE FROM employees WHERE id = 1; -- Removes specific row
TRUNCATE TABLE employees;           -- Removes all rows, resets identity
DROP TABLE employees;               -- Deletes table structure
`

4. What are constraints in SQL?
   - Rules enforced on columns: NOT NULL, UNIQUE, CHECK, DEFAULT, PRIMARY KEY, FOREIGN KEY.

5. What is normalization?
   - Organizing data to reduce redundancy.
   - Example: Splitting customer and orders into separate tables.

6. What is denormalization?
   - Combining tables for faster reads, sacrificing redundancy.

7. What is a primary key vs. foreign key?
`sql
CREATE TABLE orders (
  order_id INT PRIMARY KEY,
  customerid INT FOREIGN KEY REFERENCES customers(customerid)
);
`

8. What is a unique key?
   - Ensures column values are unique.
`sql
CREATE TABLE users (
  email VARCHAR(100) UNIQUE
);
`

9. What is the difference between HAVING and WHERE?
`sql
SELECT department, COUNT(*) 
FROM employees 
WHERE active = 1 
GROUP BY department 
HAVING COUNT(*) > 5;
`

10. What is a join? Types?
`sql
-- INNER JOIN
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- LEFT JOIN
SELECT * FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;
`

---

🧪 SQL Queries & Examples (11–20)

11. Find the second highest salary.
`sql
SELECT MAX(salary) 
FROM employees 
WHERE salary < (SELECT MAX(salary) FROM employees);
`

12. Find duplicate records.
`sql
SELECT name, COUNT(*) 
FROM students 
GROUP BY name 
HAVING COUNT(*) > 1;
`

13. Get department-wise highest salary.
`sql
SELECT department, MAX(salary) 
FROM employees 
GROUP BY department;
`

14. Retrieve employees who joined in the last 30 days.
`sql
SELECT * 
FROM employees 
WHERE join_date >= DATEADD(DAY, -30, GETDATE());
`

15. Find employees with names starting with 'A'.
`sql
SELECT * 
FROM employees 
WHERE name LIKE 'A%';
`

16. Get employees not assigned to any project.
`sql
SELECT * 
FROM employees 
WHERE project_id IS NULL;
`

17. Find employees who are also managers.
`sql
SELECT * 
FROM employees 
WHERE id IN (SELECT DISTINCT manager_id FROM employees);
`

18. Fetch odd-numbered rows.
`sql
SELECT * 
FROM (SELECT *, ROW_NUMBER() OVER (ORDER BY id) AS rn FROM employees) AS temp
WHERE rn % 2 = 1;
`

19. Find 3rd highest salary without TOP.
`sql
SELECT salary 
FROM employees e1 
WHERE 2 = (SELECT COUNT(DISTINCT salary) FROM employees e2 WHERE e2.salary > e1.salary);
`

20. Get project-wise employee count.
`sql
SELECT project_id, COUNT(*) 
FROM employees 
GROUP BY project_id;
`

---

⚙️ SQL Server-Specific Features (21–30)

21. What is a stored procedure?
`sql
CREATE PROCEDURE GetEmployeeById @id INT
AS
BEGIN
  SELECT * FROM employees WHERE id = @id;
END;
`

22. What is a trigger?
`sql
CREATE TRIGGER trgafterinsert
ON employees
AFTER INSERT
AS
BEGIN
  PRINT 'New employee added';
END;
`

23. What is a view?
`sql
CREATE VIEW ActiveEmployees AS
SELECT * FROM employees WHERE active = 1;
`

24. What is a CTE (Common Table Expression)?
`sql
WITH SalesCTE AS (
  SELECT salesperson, SUM(sales) AS total_sales
  FROM orders
  GROUP BY salesperson
)
SELECT * FROM SalesCTE WHERE total_sales > 10000;
`

25. What is a temporary table?
`sql
CREATE TABLE #TempEmployees (
  id INT, name VARCHAR(100)
);
`

26. Difference between local and global temp tables?
   - Local: #temp, visible to current session.
   - Global: ##temp, visible to all sessions.

27. What is SQL Server Agent?
   - Automates jobs like backups, alerts, and scheduled tasks.

28. What is SQL Profiler?
   - Tool to monitor and analyze SQL Server events.

29. What is a recursive stored procedure?
`sql
CREATE PROCEDURE Factorial (@n INT, @result INT OUTPUT)
AS
BEGIN
  IF @n <= 1 SET @result = 1;
  ELSE
  BEGIN
    DECLARE @temp INT;
    EXEC Factorial @n - 1, @temp OUTPUT;
    SET @result = @n * @temp;
  END
END;
`

30. What is a synonym?
`sql
CREATE SYNONYM Emp FOR dbo.Employees;
SELECT * FROM Emp;
`

---

🔐 Transactions & Security (31–40)

31. What is a transaction?
`sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
`

32. What is ACID?
   - Atomicity, Consistency, Isolation, Durability.

33. What is isolation level?
`sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
`

34. What is deadlock?
   - Two processes block each other; SQL Server resolves by killing one.

35. How to handle errors?
`sql
BEGIN TRY
  -- SQL statements
END TRY
BEGIN CATCH
  SELECT ERROR_MESSAGE();
END CATCH;
`

36. What is SQL Injection?
   - Malicious input to manipulate queries.
   - Prevent using parameterized queries.

37. What are authentication modes?
   - Windows Authentication and Mixed Mode.

38. What is role-based access control?
`sql
CREATE ROLE Manager;
GRANT SELECT ON employees TO Manager;
EXEC sp_addrolemember 'Manager', 'John';
`

39. What is encryption in SQL Server?
   - Protects sensitive data using Transparent Data Encryption (TDE).

40. What is row-level security?
   - Restricts access to rows based on user identity.

---

📊 Performance & Optimization (41–50)

41. What is an index?
`sql
CREATE INDEX idx_name ON employees(name);
`

42. Clustered vs. Non-clustered index?
   - Clustered: Sorts data physically.
   - Non-clustered: Separate structure with pointers.

43. What is execution plan?
   - Visual representation of query performance.

44. What is parameter sniffing?
   - SQL Server caches plan based on first parameter value.

45. What is batch mode execution?
   - Processes rows in batches for better performance.

46. What is adaptive query processing?
   - SQL Server adjusts execution plans based on runtime feedback.

47. How to calculate running totals?
`sql
SELECT name, salary,
       SUM(salary) OVER (ORDER BY name) AS running_total
FROM employees;
`

48. What is COALESCE?
`sql
SELECT COALESCE(NULL, NULL, 'default');
`

49. Difference between ISNULL and NULLIF?
`sql
SELECT ISNULL(NULL, 'default'); -- returns 'default'
SELECT NULLIF(5, 5);            -- returns NULL
`

50. What are temporal tables?
   - Automatically track history of changes.
`sql
CREATE TABLE EmployeeHistory (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  PERIOD FOR SYSTEM_TIME (ValidFrom, ValidTo)
) WITH (SYSTEM_VERSIONING = ON);
`

---

Would you like me to turn this into a downloadable PDF or quiz format for practice?
