## Objective

The application contains a SQL Injection vulnerability in the product category filter, exploitable through a UNION attack. The goal of this lab was to make the database return its full version banner, specifically the exact string identifying it as an Oracle Database instance.

---

## 1. Determining the Number of Columns

Before attempting anything version-specific, I needed to confirm how many columns the original query returns, since any `UNION SELECT` has to match that count exactly.

Testing showed the query accepts:

```
' UNION SELECT NULL,NULL--
```

confirming the original query returns **2 columns**.

<img width="1456" height="816" alt="image" src="https://github.com/user-attachments/assets/62869c68-2576-48fd-a94c-52e33c7135de" />

---

this when i tried 3 columns which assure that it is 2 col

2. First Attempt — Guessing the Wrong Source

My first idea was to pull version info from `v$instance`, a system view Oracle exposes with details about the running database instance:

```
' UNION SELECT VERSION,NULL FROM v$instance--
```

This wasn't the right combination for this lab. It did teach me the distinction between two similarly-named Oracle system views, though: `v$instance` holds general instance metadata (including a `VERSION` column), while `v$version` specifically holds the full version **banner** text which is what the lab was actually asking for.

---

## 3. Second Attempt — Using Instead of Explicit Columns

I then tried:

```
' UNION SELECT * FROM v$version--
```

This returned an **Internal Server Error**.

  

The problem here was using `*` at all. A `UNION` requires both `SELECT` statements to return the exact same number of columns, and since I already knew the original query returns 2 columns, pulling in `*` from `v$version` was a mistake, I had no control over how many columns that view would actually return, and it clearly didn't line up with 2.

---

## 4. Identifying the Correct Column — `BANNER`

Going back to Oracle-specific reference material, I used the **PortSwigger SQL injection cheat sheet** at this point to confirm the exact Oracle syntax, since Oracle's system views and requirements differ from PostgreSQL or MySQL, I found that `v$version` contains a column called `BANNER`, which holds the actual version string, something like:

```
Oracle Database 11g Express Edition Release ...
```

So the real target wasn't the view name being used as a keyword `BANNER` is simply the column name inside `v$version` that holds what I needed.

---

## 5. Keeping the Column Count Correct

Since the original query returns 2 columns, and the version string only fills one of them, I needed a second placeholder column to keep the counts matching:

```
BANNER, NULL
```

---

## 6. The `FROM dual` Question

Oracle has a strict rule that every `SELECT` statement must include a `FROM` clause, even when you're just selecting a literal value with no real table behind it. For that reason, Oracle provides a special built-in table called `DUAL`, meant exactly for this — e.g. `SELECT 'test' FROM dual`.

That rule applies when selecting a fixed value, but here I wasn't selecting a literal — I was reading an actual column (`BANNER`) out of an actual Oracle system view (`v$version`). So the `FROM` clause needed to point at `v$version` itself, not `dual`.

---

## 7. Identifying the Reflected Column

Before locking in the final payload, I confirmed which of the two columns actually shows up in the page output, using simple text markers:

```
' UNION SELECT 'aaa','bbb' FROM dual--
```

Here `dual` was appropriate, since `'aaa'` and `'bbb'` are plain literal values rather than a real table's data. Watching which marker appeared on the page (or whether both did) told me exactly where to place the version data in the real payload.

---

## 8. Final Payload

Putting all of this together — 2 columns required, `BANNER` holding the version text, `NULL` as the second placeholder, and `v$version` as the source — the final payload was:

```
' UNION SELECT BANNER, NULL FROM v$version--
```

Submitted as:

```
GET /filter?category=Gifts'UNION SELECT BANNER, NULL FROM v$version--
```

The response displayed the full Oracle version banner, and the lab was marked **Solved**.


<img width="1918" height="1058" alt="image" src="https://github.com/user-attachments/assets/7fd77fc0-254c-4dba-ae75-db64aa71e9ce" />

---

## 9. Notes

The point of this lab wasn't memorizing the exact payload, it was the reasoning that led to it. The overall progression was: confirm the injection point, determine the column count, figure out which column is reflected, identify the DBMS as Oracle, then find the Oracle-specific source for version data (`v$version`) and the specific column inside it (`BANNER`), while keeping the total column count consistent throughout with `NULL` as a placeholder.

I also leaned on the **PortSwigger SQL injection cheat sheet** to confirm Oracle's exact system view and column names Oracle's syntax and system tables are different enough from PostgreSQL/MySQL that guessing without a reference wasted time on the earlier `v$instance` and `SELECT *` attempts.
