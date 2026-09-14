# Lab- SQL injection UNION attack, retrieving multiple values in a single column

## Objective

The application contains a SQL Injection vulnerability in the product category filter. The results of the SQL query are reflected in the HTTP response, which makes a UNION-based SQL injection possible.

The database contains a `users` table with a `username` and `password` column. The goal was to retrieve all usernames and passwords, then use the `administrator` credentials to log in — but this time, only a **single column** in the query result is actually reflected back to the page, so both values would need to be squeezed into that one column.

---

## 1. Determining the Number of Columns

I started by testing the column count, the same way as previous labs. This payload returned an error:

```
' UNION SELECT NULL--
```

This one succeeded:

```
' UNION SELECT NULL,NULL--
```

That confirmed the original query returns **2 columns**.

---

## 2. Identifying the Reflected Column

Next, I needed to work out which of the two columns is actually displayed on the page. I tested:

```
' UNION SELECT 'A',NULL--
```

The letter `A` did not appear anywhere in the response. Then I tried:

```
' UNION SELECT NULL,'A'--
```

This time `A` appeared in the product listing. So column 1 isn't reflected, but **column 2 is** — meaning any data I want visible has to go into that second column.

---

## 3. Identifying the Database Version

Before building the final payload, I identified the underlying database engine using the reflected column, since the syntax for combining multiple values differs between database types:

```
' UNION SELECT NULL,version()--
```

The application returned:

```
PostgreSQL 12.22 (Ubuntu 12.22-0ubuntu0.20.04.4) ...
```

  

<img width="1917" height="1048" alt="image" src="https://github.com/user-attachments/assets/aeb499e5-5030-444f-bad5-c6c97565ecce" />


This confirmed the database is **PostgreSQL**, which matters because PostgreSQL uses the `||` operator to concatenate (join together) strings — other databases like MySQL or SQL Server use different syntax for the same thing.

---

## 4. Retrieving Usernames and Passwords

Since only one column is reflected, but I needed two pieces of data (`username` and `password`), the solution was to concatenate both values into that single reflected column, separated by a character that wouldn't naturally appear in either value — I used `~` as the separator:

```
username || '~' || password
```

Putting that into the second column of the UNION, the final payload was:

```
' UNION SELECT NULL, username || '~' || password FROM users--
```

Submitted as:

```
GET /filter?category=Lifestyle'UNION SELECT NULL, username || '~' || password FROM users--
```

The response displayed rows like:

```
administrator~gjwtxvkcsuy07aj05543
wiener~p5wj4xh8frj6zbtiktqm
```

<img width="1918" height="1072" alt="image" src="https://github.com/user-attachments/assets/628f66e7-54c0-4136-a1a6-e4082ab4cddb" />

Since both the username and password were now packed into a single visible string, I could just split on the `~` to read them apart.

---

## 5. Logging in as Administrator

I took the `administrator` username and the password extracted after the `~`, and used them on the application's login page.

The login succeeded, the page confirmed "Your username is: administrator," and the lab was marked **Solved**.

<img width="1918" height="1026" alt="image" src="https://github.com/user-attachments/assets/88328367-1dcb-412a-851a-e32beba21b2b" />

---

## 6. Final Payload

The payload that solved the lab was:

```
'UNION SELECT NULL, username || '~' || password FROM users--
```

---

## 7. Notes

This lab added a wrinkle to the standard UNION attack process: even after confirming the column count and figuring out which column gets reflected, sometimes there just isn't enough "room" in the output to display everything you want in one shot. The workaround is to combine multiple pieces of data into a single string using string concatenation, with a separator character in between so the combined value can still be read apart afterward.

The reasoning followed the same overall progression as earlier labs — confirm the injection point, work out the column count, figure out which column is actually shown on the page — but added two new steps on top: identifying the database engine so I'd know the correct concatenation syntax, and then building a concatenated string instead of a plain single value to fit both the username and password into that one reflected column.
