## Objective

The application contains a SQL Injection vulnerability in the product category filter. The goal of this lab was to use a UNION-based SQL injection to retrieve and display the database's exact version string.

---

## 1. Identifying the Injection Point

The vulnerable parameter was the `category` parameter:

```
GET /filter?category=Pets
```

I tested it by adding a single quote:

```
Pets'
```

This produced an **Internal Server Error**, confirming the input was being inserted directly into a SQL query without sanitization.

---

## 2. Finding the Correct Comment Syntax

The next step was commenting out the rest of the original query. I tried the standard `--` comment syntax a few different ways:

```
Pets'--
Pets'--
```

Neither worked. I also tried a few other approaches along the way — `NULL`, `GROUP BY` — but kept hitting errors.

After more testing, I tried:

```
Pets'#
```

This worked. `#` is MySQL's single-line comment character, so getting a successful response with `#` (while `--` failed) was a useful early clue that the backend was likely **MySQL**, before I'd confirmed the version explicitly.

---

## 3. Determining the Number of Columns

With a working comment syntax in hand, I moved on to building the actual UNION query. I tested different numbers of `NULL` placeholders to find the right column count:

```
' UNION SELECT NULL#
```

then increased the number of `NULL`s from there. I'd also tried `ORDER BY 1` through `ORDER BY 15` earlier, but those attempts just produced Internal Server Errors without giving me a clear signal either way, so I relied on the `UNION SELECT NULL,...` approach instead. The requirement throughout was that the UNION query has to return the exact same number of columns as the original query, or the database rejects it outright.

---

## 4. Retrieving the Database Version

Since the lab specifically asked for the database version, and both MySQL and Microsoft SQL Server support the `@@version` system variable, I substituted `@@version` into one of the column positions while adjusting the number of `NULL` placeholders around it.

The working payload was:

```
' UNION SELECT NULL,@@version#
```

Submitted as:

```
GET /filter?category=Pets'UNION SELECT NULL,@@version#
```

_(screenshot: Burp Repeater request showing the payload and response)_

This worked because the query needed exactly two columns: `NULL` filled the first position as a placeholder, while `@@version` in the second position returned the actual database version string. The `#` commented out the rest of the original query, including the `released = 1` restriction.

---

## 5. Result

The database version string was returned directly in the application's response, confirming the injection worked. The Burp response showed the lab banner updated to `is-solved`, and reloading the page in the browser confirmed:

> **Congratulations, you solved the lab!**

_(screenshot: browser confirming the lab is solved, with the payload `Pets'UNION SELECT NULL,@@version#` visible in the page title)_

---

## 6. Final Payload

The payload that solved the lab was:

```
' UNION SELECT NULL,@@version#
```

---

## 7.Notes

This lab was less about the final payload and more about the process of getting there testing, hitting errors, and using those errors as information rather than just noise.

A single quote is a reliable first test to check whether input is landing inside a SQL query. Comment syntax isn't universal across database engines `--` didn't work here, but `#` did, and that mismatch alone was a strong early hint that the backend was MySQL rather than PostgreSQL or Oracle. `ORDER BY` is a common technique for finding the column count, but it doesn't always give a clean signal — in this case it just produced errors without narrowing anything down, so falling back to incrementing `NULL` values in a `UNION SELECT` was more productive. And once the column count and comment syntax were both sorted, `@@version` dropped straight into the right column position gave the database version, since it's supported by both MySQL and Microsoft SQL Server.

The failed attempts here weren't wasted effort — each one narrowed down what was and wasn't true about the target. The `--` failure combined with the `#` success was effectively a fingerprinting step on its own, done before I ever explicitly queried `@@version` to confirm the DBMS. Paying attention to _why_ something failed, not just moving on to the next guess, is what made it possible to reason through this instead of brute-forcing a payload from a cheat sheet with no understanding of why it worked.
