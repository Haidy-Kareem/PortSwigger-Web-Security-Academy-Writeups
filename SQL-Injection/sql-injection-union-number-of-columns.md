## Objective

The application contains a SQL Injection vulnerability in the product category filter. The application returns the results of the SQL query directly in the HTTP response, which makes it possible to perform a SQL injection **UNION attack**.

The goal of this lab is to determine how many columns are returned by the original SQL query.

---

## 1. Understanding the Vulnerability

The application likely executes a query similar to:

```
SELECT name, description, price
FROM products
WHERE category = 'Gifts'
```

The exact query isn't visible to us, so before anything else, we need to work out how many columns its `SELECT` statement returns.

This matters because a `UNION` query must return the **exact same number of columns** as the original query. For example:

```
SELECT a, b, c FROM table1
UNION
SELECT x, y, z FROM table2
```

Both queries return three columns, so the `UNION` is valid. If the counts don't match, the database rejects the query outright.

---

## 2. Testing for SQL Injection

The vulnerable parameter was the category filter:

```
GET /filter?category=Lifestyle
```

I first tested whether the input could interfere with the SQL syntax by adding a single quote:

```
Lifestyle'
```

This broke the query and returned an **Internal Server Error**, confirming the input is inserted directly into the SQL statement without being sanitized.

<img width="1918" height="1031" alt="image" src="https://github.com/user-attachments/assets/28fe74ad-b60d-49fc-8610-805cd34bd37b" />


---

## 3. Determining the Number of Columns with ORDER BY

Next, I used the `ORDER BY` technique to work out the column count. The idea is to increment the column index one at a time:

```
ORDER BY 1
ORDER BY 2
ORDER BY 3
ORDER BY 4
```

As long as the specified column index exists in the result set, the query executes normally. Once the index exceeds the actual number of columns, the application throws an error or the response changes noticeably.

Using this technique, I found that the query broke as soon as the index went past 3 meaning the original query returns:

**3 columns**

---

## 4. Confirming with UNION SELECT

After determining the column count, I confirmed it using a `UNION SELECT` with three `NULL` values:

```
' UNION SELECT NULL,NULL,NULL--
```

This payload was inserted into the vulnerable `category` parameter, giving the full request:

```
GET /filter?category=Lifestyle'UNION SELECT NULL,NULL,NULL--
```

**Why** `**NULL**`**?** At this stage, the goal isn't to retrieve real data it's only to confirm the column count. `NULL` works as a placeholder because it's compatible with virtually any SQL data type, which gives the injected `UNION` query the best chance of succeeding once the correct number of columns is used. So the three `NULL`s line up one-to-one with the three columns returned by the original query.

<img width="1918" height="1046" alt="image" src="https://github.com/user-attachments/assets/2cfad78e-1be0-4bd5-a086-4482643020da" />


---

## 5. Final Payload

The payload that solved the lab was:

```
' UNION SELECT NULL,NULL,NULL--
```

Submitted as:

```
GET /filter?category=Lifestyle' UNION SELECT NULL,NULL,NULL--
```

The application accepted the query without error, and the lab was marked **Solved**.

---

## 6. Notes

The key lesson from this lab is that a `UNION` attack lets an attacker append the results of a second, attacker-controlled `SELECT` query onto the original query's output but only if both queries return the same number of columns.

The important techniques demonstrated in this lab were:

- Confirming the parameter is injectable by breaking the query with a single quote.
- Using `ORDER BY N` and incrementing `N` until the query breaks, to determine the number of columns.
- Using `UNION SELECT NULL,NULL,...` as a second way to confirm the column count.
- Understanding why `NULL` is the safest placeholder value it's compatible with almost any column data type.
- Recognizing that this step (finding the column count) is a **prerequisite** for a full UNION attack, not the end goal the next step would be figuring out which of those columns can actually display attacker-controlled data back in the response.
