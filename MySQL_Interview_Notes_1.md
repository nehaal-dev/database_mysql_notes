# 📚 MySQL Interview Notes
> By Nehal Khan | Full Stack PHP Laravel Developer

---

## 📋 Table of Contents
1. [WHERE vs HAVING](#1-where-vs-having)
2. [JOINs](#2-joins)
3. [Indexes](#3-indexes)
4. [Subqueries](#4-subqueries)
5. [Transactions](#5-transactions)
6. [Normalization](#6-normalization)
7. [Query Optimization](#7-query-optimization)
8. [Keys](#8-keys)

---

## 1. WHERE vs HAVING

### Core Difference:
```
WHERE  → GROUP BY se PEHLE  → Rows filter karta hai
HAVING → GROUP BY ke BAAD   → Groups filter karta hai
```

### Sample Table — `orders`:
| order_id | customer | category | status | amount |
|----------|----------|----------|--------|--------|
| 1 | Ali | Mobile | paid | 5000 |
| 2 | Sara | Mobile | unpaid | 3000 |
| 3 | John | Laptop | paid | 80000 |
| 4 | Ali | Mobile | paid | 7000 |

### WHERE Example:
```sql
-- Sirf "paid" orders ka category wise count
SELECT category, COUNT(*) as total
FROM orders
WHERE status = 'paid'       -- pehle rows filter karo
GROUP BY category;
```

### HAVING Example:
```sql
-- Sirf wo categories jisme 2 se zyada orders hain
SELECT category, COUNT(*) as total
FROM orders
GROUP BY category
HAVING COUNT(*) > 2;        -- groups filter karo
```

### Rules:
| Rule | WHERE | HAVING |
|------|-------|--------|
| Normal Column | ✅ | ✅ |
| Aggregate Function (COUNT, SUM) | ❌ | ✅ |
| Position | GROUP BY se Pehle | GROUP BY ke Baad |

### Common Mistake:
```sql
-- ❌ GALAT — Error aayega!
SELECT category, COUNT(*)
FROM orders
WHERE COUNT(*) > 2        -- WHERE aggregate use nahi kar sakta!
GROUP BY category;

-- ✅ SAHI
SELECT category, COUNT(*) as total
FROM orders
GROUP BY category
HAVING COUNT(*) > 2;
```

### Alias Kya Hai?
```sql
COUNT(*) as total
-- "total" ek temporary nickname hai
-- Table mein save nahi hota
-- Laravel mein $data->total se access hota hai
```

---

## 2. JOINs

### Sample Tables:

**`users`:**
| id | name |
|----|------|
| 1 | Ali |
| 2 | Sara |
| 3 | John |

**`orders`:**
| id | user_id | product |
|----|---------|---------|
| 1 | 1 | Mobile |
| 2 | 1 | Laptop |
| 3 | 2 | Watch |

> ⚠️ John (id=3) ka koi order nahi!

---

### INNER JOIN — Sirf Common Data:
```sql
SELECT users.name, orders.product
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```
**Result:**
| name | product |
|------|---------|
| Ali | Mobile |
| Ali | Laptop |
| Sara | Watch |
> John nahi aaya — kyunki uska order nahi!

---

### LEFT JOIN — Left Table Ka Pura Data:
```sql
SELECT users.name, orders.product
FROM users
LEFT JOIN orders ON users.id = orders.user_id;
```
**Result:**
| name | product |
|------|---------|
| Ali | Mobile |
| Ali | Laptop |
| Sara | Watch |
| John | NULL |
> John bhi aaya — NULL ke saath!

---

### RIGHT JOIN — Right Table Ka Pura Data:
```sql
SELECT users.name, orders.product
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;
```
> Right table (orders) ka pura data aata hai!

---

### Konsa JOIN Use Karein?
```
"Saare users chahiye order ho ya na ho"  → LEFT JOIN
"Sirf order karne wale users"            → INNER JOIN
"Saare orders chahiye"                   → LEFT JOIN (orders left pe)
```

### Jo Users Ne Order Nahi Kiya:
```sql
-- Method 1: LEFT JOIN + NULL (Best!)
SELECT users.name
FROM users
LEFT JOIN orders ON users.id = orders.user_id
WHERE orders.user_id IS NULL;

-- Method 2: NOT EXISTS
SELECT users.name
FROM users
WHERE NOT EXISTS (
    SELECT 1 FROM orders
    WHERE orders.user_id = users.id
);

-- Method 3: NOT IN (Slow for large tables)
SELECT users.name
FROM users
WHERE users.id NOT IN (
    SELECT user_id FROM orders
);
```

### Table.Column Syntax Kyun?
```sql
-- Jab 2 tables hon → table.column likhna zaroori!
SELECT users.name, orders.product   -- ✅ clear hai
FROM users
LEFT JOIN orders ON users.id = orders.user_id;

-- Alias se aur easy:
SELECT u.name, o.product
FROM users as u
LEFT JOIN orders as o ON u.id = o.user_id;
```

### JOIN Rule:
```
SELECT → dono tables ke columns yahan likhdo
FROM   → pehli (main) table
JOIN   → dusri table
ON     → relation (primary_key = foreign_key)
```

---

## 3. Indexes

### Index Kya Hai?
```
Index = Book ke end mein index jaisa
"Python → Page 245" → seedha wahan jao!

MySQL mein:
Bina Index → 10 lakh rows ek ek check karo (slow!)
Index ke saath → seedha record milta hai (fast!)
```

### Index Kab Lagao:
```sql
-- ✅ WHERE mein baar baar use hone wala column
CREATE INDEX idx_email ON users(email);

-- ✅ JOIN mein use hone wala column
CREATE INDEX idx_user_id ON orders(user_id);

-- ✅ ORDER BY mein use hone wala column
CREATE INDEX idx_created_at ON orders(created_at);

-- ✅ 10,000+ rows wali table
```

### Index Kab Mat Lagao:
```
❌ Choti table (100-200 rows)
❌ Har column pe
❌ Baar baar UPDATE hone wala column
```

### Index Ka Cost:
```
READ  → Fast ⚡
WRITE → Thoda slow ⚠️ (INSERT/UPDATE pe index bhi update hota hai)
```

### Syntax:
```sql
-- Banao
CREATE INDEX idx_email ON users(email);

-- Table banate waqt
CREATE TABLE users (
    id    INT PRIMARY KEY,        -- auto index
    email VARCHAR(100) UNIQUE,    -- auto index
    city  VARCHAR(100),
    INDEX idx_city (city)         -- manual index
);

-- Dekho
SHOW INDEX FROM users;

-- Delete karo
DROP INDEX idx_email ON users;
```

### Automatic Index:
```
PRIMARY KEY → Automatic index ✅
UNIQUE      → Automatic index ✅
Normal col  → Manually banao  ✅
```

---

## 4. Subqueries

### Subquery Kya Hai?
```sql
-- Ek query ke ANDAR doosri query
-- Parentheses () mein likhi jaati hai

SELECT name
FROM users
WHERE id IN (SELECT user_id FROM orders);
--           👆 YE HAI SUBQUERY
```

### Step by Step:
```
Step 1 → Andar wali query pehle chalti hai
Step 2 → Result bahar wali query mein jaata hai
Step 3 → Bahar wali query result deti hai
```

### 3 Jagah Use Hoti Hai:

**1. WHERE Mein:**
```sql
SELECT name
FROM users
WHERE id IN (
    SELECT user_id FROM orders
);
```

**2. SELECT Mein:**
```sql
SELECT
    name,
    (SELECT COUNT(*) FROM orders
     WHERE orders.user_id = users.id) as total_orders
FROM users;
```

**3. FROM Mein:**
```sql
SELECT * FROM (
    SELECT user_id, COUNT(*) as total
    FROM orders
    GROUP BY user_id
) as order_summary;
```

### Subquery vs JOIN:
```
Subquery → Simple, easy to read (chhoti tables)
JOIN     → Fast, better performance (badi tables)
```

---

## 5. Transactions

### Transaction Kya Hai?
```
Transaction = Group of queries
              Ya SAARI chalein ✅
              Ya KOI na chale ✅

Real Example:
Ali Sara ko 5000 bhejta hai
Step 1 → Ali ka balance - 5000
Step 2 → Sara ka balance + 5000

Beech mein error → ROLLBACK (sab undo!)
Sab sahi → COMMIT (sab save!)
```

### Syntax:
```sql
START TRANSACTION;    -- shuru karo

    INSERT INTO orders(user_id, product_id, amount)
    VALUES (1, 1, 2000);

    UPDATE products
    SET stock = stock - 1
    WHERE id = 1;

    UPDATE users
    SET total_orders = total_orders + 1
    WHERE id = 1;

COMMIT;               -- ✅ sab save karo
-- YA
ROLLBACK;             -- ↩️ sab undo karo
```

### COMMIT vs ROLLBACK:
```
COMMIT   → Sab sahi hua → Changes SAVE karo ✅
ROLLBACK → Kuch galat hua → Sab UNDO karo ↩️
```

### Laravel Mein:
```php
try {
    DB::transaction(function () {
        DB::table('orders')->insert([...]);
        DB::table('products')->where('id', 1)->decrement('stock', 1);
        DB::table('users')->where('id', 1)->increment('total_orders', 1);
    });
    // ✅ Automatic COMMIT

} catch (Exception $e) {
    // ❌ Automatic ROLLBACK
}
```

---

## 6. Normalization

### Normalization Kya Hai?
```
Database ko organize karna taaki:
→ Data repeat na ho ❌
→ Data sahi rahe ✅
→ Update easy ho ✅
```

---

### 1NF — First Normal Form
**Rule: Har cell mein SIRF EK value!**

```
-- ❌ Galat
| order_id | products          |
|----------|-------------------|
| 1        | Mobile, Laptop    |  ← 2 values!

-- ✅ Sahi
| order_id | product |
|----------|---------|
| 1        | Mobile  |
| 2        | Laptop  |
```

---

### 2NF — Second Normal Form
**Rule: Har column sirf Primary Key pe depend kare!**

```
-- ❌ Galat (customer_city order pe nahi Ali pe depend karta hai!)
| order_id | customer_name | customer_city | product |
|----------|--------------|---------------|---------|
| 1        | Ali          | Delhi         | Mobile  |
| 2        | Ali          | Delhi         | Laptop  |

-- ✅ Sahi — Alag tables banao!
users:  | id | name | city  |
orders: | id | user_id | product |
```

**Dependency Check Rule:**
```
Poocho → "Agar order badal jaaye, kya ye column badlega?"
product  → Haan! ✅ → ORDER pe depend karta hai
city     → Nahi! ❌ → USER pe depend karta hai → Alag table!
```

---

### 3NF — Third Normal Form
**Rule: Koi column doosre NON-KEY column pe depend na kare!**

```
-- ❌ Galat (category_name, category_id pe depend karta hai)
| id | product | category_id | category_name |
|----|---------|-------------|---------------|
| 1  | Mobile  | 1           | Electronics   |

-- ✅ Sahi — Alag table!
categories: | id | name        |
products:   | id | category_id | name |
```

### Summary:
```
1NF → Ek cell = Ek value ✅
2NF → Alag data = Alag table ✅
3NF → Har column sirf apni table pe depend kare ✅
```

---

## 7. Query Optimization

### EXPLAIN — Query Ka X-Ray:
```sql
EXPLAIN SELECT * FROM users WHERE email = 'ali@gmail.com';
```

**Result Mein Kya Dekhna Hai:**
```
rows  → Kam = Fast ⚡  | Zyada = Slow 😴
key   → NULL = Index nahi ❌ | idx_email = Index use hua ✅
type  → ALL = Slow 😴 | ref = Fast ⚡
```

### 5 Optimization Tips:

**1. Index Lagao:**
```sql
CREATE INDEX idx_email ON users(email);
```

**2. SELECT * Mat Likho:**
```sql
-- ❌ Slow
SELECT * FROM users;

-- ✅ Fast
SELECT id, name, email FROM users;
```

**3. WHERE Mein Function Mat Lagao:**
```sql
-- ❌ Slow (index kaam nahi karta)
WHERE YEAR(created_at) = 2024;

-- ✅ Fast
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
```

**4. LIMIT Use Karo:**
```sql
SELECT * FROM orders LIMIT 10;  -- ✅
```

**5. Subquery Ki Jagah JOIN:**
```sql
-- ❌ Slow
SELECT name FROM users
WHERE id IN (SELECT user_id FROM orders);

-- ✅ Fast
SELECT DISTINCT users.name
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

---

## 8. Keys

### Primary Key:
```
→ Unique ✅
→ NULL nahi ho sakta ✅
→ Ek table mein SIRF EK ✅
→ Har row ko identify karta hai ✅
```
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);
```

---

### Foreign Key:
```
→ Doosri table ki Primary Key ka reference ✅
→ Tables ko RELATE karta hai ✅
→ Galat data insert nahi hone deta ✅
```
```sql
CREATE TABLE orders (
    id      INT PRIMARY KEY,
    user_id INT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

---

### Unique Key:
```
→ Duplicate values nahi ✅
→ NULL ho sakta hai ✅
→ Multiple unique keys ho sakti hain ✅
```
```sql
CREATE TABLE users (
    id    INT PRIMARY KEY,
    email VARCHAR(100) UNIQUE  -- ✅
);
```

---

### Composite Key:
```
→ 2 ya zyada columns milke PRIMARY KEY bante hain ✅
→ Akela column unique nahi hota ✅
→ Dono milke unique hote hain ✅
```
```sql
CREATE TABLE students (
    class_id INT,
    roll_no  INT,
    name     VARCHAR(100),
    PRIMARY KEY (class_id, roll_no)  -- ✅ composite key
);
```

### Comparison:
| Key | Unique | NULL | Ek Table Mein |
|-----|--------|------|----------------|
| Primary Key | ✅ | ❌ | Sirf 1 |
| Foreign Key | ❌ | ✅ | Multiple |
| Unique Key | ✅ | ✅ | Multiple |
| Composite Key | Dono milke ✅ | ❌ | Sirf 1 |

---

## 🎯 Quick Revision — One Liners

```
WHERE     → Rows filter (GROUP BY se pehle)
HAVING    → Groups filter (GROUP BY ke baad)
INNER JOIN → Sirf common data
LEFT JOIN  → Left table pura + common
RIGHT JOIN → Right table pura + common
INDEX      → Search speed badhata hai
SUBQUERY   → Query ke andar query ()
TRANSACTION → Group of queries (COMMIT/ROLLBACK)
1NF        → Ek cell = Ek value
2NF        → Related data = Alag table
3NF        → Har column apni table pe depend kare
EXPLAIN    → Query ka X-Ray
PRIMARY KEY → Unique + Not Null
FOREIGN KEY → Doosri table ka reference
UNIQUE KEY  → Duplicate nahi
COMPOSITE   → 2+ columns milke primary key
```

---

*Notes by Nehal Khan | PHP Laravel Developer | nehaal.netlify.app*
