### Lesson 1 — MariaDB Fundamentals

**Goal:** Understand what a MariaDB DBA actually manages.

MariaDB is a **relational database management system (RDBMS)**. 

A MariaDB server stores structured data in databases and provides clients/applications with SQL access to that data.

Think of the hierarchy like this:

**MariaDB Server → Database → Tables → Rows → Columns**

For example:

```text
MariaDB Server
│
├── company
│   ├── employees
│   ├── departments
│   └── salaries
│
└── inventory
    ├── products
    └── warehouses
```

### 1. What does a MariaDB DBA do?

A DBA is responsible for things such as:

* Installing and configuring MariaDB
* Creating and managing databases
* Managing users and permissions
* Taking backups
* Restoring databases
* Monitoring performance
* Troubleshooting failures
* Managing replication
* Planning high availability
* Securing the database
* Performing upgrades
* Managing storage and capacity

A DBA's primary concerns are often summarized as:

**Availability + Performance + Security + Data Integrity + Recoverability**

### 2. MariaDB Server vs. MariaDB Client

There are two important concepts:

**Server**

The MariaDB server process actually stores and manages the data.

**Client**

The client connects to the server and sends SQL commands.

Conceptually:

```text
Application / DBA
       │
       ▼
MariaDB Client
       │
       │ SQL
       ▼
MariaDB Server
       │
       ▼
Storage
```

For example, you might connect with:

```bash
mariadb -u root -p
```

The client connects to the MariaDB server, authenticates you, and gives you a SQL prompt.

### 3. Connecting to MariaDB

After connecting, you might see:

```sql
MariaDB [(none)]>
```

You can check the server version:

```sql
SELECT VERSION();
```

You can see the databases you can access:

```sql
SHOW DATABASES;
```

You can select a database:

```sql
USE company;
```

Then:

```sql
SHOW TABLES;
```

### 4. Databases and tables

A **database** is a logical container for database objects.

A **table** stores structured data.

Example:

```sql
CREATE DATABASE company;
```

Then:

```sql
USE company;
```

Create a table:

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(100),
    salary DECIMAL(10,2)
);
```

Insert data:

```sql
INSERT INTO employees
(employee_id, name, department, salary)
VALUES
(1, 'Alice', 'IT', 75000.00);
```

Retrieve it:

```sql
SELECT * FROM employees;
```

### 5. What is the MariaDB data directory?

MariaDB needs persistent storage for its databases, tables, logs, and other files.

The location is controlled by server configuration and can vary depending on how MariaDB was installed.

As a DBA, you need to know:

* Where the data is stored
* How much disk space is available
* Where logs are stored
* Where configuration files are located
* Which storage engine the tables use

**Never assume that simply copying arbitrary database files is a valid backup strategy.** Backup and recovery procedures need to account for the storage engine and consistency.

### 6. MariaDB configuration

MariaDB's behavior is controlled by configuration files and server variables.

A DBA commonly needs to inspect variables such as:

```sql
SHOW VARIABLES;
```

For a particular variable:

```sql
SHOW VARIABLES LIKE 'max_connections';
```

You can inspect the current connection count:

```sql
SHOW STATUS LIKE 'Threads_connected';
```

These become extremely important when we reach **Lesson 4 — Configuration** and **Lesson 7 — Performance**.

### 7. Storage engines

MariaDB supports multiple storage engines.

One of the most important is **InnoDB**, which provides features such as:

* Transactions
* Row-level locking
* Foreign keys
* Crash recovery
* ACID properties

You can check a table's storage engine with:

```sql
SHOW TABLE STATUS;
```

Or inspect a particular table:

```sql
SHOW CREATE TABLE employees;
```

### 8. The DBA mindset

A beginner often thinks:

> "How do I make this SQL query work?"

A DBA eventually starts asking:

> "What happens if the server crashes?"

> "Can I restore this data?"

> "How much downtime can the business tolerate?"

> "Who has access to this database?"

> "Why is the query slow?"

> "What happens if the primary server fails?"

> "Do we have enough disk space?"

> "Can we recover yesterday's data?"

That's the difference between simply **using MariaDB** and **administering MariaDB**.

### Lesson 1 practice

If you have a MariaDB server available, practice these commands:

```sql
SELECT VERSION();

SHOW DATABASES;

SHOW VARIABLES LIKE 'datadir';

SHOW VARIABLES LIKE 'port';

SHOW VARIABLES LIKE 'max_connections';

SHOW STATUS LIKE 'Threads_connected';
```

Then create a practice database:

```sql
CREATE DATABASE dba_lab;

USE dba_lab;

CREATE TABLE test_table (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

INSERT INTO test_table VALUES
(1, 'MariaDB');

SELECT * FROM test_table;
```

Finally:

```sql
DROP DATABASE dba_lab;
```

**Important:** Only run `DROP DATABASE` on a disposable practice database.

### What you should know after Lesson 1

You should now understand:

**MariaDB Server → databases → tables → rows/columns**

and the basic DBA responsibilities:

**Configure → Secure → Backup → Monitor → Optimize → Recover → Maintain**
