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

**Why start here:** You've been handed credentials and told the database is `acmecorp`. Before writing any exploit or doing any interesting query, you need to understand the data model. You don't know what tables exist, what columns they have, or how they relate to each other. The psql meta-commands (`\dt`, `\d+`) are the fastest way to build that picture — faster than guessing table names and getting empty results.

**Step 1 — Connect to the database**

```bash
psql -h <TARGET_IP> -p <TARGET_PORT> -U acdbuser acmecorp
# Enter password: AcmeCorp2023!
```

On connection success, you get a `acmecorp=>` prompt. This confirms the credentials are valid and the database name is correct. If connection fails, check the port — PostgreSQL sometimes runs on non-standard ports (5433, 5432, or application-specific like 32452 in HTB lab setups).

**Step 2 — Discover the schema**

```sql
\dt
```

This lists all tables. You're looking for anything that sounds like organizational structure, user accounts, or business data — all of which are potential targets or needed for multi-table queries.

```
 public | departments | table | postgres
 public | dept_emp    | table | postgres
 public | employees   | table | postgres
 public | salaries    | table | postgres
 public | titles      | table | postgres
```

Five tables. The `departments` table is the obvious first target. Before writing a SELECT, check its columns so you don't guess the column name:

```sql
\d+ departments
```

Output shows two columns: `id` (integer, primary key) and `name` (varchar). Now you know the exact column names and can write a precise query instead of doing a `SELECT *` and eyeballing it.

**Step 3 — Query the ID**

```sql
SELECT id FROM departments WHERE name = 'Information Technology';
```

The result is `4`. This becomes an input to subsequent queries — write it down or keep the psql session open across questions.

---

## Question 2 — Count of IT employees

**Goal:** Count how many employees are assigned to department 4.

**Why this table:** You know `dept_emp` exists from `\dt`. The name suggests it maps employees to departments. Before writing a JOIN or subquery, verify that assumption by inspecting its structure — there may be a direct `dept_id` column you can filter on without a JOIN at all.

**Step 1 — Inspect the dept_emp table**

```sql
\d+ dept_emp
```

Key columns: `emp_id` and `dept_id`, both integers. No name or email here — just foreign keys. This means to get employee details you'd need to JOIN to `employees`, but for a simple count, you only need `dept_emp`. You don't need a JOIN at all, which keeps the query simple.

**Step 2 — Count employees in IT**

```sql
SELECT COUNT(emp_id) FROM dept_emp WHERE dept_id = 4;
```

The result is `390`. Using `COUNT(emp_id)` rather than `COUNT(*)` is a minor distinction — both work here — but `COUNT(column)` skips NULLs. In a well-designed schema there shouldn't be NULL emp_id values, but it's a good habit.

---

## Question 3 — Email of the most recently hired IT employee

**Goal:** Find the employee in IT (dept_id = 4) with the latest `hire_date` and return their email.

**Why a JOIN is needed now:** `dept_emp` only has `emp_id` and `dept_id` — no email, no hire date. Those attributes belong to the `employees` table. You need to match records across tables using the shared `emp_id` key. Verify that before assuming:

**Step 1 — Inspect the employees table**

```sql
\d+ employees
```

Key columns: `id` (primary key), `email`, `hire_date`. The `id` column is what `dept_emp.emp_id` references. Now you can write the JOIN: match `employees.id = dept_emp.emp_id`, filter by department, sort by hire date descending, and take the top result.

**Decision point:** You could also do this as a subquery (`WHERE id IN (SELECT emp_id FROM dept_emp WHERE dept_id=4) ORDER BY hire_date DESC LIMIT 1`). Both approaches are equivalent — the JOIN is slightly more idiomatic for relational data and easier to extend if you need more columns.

**Step 2 — JOIN employees to dept_emp and filter by IT**

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

**Why three tables:** `first_name` is in `employees`. Department membership is in `dept_emp`. Salary is in `salaries`. You can't get all three from any two tables — you need to chain `employees → dept_emp → salaries`. Before writing the JOIN, inspect `salaries` to confirm its structure:

**Step 1 — Inspect the salaries table**

```sql
\d+ salaries
```

Key columns: `emp_id` and `salary`. `emp_id` is a foreign key to `employees.id`. Notice it also has `from_date` and `to_date` columns — this could mean an employee has multiple salary rows over time. For this question, it doesn't matter (you're just finding the salary value), but in a real engagement you'd note this as a potential source of duplicate rows.

**Step 2 — Three-table JOIN with ORDER BY and LIMIT**

The query chains the three tables on their shared `emp_id` keys. Filter to IT department and first name William, sort ascending (lowest first), and take two rows:

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

Two rows come back. The first is the lowest-paid William; the second — `107000` — is the answer. The `LIMIT 2` + reading the second row is simpler than using `OFFSET 1 LIMIT 1`, though both work.

---

## Lessons Learned

- Always inspect the schema before querying — `\dt` → `\d+ <table>` takes 30 seconds and prevents wasted queries on wrong column names
- Not every multi-table question requires a JOIN; check if one table already has everything you need
- `ORDER BY hire_date DESC LIMIT 1` is the standard "most recent" pattern; `ORDER BY salary ASC LIMIT 2` is the "second-lowest" pattern
- Multiple salary rows per employee (from_date/to_date) is a real complication you'd handle in a real query with a `WHERE to_date IS NULL` filter

## Related Pages

- [[tools/utility/psql]] — psql client reference
- [[attack/web/advanced_sqli_character_bypasses]] — character-level SQLi techniques built on this SQL knowledge
