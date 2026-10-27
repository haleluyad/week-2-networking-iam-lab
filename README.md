# Week 2 — Networking & IAM Fundamentals

**Name:**  Haleluya Desalegn 
**Date:** 2026-10-27

---

## Task 1 — My Network Information

- **Operating system:** Windows
- **Local IPv4 address:** 10.154.4.201
- **Default gateway:** 10.154.4.128
- **Private or public?** Private — the address starts with `10.`, which is the 10.0.0.0/8 private range (RFC 1918).

---

## Task 2 — Subnetting 10.0.0.0/24 into 4 equal subnets

**Step 1 — Bits to borrow:** Need 4 subnets → 2ⁿ = 4 → n = 2 bits
**Step 2 — New prefix:** 24 + 2 = /26
**Step 3 — Host bits left:** 32 − 26 = 6
**Step 4 — Addresses per subnet:** 2⁶ = 64
**Step 5 — Block size 64:** subnets start at .0, .64, .128, .192

| # | CIDR | Network | Full range | Usable (traditional) | Broadcast | AWS usable |
|---|---|---|---|---|---|---|
| 1 | 10.0.0.0/26 | 10.0.0.0 | 10.0.0.0 – 10.0.0.63 | 10.0.0.1 – 10.0.0.62 | 10.0.0.63 | 59 |
| 2 | 10.0.0.64/26 | 10.0.0.64 | 10.0.0.64 – 10.0.0.127 | 10.0.0.65 – 10.0.0.126 | 10.0.0.127 | 59 |
| 3 | 10.0.0.128/26 | 10.0.0.128 | 10.0.0.128 – 10.0.0.191 | 10.0.0.129 – 10.0.0.190 | 10.0.0.191 | 59 |
| 4 | 10.0.0.192/26 | 10.0.0.192 | 10.0.0.192 – 10.0.0.255 | 10.0.0.193 – 10.0.0.254 | 10.0.0.255 | 59 |

**Conventional rule:** 64 − 2 (network + broadcast) = 62 usable per subnet.
**AWS rule:** AWS reserves 5 addresses per subnet, so 64 − 5 = 59 usable per subnet.

---

## Task 3 — DNS Lookup

- **Domain tested:** `example.com`
- **DNS resolution succeeded:** Yes
- **IP addresses returned:**
  - IPv6: `2606:4700:10::ac42:93f3`, `2606:4700:10::6814:179a`
  - IPv4: `104.20.23.154`, `172.66.147.243`
- **Resolver used:** `10.154.4.128` (my router / campus network)
- **What DNS does (in my own words):**
  DNS is the internet's phonebook. It converts a human-readable domain name
  (like `example.com`) into one or more IP addresses that a computer can
  connect to. Without DNS, I'd have to remember raw numbers for every website.
  The "non-authoritative" note in the output just means the answer came from
  a cache — which is normal and still correct.

---

## Task 4 — HTTPS Inspection

- **Loaded in browser:** Yes
- **HTTP status code (from curl):** `200 OK`
- **What HTTPS adds to HTTP:**
  HTTPS adds TLS on top of HTTP. This provides three things:
  (1) **Encryption** — data can't be read in transit.
  (2) **Server authentication** — the certificate proves the server is who it claims to be.
  (3) **Integrity** — data can't be silently modified in transit.

- **Why DNS resolution alone does not guarantee the site loads:**
  DNS only converts the domain name into an IP address. After that, the browser
  still has to (a) open a TCP/QUIC connection, (b) complete a TLS handshake,
  and (c) receive a valid HTTP response. Any of those steps can fail —
  connection timeout, TLS certificate error, or an HTTP error code — even
  when DNS succeeds.

---

## Task 5 — HTTP Status Codes

| Code | Meaning | Example situation |
|---|---|---|
| 200 | OK — request succeeded | Page loaded successfully |
| 301 | Moved Permanently (redirect) | Typing http://example.com redirects to https://example.com |
| 403 | Forbidden — server understood but refuses | Logged-in user tries to visit /admin without admin rights |
| 404 | Not Found | URL has a typo, or the page was deleted |
| 500 | Internal Server Error | Backend code crashed, page can't render |

**Difference between failure types:**
- **DNS failure:** the name could not be resolved to an IP. The request never leaves your machine.
- **Connection timeout:** DNS worked, but no reply came back before the time limit — the host is down, or a firewall is blocking the port.
- **TLS certificate error:** TCP connected, but the certificate failed validation (expired, wrong hostname, untrusted CA).
- **HTTP error response:** the server successfully answered and returned an error status code (4xx / 5xx).

---

## Task 6 — IAM Policy

See `s3-read-policy.json`.

1. The **`Action`** field specifies the permitted action.
2. The **`Resource`** field specifies the resources.
3. **No.** `s3:GetObject` only reads objects you already know the name of. Listing requires a separate action, `s3:ListBucket`, which is applied to the bucket ARN (not `/*`).
4. A narrow permission follows the **principle of least privilege**. If credentials leak, an attacker can only perform the exact allowed action — not delete data, change permissions, or take over the bucket. `s3:*` would allow all of that.
5. AWS access keys committed to GitHub are publicly visible. Automated bots scan GitHub constantly; leaked keys are abused within minutes to spin up expensive resources or steal data. Never commit keys, passwords, or tokens.

---

## Task 7 — Troubleshooting

1. **Domain does not resolve** → **DNS.** The name → IP step failed, so nothing downstream can run.
2. **Resolves but times out** → **Network connectivity.** DNS succeeded; the endpoint is unreachable (host down, firewall blocking, wrong port).
3. **TLS certificate warning** → **TLS.** TCP connected; certificate validation failed.
4. **HTTP 404** → **HTTP/application.** The server responded; the requested resource doesn't exist.
5. **EC2 reaches S3 but AccessDenied** → **IAM/S3 authorization.** Network is fine; the role's policy or the bucket policy is denying the read.
