# Lab- SameSite Lax bypass via method override

## **Objective**

The goal of this lab is to change the victim's account email address by exploiting a CSRF vulnerability, even though the session cookie is set with `SameSite=Lax`, which normally blocks cookies from being sent on cross-site POST requests.

## **Vulnerability Overview** 

`SameSite=Lax` cookies are still sent on top-level, "safe" cross-site navigations — most notably plain GET requests, such as a link click or a page load. The application also supports method overriding: it accepts a `_method` parameter, which lets a request sent as GET be internally treated as if it were a POST. Combining the two removes the SameSite protection entirely, because the attacker never needs to trigger a cross-site POST at all — a cross-site GET is enough, and the server itself converts it into a POST internally.

**Step 1: Confirming the App Rejects a Direct GET** I intercepted the email change request while logged in and tried resending it as a plain GET with the email parameter in the query string. The server responded with `405 Method Not Allowed`, confirming the change-email endpoint strictly requires POST under normal circumstances.

<img width="1568" height="777" alt="image" src="https://github.com/user-attachments/assets/31f6c7bd-5b1f-49d6-a7d1-c6b76ce86d03" />

**Step 2: Discovering the Method Override Parameter** I retried the same GET request, this time adding `&_method=POST` to the query string. The server responded with a 302 redirect to `/my-account`, confirming the email was updated successfully. This showed that the application silently supports overriding the HTTP method through a parameter, and that a GET request carrying `_method=POST` is processed exactly like a real POST.

<img width="1555" height="784" alt="image" src="https://github.com/user-attachments/assets/235dd40a-c29c-4502-9fbb-dc3e39664a04" />

**Step 3: Understanding Why This Bypasses SameSite=Lax** Because the entire request is technically a GET, the browser treats it as a top-level, safe cross-site navigation, so it still attaches the `SameSite=Lax` session cookie even when the navigation originates from a third-party page. The server-side method override then reinterprets that GET as a POST, so from the application's point of view a real state-changing action just took place — all without ever needing a cross-site POST, which `SameSite=Lax` would have blocked.

**Step 4: Building the Exploit** The exploit is a simple auto-submitting form that uses `method="GET"`, targets the change-email endpoint, and passes `_method=POST` alongside the new email address:

```html
<form action="https://0aa0005503c8a84584e5191c00e90032.web-security-academy.net/my-account/change-email" method="GET">
    <input type="hidden" name="_method" value="POST">
    <input type="email" name="email" value="hackeddd@gmail.com">
</form>

<script>
document.forms[0].submit();
</script>
```

Since the whole request is a GET navigation from the victim's browser, `SameSite=Lax` allows the session cookie through, and the server's method-override logic treats it as an authenticated POST to change the email.

<img width="1568" height="782" alt="image" src="https://github.com/user-attachments/assets/c5524b7b-1346-4afb-a8a6-fbd844be56dc" />

**Step 5: Delivering the Exploit and Confirming the Result** The exploit was stored on the exploit server and delivered to the victim. The email change succeeded and the lab was marked as solved.

<img width="1565" height="784" alt="image" src="https://github.com/user-attachments/assets/fbc2a5e9-39aa-474b-995d-bc11210ed82d" />

## **Root Cause**

The vulnerability comes from combining two separate design choices that are each reasonable in isolation but dangerous together: `SameSite=Lax` cookies are deliberately allowed on top-level GET navigations, and the application's method-override feature lets a GET request behave like a POST. Neither feature alone breaks the CSRF defense, but together they let an attacker fully sidestep the protection SameSite=Lax was meant to provide, since the state-changing action is delivered as a "safe" GET on the wire.

**Impact** An attacker can change a victim's account email through nothing more than a link the victim clicks or a page they land on — no form submission or POST is required — potentially leading to full account takeover via a follow-up password reset.

## **Remediation**

- Disable or tightly restrict HTTP method override functionality; if it's needed, don't honor it for state-changing endpoints that rely on SameSite cookies for CSRF protection.
- Use `SameSite=Strict` for sensitive session cookies where cross-site top-level navigation isn't a legitimate use case.
- Pair SameSite cookie protections with a proper CSRF token bound to the session, rather than relying on SameSite alone.

## **Notes**

- This lab is a good reminder that SameSite=Lax was designed with the assumption that GET requests are "safe" (non-state-changing); any feature that lets a GET perform a state change — like method override — undermines that assumption entirely.
- No CRLF injection or cookie tricks were needed here, unlike the last two labs — this one is purely about request method manipulation.
- Worth checking any app for hidden `_method`, `X-HTTP-Method-Override`, or similar override parameters/headers whenever a state-changing endpoint only accepts POST, since they're an easy way to turn a "safe" GET into a real action.



---

<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=pwned@web-security-academy.net&_method=POST";
</script>    

official solution 
