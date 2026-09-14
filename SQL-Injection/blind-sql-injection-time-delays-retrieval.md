# Lab- Blind SQL injection with time delays and information retrieval

## Objective

This lab builds directly on a simpler time-based blind SQLi scenario: the same `TrackingId` cookie is concatenated into a backend SQL query, the response gives no visible difference between true/false conditions or errors, and the only oracle is response timing. This time the goal isn't just to prove the injection exists — it's to actually **extract data**: find the `administrator` user's password from the `users` table and use it to log in.

The core technique is turning the boolean condition inside a `pg_sleep()` call into a data-extraction primitive: instead of injecting a fixed `TRUE`/`FALSE`, inject a condition that depends on the actual data (`LENGTH(password) = N`, `SUBSTRING(password, N, 1) = 'x'`), then use response time as a 1-bit-at-a-time (or 1-character-at-a-time) readout.

## Step 1: Confirming Conditional Time-Based Injection

Before extracting anything, I confirmed the injection point supports a _conditional_ delay — not just an unconditional `pg_sleep(10)`, but a `CASE WHEN` that only sleeps when a condition is true:

```
Cookie: TrackingId=dR8XcKjfPb9iVhoP' || (SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END)--
```

<img width="1600" height="773" alt="image" src="https://github.com/user-attachments/assets/c90f2c35-4a14-4f37-b850-4ea84ca1ccac" />

Since `1=1` is always true, the response should always delay — and it did (10,038 ms). This confirmed the `CASE WHEN ... THEN pg_sleep(10) ELSE pg_sleep(0) END` pattern executes correctly inside the injection point, which is the building block for every extraction step that follows.

## Step 2: Determining the Password Length

With the conditional primitive confirmed, the next step was to find out _how many characters_ to extract. I built a payload that sleeps only when the password's length equals a specific number, and used Burp Intruder's **Sniper** attack to sweep through a range of candidate lengths:

```
Cookie: TrackingId=dR8XcKjfPb9iVhoP' || (SELECT CASE WHEN (LENGTH((SELECT password FROM users WHERE username='administrator'))=§10§) THEN pg_sleep(10) ELSE pg_sleep(0) END)--
```

<img width="1600" height="773" alt="image" src="https://github.com/user-attachments/assets/65c84ded-25ff-40f7-a70a-04e0280630df" />

Running the attack against a simple-list payload of numbers, every request came back fast **except** one:

<img width="1600" height="802" alt="image" src="https://github.com/user-attachments/assets/7b24dc95-ac06-4692-a9b6-43963e87a2b4" />

Only the request testing `LENGTH(password)=20` delayed by ~10 seconds, so the administrator's password is **20 characters long**. Knowing the length in advance meant the next stage could target exactly 20 positions instead of guessing when to stop.

## Step 3: Extracting the Password Character by Character

With the length fixed, I extracted the password one character at a time using `SUBSTRING(password, N, 1)`, again driven by Intruder — this time with a payload list covering the full lowercase alphanumeric charset (`0–9`, `a–z`, 36 values), and the position number changed manually for each character:

```
Cookie: TrackingId=dR8XcKjfPb9iVhoP' || (SELECT CASE WHEN (SUBSTRING(password,1,1)='§0§') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users WHERE username='administrator')--
```

**Position 1:**

<img width="1600" height="780" alt="image" src="https://github.com/user-attachments/assets/034866a4-d6b9-45f0-8e1f-e5c0d16f5b56" />

Only the payload `0` triggered the delay, so character 1 of the password is `0`.

**Position 2:**

Changing the `SUBSTRING` offset to `2` and re-running the same 36-value attack:

<img width="1600" height="777" alt="image" src="https://github.com/user-attachments/assets/29a27dc7-2393-4a81-a993-655861ff35b4" />

Only `4` delayed, so character 2 is `4`. Repeating this exact process — incrementing the `SUBSTRING` offset from 3 up to 20 and re-running the Sniper attack each time — reconstructs the full 20-character password one position at a time, since exactly one candidate out of 36 will delay at each position.

## Step 4: Logging In and Confirming the Solve

Once all 20 characters were assembled into the full password, I logged in through the application's login form as `administrator` using the recovered password. The lab flagged as solved and the account page confirmed the login:

<img width="1600" height="714" alt="image" src="https://github.com/user-attachments/assets/bfbf216d-b0ad-4887-b56e-4006b8c57ecf" />


## Root Cause

Same underlying flaw as the simpler lab: the `TrackingId` cookie is concatenated directly into a SQL query without parameterization. What makes this lab worse in impact is that the _same_ time-based channel that merely proves injection also functions as a full read primitive — any Boolean condition can be wrapped in `CASE WHEN ... THEN pg_sleep(N) ELSE pg_sleep(0) END` and used to exfiltrate arbitrary column data one bit or character at a time, entirely without any visible output.

## Impact

Full blind SQL injection with practical data exfiltration and account takeover. An attacker doesn't need error messages or reflected content — response latency alone is enough to determine data length and then read out every character of any column (passwords, tokens, PII), and in this case to fully compromise the `administrator` account.

## Remediation

- Use parameterized queries / prepared statements for all user-controlled input reaching SQL — never string concatenation.
- Apply least-privilege database accounts so a successful injection has minimal reach even when it succeeds.
- Store passwords using a strong salted hash (bcrypt/argon2) rather than any reversible or directly comparable form, so even full extraction of the stored value doesn't yield a usable credential.
- Log and alert on statistically anomalous query execution times — a burst of ~10-second responses from one client is a strong time-based blind SQLi signal.
- Treat a WAF as defense-in-depth only, never as a substitute for fixing the underlying query construction.

## Notes

- The technique here is the natural extension of the boolean-oracle idea: once you can make the database sleep conditionally on `1=1` vs `1=2`, you can make it sleep conditionally on _any_ predicate — including one built from real column data (`LENGTH(...)`, `SUBSTRING(...)=...`). That turns a timing side-channel into a general-purpose blind read primitive.
- Determining the length first (`LENGTH(password)=N`) is what makes the character-by-character phase efficient — it defines exactly how many positions need to be swept, instead of guessing where the string ends.
- Automating the position sweep (changing the `SUBSTRING` offset and re-running the same 36-value Intruder attack) is what makes 20-character extraction practical by hand; in a real engagement this loop would normally be scripted (e.g., with Burp's Intruder "Pitchfork"/scripted extensions, or a small Python + `requests`/`sqlmap`-style harness) rather than repeated manually position by position.
- Restricting the payload set to the expected charset (here, lowercase alphanumeric) before running Intruder keeps the number of requests per character down to 36 instead of testing the full printable ASCII range.
