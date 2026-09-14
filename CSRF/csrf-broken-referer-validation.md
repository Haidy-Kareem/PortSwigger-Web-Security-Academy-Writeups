# Lab- CSRF with broken Referer validation


## **Objective** 

The email change functionality in this lab is vulnerable to CSRF. The application attempts to detect and block cross-domain requests using Referer validation, but the detection mechanism itself can be bypassed.

**Step 1: Confirming Referer Validation Exists**  
After logging in as `wiener`, I captured the legitimate email change request and noted the session cookie. I sent the request to Repeater and swapped the Referer header for an unrelated domain, `https://google.com`. The server responded with `400 Bad Request` and `"Invalid referer header"`, confirming that Referer validation is actually enforced.

<img width="1568" height="738" alt="image" src="https://github.com/user-attachments/assets/ea372ac2-c2f3-4614-a74a-ada5ffb879b7" />

<img width="1568" height="766" alt="image" src="https://github.com/user-attachments/assets/687e50e5-1a1f-452f-98b2-83f5117980fa" />

**Step 2: Testing a Subdomain Bypass**  
I tested whether the check only looks for the lab domain appearing somewhere in the Referer, rather than validating it properly. I set the Referer to a URL where the lab's domain was appended as if it were a subdomain of an attacker domain — effectively `attacker.com` wrapped around the lab ID in a way that still contains the exact lab hostname. This request was accepted with a `302 Found`, showing the validation isn't checking the real origin, just whether the expected string shows up in the header.

<img width="1568" height="781" alt="image" src="https://github.com/user-attachments/assets/5bdcffad-dcb1-4dd8-92c8-9e2f47a8af32" />

Step 3: Testing the Same Idea in a Query String — First Attempt Failed**  
I tried placing the lab ID inside a query string after an attacker domain instead (`https://attacker.com/?LAB-ID`). My first attempt used the wrong lab ID by mistake (a leftover from a previous lab), so the server correctly rejected it with `400 Bad Request` / `"Invalid referer header"`.

<img width="1568" height="734" alt="image" src="https://github.com/user-attachments/assets/34c375eb-c245-4349-8f20-a22a0bc671c0" />

**Step 4: Correcting the Lab ID — Bypass Confirmed**  
After correcting the Referer to use the actual target lab ID inside the query string (`https://attacker.com/?ACTUAL-LAB-ID`), the request was accepted with a `302 Found`. This confirmed the vulnerability precisely: the application checks whether its own domain appears anywhere inside the Referer string, not whether the Referer's actual host matches.

<img width="1568" height="782" alt="image" src="https://github.com/user-attachments/assets/fe51504a-d883-4764-8dc4-bb1980198021" />

**Step 5: Building the Browser-Based Exploit**  
A `starts-with`/`contains` bypass that works in Repeater doesn't automatically work from a real browser, because browsers can trim the query string out of the Referer depending on the page's Referrer Policy. To keep the full URL (including the injected lab ID) in the Referer sent by the browser, I used `history.pushState()` to rewrite the exploit page's visible URL to include the lab ID as a query string, without actually navigating anywhere. I also set the response header `Referrer-Policy: unsafe-url` on the exploit server so the browser would forward the full URL — including the query string — as the Referer when the form submits.

The final exploit body:

html

```html
<form action="https://0ac300d1035513ff81eb665f00210042.web-security-academy.net/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="hackeeee@gmail.com">
</form>

<script>
    history.pushState("", "", "/?0ac300d1035513ff81eb665f00210042.web-security-academy.net");
    document.forms[0].submit();
</script>
```

With the response head:

```
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Referrer-Policy: unsafe-url
```

<img width="1526" height="784" alt="image" src="https://github.com/user-attachments/assets/8bb772cc-76c5-4b02-b6d1-4c9f70dfbf3a" />

**Step 6: Delivering the Exploit and Confirming the Result**  
The exploit was stored and delivered to the victim. The forged POST request carried a Referer containing the lab's domain (thanks to `pushState` and the unsafe-url policy), the server's naive "contains" check accepted it, and the victim's email was changed. The lab was marked as solved.

<img width="1563" height="784" alt="image" src="https://github.com/user-attachments/assets/bfd89707-2b52-41cc-9870-bace084c474d" />

## **Root Cause**  

The application validates the Referer header by checking whether its own domain appears somewhere inside the header value, instead of properly parsing the URL and comparing the actual hostname. This lets an attacker place the trusted domain anywhere in an otherwise attacker-controlled URL — as a subdomain suffix or inside a query string — and still pass the check.

## **Impact**  

An attacker can silently change a victim's account email address through a hosted CSRF page, potentially leading to a follow-up password reset and full account takeover.

## **Remediation**

- Never validate Referer/Origin with substring matching; parse the URL properly and compare the exact scheme and hostname against an allowlist.
- Prefer CSRF tokens bound to the authenticated session as the primary defense, using Referer/Origin checks only as a secondary layer.
- Set a strict `Referrer-Policy` (e.g. `same-origin` or `strict-origin-when-cross-origin`) application-wide so pages don't leak full URLs to third parties by default — this doesn't fix the server bug, but reduces what attacker-controlled pages can manipulate.

## **Notes**

- This lab is a good example of why "contains" checks are dangerous for any security control involving domains — a subdomain suffix trick and a query-string trick both worked against the same flawed check.
- `history.pushState()` doesn't perform navigation; it only changes what URL the browser reports as current, which is exactly what's needed to control the Referer without triggering a real page load.
- Getting a working query-string bypass in Repeater doesn't guarantee it works from an actual browser — the browser's Referrer Policy can silently strip the query string, so the `Referrer-Policy: unsafe-url` header on the exploit response was essential here.
- One test attempt failed only because of a copy-paste mistake (wrong lab ID left over from a previous lab) — worth double-checking IDs when reusing payload templates across labs.
