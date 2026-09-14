# Lab- SameSite Strict bypass via client-side redirect

### Objective

The email change functionality in this lab is vulnerable to CSRF, even though the session cookie is set with `SameSite=Strict` — the strictest setting, which normally blocks the cookie from being sent on **any** cross-site request, including top-level GET navigations (unlike `Lax`).

### Vulnerability Overview

`SameSite=Strict` cookies are only sent when the navigation or request originates from the same site the cookie belongs to. A direct cross-site navigation from an attacker's exploit page straight to the vulnerable site would never carry the session cookie, so a normal CSRF attempt fails outright here. However, the application has a "confirmation" page (`/post/comment/confirmation?postId=...`) that performs a **client-side redirect** based on the `postId` parameter, and that parameter isn't properly restricted — it accepts path traversal sequences. This means an attacker can craft a `postId` value that, once the browser is redirected client-side, actually points at `/my-account/change-email` with attacker-chosen query parameters. Because this second redirect is triggered by JavaScript already running on the vulnerable site's own origin, the browser treats it as a same-site navigation — and same-site navigations do carry `SameSite=Strict` cookies.

### Step 1: Confirming the Change-Email Endpoint Accepts GET + Method Override

Since the final leg of the attack has to happen through a redirect (which the browser always performs as a GET), I first tested in Burp whether the endpoint would accept a GET request with the email and submit parameters in the query string, along with `_method=POST` to override the HTTP method. The request succeeded with a 302 redirect to `/my-account`, confirming the endpoint can be triggered entirely through a GET URL.

<img width="1547" height="784" alt="image" src="https://github.com/user-attachments/assets/bb2d3c5b-a19c-4d0c-9fb0-13cd0d19d965" />

### Step 2: Crafting the Path-Traversal Redirect Payload

With that confirmed, I built a `postId` value using `../../` to break out of the `/post/comment/confirmation` path and land on `/my-account/change-email`, with the email, submit, and `_method=POST` parameters appended and URL-encoded so they survive being nested inside the outer `postId` query parameter:

```
https://<lab-id>.web-security-academy.net/post/comment/confirmation?postId=/../../my-account/change-email%3femail=hacked%40gmail.com%26submit=1%26_method=POST
```

### Step 3: Building the Exploit

The exploit page itself doesn't need a form at all — it just needs to send the victim's browser to the confirmation page, which then performs the client-side redirect on the vulnerable site's own origin:

html

```html
<script>
window.location = "https://0a5700e5031523f380602671005200e8.web-security-academy.net/post/comment/confirmation?postId=/../../my-account/change-email%3femail=hacked%40gmail.com%26submit=1%26_method=POST";
</script>
```

When the victim visits the exploit page, the browser first navigates cross-site to the confirmation page (no cookie needed yet, since that page doesn't require auth to run its redirect logic). The confirmation page's own script then reads the malformed `postId` and redirects the browser again — this time to `/my-account/change-email` — but since this second navigation originates from the vulnerable site itself, it counts as same-site, so the `SameSite=Strict` session cookie is attached. The email change endpoint accepts the GET with `_method=POST`, and the victim's email is changed.

<img width="1568" height="782" alt="image" src="https://github.com/user-attachments/assets/afe7f166-0f83-4bb4-a4da-5e744c450f1b" />

### Step 4: Delivering the Exploit and Confirming the Result

The exploit was stored and delivered to the victim. The lab was marked as solved.
<img width="1568" height="783" alt="image" src="https://github.com/user-attachments/assets/1fc10587-3836-4432-8a4c-0ecd0d2a1e5b" />


### Root Cause

`SameSite=Strict` correctly blocked any direct cross-site request from reaching the vulnerable endpoint with the session cookie attached. The weakness was an unrelated feature — a client-side redirect that trusted user input (`postId`) without validating it — which let an attacker chain a harmless cross-site hop (no cookie needed) into a same-site navigation (cookie included) that lands exactly on the sensitive endpoint.

### Impact

An attacker can silently change a victim's account email address just by getting them to open a link, even though the application uses the strictest SameSite cookie setting — demonstrating that SameSite alone isn't a complete CSRF defense if the site has any client-side redirect that can be abused to "launder" a cross-site visit into a same-site one.

### Remediation

- Never build client-side redirects from unvalidated user input; strictly allowlist redirect destinations or use indirect reference tokens instead of raw paths/URLs.
- Don't rely solely on `SameSite=Strict` cookies for CSRF protection; pair them with a proper session-bound CSRF token so that even a same-site request must also prove it originated from a legitimate page/action.
- Reject GET requests with method-override parameters on sensitive, state-changing endpoints; require the real HTTP method to match the action.

### Notes

- This is the strongest SameSite setting (`Strict`) and it still fell — the bug wasn't in the cookie policy itself but in a same-site open-redirect-like feature that let the attacker "bounce" through the app's own origin.
- The path traversal in `postId` combined with the earlier method-override trick (`_method=POST` on a GET) from the SameSite Lax lab — worth remembering these techniques often stack together across labs.
- Double URL-encoding wasn't used here (unlike the CRLF injection labs) — only single encoding of `?` and `&` (`%3f`, `%26`) was needed since the payload is nested one level inside the `postId` query parameter, not passed through multiple decode stages.
