# Lab- SQL injection with filter bypass via XML encoding

## Objective

The application's stock-check feature (`POST /product/stock`) takes an XML body with a `productId` and a `storeId`, runs a SQL query using both values, and — unlike a blind injection point — actually reflects the query results back in the response (e.g. "71 units"). That makes it a textbook UNION-attack target on paper. In practice, a Web Application Firewall (WAF) sits in front of the query and returns "Attack detected" for anything that looks like SQL syntax. The real challenge here isn't finding the injection — it's getting a payload past the WAF without the backend XML/SQL parser losing the plot. The goal was to extract the `administrator` user's credentials from the `users` table and log in as them.

---

## Step 1: Confirming the Injection Point Exists — and the WAF Along With It

Baseline request, both fields numeric and unmodified, returns a normal `200 OK` with a stock count.

Adding a single quote to `productId` (`2'`) to test for injection immediately triggered a `403 Forbidden` with the body `"Attack detected"` — so a WAF is inspecting the request body, not just validating input server-side.

<img width="1600" height="773" alt="image" src="https://github.com/user-attachments/assets/6e6875f2-60a4-4907-b30c-31cb531d82ae" />

Trying a classic stacked payload directly (`2'; SELECT * FROM users --`) in `productId` was blocked the same way. The WAF is clearly pattern-matching on SQL keywords/characters in the raw request, which ruled out attacking `productId` directly and pointed toward finding either a different field or a different encoding of the same payload.

<img width="1600" height="772" alt="image" src="https://github.com/user-attachments/assets/20e7b2ac-4106-4b58-bcd6-252aa5e083c2" />

---

## Step 2: Finding an Unfiltered, Query-Evaluated Field — `storeId`

Switching focus to `storeId` and sending arithmetic instead of a quote-breakout (`1+1`) was a low-risk way to check two things at once: whether the WAF flags this field at all, and whether its value is actually evaluated by the database rather than just matched as a literal string.

The response came back `200 OK` with `"71 units"` — a different count than the baseline, consistent with `1+1` being evaluated as `2` server-side. That confirmed `storeId` is concatenated into the query and isn't guarded by the same filter that blocked `productId`.

<img width="1600" height="773" alt="image" src="https://github.com/user-attachments/assets/147e942e-dc19-4f51-a8f2-0a980aae5c02" />

---

## Step 3: Confirming the UNION Attack Still Gets Blocked in Plaintext

Sending a raw, unencoded `UNION SELECT NULL,NULL` in `storeId` still tripped the WAF. So the filter isn't scoped to a single field — it's inspecting the whole body for SQL syntax regardless of which XML element it sits in.


---

## Step 4: Bypassing the WAF with XML Hex Entity Encoding (Hackvertor)

Since the request body is XML, and XML parsers decode character entities before the resulting string is handed off to the SQL layer, encoding the payload as XML hex entities (`&#x..;`) hides the SQL keywords from the WAF's pattern matching while still arriving at the database as plain SQL after the parser decodes it.

Using the Burp Hackvertor extension's `hex_entities` encoder on `1 UNION SELECT NULL,NULL`:

<img width="1600" height="802" alt="image" src="https://github.com/user-attachments/assets/eb24bb35-ee0e-45b4-90d8-f4dfe321dc5c" />

Dropping that hex-entity string into `storeId` and sending the request returned `200 OK` with:794 units  null


Two things confirmed at once: the column count is 2 (no error), and the WAF no longer sees any raw SQL keywords to flag — the payload sailed through as harmless-looking hex entities and only became `UNION SELECT NULL,NULL` after the XML parser decoded it server-side.

<img width="1600" height="777" alt="image" src="https://github.com/user-attachments/assets/cf8f747d-e853-481b-8edc-c16390f42851" />

---

## Step 5: Confirming the `users` Table Is Reachable

Extending the same technique, encoding `1 UNION SELECT NULL,NULL FROM users` and sending it returned `200 OK` with `"0 units"` — no error, confirming the `users` table exists and is queryable through this injection point with the expected column count.

<img width="1600" height="777" alt="image" src="https://github.com/user-attachments/assets/ba8f37ca-6c1b-41a3-96e6-9f58e1584e84" />

---

## Step 6: Extracting Credentials

Final payload, built the same way — encode with Hackvertor, drop into `storeId`:Input: 1 UNION SELECT username || '~' || password FROM users



<img width="1600" height="780" alt="image" src="https://github.com/user-attachments/assets/2d02d904-2f63-4825-9165-3a809f89202e" />

The response returned all three accounts, `username~password` per row, concatenated with `||` so each pair fits in the single available column:administrator~[password]  
wiener~[password]  
carlos~[password]  
794 units


<img width="1600" height="782" alt="image" src="https://github.com/user-attachments/assets/701efc9a-477e-43b3-84e6-d11646761130" />

---

## Step 7: Confirming the Solve

Logging in as `administrator` with the extracted password confirmed the lab status changed to **Solved**.

<img width="1600" height="710" alt="image" src="https://github.com/user-attachments/assets/dc563a05-203b-467b-a497-efd2aa012129" />

---

## Root Cause

`storeId` (and `productId`) are concatenated directly into a SQL query without parameterization, and the WAF in front of the application relies on inspecting the raw request body for SQL syntax rather than the value the application will actually evaluate. Because the body is XML, the XML parser decodes character entities before the value reaches the SQL layer — meaning the WAF and the database are effectively looking at two different strings. Anything the WAF's signature-based filter doesn't recognize in its encoded form sails straight through and gets decoded into live SQL on the other side.

---

## Impact

Full UNION-based SQL injection with WAF bypass. An attacker can read arbitrary data from any table the database user can access — here, all registered users' credentials — and use them to fully compromise accounts, including administrative ones. The presence of a WAF gave no real protection since it inspected the wrong representation of the input.

---

## Remediation

- Use parameterized queries / prepared statements for all user-controlled input reaching SQL — this closes the vulnerability regardless of encoding, so no WAF signature can be bypassed around it.
- Never rely on a WAF as the primary defense against injection; treat it strictly as defense-in-depth on top of secure query construction.
- If a WAF is used, it must normalize/decode input the same way the downstream parser will (XML entities, URL encoding, Unicode normalization, etc.) before applying its signatures — inspecting the pre-decoded body is a bypassable illusion of protection.
- Apply least-privilege database accounts so a successful injection has minimal reach (e.g. no access to a `users` table from a stock-check query's context).

---

## Notes

- A WAF blocking the obvious form of a payload isn't proof the injection is unreachable — it's a signal to change the payload's representation, not necessarily its target field. Testing a second, non-flagged field (`storeId`) first confirmed the injection was real before spending effort on evasion.
- Arithmetic (`1+1` → `2`) is a cheap, filter-safe way to confirm a parameter is being evaluated by the database rather than just accepted as an opaque string, without tripping any SQL-keyword signature.
- Since the request body is XML, hex character entities (`&#x..;`) are decoded by the XML parser before the SQL layer ever sees the string — this is the crux of the bypass: the WAF inspects the entity-encoded form (which contains no recognizable SQL keywords), while the database receives the fully decoded, syntactically valid SQL.
- Concatenating `username || '~' || password` into one column works around the 2-column constraint discovered via the `UNION SELECT NULL,NULL` column-count probe, letting both fields exfiltrate through a single displayed value per row.
