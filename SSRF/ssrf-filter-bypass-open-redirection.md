# Lab- SSRF with filter bypass via open redirection vulnerability

## Objective

The stock-check feature (`stockApi` parameter) fetches data from an internal system. This time the anti-SSRF defense only allows `stockApi` to point at **local** (same-origin) URLs — anything with an external host gets rejected outright. The goal is to reach the internal admin interface at `http://192.168.0.12:8080/admin` and delete the user `carlos`.

## Step 1: Finding the Open Redirect

The product page has a `nextProduct` endpoint that redirects based on a `path` parameter. Sent as-is with a normal in-app path, it behaves as expected:

```
GET /product/nextProduct?currentProductId=7&path=/product?productId=8
```

→ `302 Found`, `Location: /product?productId=8`.

<img width="1917" height="797" alt="image" src="https://github.com/user-attachments/assets/52d2955a-b767-447d-80a1-a24a7ef43ae9" />

To check whether `path` is actually validated, I swapped it for an unrelated external domain:

```
GET /product/nextProduct?currentProductId=7&path=google.com
```

This also redirected — `302 Found`, `Location: google.com` — confirming `path` isn't validated at all, i.e. a genuine open redirect.

<img width="1917" height="895" alt="image" src="https://github.com/user-attachments/assets/f763e78b-9db8-4113-b999-2b213a526f8f" />

I followed the crafted URL in an actual browser to confirm it's exploitable outside of Burp too, and it landed straight on Google.

<img width="1917" height="811" alt="image" src="https://github.com/user-attachments/assets/e777c56d-3367-4f81-aa8f-81853d5a86dc" />

## Step 2: Confirming the Redirect Accepts Internal Addresses

Since `path` isn't validated at all, I tried the internal admin address directly:

```
GET /product/nextProduct?currentProductId=7&path=http://192.168.0.12:8080/admin
```

Same result — `302 Found`, `Location: http://192.168.0.12:8080/admin`. So the open redirect will happily send anyone (including the backend) to an internal-network address.

<img width="1917" height="897" alt="image" src="https://github.com/user-attachments/assets/c2d1632e-7b57-484e-a927-9f94a5a49b63" />

## Step 3: First Attempt — Full URL in stockApi (Failed)

I tried handing `stockApi` the complete absolute URL to the redirect endpoint:

```
stockApi=https://<lab-id>.web-security-academy.net/product/nextProduct?currentProductId=7&path=http://192.168.0.12:8080/admin
```

This got rejected — `400 Bad Request`, `"Invalid external stock check url 'Invalid URL'"`. Passing it as a full `https://host/...` value didn't satisfy whatever local-URL check the filter does here.

<img width="1917" height="892" alt="image" src="https://github.com/user-attachments/assets/70d4201b-df3e-41da-a4d0-fdd998331de0" />

## Step 4: The Working Payload — Relative Path + Double Parameter Encoding

Instead of a full URL, I gave `stockApi` a **relative** path to the same `nextProduct` endpoint — no scheme, no host, so it's unambiguously "local." The inner `&` (the one separating `currentProductId` from `path`) needed to be percent-encoded as `%26`, otherwise it would get parsed as a second top-level parameter of the outer request instead of staying part of the `stockApi` value:

```
stockApi=/product/nextProduct?currentProductId=7%26path=http://192.168.0.12:8080/admin
```

This worked. The backend fetched the relative URL, the app's own redirect handler decoded `%26` back into `&`, read `path=http://192.168.0.12:8080/admin`, and issued the redirect — which the stock-check backend followed itself, landing on the internal admin page and returning its HTML, including delete links for every user.

<img width="1916" height="802" alt="image" src="https://github.com/user-attachments/assets/bee5782c-10f1-452c-8640-bb11a872dbcc" />

## Step 5: Deleting carlos

With the delete endpoint confirmed (`/admin/delete?username=carlos`), I pointed the same relative-redirect payload at it directly:

```
stockApi=/product/nextProduct?currentProductId=7%26path=http://192.168.0.12:8080/admin/delete?username=carlos
```

<img width="1917" height="901" alt="image" src="https://github.com/user-attachments/assets/9e539124-0a72-404b-9ca6-4020f6ad47d7" />

## Step 6: Confirming the Result

The lab was marked as solved after the delete request went through.

<img width="1917" height="962" alt="image" src="https://github.com/user-attachments/assets/b21ac4d4-12c1-4684-86c2-7470c6df1664" />

## Root Cause

The `stockApi` filter only checks whether the URL _looks_ local — no scheme, no external host — and stops there. "Local" isn't the same thing as "safe": the app has its own endpoint (`nextProduct`) that performs an unvalidated redirect based on attacker-controlled input. A local URL that itself redirects arbitrarily lets the backend get handed off to any destination once it follows that redirect, with no re-validation applied to where it actually ends up.

## Impact

By chaining the open redirect with the SSRF filter, an attacker can bypass a same-origin/local-only restriction entirely, reach internal-network services and admin interfaces, and perform unauthorized administrative actions on systems that were never meant to be reachable from outside.

## Remediation

- Fix the open redirect at the source: validate `path` (and any redirect target) against an allowlist of expected destinations instead of passing it through unchecked.
- If the backend fetcher follows redirects at all, re-apply the full SSRF/local-URL validation to every redirect target, not just the first URL requested.
- Prefer disabling automatic redirect-following for server-side requests built from user input; if redirects must be followed, cap the count and re-validate each hop.
- Resolve and validate the final destination host/IP rather than just the string form of the initial URL.

## Notes

- The key insight here is that "local URL" filters only cover the _first hop_. Any redirect anywhere in the app — even one that looks completely unrelated to the SSRF feature — extends the attack surface the filter was supposed to close off.
- The `%26` trick is the same idea as double URL-encoding in general: make sure the payload survives being embedded _inside_ another parameter instead of getting split off and parsed separately too early.
- The full absolute-URL attempt (Step 3) failing first is worth keeping in the write-up — it's what made clear the filter cares about the URL's _form_, not just its destination, which is what pointed toward trying a relative path instead.
