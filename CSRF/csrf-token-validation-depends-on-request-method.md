# Lab- CSRF where token validation depends on request method

## Objective

The application's email change functionality is protected by a CSRF token, but the protection isn't applied consistently across every way the action can be triggered. The lab's target was to prove that the token check could be bypassed entirely and to actually change a victim's email address using a forged, self-triggering request hosted on an external page — not just demonstrate that a token was missing.

---

## 1. Capturing the Legitimate Request

Before trying to break anything, I needed to see what the protected request normally looked like. I logged into the lab account (`wiener:peter`), turned on Burp's intercept, and submitted an email change through the account settings page.

The intercepted request confirmed the endpoint and how the token was being sent:

```
POST /my-account/change-email HTTP/1.1
Host: 0a90000e03678bab80ed265f00f40074.web-security-academy.net
Cookie: session=SaQSPbuO0bZMkRVaHhOvOiKW9vKmN5Iq
Content-Type: application/x-www-form-urlencoded

email=haidy1289%40gmail.com&csrf=cEwfQvmtfUwdjB4EmGJ8wvZNWzPFIYVx
```

Both `email` and `csrf` were sent in the request body, over `POST`. This told me the token check was almost certainly tied to this specific request shape which is exactly what I wanted to test next.

<img width="1512" height="790" alt="image" src="https://github.com/user-attachments/assets/34518146-0abe-406f-b96b-fa2a456ae18b" />


---

## 2. Testing Whether the Method Itself Mattered

Rather than guessing at ways to forge or predict the token, I tested a simpler question first: does the server actually validate `csrf` on every method that reaches this route, or only on `POST`? Many CSRF checks are implemented as middleware attached to specific route handlers, and it's a common mistake to only wire that middleware into the `POST` handler while forgetting that the same underlying action can sometimes be reached another way.

I used Burp's **"Change request method"** option on the intercepted request to convert it from `POST` to `GET`. Burp automatically moved the parameters into the query string and dropped the request body which meant the `csrf` parameter was no longer being sent as part of a validated body, just sitting in the URL:

```
GET /my-account/change-email?email=haidy1289%40gmail.com&csrf=cEwfQvmtfUwdjB4EmGJ8wvZNWzPFIYVx HTTP/1.1
Host: 0a90000e03678bab80ed265f00f40074.web-security-academy.net
Cookie: session=SaQSPbuO0bZMkRVaHhOvOiKW9vKmN5Iq
```

<img width="1552" height="784" alt="image" src="https://github.com/user-attachments/assets/d0c1b0a7-5ac4-4384-957d-089a65e5cef6" />

---

## 3. Confirming the Bypass

I forwarded the request and checked the account page. The email address had changed successfully proving the server wasn't actually validating the `csrf` value on this code path at all. This confirmed the core vulnerability: token validation was implemented only for the `POST` handler, and the `GET` version of the same action skipped it entirely.

This meant the validation logic was effectively:

```
POST  →  csrf checked
GET   →  csrf ignored
```

Since a `GET` request can be triggered passively by a victim's browser with zero interaction, this was enough on its own to build a working CSRF exploit — no need to predict, steal, or reuse a token at all.

---

## 4. Building the Exploit

Because the vulnerable action could now be triggered with a plain `GET` request, I didn't need a self-submitting HTML form (the usual CSRF approach for `POST` endpoints). Any HTML element that causes the browser to automatically issue a `GET` request would work, so I used the simplest one — an `<img>` tag pointed at the vulnerable URL:

```html
<img src="https://0a90000e03678bab80ed265f00f40074.web-security-academy.net/my-account/change-email?email=haidykareem1289@gmail.com">
```

I made sure to use a different email address than the one I'd already tested with, since the lab won't let you register an email that's already taken by another account.

The `<img>` tag isn't meaningful here because the response is an image — it's used purely because the browser will fetch whatever URL is in `src` automatically, with no click and no confirmation needed from the victim. This wouldn't have worked if the endpoint only accepted `POST`, since an `<img>` tag has no way to send a request body or change its method.

I pasted this into the exploit server's response body.

<img width="1568" height="783" alt="image" src="https://github.com/user-attachments/assets/5f4cd58b-314b-4f0d-b4ed-b360f1b98597" />

---

## 5. Delivering the Exploit

I clicked **Store**, then **Deliver exploit to victim**. The exploit server hosted the page and served it to the simulated victim. As soon as the victim's browser rendered the page, it loaded the `<img>` tag, fired the `GET /my-account/change-email` request using the victim's own session cookie, and changed their email without the victim doing anything beyond visiting the page.

The lab flagged as solved immediately after delivery.

<img width="1556" height="784" alt="image" src="https://github.com/user-attachments/assets/69637500-eb96-48f9-afb5-f036e7279600" />

---

## 6. Final Payload

The exploit that solved the lab:

```html
<img src="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=attacker-controlled@gmail.com">
```

Delivered as the body of a page hosted on the exploit server, exploiting the fact that the target endpoint validates `csrf` on `POST` but silently skips that check on `GET`.

---

## 7. Notes

The most useful thing this lab reinforced is that a CSRF token existing in the codebase proves nothing on its own what matters is whether the _check_ runs on every path that reaches the sensitive action. Here the token itself was fine and unpredictable; the actual bug was that the validation was wired into the `POST` handler only, and the app happened to also accept the same logical action over `GET`, which never ran that check.

This is also why I tested method tampering before trying anything more complex like token leakage or referer bypasses it's a cheap, fast test (Burp's "Change request method" does it in one click) and it should be one of the first things tried against any state-changing endpoint during a CSRF assessment, even one that already looks protected at first glance.

The exploit delivery mechanism mattered too: because the bypass turned this into a `GET`-based action, the classic CSRF auto-submitting form wasn't even necessary a single `<img>` tag was enough, which makes this class of bug easier to weaponize than a `POST`-only CSRF, since it doesn't require JavaScript or a form submission to fire, just the victim loading a page.
