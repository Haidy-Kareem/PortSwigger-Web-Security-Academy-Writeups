# Lab- SQL injection vulnerability allowing login bypass

## Subverting Application Logic

## 1. Understanding the Concept

One of the uses of SQL Injection is to **subvert application logic**.

This means that instead of simply retrieving additional data from the database, an attacker changes the logic of the SQL query so that the application behaves differently from what the developer intended.

A common example is a login form.

Suppose a user enters:

```
Username: wiener
Password: bluecheese
```

The application might construct the following SQL query:

```sql
SELECT * FROM users
WHERE username = 'wiener'
AND password = 'bluecheese'
```

The application expects the database to return a user only when **both conditions** are true:

```
username = wiener
AND
password = bluecheese
```

If a matching user is returned, the application considers the login successful. Otherwise, the login is rejected.

---

## 2. The Problem

If the application directly inserts user-controlled input into the SQL query, an attacker may be able to modify the query's structure.

The attacker may know the username of a privileged account, such as:

```
administrator
```

but not know its password.

Instead of trying to discover the password, the attacker can attempt to **remove the password condition from the SQL query**.

The goal is to turn:

```sql
SELECT * FROM users
WHERE username = 'administrator'
AND password = 'unknown'
```

into something effectively equivalent to:

```sql
SELECT * FROM users
WHERE username = 'administrator'
```

---

## 3. The Role of the SQL Comment

SQL supports comments, and in this context:

```
--
```

can be used to comment out the remainder of the query.

For example:

```sql
SELECT * FROM users
WHERE username = 'administrator'--
AND password = ''
```

Everything after `--` is treated as a comment.

Therefore, the database effectively processes:

```sql
SELECT * FROM users
WHERE username = 'administrator'
```

The password check is no longer part of the executed condition.

---

## 4. Why the Single Quote Is Important

The application originally places the username inside quotes:

```sql
WHERE username = '$username'
```

If the attacker submits:

```
administrator'--
```

the application may construct:

```sql
WHERE username = 'administrator'--'
```

The first `'` after `administrator` closes the original SQL string.

Then:

```
--
```

comments out everything that follows.

So the attacker is effectively changing the structure of the original query.

The important idea is:

```
'       → closes the string
--      → comments out the remaining SQL
```

---

# Lab: SQL Injection Vulnerability Allowing Login Bypass

## Objective

The lab contains a SQL Injection vulnerability in the login functionality.

The goal is to log in as the `administrator` user without knowing the administrator's password.

---

## 1. Identify the Login Functionality

The application provides a login form with two inputs:

```
Username
Password
```

Normally, the backend performs a query similar to:

```sql
SELECT * FROM users
WHERE username = 'wiener'
AND password = 'bluecheese'
```

The important observation is that both values are inserted into the SQL query.

---

## 2. Identify the Target Account

The lab requires logging in as:

```
administrator
```

However, the administrator's password is unknown.

Instead of trying to guess the password, the objective is to manipulate the SQL query so that the password check is ignored.

---

## 3. Inject into the Username

I entered the following as the username:

```
administrator'--
```

The password can be left blank.

The application then constructs a query similar to:

```sql
SELECT * FROM users
WHERE username = 'administrator'--'
AND password = ''
```

Because `--` starts a SQL comment, the following part:

```sql
AND password = ''
```

is ignored.

The effective query becomes:

```sql
SELECT * FROM users
WHERE username = 'administrator'
```

---

## 4. Result

The database returns the `administrator` user's record because the query only checks the username.

The application receives a valid user record and therefore considers the authentication successful.

I was successfully logged in as the administrator without knowing the administrator's password.

The lab was marked as **Solved**.

---

## 5. What This Lab Demonstrates

This lab demonstrates how SQL Injection can be used to **subvert application logic**.

The original authentication logic was:

```
Username correct
        AND
Password correct
        ↓
Login
```

After the injection, the effective logic became:

```
Username = administrator
        ↓
Login
```

The vulnerability exists because user input is being treated as part of the SQL syntax instead of being safely handled as data.

### Notes

- SQL Injection can modify the logic of a SQL query.
- A single quote can interact with a string-based SQL context.
- `-` can comment out the remainder of a query.
- This can remove security checks such as password verification.
- The database is simply executing the modified query; the vulnerability is caused by the application's unsafe construction of SQL.
- The proper defense is to use **parameterized queries / prepared statements**, which keep SQL structure separate from user-supplied data.
