# Lab- Basic SSRF against the local server

## Objective

The application has a stock-check feature on product pages that fetches data from an internal system (`stockApi` parameter). The goal is to abuse this server-side request to reach the internal admin interface at `http://localhost/admin` and delete the user `carlos`.

## Step 1: Locating the Admin Interface

I first sent the stock-check request with a benign `stockApi=http://localhost/admin` value to Repeater. The server followed through and fetched the internal admin page, returning its HTML content back in the response — including a list of users (`wiener`, `carlos`) each with a "Delete" link pointing to `/admin/delete?username=<name>`. This confirmed the SSRF: the server was willing to fetch and return content from an internal-only endpoint that shouldn't be reachable from outside.

<img width="1917" height="861" alt="image" src="https://github.com/user-attachments/assets/748149bb-648f-48a5-bafc-1a18c95f2fb2" />

## Step 2: Extracting the Delete Path

From the returned HTML, I copied the exact delete path for the target user:

```
/admin/delete?username=carlos
```

This is the internal admin action the SSRF needed to trigger.

## Step 3: Triggering the Delete via SSRF

I modified the `stockApi` parameter to point directly at that delete path instead of just the admin homepage:

```
stockApi=http://localhost/admin/delete?username=carlos
```

Sending this request made the server internally issue the delete request to its own admin interface on `carlos`'s behalf. The response was a `302 Found` redirecting to `/admin`, along with a new session cookie being set — consistent with the delete action completing successfully.

<img width="1917" height="920" alt="image" src="https://github.com/user-attachments/assets/bb91af9e-9f96-4721-9815-50ffb0c2ad2c" />

## Step 4: Confirming the Result

The lab was marked as solved after the request went through.

<img width="1917" height="862" alt="image" src="https://github.com/user-attachments/assets/2f6dee46-5f52-4f90-a1fc-de8ace8523d1" />

## Root Cause

The stock-check feature accepts a fully attacker-controlled URL (`stockApi`) and makes a server-side HTTP request to it with no validation of the host, scheme, or destination. Because the server itself is trusted by the internal admin interface (which likely only checks that the request comes from `localhost`), an attacker can use the server as a proxy to reach and interact with internal endpoints that are otherwise inaccessible from the outside — including state-changing actions like deleting a user.

## Impact

An attacker with no direct access to the internal network can still fully interact with internal-only admin functionality by routing requests through the vulnerable server, potentially leading to unauthorized administrative actions (as demonstrated here), internal service enumeration, or further internal reconnaissance depending on what else is reachable on `localhost` or the internal network.

## Remediation

- Never let user input directly control the destination of a server-side request; validate against a strict allowlist of permitted hosts/URLs.
- Don't rely on `localhost`/internal-IP checks alone to authenticate internal services — internal endpoints should still require proper authentication, not just network-origin trust.
- Apply network-level segmentation so the application server can't reach sensitive internal admin interfaces at all unless explicitly required.
- Disable following of unexpected redirects and restrict the schemes/ports the server-side request can use (e.g., block `file://`, restrict to expected internal service ports only).

## Notes

- This is a very direct SSRF pattern: the vulnerable parameter's whole job is to be an external URL fetched by the server, so there's no need for path traversal, encoding tricks, or bypasses — just pointing it at the internal target is enough.
- The internal admin interface trusted `localhost` as sufficient authorization, which is exactly the assumption SSRF vulnerabilities are built to break — the app itself becomes the "internal user" making the request.
- Useful technique: when reflecting back the fetched content (like the admin HTML here), read the response carefully — it directly revealed the exact delete-action URL format needed for the exploit, no guessing required.
