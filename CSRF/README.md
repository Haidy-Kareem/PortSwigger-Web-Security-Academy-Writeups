# CSRF (Cross-Site Request Forgery)

This folder contains write-ups for CSRF labs from PortSwigger Web Security 
Academy, covering token bypass techniques, SameSite cookie misconfigurations, 
and Referer validation weaknesses.

## Covered Concepts
- Token validation bypass (method-based, presence-based)
- Token not tied to user session
- Token tied to non-session cookie
- Token duplicated in cookie
- SameSite Lax/Strict bypass techniques
- Referer-based validation bypass

## Write-ups
- [Lab: CSRF where token validation depends on request method](csrf-token-validation-depends-on-request-method.md)
- [Lab: CSRF where token validation depends on token being present](csrf-token-validation-depends-on-token-being-present.md)
- [Lab: CSRF where token is not tied to user session](csrf-token-not-tied-to-user-session.md)
- [Lab: CSRF where token is tied to non-session cookie](csrf-token-tied-to-non-session-cookie.md)
- [Lab: CSRF where token is duplicated in cookie](csrf-token-duplicated-in-cookie.md)
- [Lab: SameSite Lax bypass via method override](csrf-samesite-lax-bypass-method-override.md)
- [Lab: SameSite Strict bypass via client-side redirect](csrf-samesite-strict-bypass-client-side-redirect.md)
- [Lab: CSRF where Referer validation depends on header being present](csrf-referer-validation-depends-on-header-being-present.md)
- [Lab: CSRF with broken Referer validation](csrf-broken-referer-validation.md)
