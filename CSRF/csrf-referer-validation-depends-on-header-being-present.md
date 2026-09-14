# Lab- CSRF where Referer validation depends on header being present

## **Objective**  

The email change functionality in this lab is vulnerable to CSRF. The application attempts to block cross-domain requests by validating the Referer header, but it has an insecure fallback: if the Referer header is missing from the request, the validation step is skipped entirely and the request is processed as legitimate.

## **Step 1: Testing the Referer Defense in Repeater**  
I sent the email change request to Burp Repeater and manually removed the Referer header from the request. After resending it, the server still responded with a 302 redirect to `/my-account`, confirming that the email change was accepted even with no Referer header present at all. This confirmed the insecure fallback: the app doesn't reject requests missing the header, it simply stops checking.

<img width="1568" height="758" alt="image" src="https://github.com/user-attachments/assets/eee3a8ce-db90-4712-aebd-d48c672e9472" />

## **Step 2: Building the Exploit**  
To exploit this from a third-party page, the victim's browser needs to send the change-email request without a Referer header. I added a `<meta name="referrer" content="never">` tag inside the exploit page's head — `"never"` is an older alias for `"no-referrer"`, and it instructs the browser not to send a Referer header at all on submission. The exploit page uses a form pointing at the change-email endpoint with the attacker's chosen email address.

<img width="1568" height="782" alt="image" src="https://github.com/user-attachments/assets/9bf03044-3967-49ec-b099-df54112fec80" />

<img width="1566" height="784" alt="image" src="https://github.com/user-attachments/assets/05cc4f68-b85c-491e-9ee7-3b9c2a01f564" />

## **Step 3: Delivering the Exploit and Confirming the Result**  
The exploit was stored on the exploit server and delivered to the victim. Because the meta referrer tag suppressed the Referer header on submission, the application processed the forged request as valid and updated the victim's email address. The lab was marked as solved.

<img width="1568" height="774" alt="image" src="https://github.com/user-attachments/assets/4bcd2cb5-4fcb-4753-86ce-a1f2a9f4cc90" />

## **Root Cause**  

The application's CSRF defense follows a fail-open pattern: it validates the Referer header only when it's present, and does nothing when it's missing. Referer-based validation is fragile because the header can be legitimately stripped in several ways (privacy settings, browser extensions, proxies, or a no-referrer/never meta tag), and treating its absence as "nothing to check" removes the protection entirely for an attacker who controls how the request is generated.

**Impact**  
An attacker can silently change a victim's account email address by luring them to a malicious page, then trigger a password reset to the new address and take over the account.

## **Remediation**

- Don't rely on the Referer header as the sole CSRF defense; use unpredictable, per-session CSRF tokens validated on every state-changing request.
- If Referer/Origin checks are used as a supplementary defense, reject the request outright when the header is missing instead of letting it through.
- Apply the SameSite cookie attribute (Lax or Strict) to session cookies to reduce the browser's willingness to send them on cross-site requests.

## **Notes**

- The Referer header was removed two different ways across testing: manually deleted in Burp Repeater to confirm the server-side flaw, then suppressed at the browser level using a meta referrer tag in the actual exploit — both achieve the same result of an absent header.
- `content="never"` and `content="no-referrer"` are equivalent values for the `referrer` meta tag; either works for this bypass.
