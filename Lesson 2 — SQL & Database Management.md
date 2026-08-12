# Lesson 2 — SQL & Database Management

**Goal:** Learn the SQL operations a MariaDB DBA uses to create, modify, inspect, and remove database objects.

---

## 1. Database Management

First, see the databases:

```sql
SHOW DATABASES;
```

Create a database:

```sql
CREATE DATABASE company;
```

Select it:

```sql
USE company;
```

Check the currently selected database:

```sql
SELECT DATABASE();
```

Delete a database:

```sql
DROP DATABASE company;
```

⚠️ `DROP DATABASE` permanently removes the database and its objects. Never run it against a production database unless you have explicitly verified the operation.

---

## 2. Creating Tables

Let's create a realistic employee table:

```sql
CREATE TABLE employees (
    employee_id INT NOT NULL AUTO_INCREMENT,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100),
    department VARCHAR(50),
    salary DECIMAL(10,2),
    hire_date DATE,
    PRIMARY KEY (employee_id)
);
```

Breakdown:

* `INT` → integer
* `VARCHAR(50)` → variable-length text
* `DECIMAL(10,2)` → numeric value with 2 decimal places
* `DATE` → date
* `NOT NULL` → value is required
* `AUTO_INCREMENT` → MariaDB generates the ID
* `PRIMARY KEY` → uniquely identifies each row

---

## 3. Inspecting Tables

List tables:

```sql
SHOW TABLES;
```

Describe a table:

```sql
DESCRIBE employees;
```

You can also use:

```sql
SHOW COLUMNS FROM employees;
```

For the exact table definition:

```sql
SHOW CREATE TABLE employees;
```

As a DBA, `SHOW CREATE TABLE` is particularly useful when investigating how an existing table was configured.

---

## 4. Inserting Data

Insert one employee:

```sql
INSERT INTO employees
(first_name, last_name, email, department, salary, hire_date)
VALUES
('Alice', 'Johnson', 'alice@example.com', 'IT', 75000.00, '2025-01-15');
```

Insert several rows:

```sql
INSERT INTO employees
(first_name, last_name, email, department, salary, hire_date)
VALUES
('Bob', 'Smith', 'bob@example.com', 'HR', 65000.00, '2025-02-01'),
('Carol', 'Brown', 'carol@example.com', 'IT', 82000.00, '2025-03-10'),
('David', 'Wilson', 'david@example.com', 'Finance', 70000.00, '2025-04-20');
```

Check the data:

```sql
SELECT * FROM employees;
```

---

## 5. Selecting Data

Select specific columns:

```sql
SELECT first_name, last_name, department
FROM employees;
```

Filter rows:

```sql
SELECT *
FROM employees
WHERE department = 'IT';
```

Find employees earning more than $70,000:

```sql
SELECT first_name, last_name, salary
FROM employees
WHERE salary > 70000;
```

Multiple conditions:

```sql
SELECT *
FROM employees
WHERE department = 'IT'
AND salary > 70000;
```

---

## 6. Updating Data

Suppose Alice receives a raise:

```sql
UPDATE employees
SET salary = 80000
WHERE employee_id = 1;
```

Verify:

```sql
SELECT *
FROM employees
WHERE employee_id = 1;
```

### DBA warning

Be extremely careful with:

```sql
UPDATE employees
SET salary = 80000;
```

There is **no `WHERE` clause**, so every employee could receive the new salary.

A DBA should develop the habit:

**Before UPDATE → run the equivalent SELECT.**

For example:

```sql
SELECT *
FROM employees
WHERE department = 'IT';
```

Then, if the result is exactly what you intend:

```sql
UPDATE employees
SET salary = salary * 1.10
WHERE department = 'IT';
```

---

## 7. Deleting Data

Delete one employee:

```sql
DELETE FROM employees
WHERE employee_id = 4;
```

Again, verify first:

```sql
SELECT *
FROM employees
WHERE employee_id = 4;
```

Then delete.

Be very careful with:

```sql
DELETE FROM employees;
```

That deletes **all rows** in the table.

---

# 8. ALTER TABLE

Database requirements change over time.

Suppose we need to add a phone number:

```sql
ALTER TABLE employees
ADD phone VARCHAR(20);
```

Check:

```sql
DESCRIBE employees;
```

Rename a column:

```sql
ALTER TABLE employees
RENAME COLUMN phone TO phone_number;
```

Modify a column:

```sql
ALTER TABLE employees
MODIFY phone_number VARCHAR(30);
```

Rename the table:

```sql
RENAME TABLE employees TO staff;
```

Rename it back:

```sql
RENAME TABLE staff TO employees;
```

---

# 9. Constraints

Constraints protect data integrity.

### Primary Key

```sql
PRIMARY KEY (employee_id)
```

A primary key uniquely identifies each row.

### NOT NULL

```sql
first_name VARCHAR(50) NOT NULL
```

The column cannot contain `NULL`.

### UNIQUE

Suppose every employee email must be unique:

```sql
ALTER TABLE employees
ADD CONSTRAINT unique_email UNIQUE (email);
```

Now MariaDB prevents duplicate email values.

### Foreign Key

Let's create departments:

```sql
CREATE TABLE departments (
    department_id INT PRIMARY KEY AUTO_INCREMENT,
    department_name VARCHAR(50) NOT NULL UNIQUE
);
```

Now we could have employees reference a department.

A foreign key establishes a relationship between tables:

```text
departments
     │
     │ department_id
     ▼
employees
```

Example:

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50) NOT NULL,
    department_id INT,
    FOREIGN KEY (department_id)
        REFERENCES departments(department_id)
);
```

This is one of the fundamental concepts behind relational databases.

---

# 10. Indexes

Indexes are extremely important for DBAs.

Imagine a table containing **10 million employees**.

You run:

```sql
SELECT *
FROM employees
WHERE email = 'alice@example.com';
```

Without an appropriate index, MariaDB may need to examine many rows.

Create an index:

```sql
CREATE INDEX idx_employee_email
ON employees(email);
```

Now MariaDB has an index it can potentially use to locate matching rows more efficiently.

Check indexes:

```sql
SHOW INDEX FROM employees;
```

Remove an index:

```sql
DROP INDEX idx_employee_email ON employees;
```

### Important DBA principle

**Indexes can make reads faster, but they aren't free.**

Indexes consume:

* Disk space
* Memory/cache
* CPU during maintenance
* Additional work during `INSERT`, `UPDATE`, and `DELETE`

Therefore, a DBA shouldn't simply create indexes on every column.

---

# 11. Transactions

Transactions are fundamental to database administration.

Suppose we transfer $500 between two accounts.

We don't want:

```text
Account A: -$500
Account B: unchanged
```

if something fails halfway through.

Instead:

```sql
START TRANSACTION;

UPDATE accounts
SET balance = balance - 500
WHERE account_id = 1;

UPDATE accounts
SET balance = balance + 500
WHERE account_id = 2;

COMMIT;
```

`COMMIT` makes the transaction permanent.

If something goes wrong:

```sql
ROLLBACK;
```

This is one reason transactional storage engines such as **InnoDB** are so important.

---

# 12. Useful DBA Inspection Commands

These are worth memorizing.

### Databases

```sql
SHOW DATABASES;
```

### Tables

```sql
SHOW TABLES;
```

### Table structure

```sql
DESCRIBE employees;
```

### Exact table definition

```sql
SHOW CREATE TABLE employees;
```

### Indexes

```sql
SHOW INDEX FROM employees;
```

### Server variables

```sql
SHOW VARIABLES;
```

### Server status

```sql
SHOW STATUS;
```

### Current connections/queries

```sql
SHOW PROCESSLIST;
```

These commands will become increasingly important when we start troubleshooting performance.

---

# 13. DBA Safety Rules

Start developing these habits now:

### Rule 1 — Never blindly run UPDATE

First:

```sql
SELECT ...
WHERE ...
```

Then:

```sql
UPDATE ...
WHERE ...
```

### Rule 2 — Never blindly run DELETE

First determine exactly which rows will be affected.

### Rule 3 — Be extremely careful with DROP

These are destructive:

```sql
DROP DATABASE ...
DROP TABLE ...
```

### Rule 4 — Understand indexes

Don't create indexes randomly.

### Rule 5 — Understand transactions

For important multi-step changes:

```sql
START TRANSACTION;
...
COMMIT;
```

or:

```sql
ROLLBACK;
```

---

# Lesson 2 Lab

Create a database:

```sql
CREATE DATABASE dba_lab;

USE dba_lab;
```

Create a table:

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    department VARCHAR(50),
    salary DECIMAL(10,2)
);
```

Insert five employees.

Then practice:

```sql
SELECT * FROM employees;
```

```sql
SELECT *
FROM employees
WHERE salary > 70000;
```

Create an index:

```sql
CREATE INDEX idx_department
ON employees(department);
```

Inspect it:

```sql
SHOW INDEX FROM employees;
```

Then add a column:

```sql
ALTER TABLE employees
ADD email VARCHAR(100);
```

Finally inspect the table:

```sql
SHOW CREATE TABLE employees;
```

---

## Lesson 2 — What you should remember

The most important DBA concepts from this lesson are:

**DDL**
→ `CREATE`, `ALTER`, `DROP`

**DML**
→ `INSERT`, `UPDATE`, `DELETE`

**Queries**
→ `SELECT`, `WHERE`, `ORDER BY`, etc.

**Integrity**
→ Primary keys, foreign keys, `NOT NULL`, `UNIQUE`

**Performance**
→ Indexes

**Safety**
→ Transactions, `COMMIT`, `ROLLBACK`

**Administration**
→ `SHOW DATABASES`, `SHOW TABLES`, `SHOW CREATE TABLE`, `SHOW INDEX`, `SHOW PROCESSLIST`

### Next: Lesson 3 — MariaDB Users & Security

We'll learn **`CREATE USER`, `GRANT`, `REVOKE`, roles, privileges, authentication, least privilege, and how to secure a MariaDB server like a real production DBA.**
