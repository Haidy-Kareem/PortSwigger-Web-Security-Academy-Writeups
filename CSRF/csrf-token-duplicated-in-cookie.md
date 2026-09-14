# Lab- CSRF where token is duplicated in cookie

## **Objective**

The goal of this lab is to change the victim's account email address by exploiting a CSRF vulnerability in the email change functionality, which relies on the insecure "double submit cookie" pattern for CSRF protection.

## **Vulnerability Overview**

Instead of tying the CSRF token to the server-side session, the application implements the "double submit" technique: it sends a `csrf` cookie to the browser and expects the same value to be submitted as a `csrf` request parameter. The server's check is simply whether the cookie value and the parameter value match — it never verifies that this value was actually issued to the current session. Since an attacker can choose any arbitrary value, all they need is a way to make the victim's browser hold a `csrf` cookie equal to a value the attacker also controls in the forged request. If the attacker can set that cookie in the victim's browser, the double-submit check becomes meaningless, because both sides of the comparison are attacker-controlled.

**Step 1: Testing the Double-Submit Behavior in Repeater**  
After logging in as `wiener`, I intercepted the email change request in Burp Suite. It included a `csrf` cookie alongside a `csrf` body parameter. I sent the request to Repeater and tried changing the `csrf` cookie and the `csrf` parameter to a matching arbitrary value (`csrf=1` in both places) while keeping the real `session` cookie. The server accepted the request and returned a 302 redirect to `/my-account`, confirming that any value works as long as the cookie and the parameter match — the token itself carries no real per-session meaning.

<img width="1568" height="606" alt="image" src="https://github.com/user-attachments/assets/2e38f305-4805-46f7-bacd-184147532f00" />

<img width="1568" height="618" alt="image" src="https://github.com/user-attachments/assets/f71be2ef-e54a-41cd-abab-4ba4e89214f4" />

**Step 2: Finding a Way to Set the Cookie in the Victim's Browser**  
As in the earlier lab, the search functionality on this application reflects the search term into the response in a way that allows CRLF injection, letting an attacker inject an arbitrary `Set-Cookie` header. This time, the payload only needs to set a `csrf` cookie to a value the attacker also controls, since there's no session-binding to work around — the check is purely cookie-versus-parameter.

The injection URL used was:

```
/?search=test%0d%0aSet-Cookie: csrf=1;SameSite=None
```

When the victim's browser requests this URL, the server's response smuggles in an extra `Set-Cookie: csrf=1; SameSite=None` header, which the browser stores.


**Step 3: Building the Exploit**  
The exploit combines the cookie-injection request with an auto-submitting form targeting the change-email endpoint, using the same `csrf` value (`1`) that was just planted as a cookie:

```html
<img src="<https://0a6b00d203fcd5da805e44d500620095.web-security-academy.net/?search=test%0d%0aSet-Cookie:> csrf=1;SameSite=None" onerror="document.forms[0].submit()">

<form action="<https://0a6b00d203fcd5da805e44d500620095.web-security-academy.net/my-account/change-email>" method="POST">
    <input required type="email" name="email" value="hacked@gmail.com">
    <input required type="hidden" name="csrf" value="1">
</form>
```

The `<img>` tag triggers a GET request to the vulnerable search endpoint, injecting the `csrf` cookie into the victim's browser. Because the image path isn't a real image, it fails to load, which fires `onerror` and submits the hidden form. At that point, the victim's browser sends the POST request carrying the real `session` cookie, the newly injected `csrf` cookie, and the matching `csrf=1` body parameter — satisfying the double-submit check even though the value was never legitimately issued to that session.

<img width="1568" height="783" alt="image" src="https://github.com/user-attachments/assets/87ebcc1b-a3ee-4474-b44a-2c4af3dd52c8" />

**Step 4: Delivering the Exploit and Confirming the Result**  
The exploit was stored on the exploit server and delivered to the victim. The email change went through, and the lab was marked as solved.

<img width="1568" height="783" alt="image" src="https://github.com/user-attachments/assets/4e472ec9-cc34-4ec2-ac23-dc35786f2c65" />

 

## Root Cause 

The double-submit cookie pattern only proves that the same origin that set the cookie is the one submitting the parameter — it does not prove that the value was ever assigned by the server to a specific, authenticated session. Because the application also had a separate flaw that let an attacker set arbitrary cookies on the victim's browser (via CRLF injection in the search response), the attacker could satisfy both sides of the double-submit check with a self-chosen value, completely defeating the protection.

**Impact**  
An attacker can silently change a victim's account email address by delivering a single malicious link, and from there potentially trigger a password reset to take over the account.

## **Remediation**

- Do not rely on double-submit cookies as the sole CSRF defense; tie the CSRF token to the server-side session so an attacker cannot generate a valid token/cookie pair on their own.
- Eliminate the header injection vulnerability in the search functionality by never reflecting user input directly into HTTP response headers.
- Set CSRF cookies with `HttpOnly` where applicable and enforce strict `SameSite` settings to reduce the attack surface for cross-site cookie manipulation.

## **Notes**

- This lab is essentially a variation of the earlier "csrfKey tied to non-session cookie" lab: same root weakness (search endpoint allows `Set-Cookie` injection via CRLF), different CSRF-defense mechanism being bypassed (double-submit cookie instead of session-bound token).
- No incognito/second account was needed here, since the double-submit check doesn't reference any server-side stored value — any self-chosen `csrf` value works as long as the cookie and parameter match.
- The `onerror` trick on the `<img>` tag remains the key technique to sequence "inject cookie first, then submit form" in a single page load without user interaction.
