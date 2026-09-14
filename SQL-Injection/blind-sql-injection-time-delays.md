#  Lab- Blind SQL injection with time delays

## Objective

The application uses a `TrackingId` cookie in a backend SQL query, but the results of that query are never returned to the user and the application behaves identically whether the query succeeds, returns rows, or errors out. This makes the injection **fully blind** — there is no error message and no visible content difference to lean on. The only usable oracle is _time_: if I can make the database pause before responding, I can infer that my injected SQL actually executed. The goal is to make the query delay the response by 10 seconds.

## Step 1: Confirming the Injection Point

The `TrackingId` cookie value is reflected into a SQL query server-side. Since there's no visible output at all (not even an error), the plan was to test time-delay syntax for several common database engines one at a time and watch the response time in Burp, rather than guessing the backend up front.

## Step 2: Testing MySQL Syntax — `OR SLEEP(10)`

First attempt, assuming a MySQL-style backend:

```
Cookie: TrackingId=uN9TODvkyfn2W6Ty' OR SLEEP(10)--
```

<img width="1917" height="927" alt="image" src="https://github.com/user-attachments/assets/b8260d62-7b1b-4995-8dca-90af0a56bc86" />

The response came back in **82 milliseconds** — essentially instant, no delay at all. Either the syntax was silently ignored, or `SLEEP()` isn't a valid function in this DBMS. This was a signal to rule MySQL out rather than a dead end.

## Step 3: Testing Microsoft SQL Server Syntax — `WAITFOR DELAY`

Next, I tried MSSQL's time-delay syntax, which doesn't require an `OR`/quote-breakout in the same way since `WAITFOR` is a statement, not part of a boolean expression:

```
Cookie: TrackingId=uN9TODvkyfn2W6Ty'WAITFOR DELAY '0:0:10'--
```

<img width="1916" height="940" alt="image" src="https://github.com/user-attachments/assets/194accbe-12a0-413f-85af-33702dfad007" />

Again, the response returned in **79 milliseconds** — no delay. This ruled out MSSQL as well.

## Step 4: Testing Oracle Syntax — `DBMS_PIPE.RECEIVE_MESSAGE`

Oracle doesn't have a native `SLEEP()`, so its usual time-delay trick is calling the `DBMS_PIPE.RECEIVE_MESSAGE` package function, which blocks for the given number of seconds:

```
Cookie: TrackingId=uN9TODvkyfn2W6Ty' || DBMS_PIPE.RECEIVE_MESSAGE('x',10)--
```

<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/d08994cc-b2c8-46fb-a140-e3ac17b8e97d" />

Still **84 milliseconds** — no delay. Three engines ruled out, one to go.

## Step 5: Testing PostgreSQL Syntax — `pg_sleep(10)`

The last common candidate was PostgreSQL, whose delay function is `pg_sleep(seconds)`. Since the field is concatenated into an existing string value (not sitting in a bare boolean expression like the `OR` attempts), I broke out of the string with `||` (Postgres's string-concatenation operator) directly followed by the function call, then commented out the rest of the query:

```
Cookie: TrackingId=uN9TODvkyfn2W6Ty'||pg_sleep(10)--
```

<img width="1917" height="932" alt="image" src="https://github.com/user-attachments/assets/de720dd0-7c35-4627-a46e-417b60dcb500" />

This time the response took **10,088 milliseconds** — a clean ~10 second delay, matching the payload exactly. This confirmed two things at once: the injection point executes arbitrary SQL, and the backend database is **PostgreSQL**.

## Step 6: Confirming the Solve

Reloading the lab confirmed the status changed to solved.

<img width="1917" height="847" alt="image" src="https://github.com/user-attachments/assets/2b489bf5-5ebd-4bf8-905f-cf722f21363f" />

## Root Cause

The `TrackingId` cookie value is concatenated directly into a SQL query string without sanitization or parameterization. Because the query executes synchronously before the server sends its response, any time-consuming operation injected into it (like `pg_sleep()`) directly delays the HTTP response — turning response latency into a full data-exfiltration side channel, even with zero visible output difference.

## Impact

Full blind SQL injection. Even without any error messages or content-based oracle, an attacker can use time-based inference (`pg_sleep()` combined with conditional logic, e.g. `CASE WHEN (condition) THEN pg_sleep(10) ELSE pg_sleep(0) END`) to extract arbitrary data from the database one bit/character at a time — including credentials, session tokens, or any other table contents.

## Remediation

- Use parameterized queries / prepared statements for all user-controlled input reaching SQL, never string concatenation.
- Apply least-privilege database accounts so even a successful injection has minimal reach.
- Enable database query logging and alerting on statistically anomalous query execution times, which can catch time-based blind SQLi attempts in production.
- Consider a Web Application Firewall as defense-in-depth, but never as a substitute for fixing the underlying query construction.

## Notes

- Blind, content-blind injection points (no error, no reflected output) still leak information through **timing** — response latency is itself an oracle. This is the core lesson of this lab versus the earlier error-based ones.
- Since the DBMS engine wasn't stated upfront, testing each major engine's own delay syntax one at a time (`SLEEP()` → MySQL, `WAITFOR DELAY` → MSSQL, `DBMS_PIPE.RECEIVE_MESSAGE` → Oracle, `pg_sleep()` → PostgreSQL) is a fast, reliable way to both confirm the injection _and_ fingerprint the backend simultaneously — a non-delay response rules a candidate engine out cleanly.
- The syntax used to reach the function mattered as much as the function itself: `OR SLEEP(10)` and `WAITFOR DELAY` were tried as Boolean/statement injections, but the working payload needed Postgres's own string-concatenation operator (`||`) to break out of the existing string context before the function call would execute — a reminder to match the _injection context_ (string vs. numeric vs. statement) to the target syntax, not just the target DBMS.
