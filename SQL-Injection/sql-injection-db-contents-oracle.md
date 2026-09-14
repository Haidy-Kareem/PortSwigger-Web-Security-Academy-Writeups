# Lab- SQL injection attack, listing the database contents on Oracle

## Objective

The application contains a SQL Injection vulnerability in the product category filter, exploitable through a UNION attack. This lab specifically targets an **Oracle database**, which uses different system views for schema enumeration than PostgreSQL or MySQL. As with the non-Oracle version of this lab, no table or column names were given — I had to enumerate the schema from scratch to find and retrieve the `administrator` user's real credentials.

---

## 1. Researching Oracle's Schema Views

Since Oracle doesn't expose `INFORMATION_SCHEMA` the way PostgreSQL and MySQL do, I needed to find Oracle's equivalent before attempting anything. I looked up Oracle's official documentation for `ALL_TABLES`, which describes the relational tables accessible to the current user.

The documentation confirmed the relevant view structure, showing columns like `OWNER`, `TABLE_NAME`, `TABLESPACE_NAME`, `CLUSTER_NAME`, `IOT_NAME`, and `STATUS`, and also pointed out related views:

- `DBA_TABLES` — describes all relational tables in the database
- `USER_TABLES` — describes the tables owned by the current user
- `ALL_TABLES` — describes tables accessible to the current user

For column-level metadata specifically (which is what I actually needed to enumerate columns, not just table names), the equivalent Oracle view is `ALL_TAB_COLUMNS`.

<img width="1456" height="820" alt="image" src="https://github.com/user-attachments/assets/b4be2b3d-966f-47c6-b3b2-52399fa6c0cd" />

---

## 2. Confirming the Injection Point and Column Count

As with the other UNION labs, I confirmed the injection point and column count first before attempting any schema enumeration:

```
Pets'UNION SELECT NULL,NULL--
```

This confirmed the original query returns **2 columns**.

---

## 3. Enumerating the Schema with ALL_TAB_COLUMNS

I built the Oracle equivalent of the `INFORMATION_SCHEMA.COLUMNS` query I'd used in the non-Oracle lab, substituting in Oracle's own view name:

```
'UNION SELECT table_name, column_name FROM all_tab_columns--
```

Submitted as:

```
GET /filter?category=Gifts'UNION%20SELECT%20table_name,%20column_name%20FROM%20all_tab_columns--
```

This returned a massive dump of table/column pairs — hundreds of entries from Oracle's own built-in system views, things like `ALL_UPDATABLE_COLUMNS` → `DELETABLE`, `INSERTABLE`, `OWNER`, `TABLE_NAME`, `UPDATABLE`, followed by `ALL_USERS` → `CREATED`, `USERNAME`, `USER_ID`, and then a long run of `ALL_USTATS` entries (`ASSOCIATION`, `COLUMN_NAME`, `OBJECT_NAME`, `OBJECT_OWNER`, `OBJECT_TYPE`).

<img width="1460" height="812" alt="image" src="https://github.com/user-attachments/assets/d08ddead-4feb-4cd9-88d7-7fa056f24de6" />

Given the sheer size of the output, I used Firefox's in-page search (Ctrl+F) again to hunt through it for anything credential-related rather than scrolling manually through hundreds of rows.

---

## 4. First Attempt — Chasing `ALL_USERS`

The `ALL_USERS` → `USERNAME` pairing looked like an obvious first candidate, since it's an Oracle system view literally named "users." However, on checking Oracle's documentation more closely (and from prior knowledge of Oracle's system views), `ALL_USERS` only exposes metadata like `USERNAME`, `USER_ID`, and `CREATED` — it has no password column at all, since Oracle doesn't expose user password hashes through this view for security reasons. This was a dead end before I even tried a payload against it.

---

## 5. Second Attempt — `KU$_ROLE_VIEW`

Continuing to search through the enumerated dump, I found a much more promising-looking system view: `KU$_ROLE_VIEW`, which appeared repeatedly with columns including `CTIME`, `DEFGRP_NUM`, `DEFGRP_SEQ_NUM`, `DEFROLE`, `EXPTIME`, `EXT_USERNAME`, `LCOUNT`, `LTIME`, `NAME`, `PACKAGE`, and — critically — `PASSWORD`.

<img width="1456" height="825" alt="image" src="https://github.com/user-attachments/assets/24e4a475-3507-4e80-b7d3-aa9ce6355b3a" />


<img width="1456" height="839" alt="image" src="https://github.com/user-attachments/assets/d1a1b28b-aada-452b-a87d-3e8b392d56b1" />


This looked exactly like what I needed — a role view with both a `NAME` and a `PASSWORD` column. I built the payload:

```
'UNION SELECT NAME, PASSWORD FROM KU$_ROLE_VIEW--
```

This did **not** work for solving the lab. `KU$_ROLE_VIEW` is part of Oracle's Data Pump export/import metadata views, and while it does have a column literally called `PASSWORD`, it isn't populated with actual usable credential data in this context — another dead end, despite looking exactly right on paper.

---

## 6. Finding the Real Application Table

Going back to the full `table_name, column_name` dump, I kept searching, this time for `users` specifically, to try to spot the lab's actual custom users table hidden among the Oracle system view noise (`TABQUOTAS`, `TRANSPORTABLE_EXPORT_OBJECTS`, `TRANSPORTABLE_EXPORT_PATHS`, etc.).

At match 23 of 79, I found it:

```
USERS_WJIWCZ → EMAIL
USERS_WJIWCZ → PASSWORD_YBKORB
USERS_WJIWCZ → USERNAME_OKZVSD
```

<img width="1456" height="825" alt="image" src="https://github.com/user-attachments/assets/92c32504-3041-4d69-a848-16659af6c426" />


Just like in the non-Oracle version of this lab, the randomized suffixes (`_WJIWCZ`, `_YBKORB`, `_OKZVSD`) were the giveaway that this was the application's real, custom users table — not something Oracle ships by default.

---

## 7. Final Payload — Retrieving Real Credentials

I built the final payload using the discovered table and column names:

```
' UNION SELECT USERNAME_OKZVSD, PASSWORD_YBKORB FROM USERS_WJIWCZ--
```

Submitted as:

```
GET /filter?category=Gifts'%20UNION%20SELECT%20USERNAME_OKZVSD,%20PASSWORD_YBKORB%20FROM%20USER...
```

The response displayed real application user data mixed into the product listing:

```
administrator → 460yuvy979yr3e5clogx
carlos → awauov1dnb7pc7rlz5xy
wiener → 7vhytpqiwc2mcih53v30
```

<img width="1477" height="812" alt="image" src="https://github.com/user-attachments/assets/19c9c9e0-b629-4606-b18c-5e3bf686f2b0" />


---

## 8. Logging In as Administrator

I used the retrieved `administrator` username and password on the application's login page.

The login succeeded — the URL redirected to `/my-account?id=administrator`, the page displayed "Your username is: administrator," and the banner read **"Congratulations, you solved the lab!"**

<img width="1474" height="812" alt="image" src="https://github.com/user-attachments/assets/b59b9da0-fd52-4de4-81b0-43d3f3b216e0" />


---

## 9. Final Payload

The payload that solved the lab was:

```
' UNION SELECT USERNAME_OKZVSD, PASSWORD_YBKORB FROM USERS_WJIWCZ--
```

---

## 10. Notes

This lab reinforced the same enumeration methodology from the non-Oracle version, but forced me to learn Oracle's specific system view naming conventions instead of relying on the standard `INFORMATION_SCHEMA` that PostgreSQL and MySQL both support. Oracle uses its own view, `ALL_TAB_COLUMNS`, for the same purpose — I confirmed this by referencing Oracle's official documentation for `ALL_TABLES` and its related views before writing any payload, rather than guessing at Oracle syntax.

I also ran into two separate dead ends that looked promising on the surface: `ALL_USERS`, which sounds like exactly the table I wanted but doesn't expose password data at all, and `KU$_ROLE_VIEW`, which does have a literal `PASSWORD` column but is part of Oracle's Data Pump internals rather than a source of real, usable login credentials. Both taught me the same lesson from the earlier lab — a column name matching what I'm looking for doesn't guarantee the query will actually return something exploitable, and each failed attempt still narrowed down where the real data wasn't, which mattered as much as eventually finding where it was.

Oracle's system view landscape is considerably larger and more specialized than PostgreSQL's, which made this enumeration noisier and slower — table names like `KU$_ROLE_VIEW`, `TRANSPORTABLE_EXPORT_OBJECTS`, and `ALL_USTATS` are all legitimate, database-specific metadata views that have nothing to do with the application's actual user accounts. Recognizing the randomized suffix pattern (`_WJIWCZ`, `_OKZVSD`, `_YBKORB`) as the marker of an application-defined table, rather than an Oracle-native one, was the key signal that separated the real target from the surrounding noise.
