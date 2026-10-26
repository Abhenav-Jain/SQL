import pypandoc

content = r"""# SQL Premium Notes — Joins, Aggregation, Filtering & Data Transformation

> **Session revision notes | MySQL-focused | Placement + interview preparation**
>
> These notes cover the SQL concepts practised in this session: multi-table JOINs, aggregation, duplicate detection, arithmetic, rounding, CASE expressions, unit conversion, and finding unmatched records.

---

## 1. SQL Query Thinking: Start With the Requirement

Before writing SQL, translate the question into a sequence of operations.

| Requirement wording | SQL concept to consider |
|---|---|
| Display columns | `SELECT` |
| Read from a table | `FROM` |
| Combine related tables | `JOIN ... ON` |
| Keep rows meeting a condition | `WHERE` |
| Calculate totals/counts/averages | Aggregate functions |
| One result per category/group | `GROUP BY` |
| Filter aggregated groups | `HAVING` |
| Sort results | `ORDER BY` |
| Does not exist / no matching record | `NOT EXISTS` or `LEFT JOIN ... IS NULL` |
| Convert abbreviations into labels | `CASE WHEN` |
| Round a calculation | `ROUND()` |

### A reliable problem-solving checklist

1. **What should one output row represent?** One patient, one admission, one province, or one duplicate-name group?
2. **Which table contains the starting entities?**
3. **Which tables are needed to fetch the requested columns?**
4. **What are the relationship keys?** Identify primary-key/foreign-key pairs.
5. **Do we need every left-side row, or only matching rows?**
6. **Is the condition row-level (`WHERE`) or group-level (`HAVING`)?**
7. **Do calculations need rounding or aliases?**
8. **Could NULL values change the result?**

---

## 2. SELECT, FROM, Aliases and CONCAT

### Basic structure

```sql
SELECT column1, column2
FROM table_name;
```

### Table aliases

Aliases make multi-table queries easier to read.

```sql
SELECT p.patient_id, p.first_name
FROM patients AS p;
```

`AS` is optional for table aliases in MySQL:

```sql
FROM patients p
```

### CONCAT: build a full name

```sql
SELECT
    CONCAT(first_name, ' ', last_name) AS patient_name
FROM patients;
```

- `CONCAT()` combines strings.
- `' '` inserts a space between first and last name.
- `AS patient_name` names the output column.

**NULL note:** In MySQL, `CONCAT()` returns NULL if any argument is NULL. If nullable name fields must be handled, consider `CONCAT_WS()`:

```sql
SELECT CONCAT_WS(' ', first_name, last_name) AS patient_name
FROM patients;
```

`CONCAT_WS()` skips NULL arguments (but does not skip empty strings).

---

## 3. JOIN Fundamentals

A JOIN combines rows from tables using a relationship.

### 3.1 Two-table JOIN

```sql
SELECT
    p.patient_id,
    p.first_name,
    a.diagnosis
FROM patients p
JOIN admissions a
    ON p.patient_id = a.patient_id;
```

This matches a patient row to admission rows having the same `patient_id`.

### 3.2 Three-table JOIN

For every admission, show patient name, diagnosis, and the attending doctor's name.

Tables:

- `patients(patient_id, first_name, last_name)`
- `admissions(patient_id, attending_doctor_id, diagnosis)`
- `doctors(doctor_id, first_name, last_name)`

```sql
SELECT
    CONCAT(p.first_name, ' ', p.last_name) AS patient_name,
    a.diagnosis,
    CONCAT(d.first_name, ' ', d.last_name) AS doctor_name
FROM doctors d
JOIN admissions a
    ON a.attending_doctor_id = d.doctor_id
JOIN patients p
    ON p.patient_id = a.patient_id;
```

**How to read it:**

1. Start with `doctors d`.
2. Match each doctor to admissions using `a.attending_doctor_id = d.doctor_id`.
3. Match each admission to its patient using `p.patient_id = a.patient_id`.
4. Select the requested columns.

The order of JOINs can often be rearranged if the relationships and join types are preserved. Choose an order that makes the relationships easy to understand.

### 3.3 General three-table template

```sql
SELECT ...
FROM table1 t1
JOIN table2 t2
    ON t1.key = t2.key
JOIN table3 t3
    ON t2.other_key = t3.key;
```

For three tables, you commonly need two JOIN clauses and two `ON` conditions. The exact keys depend on the schema.

### 3.4 INNER JOIN vs LEFT JOIN

**INNER JOIN** (written as `JOIN`) returns only rows that have a match on both sides.

```sql
SELECT p.patient_id, a.diagnosis
FROM patients p
INNER JOIN admissions a
    ON p.patient_id = a.patient_id;
```

**LEFT JOIN** keeps every row from the left table. If no right-side match exists, right-side columns are returned as NULL.

```sql
SELECT p.patient_id, p.first_name, a.admission_id
FROM patients p
LEFT JOIN admissions a
    ON p.patient_id = a.patient_id;
```

Example:

`patients`

| patient_id | first_name |
|---:|---|
| 1 | Rahul |
| 2 | Aman |
| 3 | Rohit |
| 4 | Karan |
| 5 | Arjun |

`admissions`

| admission_id | patient_id |
|---:|---:|
| 101 | 1 |
| 102 | 2 |
| 103 | 4 |

LEFT JOIN result (simplified):

| patient_id | first_name | admission_id |
|---:|---|---:|
| 1 | Rahul | 101 |
| 2 | Aman | 102 |
| 3 | Rohit | NULL |
| 4 | Karan | 103 |
| 5 | Arjun | NULL |

**Key distinction:** A LEFT JOIN does not automatically mean “find missing records.” It first preserves all left-table rows. Add a NULL filter when you specifically want unmatched rows.

### 3.5 One-to-many JOINs can multiply rows

One patient may have multiple admissions. A JOIN then returns one row per matching patient-admission pair.

For example, if patient 1 has three admissions, patient 1 can appear three times. This is expected—not necessarily a duplicate-data problem.

Before using `COUNT()`, ask: **Am I counting patients, admissions, or joined rows?**

---

## 4. Find Records With No Match: LEFT JOIN + IS NULL

Question pattern:

> Show patients whose `patient_id` does not exist in `admissions.patient_id`.

### Solution

```sql
SELECT
    p.patient_id,
    p.first_name,
    p.last_name
FROM patients p
LEFT JOIN admissions a
    ON p.patient_id = a.patient_id
WHERE a.patient_id IS NULL;
```

### Understand it step by step

**Step 1 — LEFT JOIN**

```sql
FROM patients p
LEFT JOIN admissions a
    ON p.patient_id = a.patient_id
```

All patients remain in the result. Matching admissions are attached. Patients with no matching admission get NULL in the `a` columns.

**Step 2 — Keep only unmatched rows**

```sql
WHERE a.patient_id IS NULL
```

Because `a.patient_id` comes from the right table, it is NULL when no admission matched.

### Why check a right-table column?

Use a right-table column that is guaranteed non-NULL for a real matched row, typically its primary key or the joined foreign key if that column is non-NULL.

### Common wrong approach

```sql
-- Incorrect for this requirement
SELECT ...
FROM patients p
JOIN admissions a
WHERE p.patient_id != a.patient_id;
```

Why it is wrong:

- `!=` asks for unequal values; it does not mean “no matching row exists.”
- An INNER JOIN already requires a matching row under its join condition.
- The correct task is an anti-match: retain patients for whom no matching admission exists.

### Equivalent approach: NOT EXISTS

```sql
SELECT
    p.patient_id,
    p.first_name,
    p.last_name
FROM patients p
WHERE NOT EXISTS (
    SELECT 1
    FROM admissions a
    WHERE a.patient_id = p.patient_id
);
```

Read it as: return a patient only if no matching admission row exists.

### Alternative: NOT IN

```sql
SELECT
    p.patient_id,
    p.first_name,
    p.last_name
FROM patients
WHERE patient_id NOT IN (
    SELECT patient_id
    FROM admissions
);
```

**Important NULL caveat:** If the subquery returns even one NULL, `NOT IN` can evaluate to UNKNOWN for comparisons and unexpectedly return no rows. `NOT EXISTS` avoids this specific problem and is usually a safer default for anti-match logic.

### Which one should you use?

| Method | Good to know |
|---|---|
| `LEFT JOIN ... IS NULL` | Very visual; makes unmatched rows easy to understand |
| `NOT EXISTS` | Expresses “no matching row exists”; NULL-safe for this pattern |
| `NOT IN` | Concise, but beware of NULL values in the subquery |

---

## 5. GROUP BY and Aggregate Functions

Aggregate functions calculate a value over a set of rows.

| Function | Purpose |
|---|---|
| `COUNT(*)` | Count rows |
| `COUNT(column)` | Count non-NULL values in a column |
| `SUM(column)` | Add values |
| `AVG(column)` | Calculate average |
| `MIN(column)` | Smallest value |
| `MAX(column)` | Largest value |

### 5.1 Count patients by province

```sql
SELECT
    pn.province_name,
    COUNT(p.patient_id) AS patient_count
FROM patients p
JOIN province_names pn
    ON p.province_id = pn.province_id
GROUP BY pn.province_id, pn.province_name
ORDER BY patient_count DESC;
```

**Query understanding:**

1. JOIN patients to province names using `province_id`.
2. Group rows by province.
3. Count patient IDs in each province.
4. Sort the largest count first.

`DESC` means descending; `ASC` means ascending.

### Why group by both ID and name?

Grouping by both `province_id` and `province_name` is explicit and safe when selecting both. Grouping only by the name could merge distinct provinces if names were not unique.

### 5.2 GROUP BY multiple columns

To group patients by the combination of first name and last name:

```sql
SELECT
    first_name,
    last_name,
    COUNT(patient_id) AS name_count
FROM patients
GROUP BY first_name, last_name;
```

Each distinct `(first_name, last_name)` pair forms one group.

---

## 6. Duplicate Detection With GROUP BY and HAVING

Question pattern:

> Display first name, last name, and the number of duplicate patients based on identical first and last names.

```sql
SELECT
    first_name,
    last_name,
    COUNT(patient_id) AS num_of_duplicates
FROM patients
GROUP BY first_name, last_name
HAVING COUNT(patient_id) >= 2;
```

### Example input

| first_name | last_name |
|---|---|
| John | Smith |
| Rahul | Jain |
| John | Smith |
| Aman | Sharma |
| Rahul | Jain |
| John | Smith |

### Grouped result before HAVING

| first_name | last_name | count |
|---|---|---:|
| John | Smith | 3 |
| Rahul | Jain | 2 |
| Aman | Sharma | 1 |

### After HAVING

| first_name | last_name | num_of_duplicates |
|---|---|---:|
| John | Smith | 3 |
| Rahul | Jain | 2 |

### Why HAVING instead of WHERE?

- `WHERE` filters individual rows before grouping.
- `HAVING` filters groups after aggregation.

This is not valid in the usual way:

```sql
-- Do not use WHERE to filter an aggregate count
WHERE COUNT(patient_id) >= 2
```

Use:

```sql
HAVING COUNT(patient_id) >= 2
```

### Why not GROUP BY CONCAT(first_name, ' ', last_name)?

This may group by the combined name, but selecting `first_name` and `last_name` separately while grouping only by the expression can fail under MySQL's `ONLY_FULL_GROUP_BY` mode.

Prefer:

```sql
GROUP BY first_name, last_name
```

This also expresses the intended two-column grouping clearly.

**Important interpretation:** This query counts rows with the same name. It does not prove that the records belong to the same real person. Two different people can share a name.

---

## 7. WHERE vs HAVING

| WHERE | HAVING |
|---|---|
| Filters individual rows | Filters grouped results |
| Applied before grouping | Applied after grouping |
| Normally does not use aggregate conditions | Commonly uses aggregate conditions |
| Example: `WHERE gender = 'M'` | Example: `HAVING COUNT(*) > 1` |

Example combining both:

```sql
SELECT
    province_id,
    COUNT(*) AS male_patient_count
FROM patients
WHERE gender = 'M'
GROUP BY province_id
HAVING COUNT(*) >= 5;
```

Meaning:
1. Keep male patients.
2. Group them by province.
3. Count patients in each province.
4. Keep provinces with at least five male patients.

---

## 8. Arithmetic Operators in SQL

SQL supports standard arithmetic operators.

| Operation | Operator | Example |
|---|---|---|
| Addition | `+` | `10 + 5` |
| Subtraction | `-` | `10 - 5` |
| Multiplication | `*` | `10 * 5` |
| Division | `/` | `10 / 5` |
| Modulo / remainder | `%` | `10 % 3` |

Examples:

```sql
SELECT salary * 12 AS annual_salary
FROM employees;
```

```sql
SELECT total_marks / 5 AS average_marks
FROM students;
```

### Percentage calculation

```sql
SELECT
    obtained_marks * 100.0 / total_marks AS percentage
FROM students;
```

Using `100.0` helps produce a decimal calculation. Division-by-zero must also be considered if `total_marks` can be zero:

```sql
SELECT
    obtained_marks * 100.0 / NULLIF(total_marks, 0) AS percentage
FROM students;
```

`NULLIF(total_marks, 0)` returns NULL when the denominator is zero, avoiding a divide-by-zero calculation.

### Aggregate arithmetic

```sql
SELECT
    SUM(salary) / COUNT(*) AS average_salary
FROM employees;
```

In practice, `AVG(salary)` is clearer:

```sql
SELECT AVG(salary) AS average_salary
FROM employees;
```

---

## 9. ROUND, CEIL, FLOOR and TRUNCATE

### ROUND()

Syntax:

```sql
ROUND(number, decimal_places)
```

Examples:

```sql
SELECT ROUND(12.567, 2); -- 12.57
SELECT ROUND(12.567, 1); -- 12.6
SELECT ROUND(12.567, 0); -- 13
```

### Related functions

| Function | Meaning | Example |
|---|---|---|
| `ROUND(x, d)` | Round to `d` decimal places | `ROUND(12.567, 2)` → `12.57` |
| `CEIL(x)` / `CEILING(x)` | Smallest integer greater than or equal to x | `CEIL(12.2)` → `13` |
| `FLOOR(x)` | Largest integer less than or equal to x | `FLOOR(12.9)` → `12` |
| `TRUNCATE(x, d)` | Cut off digits after `d` decimal places | `TRUNCATE(12.567, 2)` → `12.56` |

Rounding behavior can depend on numeric type and database implementation for some halfway cases. For ordinary placement questions, know the function purpose and syntax.

---

## 10. CASE WHEN: Convert Abbreviations Into Full Labels

Question pattern:

> Display gender in non-abbreviated form.

```sql
SELECT
    first_name,
    last_name,
    CASE
        WHEN gender = 'M' THEN 'MALE'
        WHEN gender = 'F' THEN 'FEMALE'
        ELSE gender
    END AS gender
FROM patients;
```

### How CASE works

- SQL checks conditions from top to bottom.
- When a condition is true, its `THEN` value is returned.
- If none matches, `ELSE` is returned.
- `END` closes the expression.

### Why include ELSE?

It preserves unexpected values instead of turning them into NULL. If the data is guaranteed to contain only `M` and `F`, an ELSE may not change the result, but it is still useful defensive handling.

### CASE can also create categories

```sql
SELECT
    patient_id,
    age,
    CASE
        WHEN age < 18 THEN 'Minor'
        WHEN age < 60 THEN 'Adult'
        ELSE 'Senior'
    END AS age_group
FROM patients;
```

The order matters: the first matching condition wins.

### MySQL IF() alternative

```sql
SELECT
    IF(gender = 'M', 'MALE', 'FEMALE') AS gender
FROM patients;
```

`IF()` handles a simple two-way condition. `CASE` is more flexible and portable across SQL databases.

---

## 11. Complete Practical Example: Patient Data Transformation

Requirement:

- Display patient's full name.
- Convert height from centimetres to feet, rounded to 1 decimal.
- Convert weight from kilograms to pounds, rounded to 0 decimals.
- Display birth date.
- Expand gender abbreviations.

Conversion rules:
- Feet = centimetres / 30.48
- Pounds = kilograms × 2.205

```sql
SELECT
    CONCAT(first_name, ' ', last_name) AS patient_name,
    ROUND(height / 30.48, 1) AS height_in_feet,
    ROUND(weight * 2.205, 0) AS weight_in_pounds,
    birth_date,
    CASE
        WHEN gender = 'M' THEN 'MALE'
        WHEN gender = 'F' THEN 'FEMALE'
        ELSE gender
    END AS gender
FROM patients;
```

### Understand each expression

| Expression | What it does |
|---|---|
| `CONCAT(first_name, ' ', last_name)` | Combines first and last name |
| `height / 30.48` | Converts centimetres to feet |
| `ROUND(..., 1)` | Keeps one decimal place |
| `weight * 2.205` | Converts kilograms to pounds |
| `ROUND(..., 0)` | Rounds to a whole number |
| `CASE WHEN` | Converts gender codes to labels |

---

## 12. SQL Logical Query Processing Order

A useful simplified logical order is:

1. `FROM` (including JOINs and ON conditions)
2. `WHERE`
3. `GROUP BY`
4. `HAVING`
5. `SELECT`
6. `DISTINCT`
7. `ORDER BY`
8. `LIMIT`

This explains why:
- `WHERE` filters rows before grouping.
- `HAVING` filters groups after grouping.
- A SELECT alias is not available in `WHERE` in standard SQL processing.
- MySQL permits SELECT aliases in `HAVING` and `ORDER BY` in many cases, but using the aggregate expression in `HAVING` is clear and portable.

---

## 13. Common Mistakes and Their Fixes

### Mistake 1 — Missing FROM before JOIN

Incorrect:

```sql
SELECT ...
JOIN patients, province_names ON ...;
```

Correct:

```sql
SELECT ...
FROM patients p
JOIN province_names pn
    ON p.province_id = pn.province_id;
```

### Mistake 2 — Wrong JOIN key

Do not join tables just because they both contain columns with similar names. Identify the actual relationship from the schema.

### Mistake 3 — Using `!=` to find missing records

Incorrect:

```sql
WHERE p.patient_id != a.patient_id
```

Correct anti-match patterns:

```sql
LEFT JOIN admissions a
    ON p.patient_id = a.patient_id
WHERE a.patient_id IS NULL;
```

or `NOT EXISTS`.

### Mistake 4 — Selecting an ungrouped column

If a query groups by `first_name, last_name`, selecting unrelated non-aggregated columns may fail or produce invalid/ambiguous results.

### Mistake 5 — Using WHERE for aggregate filters

Incorrect:

```sql
WHERE COUNT(*) >= 2
```

Correct:

```sql
HAVING COUNT(*) >= 2
```

### Mistake 6 — Forgetting that COUNT(column) ignores NULL

- `COUNT(*)` counts rows.
- `COUNT(patient_id)` counts non-NULL patient IDs.
- `COUNT(DISTINCT patient_id)` counts distinct non-NULL patient IDs.

### Mistake 7 — Forgetting NULL behavior in NOT IN

If the subquery can return NULL, prefer `NOT EXISTS` or filter NULLs explicitly inside the subquery.

### Mistake 8 — Using `result[i]` before creating vector space (programming analogy)

In C++, an empty vector cannot be indexed at position `i`; use `push_back()` or pre-size the vector. This is not SQL, but the same habit applies: understand whether an operation creates rows/space or merely accesses existing ones.

---

## 14. Placement Patterns to Memorise

### Pattern A — Count records by category

```sql
SELECT category, COUNT(*) AS total
FROM items
GROUP BY category
ORDER BY total DESC;
```

### Pattern B — Find duplicate combinations

```sql
SELECT col1, col2, COUNT(*) AS occurrences
FROM table_name
GROUP BY col1, col2
HAVING COUNT(*) > 1;
```

### Pattern C — Find entities with no related records

```sql
SELECT t1.*
FROM table1 t1
LEFT JOIN table2 t2
    ON t1.id = t2.t1_id
WHERE t2.t1_id IS NULL;
```

### Pattern D — Same anti-match using NOT EXISTS

```sql
SELECT t1.*
FROM table1 t1
WHERE NOT EXISTS (
    SELECT 1
    FROM table2 t2
    WHERE t2.t1_id = t1.id
);
```

### Pattern E — Join three related tables

```sql
SELECT ...
FROM table1 t1
JOIN table2 t2
    ON t1.id = t2.t1_id
JOIN table3 t3
    ON t2.other_id = t3.id;
```

### Pattern F — Conditional label

```sql
SELECT
    CASE
        WHEN condition1 THEN 'Label 1'
        WHEN condition2 THEN 'Label 2'
        ELSE 'Other'
    END AS category
FROM table_name;
```

### Pattern G — Calculate and round

```sql
SELECT ROUND(value * conversion_factor, 2) AS converted_value
FROM table_name;
```

---

## 15. Active Recall: Test Yourself

Try each question before looking at the answer. The schemas below are illustrative.

### Q1 — Two-table JOIN

Tables:
- `employees(employee_id, department_id, name)`
- `departments(department_id, department_name)`

Display employee name and department name.

<details>
<summary>Answer</summary>

```sql
SELECT
    e.name,
    d.department_name
FROM employees e
JOIN departments d
    ON e.department_id = d.department_id;
```
</details>

### Q2 — Three-table JOIN

Tables:
- `students(student_id, name)`
- `enrollments(student_id, course_id)`
- `courses(course_id, course_name)`

Display student name and course name.

<details>
<summary>Answer</summary>

```sql
SELECT
    s.name,
    c.course_name
FROM students s
JOIN enrollments e
    ON s.student_id = e.student_id
JOIN courses c
    ON e.course_id = c.course_id;
```
</details>

### Q3 — Find customers who never ordered

Tables:
- `customers(customer_id, name)`
- `orders(order_id, customer_id)`

<details>
<summary>Answer: LEFT JOIN</summary>

```sql
SELECT
    c.customer_id,
    c.name
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
WHERE o.customer_id IS NULL;
```
</details>

### Q4 — Find duplicate email addresses

Table: `users(user_id, email)`

<details>
<summary>Answer</summary>

```sql
SELECT
    email,
    COUNT(*) AS occurrences
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```
</details>

### Q5 — Count admissions per patient, including patients with zero admissions

Tables: `patients(patient_id)` and `admissions(admission_id, patient_id)`.

<details>
<summary>Answer</summary>

```sql
SELECT
    p.patient_id,
    COUNT(a.admission_id) AS admission_count
FROM patients p
LEFT JOIN admissions a
    ON p.patient_id = a.patient_id
GROUP BY p.patient_id;
```

Use `COUNT(a.admission_id)`, not `COUNT(*)`: for a patient with no admission, the LEFT JOIN still creates one output row, but `a.admission_id` is NULL. `COUNT(column)` ignores that NULL and returns zero.
</details>

### Q6 — Calculate percentage

Table: `exam_results(student_id, obtained_marks, total_marks)`

<details>
<summary>Answer</summary>

```sql
SELECT
    student_id,
    ROUND(
        obtained_marks * 100.0 / NULLIF(total_marks, 0),
        2
    ) AS percentage
FROM exam_results;
```
</details>

### Q7 — Convert a code to a label

Table: `orders(order_id, status)` where status codes are `P`, `C`, and `X`.

<details>
<summary>Answer</summary>

```sql
SELECT
    order_id,
    CASE
        WHEN status = 'P' THEN 'Pending'
        WHEN status = 'C' THEN 'Completed'
        WHEN status = 'X' THEN 'Cancelled'
        ELSE 'Unknown'
    END AS status_label
FROM orders;
```
</details>

---

## 16. Seven-Day Practice Roadmap

A short plan to strengthen these concepts through implementation.

| Day | Focus | Practice target |
|---|---|---|
| 1 | SELECT, WHERE, CONCAT, CASE | 8–10 questions |
| 2 | INNER JOIN, LEFT JOIN | 8–10 questions |
| 3 | Three-table and multi-table JOINs | 8 questions |
| 4 | COUNT, SUM, AVG, GROUP BY | 8–10 questions |
| 5 | HAVING, duplicate detection | 8 questions |
| 6 | NOT EXISTS, NOT IN, LEFT JOIN + NULL | 8 questions |
| 7 | Mixed revision under time limit | 10–15 questions |

### How to practise each question

1. Read the requirement and identify the expected output row.
2. Write the table names and their join keys.
3. Decide INNER JOIN vs LEFT JOIN.
4. Write a basic query first.
5. Add filters, grouping, calculations, and sorting.
6. Test edge cases: duplicates, NULLs, no matches, and multiple matches.
7. Explain the query aloud in plain English.

---

## 17. Final One-Page Revision Sheet

```sql
-- FULL NAME
CONCAT(first_name, ' ', last_name)

-- TWO-TABLE JOIN
FROM table1 t1
JOIN table2 t2 ON t1.id = t2.t1_id

-- THREE-TABLE JOIN
FROM table1 t1
JOIN table2 t2 ON t1.id = t2.t1_id
JOIN table3 t3 ON t2.other_id = t3.id

-- KEEP ALL LEFT TABLE ROWS
LEFT JOIN table2 t2 ON t1.id = t2.t1_id

-- FIND ROWS WITH NO MATCH
LEFT JOIN table2 t2 ON t1.id = t2.t1_id
WHERE t2.t1_id IS NULL

-- SAME IDEA WITH NOT EXISTS
WHERE NOT EXISTS (
    SELECT 1
    FROM table2 t2
    WHERE t2.t1_id = t1.id
)

-- COUNT BY GROUP
SELECT category, COUNT(*)
FROM table_name
GROUP BY category

-- FILTER GROUPS
HAVING COUNT(*) >= 2

-- DUPLICATE COMBINATIONS
GROUP BY first_name, last_name
HAVING COUNT(*) > 1

-- MULTIPLICATION / DIVISION
price * quantity
obtained_marks / total_marks

-- ROUNDING
ROUND(value, 2)

-- CONDITIONAL MAPPING
CASE
    WHEN code = 'A' THEN 'Active'
    WHEN code = 'I' THEN 'Inactive'
    ELSE 'Unknown'
END

-- SORT
ORDER BY column_name DESC
```

### The most important mental models

- **JOIN:** Match related rows using keys.
- **LEFT JOIN:** Preserve every row from the left table.
- **LEFT JOIN + IS NULL:** Keep left-side rows without a match.
- **GROUP BY:** Form groups of rows.
- **COUNT:** Count rows or non-NULL values, depending on the expression.
- **HAVING:** Filter groups.
- **CASE:** Return a value based on conditions.
- **ROUND:** Control decimal precision.
- **NOT EXISTS:** Check that a matching row does not exist.

> **Final advice:** Do not memorise only query syntax. For every query, be able to explain what one output row represents, why each JOIN key is correct, and what happens when a match is missing or repeated. That understanding is what makes SQL problem-solving reliable in placement tests and interviews.
"""

output_path = "/mnt/data/SQL_Premium_Notes_Session.md"
pypandoc.convert_text(content, "md", format="md", outputfile=output_path, extra_args=["--standalone"])
print(f"Created: {output_path}")
