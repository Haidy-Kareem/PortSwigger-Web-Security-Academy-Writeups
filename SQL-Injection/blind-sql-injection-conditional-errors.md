# Lab- Blind SQL injection with conditional errors

## Objective

The application uses a `TrackingId` cookie in a backend SQL query without proper sanitization. Unlike the previous blind SQLi lab, the application doesn't show any visible difference in its response (no "Welcome back" message, no row-count difference) — the only signal available is whether the database throws an error or not. The database here is Oracle. The goal is to extract the administrator's password and log in as them.

## Vulnerability Overview

Since the app gives no visual Boolean indicator, I needed a different signal: HTTP status code. Oracle lets you force a deliberate database error using `1/0` (division by zero) inside a `CASE WHEN` expression. If I make the error only trigger when my injected condition is true, the app returns `HTTP 500` for true and `HTTP 200` for false — a clean Boolean oracle based purely on status code.

## Step 1: Confirming Error-Based Injection Works

I sent the request to Repeater and tested a false condition first, `CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END`, injected into the `TrackingId` cookie using Oracle string concatenation (`||`):

```
TrackingId=q0q2wyqBZWHeHUa'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
```

This returned `HTTP 200 OK`, as expected since the condition is false.

<img width="1917" height="911" alt="image" src="https://github.com/user-attachments/assets/fc105af6-5887-4252-b257-28c724598708" />


I then flipped it to a true condition, `(1=1)`, keeping everything else the same. This time the response was `HTTP 500 Internal Server Error`, confirming the error-based Boolean channel works: true conditions trigger the divide-by-zero error, false conditions don't.

<img width="1917" height="917" alt="image" src="https://github.com/user-attachments/assets/92f01db5-128b-48a4-ade3-9f8ef072252b" />


## Step 2: Determining the Password Length

Before extracting characters, I needed the password length. I used:

```sql
CASE WHEN LENGTH(password)=20 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator'
```

Rather than testing lengths one by one manually, I sent this to Burp Intruder as a **Sniper attack**, with the number after `LENGTH(password)=` as the payload position, and a simple payload list of numbers (I used a reasonable range). I looked through the results for the request that returned `HTTP 500` instead of `200` — that was the length that made the condition true. Length **20** returned the 500.

<img width="1917" height="936" alt="image" src="https://github.com/user-attachments/assets/3cff8f8d-bf22-44ef-b980-dc429e05f528" />


## Step 3: Testing a Single Character Manually First

Before automating, I confirmed the character-extraction payload structure worked with a single manual test in Repeater:

```sql
CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator'
```

This returned `200 OK` (character 1 isn't `a`), confirming the query executes correctly and just needs the right character/position to trigger 500.


## Step 4: Automating Character Extraction with Intruder — Full Setup

This is the part worth detailing carefully so it's reproducible:

1. **Send the base request to Intruder.** Right-click the request in Repeater (or Proxy history) with the SQLi payload in the `TrackingId` cookie, and choose "Send to Intruder."
    
2. **Set the attack type to Cluster Bomb.** In the Intruder tab, change the attack type dropdown from "Sniper" to **Cluster bomb** — this lets you combine two independent payload lists (position × character) instead of testing one variable at a time.
    
3. **Mark two payload positions in the request.** In the Positions tab, the cookie value needs two separate injection markers:
    

```
TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,§1§,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

- The first `§...§` marks the **character position** (currently showing `1`).
- The second `§...§` marks the **character being tested** (currently showing `a`). Use "Clear §" first to remove any auto-added markers, then manually select each value and click "Add §" to mark exactly those two.

4. **Configure Payload Set 1 (positions).** In the Payloads tab, select payload position "1," payload type "Simple list," and add the numbers `1` through `20` (matching the password length found in Step 2).
    
5. **Configure Payload Set 2 (characters).** Switch to payload position "2," payload type "Simple list," and add all lowercase letters `a`–`z` plus digits `0`–`9` (36 values total). You can type them individually or paste a prepared list into the "Paste" box.
    
6. **Start the attack.** With 20 positions × 36 characters, this sends 720 requests total.
    
7. **Read the results by status code, not by Grep-Match this time.** Unlike the earlier "conditional responses" lab (which used Grep-Match on "Welcome back"), this lab's signal is the HTTP status code itself. Sort the Results table by the **Status code** column — any row showing `500` instead of `200` marks the correct character for that position. Click each 500 row to see which position and character combination produced it.
    

<img width="1917" height="922" alt="image" src="https://github.com/user-attachments/assets/42922c82-643e-4df6-8413-d69f60af4467" />

## Step 5: Reconstructing the Password

Going through the 20 positions and picking out the character that returned `500` at each one produced the full password:

```
bjb2yukybltj2jnvjmnq
```

## Step 6: Logging In and Confirming the Result

I logged into the application as `administrator` using the extracted password, and the lab was marked as solved.

<img width="1916" height="831" alt="image" src="https://github.com/user-attachments/assets/3aff764a-b192-4912-b1b0-740c8ebdb571" />

## Root Cause

The application concatenates the `TrackingId` cookie value directly into a SQL query without parameterization. Even though the app never displays query results or gives a visible content difference, allowing a raw database error to reach the HTTP response (as a 500 status) still leaks a full Boolean channel — an attacker doesn't need visible data, just _any_ observable difference in behavior.

## Impact

An attacker can fully compromise the `administrator` account by extracting the password through blind, error-based SQL injection, without ever seeing actual query output — only the presence or absence of a server error.

## Remediation

- Use parameterized queries/prepared statements everywhere user input reaches a SQL query; never concatenate raw input into SQL strings.
- Suppress detailed error pages and avoid letting database errors surface as distinguishable HTTP status codes to the client — return a generic error response regardless of the underlying cause.
- Apply least-privilege database accounts so that even a successful injection has minimal reach.

## Notes

- This lab is the Oracle-flavored sibling of the earlier "conditional responses" lab — same overall attack shape (position × character extraction via Intruder Cluster Bomb), but the true/false signal changed from a text string ("Welcome back") to an HTTP status code (500 vs 200), so Grep-Match wasn't used here — sorting/filtering by the Status code column in the Intruder results did the same job.
- Oracle-specific syntax mattered: `||` for string concatenation (instead of `+` or `CONCAT()`), and `TO_CHAR(1/0)` as the deliberate error trigger inside the `CASE WHEN` expression.
- `SELECT ... FROM dual` was used for the initial true/false sanity checks since Oracle requires a `FROM` clause even for expressions with no real table involved.
- Determining password length first (via a similar Intruder sweep over candidate lengths) saved time before jumping into the full 20×36 character grid.
