---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Introduction to PostgreSQL

Target credentials: connect as `acdbuser` with password `AcmeCorp2023!` to the `acmecorp` database on the spawned target.

This lab is fully CLI-based — `psql` is both the required tool and the practical tool. There is no GUI requirement.

## Tool Installation

```bash
# Check if installed
psql --version || echo "not installed"

# Install (Kali / Parrot)
sudo apt install postgresql-client-13 -y

# Verify
psql --version
```

See [[tools/utility/psql]] for full reference.

---

## Question 1 — ID of the Information Technology department

**Goal:** Query the `departments` table and return the `id` for the Information Technology row.

**Step 1 — Connect to the database**

```bash
psql -h <TARGET_IP> -p <TARGET_PORT> -U acdbuser acmecorp
# Enter password: AcmeCorp2023!
```

**Step 2 — Discover the schema**

```sql
\dt
```

Output shows five tables: `departments`, `dept_emp`, `employees`, `salaries`, `titles`.

```sql
\d+ departments
```

The table has two columns: `id` (integer) and `name` (varchar).

**Step 3 — Query the ID**

```sql
SELECT id FROM departments WHERE name = 'Information Technology';
```

The result is `4`.

---

## Question 2 — Count of IT employees

**Goal:** Count how many employees are assigned to department 4.

**Step 1 — Inspect the dept_emp table**

```sql
\d+ dept_emp
```

Relevant columns: `emp_id` and `dept_id`, both integers. `dept_id` is a foreign key to `departments.id`.

**Step 2 — Count employees in IT**

```sql
SELECT COUNT(emp_id) FROM dept_emp WHERE dept_id = 4;
```

The result is `390`.

---

## Question 3 — Email of the most recently hired IT employee

**Goal:** Find the employee in IT (dept_id = 4) with the latest `hire_date` and return their email.

**Step 1 — Inspect the employees table**

```sql
\d+ employees
```

Key columns: `id`, `email`, `hire_date`. The `id` column is the primary key referenced by `dept_emp.emp_id`.

**Step 2 — JOIN employees to dept_emp on id and filter by IT**

```sql
SELECT email
FROM employees
JOIN dept_emp ON employees.id = dept_emp.emp_id
WHERE dept_id = 4
ORDER BY hire_date DESC
LIMIT 1;
```

The result is `aealvarado28@acme.corp`.

---

## Question 4 — Salary of the second-lowest paid William in IT

**Goal:** Among all employees in IT with `first_name = 'William'`, return the second-lowest salary.

**Step 1 — Inspect the salaries table**

```sql
\d+ salaries
```

Key columns: `emp_id` and `salary`. `emp_id` is a foreign key to `employees.id`.

**Step 2 — Three-table JOIN with ORDER BY and LIMIT**

Join `employees`, `dept_emp`, and `salaries` together on their shared `emp_id` keys. Filter to IT department and first name William, then sort ascending and take two rows:

```sql
SELECT salary
FROM employees
JOIN dept_emp ON employees.id = dept_emp.emp_id
JOIN salaries ON dept_emp.emp_id = salaries.emp_id
WHERE dept_emp.dept_id = 4
  AND first_name = 'William'
ORDER BY salary ASC
LIMIT 2;
```

The first row is the lowest paid William; the second row — `107000` — is the second-lowest.

---

## Lessons Learned

- `\dt` lists tables; `\d+ <table>` shows columns, types, and foreign keys — always inspect before querying
- Three-table JOINs require tracing foreign key relationships; `\d+` is the fastest way to see them
- `ORDER BY ... DESC LIMIT 1` is the standard pattern for "most recent" or "highest" row queries

## Related Pages

- [[tools/utility/psql]] — psql client reference
- [[attack/web/advanced_sqli_character_bypasses]] — character-level SQLi techniques
