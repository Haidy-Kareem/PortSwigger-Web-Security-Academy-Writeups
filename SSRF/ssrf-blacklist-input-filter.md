# Lab- SSRF with blacklist-based input filter

## Objective

The application has a stock-check feature (`stockApi` parameter) that fetches data from an internal system. This time the developer added two weak anti-SSRF defenses — a blacklist blocking obvious references to `localhost`/`127.0.0.1`, and a blacklist blocking the literal string `admin` in the path. The goal is to bypass both filters to reach `http://localhost/admin` and delete the user `carlos`.

## Step 1: Bypassing the Hostname Blacklist

A direct request to `http://localhost/admin` or `http://127.0.0.1/admin` gets blocked by the filter, since those are the obvious values it's checking for. Since a naive hostname blacklist matches on specific _strings_ rather than the resolved _IP address_, I worked through a list of alternate representations of the loopback address that a string match would likely miss:

|Category|Examples I tried|
|---|---|
|Shorthand / partial IPv4|`127.1`, `127.0.1`, `0`, `0.0.0.0`|
|Decimal / octal / hex encodings of `127.0.0.1`|`2130706433`, `0177.0.0.1`, `0x7f000001`|
|IPv6 loopback forms|`[::1]`, `[0000::1]`, `[::ffff:127.0.0.1]`, `[0:0:0:0:0:ffff:127.0.0.1]`|
|DNS names that resolve to loopback|`localtest.me`, `localh.st`|
|DNS-rebinding-style services|`<anything>.127.0.0.1.nip.io`|
|Host aliases some resolvers accept|`ip6-localhost`, `ip6-loopback`|

Most of these were either blocked by the filter or didn't resolve the way I needed. The one that worked was the shorthand loopback notation `127.1` — a valid alias the OS still resolves to `127.0.0.1`, but which doesn't match a naive blacklist looking for `localhost` or the full `127.0.0.1` string:

```
stockApi=http://127.1/
```

This request succeeded with `HTTP 200 OK` and returned the internal admin page's full HTML content, confirming the hostname filter was bypassed.
<img width="1916" height="912" alt="image" src="https://github.com/user-attachments/assets/82421651-1a22-471c-9bbc-ccd6fd38e644" />


**Why this worked:** any of the forms above would resolve to the same forbidden address as `localhost`, which is exactly why an allowlist on the _resolved_ IP (rather than a blacklist on the string) is the only reliable fix — see Remediation below.

## Step 2: Bypassing the Path Blacklist on "admin"

With the hostname bypassed, appending `/admin` directly still got blocked by the second filter, which checks for the literal word `admin` in the path. This time the underlying idea was the same as Step 1 — the filter checks a string before it's fully normalized, so I worked through a set of encoding variants of `admin`, starting from a single percent-encoded character and adding more encoding layers each time it got blocked:

```
%61dmin
%2561dmin
%256164min
%2561256dmin
%25612564dmin
%2561252564dmin
```

The single-encoded and partially-encoded variants were still caught by the filter. What worked was full double URL-encoding of the leading character:

- Single encoding of `a` is `%61`
- Encoding the `%` itself again turns `%61` into `%2561`

So `admin` became `%2561dmin`. Many servers decode URL-encoded input twice across different processing layers — the filter checks the raw/first-decoded value (which still looks like `%61dmin`, not `admin`, so it passes), but the backend routing eventually decodes it a second time into the literal `admin` path segment. I sent:

```
stockApi=http://127.1/%2561dmin
```

This succeeded and returned the admin users page, revealing the exact delete link for the target user: `/admin/delete?username=carlos`.

<img width="1917" height="892" alt="image" src="https://github.com/user-attachments/assets/4c5cc638-ebc1-49a5-ab5c-24d58b809ba3" />

**Why this worked:** the common thread across both steps is that **the filter inspects the request at one point in the pipeline, but the destination is resolved at another** — any gap between those two points is exploitable, whether it's an unrecognized hostname alias or a decode pass the filter didn't account for.

## Step 3: Triggering the Delete via SSRF

Combining both bypasses, I sent the full delete request:

```
stockApi=http://127.1/%2561dmin/delete?username=carlos
```

The response was a `302 Found` redirecting to `/admin`, confirming the delete action on the internal admin interface succeeded.

<img width="1917" height="771" alt="image" src="https://github.com/user-attachments/assets/93e54b0c-7064-41ec-aa89-06275451a798" />

## Step 4: Confirming the Result

The lab was marked as solved after the request went through.

<img width="1917" height="852" alt="image" src="https://github.com/user-attachments/assets/f84065c4-2354-41ac-a613-65bc94453c9b" />

## Root Cause

Both defenses relied on blacklisting specific literal strings instead of properly validating the parsed, fully-decoded destination of the request. A blacklist on hostnames missed alternate representations of the same loopback address (`127.1` instead of `127.0.0.1`/`localhost`), and a blacklist on path keywords missed the fact that decoding happens more than once across the request-processing pipeline, so a double-encoded value slips past a filter that only checks a single decode pass.

## Impact

An attacker can still reach the same sensitive internal admin functionality as the fully unprotected version of this bug, just with a couple of encoding/formatting tricks — demonstrating that blacklist-based SSRF defenses are fundamentally fragile, since there are many equivalent ways to represent the same forbidden host or path.

## Remediation

- Never rely on blacklisting known-bad strings for SSRF protection; validate against an allowlist of expected, fully-resolved hosts and paths instead.
- Normalize and fully decode input (repeatedly, until stable) _before_ applying any filtering logic, so encoding tricks like double URL-encoding can't slip through.
- Resolve hostnames to their actual IP addresses before validation, and reject any request whose resolved IP falls in loopback, link-local, or other internal ranges — rather than pattern-matching on the string form of the host.
- Apply defense in depth: even with a perfect input filter, internal admin interfaces should still require their own authentication rather than trusting the network path alone.

## Notes

- This lab is a good demonstration of the general SSRF bypass toolkit for blacklist-style filters: alternate IP/hostname representations (see the table in Step 1) for hostname filters, and encoding tricks (single/double URL-encoding, case variation, path traversal segments) for keyword filters in the path.
- The two bypasses were independent and stacked cleanly — fixing the hostname check alone or the path check alone wouldn't have been enough; both needed a workaround simultaneously.
- `127.1` is a lesser-known but valid shorthand for `127.0.0.1` — worth keeping in a mental list of loopback aliases alongside `0`, `0.0.0.0`, and IPv6 equivalents like `[::1]`, since blacklists rarely account for all of them.
