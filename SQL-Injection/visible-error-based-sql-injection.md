# Lab- Visible error-based SQL injection

## Objective

The application uses a `TrackingId` cookie for analytics, and this value is inserted directly into a SQL query run against the backend database. The results of the query aren't returned to the response, but the application is running with verbose error messages enabled, so injection can be confirmed and exploited by reading the database errors it throws back. The database also has a separate `users` table with `username` and `password` columns. The goal is to leak the `administrator` password via these error messages and log in as them.

## Step 1: Identifying the Injection Point

Before touching the cookie value, I sent a plain baseline request through Repeater to get a feel for normal behavior, then started modifying `TrackingId` to see how the app reacted to broken input. Even small changes to the cookie were enough to knock the query out of shape and produce a server error, which was the first sign the value was being concatenated straight into a SQL statement rather than being treated as an opaque token.

<img width="1568" height="758" alt="image" src="https://github.com/user-attachments/assets/59ff7c67-84c5-4d13-84ea-164ab3c7ea34" />

## Step 2: Determining the Query Structure

Next I used `ORDER BY` to probe how many columns the underlying `SELECT` was working with, in case a `UNION`-based approach turned out to be the way in:

```
TrackingId=...'ORDER BY 1--
```

This came back `200 OK` with the lab still unsolved, meaning the column position was valid and the query tolerated the ordering clause without erroring.

<img width="1568" height="761" alt="image" src="https://github.com/user-attachments/assets/21e80dd0-2830-4c24-8dba-efaa50923e5a" />

## Step 3: Attempting UNION-Based Extraction with CAST

I tried forcing a type-mismatch error to leak data by wrapping the password subquery in `CAST()`, since PostgreSQL will happily tell you what it tried to convert when the cast fails:

```
TrackingId=...' UNION SELECT CAST((SELECT password FROM users WHERE username = 'administrator') AS int)--
```

This payload was long enough that it ran into the cookie's character limit and got cut off mid-string before reaching the server, producing an "unterminated string literal" error instead of the type-mismatch error I was aiming for.

<img width="1568" height="742" alt="image" src="https://github.com/user-attachments/assets/dfe63ff1-29d0-4e19-a39c-6180da31797d" />

## Step 4: Direct UNION SELECT of the Password

As a simpler check I also tried pulling the password straight out with no casting:

```
TrackingId=...' UNION SELECT password FROM users--
```

This one reached the server fine, but since the lab never reflects query results back into the page, there was nothing to actually read — confirming this approach wasn't going to work here even though the injection itself was accepted.

## Step 5: Switching to AND + CAST

I moved to a shorter `AND`-based CAST payload instead of `UNION`, hoping the reduced length would fit inside the cookie's limit:

```
TrackingId=...'AND CAST((SELECT password FROM users LIMIT 1) AS int)--
```

It was still too long — Burp showed the payload arriving truncated at the server, again producing an "unterminated string literal" error rather than a useful type-conversion error.

<img width="1568" height="756" alt="image" src="https://github.com/user-attachments/assets/9a8e52d4-8e92-4580-bab6-d995a498bb21" />


## Step 6: Targeting the Administrator Directly

I tried filtering the subquery to the `administrator` user specifically, on the theory that a single guaranteed row would sidestep any "multiple rows" issues later on:

```
TrackingId=...'CAST((SELECT password FROM users WHERE username = 'administrator') AS int)
```

Same problem as before — the payload was too long for the cookie field and got cut off before the closing parts of the query, so the server only ever saw a broken string rather than my actual CAST expression.

<img width="1568" height="766" alt="image" src="https://github.com/user-attachments/assets/f33d95d0-114d-4c62-b0b1-9b7ab9fb6465" />


## Step 7: Shortening the Payload

Realising the character limit was the real blocker, I dropped the `username = 'administrator'` filter entirely and went back to the shortest possible form of the CAST trick:

```
TrackingId=...'||CAST((SELECT password FROM users)AS int)--
```

This time the full payload reached the database, and the error came back as:

```
ERROR: more than one row returned by a subquery used as an expression
```

This was actually a great result even though it wasn't the leak yet — it confirmed the injection works, `CAST` executes, and the subquery runs; the only remaining problem was that `users` has more than one row and the subquery needs to return exactly one.

## Step 8: Adding LIMIT 1 — Leaking the Password

The fix was obvious from the previous error, so I added `LIMIT 1` to force the subquery down to a single row:

```
TrackingId=...'||CAST((SELECT password FROM users LIMIT 1)AS int)--
```

Since the password column is text and can't be cast to `int`, PostgreSQL threw a type-conversion error — and, crucially, that error message includes the exact string it tried to convert:

```
ERROR: invalid input syntax for type integer: "sOq2quecxuqunz1pp033"
```

That's the administrator's password, leaked straight out of the error message.

<img width="1568" height="740" alt="image" src="https://github.com/user-attachments/assets/cd802557-f60a-4589-9a05-ab3ac8f3a06e" />


## Step 9: Logging In as Administrator

I logged into the application as `administrator` using the leaked password, which solved the lab.

<img width="1568" height="783" alt="image" src="https://github.com/user-attachments/assets/62cd89bc-55d2-4704-838e-101198181c47" />

## Root Cause

The application concatenates the `TrackingId` cookie value directly into a SQL query without sanitization or parameterization. Because the app runs with detailed database errors surfaced back to the client, an attacker doesn't need the query results reflected anywhere — the error messages themselves become an oracle. Wrapping a subquery in `CAST()` to an incompatible type turns any returned value into free-text leaked through the resulting error.

## Impact

An attacker can extract arbitrary data from the database — including other users' credentials — without needing the application to display query results anywhere, purely by reading the verbose error output the database returns on a failed type conversion.

## Remediation

- Use parameterized queries (prepared statements) for all database access instead of string concatenation.
- Disable verbose database error messages in production; return a generic error to the client and log details server-side only.
- Apply least-privilege database accounts so even a successful injection has minimal reach.
- Validate/allowlist cookie and other client-supplied values before they ever reach a query.

## Notes

- The cookie field has a real character limit, and hitting it silently truncates the payload mid-string rather than rejecting it outright — the server then reports an "unterminated string literal" error that looks like a syntax problem but is actually just "your payload got cut off." Worth checking payload length first next time instead of debugging the SQL itself when this error shows up.
- The `CAST()` to an incompatible type trick is a solid general technique for error-based leaks: the database's own error message becomes the exfiltration channel, so you don't need the app to reflect anything back to you.
- The "more than one row returned by a subquery" error was actually useful signal, not just noise — it confirmed every part of the payload was working and pointed directly at `LIMIT 1` as the fix, similar to how the AttributeError/NameError messages guided the SSTI lab.
