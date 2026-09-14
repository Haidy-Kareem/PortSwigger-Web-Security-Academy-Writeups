# Lab- SQL injection UNION attack, retrieving data from other tables

## Objective

The application contains a SQL Injection vulnerability in the product category filter. Because the application returns the results of the SQL query directly in the HTTP response, a UNION-based SQL injection can be used to retrieve data from **other database tables** — not just the `products` table the original query was written for.

The lab provided the target table and column names in advance:

```
Table: users

Columns:
- username
- password
```

The goal was to retrieve the credentials of the `administrator` user and use them to log in.

---

## 1. Reconnaissance — Building on Previous Labs

Before attempting the final attack, I relied on the two techniques established in earlier labs against this same application:

1. **Determining the number of columns** returned by the original query, using `ORDER BY` / `UNION SELECT NULL,...` — previously confirmed to be **3 columns**.
2. **Identifying which columns are compatible with string data** and reflected in the visible response — previously confirmed to include a column that displays string values correctly.

This groundwork mattered because a `UNION SELECT` has two hard requirements:

- The injected query must return the **same number of columns** as the original query.
- The corresponding columns must have **compatible data types**.

Since both conditions were already satisfied from the earlier labs, I could move straight to extracting real data instead of re-deriving the query structure from scratch.

---

## 2. Exploitation

Because the lab explicitly supplied the table (`users`) and column names (`username`, `password`), there was no need to enumerate the database schema manually — the target was already known.

I used the following payload in the `category` parameter:

```
' UNION SELECT USERNAME,PASSWORD FROM USERS--
```

Submitted as:

```
GET /filter?category=Gifts'UNION SELECT USERNAME,PASSWORD FROM USERS--
```

### Payload Breakdown

- `'` — closes the original quoted `category` string, breaking out of the intended value.
- `UNION SELECT` — appends the results of a second, attacker-controlled query onto the original query's results.
- `USERNAME,PASSWORD` — selects the two specific pieces of data being targeted.
- `FROM USERS` — pulls that data from the `users` table instead of `products`.
- `-` — comments out the rest of the original query (including the `released = 1` condition), so it doesn't interfere with the injected `SELECT`.

---

## 3. Retrieved Data

The injected query caused the application to render the contents of the `users` table directly inside the product listing page, alongside the normal product entries.

Among the returned records was the `administrator` account, along with its password hash-looking value:

```
carlos → uka618nkvhsmp1dmn6lq
administrator → dhpqgwdabw603ynnp2q1
```

<img width="1918" height="1032" alt="image" src="https://github.com/user-attachments/assets/3fe333a9-36b8-46cd-82b6-2b8faaeb3cce" />


---

## 4. Logging In

I took the retrieved `administrator` credentials and used them on the application's **My Account** login page.

The login succeeded, the page confirmed:

> **Your username is: administrator**

and the lab was marked **Solved**.

<img width="1917" height="1021" alt="image" src="https://github.com/user-attachments/assets/2baaf6bf-fead-43dc-b218-249325f01cef" />


---

## 5. Final Payload

The payload that solved the lab was:

```
' UNION SELECT USERNAME,PASSWORD FROM USERS--
```

---

## 6. Notes

The important concept demonstrated here is that a UNION-based SQL injection attack is rarely a single payload it's usually performed in **stages**. First, you understand the shape of the original query (column count, data types), and only once that structure is known do you construct a compatible injected query that pulls the actual data you're after. Skipping the earlier reconnaissance steps and jumping straight to `UNION SELECT username, password FROM users` would likely fail if the column count or data types didn't line up.
