# Lab- Blind SQL injection with conditional responses


## Objective

The application uses a `TrackingId` cookie in a SQL query, but never displays the query's results directly and shows no error messages. The only signal available is a "Welcome back" message that appears in the page when the query returns a row. The goal is to exploit this Boolean-based blind SQL injection to recover the administrator's password and log in as them.

## Step 1: Confirming the Boolean Behavior

I sent a request to `/filter?category=Gifts` with the `TrackingId` cookie modified to:

```
TrackingId=aRs4Tyr2yWomp05P' AND '1'='1
```

This is a true condition, and the response contained "Welcome back!" in the page. Changing it to a false condition (`'1'='2'`) made the "Welcome back!" text disappear from the response. This confirmed the injection point and the Boolean indicator.

<img width="1916" height="922" alt="image" src="https://github.com/user-attachments/assets/b7d4fa01-2f84-4f90-bed6-dbbf13ff59a5" />
<img width="1915" height="961" alt="image" src="https://github.com/user-attachments/assets/f74a742d-68a9-4cb4-bb1e-d0da90b3b62b" />

## Step 2: Determining the Password Length

Using the same true/false approach, I tested:

```
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>N)='a
```

increasing `N` until the condition flipped from true to false. The password length turned out to be exactly 20 characters — `LENGTH(password)>19` was true, `LENGTH(password)>20` was false.

<img width="1916" height="962" alt="image" src="https://github.com/user-attachments/assets/237ddcd2-fc5c-4416-ba09-7bf3355b0096" />

## Step 3: Setting Up Character Extraction With SUBSTRING()

To pull out each character individually, I used:

```
SUBSTRING((SELECT password FROM users WHERE username='administrator'),POSITION,1)
```

`POSITION` is the character's index in the password (1, 2, 3, ...) and the final `1` means "extract exactly one character". I tested this manually first with a Sniper attack in Intruder to make sure the syntax worked before scaling up.

<img width="1916" height="952" alt="image" src="https://github.com/user-attachments/assets/f39595bc-8d97-45ef-8ef3-f1baebc13b5e" />

## Step 4: Automating With Burp Intruder — Full Setup

Doing this manually for 20 positions × 36 possible characters (a–z, 0–9) would mean up to 720 requests, so I automated it in Intruder. Here's exactly how it was configured:

**Attack type:** Cluster bomb — this is needed because there are two independent variables (the character position, and the character being tested), and Cluster bomb tries every combination of the two payload sets against each other.

**Request template (Positions tab):** the modified `TrackingId` cookie with two payload markers:

```
TrackingId=xyz' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),§1§,1)='§a§
```

The `§1§` marks where the position number goes, and `§a§` marks where the tested character goes.

**Payload set 1 (positions):** a simple numeric list from `1` to `20`, matching the known password length.

**Payload set 2 (characters):** a custom list containing all lowercase letters `a`–`z` and digits `0`–`9` (36 values total, based on the lab's hint that the password only contains lowercase alphanumeric characters).

**Grep - Match (Intruder → Settings → Grep - Match):** added the string `Welcome back` as a match rule. This doesn't generate anything itself — it just scans every response Intruder receives and flags whether that exact text is present, giving a quick visual column to check instead of opening every response manually.

**Running the attack:** with Cluster bomb, Intruder tried position 1 against every character, then position 2 against every character, and so on — 20 × 36 = 720 requests in total. For each position, exactly one character combination came back with "Welcome back!" present (flagged by Grep-Match), and that was the correct character for that position.

<img width="1915" height="946" alt="image" src="https://github.com/user-attachments/assets/c453c39a-2a60-4151-9ba8-186462f17c6a" />


<img width="1917" height="975" alt="image" src="https://github.com/user-attachments/assets/59e8d410-7804-46dc-911c-25a8ec527dec" />


## Step 5: Reconstructing the Password

After the attack finished, I went through the results and picked out, for each position (1–20), which character produced a "Welcome back!" match:

|Position|Character|Position|Character|
|:-:|:-:|:-:|:-:|
|1|k|11|g|
|2|m|12|v|
|3|h|13|p|
|4|t|14|z|
|5|a|15|d|
|6|j|16|k|
|7|1|17|a|
|8|a|18|y|
|9|z|19|z|
|10|f|20|a|

Reading positions 1 through 20 in order gives the full password:

```
kmhtaj1azfgvpzdkayza
```

## Step 6: Logging in as Administrator

I logged in using the `administrator` username and the extracted password. The lab confirmed success, showing "Your username is: administrator" and marking the lab as solved.

<img width="1917" height="852" alt="image" src="https://github.com/user-attachments/assets/24784b13-c87a-4743-893b-ee1dbcbf65b9" />


## Root Cause

The application builds a SQL query directly from the unsensitized `TrackingId` cookie value. Even though the query's actual output is never shown to the user, the application's behavior (showing or hiding "Welcome back") still leaks one bit of information per request which is enough to reconstruct arbitrary data given enough requests.

## Impact

An attacker can extract sensitive data from the database, in this case, a full user password, purely by observing true/false differences in the application's behavior, without ever seeing direct query output. Depending on the schema and privileges, this technique can be used to dump entire tables, not just a single password.

## Remediation

- Use parameterized queries / prepared statements for all database access; never build SQL from raw cookie or parameter values.
- Apply strict input validation and allow listing on values used in queries, even for "internal" fields like tracking cookies.
- Avoid exposing any behavioral difference (message text, response time, status code, etc.) that depends on unsensitized user input reaching the database.

## Notes

- The key mental shift from earlier UNION-based SQLi labs was that there's no data to read directly  every question has to be phrased as a true/false condition, and the app's one behavioral tell ("Welcome back") is the only channel available.
- Ran into a cookie-formatting mistake early on where multiple `TrackingId=` values got concatenated by accident instead of replacing the original  fixed by keeping exactly one `TrackingId=...` entry and leaving the `session` cookie untouched.
- `SUBSTRING(value, start, length)`  remember only the middle argument (`start`/position) changes across requests; the last argument stays `1` since only one character is being tested at a time.
- Cluster bomb is the right attack type specifically because two independent payload sets need every combination tested against each other  Sniper only varies one position at a time and wouldn't cover both position and character together.
- Grep-Match doesn't need any special server cooperation  it's purely Burp scanning responses client-side for text you specify, so it works for any Boolean indicator, not just this lab's exact wording.
