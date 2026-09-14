# Lab- Basic SSRF against another back-end system

## Objective

The application has a stock-check feature (`stockApi` parameter) that fetches data from an internal system, similar to the earlier basic SSRF lab. This time, the admin interface isn't on `localhost` — it's on an unknown host somewhere in the internal `192.168.0.X` range, on port `8080`. The goal is to scan that range to find the admin interface, then use it to delete the user `carlos`.

## Step 1: Scanning the Internal Range with Intruder

Since the exact internal IP wasn't known, I sent the stock-check request to Burp Intruder and marked the last octet of the IP as the payload position, targeting port 8080 directly (since that was given in the lab description):

```
stockApi=http://192.168.0.§X§:8080/admin
```

I used a **Sniper attack** with a simple numeric payload list covering the likely host range (`0`–`255`). Most requests came back with `500` (connection refused / no service on that IP), but one request returned `200 OK` with a much larger response length than the rest — that response contained the actual admin interface HTML, revealing the correct internal host.

<img width="1477" height="847" alt="image" src="https://github.com/user-attachments/assets/7afe384b-7ed4-4319-8cec-eace51181f2b" />

## Step 2: Extracting the Correct Host and Delete Path

From the successful response, I got both the working internal IP and the exact delete path for the target user:

```
http://192.168.0.15:8080/admin/delete?username=carlos
```

## Step 3: Triggering the Delete via SSRF

I sent this URL as the `stockApi` value in Repeater:

```
stockApi=http://192.168.0.15:8080/admin/delete?username=carlos
```

The response was a `302 Found` redirecting to `http://192.168.0.15:8080/admin`, confirming the delete action was accepted by the internal admin interface.

<img width="1917" height="926" alt="image" src="https://github.com/user-attachments/assets/c4cb6082-9126-46b6-a39b-40c76c32afac" />

## Step 4: Confirming the Result

The lab was marked as solved after the request went through.

<img width="1917" height="846" alt="image" src="https://github.com/user-attachments/assets/21fa9fd2-9c6f-4e99-bb7d-e9cd23b530e7" />

## Root Cause

The stock-check feature accepts a fully attacker-controlled URL and makes a server-side HTTP request to it, with no restriction on which internal hosts it's allowed to reach. Because the vulnerable server sits on the same internal network as the admin back-end, it can act as a pivot point — an attacker with no direct network access to `192.168.0.0/24` can still enumerate and interact with hosts on that range purely through the SSRF, since the vulnerable server does the actual network traversal on the attacker's behalf.

## Impact

An attacker can use the vulnerable server as an internal network scanner and proxy: first mapping out which internal hosts exist and what's running on them (via response differences), then directly interacting with any internal service found — in this case, performing an unauthorized administrative delete action on another back-end system entirely separate from the main application server.

## Remediation

- Restrict outbound requests from the stock-check feature to an explicit allowlist of known, required internal hosts and ports — never allow arbitrary internal IPs or ports.
- Segment the network so the public-facing application server has no route to internal admin interfaces unless explicitly necessary.
- Require proper authentication on internal admin interfaces regardless of the request's origin, rather than trusting requests based on network location alone.
- Rate-limit or monitor for patterns consistent with internal network scanning (many sequential requests to different internal IPs).

## Notes

- This lab is a natural extension of the previous "basic SSRF against the local server" lab — same vulnerable parameter and technique, but this time the target isn't a known address (`localhost`), so the attack needed an extra reconnaissance step: using Intruder to brute-force the last octet of the internal subnet and spot the outlier response (different status code and/or significantly different response length) that indicated a real service was listening.
- Filtering Intruder results by **response length** was more useful here than looking at every response individually — the real admin interface's HTML made its response noticeably larger than the generic connection-failure responses from empty IPs.
- Worth remembering as a general technique: when an SSRF target host is unknown but the port is likely fixed (as hinted in the lab description), sweep the host octet(s) with Intruder rather than trying to guess or brute-force the port too — narrowing the unknown variable first keeps the scan fast.
