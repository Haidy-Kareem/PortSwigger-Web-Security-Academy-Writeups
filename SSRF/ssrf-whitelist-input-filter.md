# Lab- SSRF with whitelist-based input filter

## Description

The application contains a Server-Side Request Forgery (SSRF) vulnerability in the stock-check functionality.

The application allows the user to provide a URL through the `stockApi` parameter. The server then makes a request to the supplied URL.

To prevent SSRF attacks, the application uses a **whitelist-based filter** that only allows requests to the trusted domain:

```text
stock.weliketoshop.net
```

However, the validation can be bypassed because the application processes and decodes the URL differently during validation and during the actual request.

The bypass relies on URL parsing behavior involving:

- The `@` character
- URL encoding
- Double URL encoding
- The `#` fragment delimiter

This allows an attacker to make the whitelist accept a URL containing the trusted hostname while causing the backend to interpret the URL differently.

## Attack Scenario

The intended request is to access the internal administrative interface:

```text
http://localhost/admin
```

However, a direct request to `localhost` is blocked by the whitelist.

The following request is therefore rejected:

```text
http://127.0.0.1/
```

The application expects the hostname to belong to the trusted domain:

```text
stock.weliketoshop.net
```

The vulnerability can be exploited by manipulating the URL structure so that the whitelist validation and the backend URL parser interpret the same input differently.

## Steps to Reproduce

### 1. Access the stock-check functionality

Open a product and click **Check stock**.

Intercept the request using Burp Suite and send it to **Repeater**.

The request contains the `stockApi` parameter:

```text
stockApi=http://stock.weliketoshop.net:8080/product/stock/check?productId=1&storeId=1
```

### 2. Test the whitelist

Change the value to:

```text
http://127.0.0.1/
```

The application rejects the request because `127.0.0.1` is not included in the whitelist.

### 3. Test URL user information

Change the URL to place the trusted hostname before an `@`, testing whether the parser and the whitelist agree on which part is the host:

```text
http://stock.weliketoshop.net...@localhost/admin
```

The request is accepted, `HTTP/2 200 OK`:

<img width="1917" height="930" alt="image" src="https://github.com/user-attachments/assets/981a9ace-f1e3-45bf-83ca-eed79d1bcac8" />

This demonstrates that the application supports the `userinfo@hostname` URL syntax — the whitelist is satisfied by seeing `stock.weliketoshop.net` appear in the string, even though it's sitting in the userinfo portion rather than being the actual host.

### 4. Test the fragment character

Add a raw `#` character before the trusted hostname:

```text
stockApi=http://localhost:80#@stock.weliketoshop.net/admin
```

The whitelist rejects this with a `400 Bad Request`:

<img width="1917" height="747" alt="image" src="https://github.com/user-attachments/assets/39fb61c6-0ea2-4ec6-9df4-632498fe0b6f" />

This indicates the filtering logic is sensitive to the fragment delimiter — sent raw, it gets caught.

### 5. Double-encode the `#`

Instead of using `#` directly, use:

```text
%2523
```

This represents a double-encoded `#`. The decoding process is:

```text
%2523  →  %23  (first decode)  →  #  (second decode)
```

The filter sees `%2523` (not `#`), passes it, but the backend eventually decodes it twice and treats it as the real fragment delimiter — splitting the URL differently than the whitelist did.

### 6. Construct the SSRF payload

The final payload used for the lab is based on the following structure:

```text
http://localhost:80%2523@stock.weliketoshop.net/admin
```

The important components are:

- `localhost:80` — the internal server being targeted
- `%2523` — the double-encoded `#` used to manipulate URL parsing
- `@stock.weliketoshop.net` — the trusted hostname used to satisfy the whitelist
- `/admin` — the administrative path being requested

The key point is that `/admin` represents the **path being requested**, while `localhost` represents the **internal server**. The URL manipulation causes the validation stage and the later URL parsing stage to interpret these components differently.

### 7. Access the administrative interface

Sending the payload from Step 6 reaches the internal admin interface and returns the full users list, with delete links for each user:

<img width="1917" height="811" alt="image" src="https://github.com/user-attachments/assets/5d3f39d1-2f83-4ba5-9009-64c73cf3b3ce" />

With the delete endpoint confirmed (`/admin/delete?username=carlos`), the same payload was extended to trigger the delete directly:

```text
stockApi=http://localhost:80%2523@stock.weliketoshop.net/admin/delete?username=carlos
```

The response is a `302 Found` redirecting to `/admin`, confirming the delete succeeded:

<img width="1912" height="727" alt="image" src="https://github.com/user-attachments/assets/c8169b85-95f9-4e48-902c-73a7cc4857d7" />

The lab was then marked as solved:

<img width="1916" height="841" alt="image" src="https://github.com/user-attachments/assets/5b7d6986-46b6-4a40-9565-44a703191bc5" />

## Why the Bypass Works

The vulnerability exists because the application does not use a single consistent interpretation of the URL.

The whitelist validates one representation of the URL, while later processing decodes and parses the URL again.

The attacker takes advantage of this discrepancy. The `@` character allows the URL to contain a trusted hostname as part of the URL structure. The double-encoded `%2523` is decoded later into `%23` and then `#`, changing how the URL is interpreted.

This creates a difference between what the whitelist validates and what the backend eventually processes.

## Impact

Successful exploitation of this vulnerability allows an attacker to:

- Bypass the SSRF protection mechanism
- Send requests to internal services
- Access internal administrative endpoints
- Potentially perform unauthorized administrative actions
- Access services that are not directly exposed to external users

Depending on the internal services available, SSRF can potentially lead to significant compromise of internal infrastructure.

## Remediation

The application should avoid relying solely on string-based URL whitelisting.

Recommended protections include:

1. Parse and normalize the URL exactly once before validation.
2. Validate the final resolved hostname and IP address.
3. Do not allow ambiguous URL syntax such as userinfo unless required.
4. Decode URL encoding before performing security validation.
5. Validate both hostname and resolved IP address.
6. Block requests to private, loopback, link-local, and other internal IP ranges.
7. Disable redirects or validate every redirect destination.
8. Use a strict allowlist of exact destinations rather than substring-based validation.

## Proof of Concept

The vulnerable parameter is:

```text
stockApi
```

The SSRF bypass payload used in the lab:

```text
http://localhost:80%2523@stock.weliketoshop.net/admin
```

The vulnerability demonstrates that a whitelist-based SSRF defense can be bypassed when URL parsing and decoding are performed inconsistently between the validation layer and the backend request handler.
