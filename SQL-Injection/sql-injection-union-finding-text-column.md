# Lab- SQL injection UNION attack, finding a column containing text

## Objective

The application contains a SQL Injection vulnerability in the product category filter. The results of the SQL query are returned directly in the HTTP response, which makes it possible to perform a SQL injection **UNION attack**.

The goal of this lab is to identify which column in the result set is compatible with string data, by making a lab-provided random value appear in the query results.

---

## 1. Understanding the Attack

A `UNION` attack requires the injected query to return the **same number of columns** as the original query.

From a previous lab against this same application, I had already determined that the original query returns **3 columns**.

The next step was to figure out **which of those three columns** can actually hold and display string data — since not every column in the underlying table is necessarily a string type, and only some columns are reflected back in the visible page output.

---

## 2. Testing for SQL Injection

The vulnerable parameter was again the category filter:

```
GET /filter?category=Pets
```

I confirmed the injection point was still exploitable by submitting a single quote:

```
Pets'
```

This produced an **Internal Server Error**, confirming the input is inserted directly into the SQL query without sanitization.

<img width="1918" height="957" alt="image" src="https://github.com/user-attachments/assets/0da6830c-a2e0-4fb3-96f0-50cf999d0972" />


The lab also provided a specific random string to use as a probe value:

```
4UFtPO
```

---

## 3. Testing the Columns

Since the original query returns 3 columns, I needed to test each column position individually to find which one accepts and reflects string data. The approach was to place the probe string `4UFtPO` into one column position at a time, using `NULL` as a placeholder for the other two:

```
' UNION SELECT '4UFtPO',NULL,NULL--
' UNION SELECT NULL,'4UFtPO',NULL--
' UNION SELECT NULL,NULL,'4UFtPO'--
```

The goal was simple: whichever version of this payload caused `4UFtPO` to actually appear on the page told me both that the column supports string data _and_ that it's one of the columns rendered in the visible response some columns might accept strings but never get displayed.

---

## 4. Confirming the Correct Column

The successful payload was:

```
' UNION SELECT NULL,'4UFtPO',NULL--
```

Submitted as:

```
GET /filter?category=Pets'UNION SELECT NULL,'4UFtPO',NULL--
```

This payload breaks down as:

```
Column 1 → NULL
Column 2 → '4UFtPO'
Column 3 → NULL
```

The value `4UFtPO` appeared directly in the application's product listing, confirming that the **second column** is compatible with string data and is reflected in the response.

  
<img width="1918" height="1007" alt="image" src="https://github.com/user-attachments/assets/eb4278e8-998d-4e0b-953e-1e14377b80dd" />


---

## 5. Final Payload

The payload that solved the lab was:

```
' UNION SELECT NULL,'4UFtPO',NULL--
```

The application displayed the probe value, and the lab was marked **Solved**.

---

## 6. Notes

The key lesson from this lab is that knowing the column _count_ isn't enough to exploit a UNION-based SQL injection you also need to know **which specific column(s)** can carry string data and are actually shown back to the user.

The important techniques demonstrated in this lab were:

- Reconfirming the injection point still behaves as expected before building on it.
- Using a unique, lab-provided random string as a reliable probe value since it's not something that could already exist naturally in the data, any appearance of it in the response is unambiguous proof of success.
- Testing each column position individually, one at a time, rather than guessing.
- Using `NULL` as a placeholder for columns not currently being tested.
- Understanding that a column being "string-compatible" and a column being "visible in the response" are two separate things this payload proved both for column 2 specifically.
