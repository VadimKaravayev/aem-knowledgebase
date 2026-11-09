# Database normalization: the normal forms in plain terms

Normalization is the process of organizing tables to **reduce data duplication and prevent inconsistencies**. The main idea: **store each fact in the right place, preferably only once.** Facts that belong together go in one table; facts that are independent of each other get separate tables.

Normal forms are cumulative: a table in 3NF is also in 2NF and 1NF, a table in BCNF is also in 3NF, and so on.

## Terms in plain words

| Term | Plain meaning | Example |
|---|---|---|
| **Key / candidate key** | A minimal set of columns that uniquely identifies a row | `employee_id` |
| **Composite key** | A key made of more than one column | `(order_id, product_id)` |
| **Superkey** | Any set of columns that uniquely identifies a row, a key plus possibly extra columns | `(employee_id, name)` |
| **Non-key attribute** | A column that is not part of any candidate key | `employee_name` |
| **Functional dependency `A → B`** | "If you know A, you know B." Same A always means same B | `dept_id → dept_name` |
| **Determinant** | The left side of a dependency, the thing that decides | `dept_id` in the line above |
| **Transitive dependency** | A → B → C: the key decides B, and B decides C, so C depends on the key only *through* B | `emp_id → dept_id → dept_name` |
| **Multi-valued dependency** | One value is linked to a *set* of values, independently of other columns | an employee has a set of skills |

## Why normalize

- **Avoid duplication**: don't repeat the same information unnecessarily.
- **Prevent update anomalies**: renaming a department shouldn't require updating 100 employee rows (and miss one, leaving two names for the same department).
- **Prevent insertion anomalies**: you should be able to add a department before it has any employees.
- **Prevent deletion anomalies**: deleting the last employee of a department shouldn't erase the department's information.

All three anomalies have the same cause: one table stores two different kinds of facts, so they can't be changed independently.

## The six normal forms

### 1NF: one value per cell

Each cell holds a single value; no lists, no repeating groups such as `phone1, phone2, phone3`.

**Before: `orders`**

| order_id | customer | products |
|---|---|---|
| 1 | C1 | pen, ink, paper |

"Which orders contain ink?" now means string parsing, and you can't store a quantity per product.

**After: `orders`** (one row per order-product pair)

| order_id | customer | product |
|---|---|---|
| 1 | C1 | pen |
| 1 | C1 | ink |
| 1 | C1 | paper |

This is valid 1NF, but `customer` now repeats on every line of the order. 2NF below removes that repetition.

### 2NF: depend on the whole key

Only relevant when a key has more than one column. Every non-key column must depend on the **whole** key, not just part of it.

**Before: `orders`** (the 1NF result plus product ID and quantity; key `(order_id, product_id)`)

| order_id | product_id | customer | product_name | quantity |
|---|---|---|---|---|
| 1 | P1 | C1 | Pen | 3 |
| 1 | P2 | C1 | Ink | 1 |
| 2 | P1 | C2 | Pen | 5 |

`quantity` needs both columns (how many pens *in this order*). The other two depend on only part of the key:

- `customer` depends on `order_id` alone, so it repeats on every line of the order.
- `product_name` depends on `product_id` alone, so "Pen" repeats on every order and can drift.

**After: `orders`**

| order_id | customer |
|---|---|
| 1 | C1 |
| 2 | C2 |

**After: `order_items`**

| order_id | product_id | quantity |
|---|---|---|
| 1 | P1 | 3 |
| 1 | P2 | 1 |
| 2 | P1 | 5 |

**After: `products`**

| product_id | product_name |
|---|---|
| P1 | Pen |
| P2 | Ink |

`customer` now appears once per order, not once per order line. If C1 places ten orders, `C1` shows up in ten `orders` rows. That isn't duplication: it's a reference (a foreign key), like `product_id` in `order_items`. Duplication means repeating *facts about* the customer, such as name or address. Those belong in a `customers(customer_id, name, address)` table, stored once.

### 3NF: no transitive dependencies

A non-key column must not depend on another non-key column. Every non-key column describes "the key, the whole key, and nothing but the key".

**Before: `employees`**

| emp_id | name | dept_id | dept_name |
|---|---|---|---|
| 10 | Employee A | D1 | Sales |
| 11 | Employee B | D1 | Sales |
| 12 | Employee C | D2 | Support |

`emp_id → dept_id → dept_name`: the department name is a fact about the department, not about the employee. Renaming Sales means editing every Sales row, and a new department with no staff has nowhere to live.

**After: `employees`**

| emp_id | name | dept_id |
|---|---|---|
| 10 | Employee A | D1 |
| 11 | Employee B | D1 |
| 12 | Employee C | D2 |

**After: `departments`**

| dept_id | dept_name |
|---|---|
| D1 | Sales |
| D2 | Support |
| D3 | Marketing |

D3 can now exist with no employees.

This one split fixes all three anomalies from "Why normalize".

### BCNF (Boyce–Codd): every determinant is a superkey

A stricter 3NF: **whatever decides another column must itself identify the row.** 3NF lets a dependency slip through when the decided column is part of a key; BCNF doesn't.

Rules: each instructor teaches exactly one course; a student takes a course from one instructor.

**Before: `enrolments`** (key `(student, course)`)

| student | course | instructor |
|---|---|---|
| S1 | Databases | T1 |
| S2 | Databases | T1 |
| S1 | Networks | T2 |

`instructor → course`, but `instructor` is not a superkey. The table is in 3NF (`course` is part of a key) yet "T1 teaches Databases" is stored once per student, and an instructor with no students yet can't be recorded.

**After: `instructors`**

| instructor | course |
|---|---|
| T1 | Databases |
| T2 | Networks |

**After: `enrolments`**

| student | instructor |
|---|---|
| S1 | T1 |
| S2 | T1 |
| S1 | T2 |

Trade-off: the rule "a student has one instructor per course" can no longer be enforced by a single table's key. BCNF sometimes costs a constraint, which is why 3NF is often accepted instead.

### 4NF: separate independent lists

Don't put two **unrelated** multi-valued facts in one table.

An employee has a set of skills and, independently, a set of languages.

**Before: `employee_skills_languages`**

| emp | skill | language |
|---|---|---|
| E1 | Java | English |
| E1 | Java | French |
| E1 | SQL | English |
| E1 | SQL | French |

Every skill is paired with every language: 4 rows to say 2 + 2 things. Adding one skill means adding a row per language; forget one and the data implies a link that doesn't exist.

**After: `employee_skills`**

| emp | skill |
|---|---|
| E1 | Java |
| E1 | SQL |

**After: `employee_languages`**

| emp | language |
|---|---|
| E1 | English |
| E1 | French |

### 5NF: no redundant combinations

If a big table can be rebuilt exactly by joining smaller tables, store the smaller tables instead. The big table is just storing combinations you could work out.

Rule: *if a supplier supplies a part, and the supplier works on a project, and the project uses that part, then the supplier supplies that part to that project.*

**Before: `supply`**

| supplier | part | project |
|---|---|---|
| S1 | Bolt | X |
| S1 | Nut | X |
| S1 | Bolt | Y |
| S2 | Bolt | X |

Under that rule every row is derivable. If S2 starts working on project Y, the rule says `(S2, Bolt, Y)` must exist too; forget to insert it and the table contradicts the rule.

**After: `supplier_parts`**

| supplier | part |
|---|---|
| S1 | Bolt |
| S1 | Nut |
| S2 | Bolt |

**After: `supplier_projects`**

| supplier | project |
|---|---|
| S1 | X |
| S1 | Y |
| S2 | X |

**After: `project_parts`**

| project | part |
|---|---|
| X | Bolt |
| X | Nut |
| Y | Bolt |

Joining all three rebuilds `supply` exactly, the same four rows (`S1, Nut, Y` doesn't appear because project Y doesn't use nuts). Each fact is now stored once, and the derived combinations follow automatically. Without that business rule, the three-column table is the only correct design: the split would produce rows that never happened. 5NF depends on a real rule in the domain, not on the data you happen to have.

## What matters in practice

- **3NF is the usual target.** BCNF improves on it in the rare cases where they differ; 4NF and 5NF fix specialized problems, mostly many-to-many tables with three or more columns.
- Normalization optimizes **integrity and maintainability**, not query speed. More tables means more joins.
- **Denormalization** (deliberately copying data back, e.g. storing `dept_name` on an order for reporting or keeping a pre-joined read table) is a valid choice *after* normalizing, when it's done knowingly and something keeps the copies in sync. Denormalized by accident and denormalized by design are different things.
- Historical snapshots aren't duplication: an invoice storing the price *at the time of sale* records a different fact from the product's current price.
- A quick test for any column: "Is this a fact about the thing this table's key identifies?" If no, it belongs in another table.
