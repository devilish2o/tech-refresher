# Lesson 011 — Networking: diagnose DNS, TCP, TLS and certificate validation

**Date:** 2026-11-22  
**Domain:** Networking / TLS / certificates / troubleshooting  
**Duration:** 30 minutes  
**Environment:** Debian 12 on the user's own VPS; read-only outbound checks to the reserved public test hostname `example.com:443`  
**Version assumptions:** Debian 12 ships OpenSSL 3.0; commands use OpenSSL 3.x and a modern curl build. Check local versions because vendor builds can differ.

## Objective

Build a layered diagnosis for an HTTPS connection without weakening certificate validation. Distinguish name resolution, TCP reachability, TLS negotiation, certificate-chain/hostname verification and the HTTP response.

You will practise:

1. Confirming DNS resolution with the system resolver.
2. Performing one normal TLS handshake with explicit SNI and hostname verification.
3. Inspecting the leaf certificate's subject, issuer, validity and Subject Alternative Name (SAN).
4. Demonstrating that a hostname mismatch is rejected.
5. Separating a successful TLS handshake from an HTTP/application response.

**Safety boundary:** These are read-only checks against the reserved public test host `example.com`. They do not scan ports or modify services. Do not use `curl -k`, `--insecure`, disabled verification, or production hostnames as shortcuts. Do not paste private endpoint details or certificate material.

## 30-minute plan

| Time | Activity |
|---|---|
| 0–4 min | Confirm OpenSSL, curl and resolver tools |
| 4–8 min | Check DNS resolution |
| 8–15 min | Perform a verified TLS handshake and inspect the certificate |
| 15–20 min | Run an intentional hostname-mismatch negative test |
| 20–26 min | Compare TLS results with the HTTP response and classify failures |
| 26–30 min | Self-check and record the evidence |

## References

1. [OpenSSL 3.0 — s_client](https://docs.openssl.org/3.0/man1/openssl-s_client/) — SNI, certificate verification, hostname verification and diagnostic handshake options.
2. [OpenSSL 3.0 — x509](https://docs.openssl.org/3.0/man1/openssl-x509/) — inspect certificate subject, issuer, validity dates, SAN and expiry checks.

The `-servername` option sets TLS SNI. `-verify_return_error` enables certificate-chain verification and aborts on verification errors; `-verify_hostname` checks the expected hostname. The x509 command can inspect the certificate and test whether it expires within a specified number of seconds. <https://docs.openssl.org/3.0/man1/openssl-s_client/> <https://docs.openssl.org/3.0/man1/openssl-x509/>

## Practical exercise

### 1. Preflight — 4 minutes

SSH to your Debian 12 VPS. These checks do not change the host.

```bash
date -Is
openssl version
curl --version | head -n 1
command -v openssl curl getent
```

Expected:
- OpenSSL 3.x on the Debian 12 host.
- curl is available.
- getent is available as part of the system's name-service tooling.

If curl or OpenSSL is missing, do not install packages during this lesson. Use the tool that is present and record the limitation.

### 2. Check DNS — 4 minutes

Use the system resolver, not a hard-coded IP:

```bash
getent ahosts example.com
```

Expected: one or more IPv4/IPv6 address records.

Interpretation:
- No result: investigate resolver configuration and DNS reachability before TLS.
- An address is returned: name resolution succeeded, but TCP and TLS are not yet proven.

### 3. Perform a verified TLS handshake — 7 minutes

Run one normal HTTPS handshake:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -verify_hostname example.com \
  -verify_return_error \
  -brief </dev/null
```

Expected:
- A negotiated TLS protocol (normally TLS 1.2 or TLS 1.3).
- A negotiated cipher suite.
- Certificate verification succeeds.

The important flags:
- `-connect`: TCP destination.
- `-servername`: sends SNI so a virtual-hosted server can select the intended certificate.
- `-verify_hostname`: checks that the certificate is valid for the expected DNS name.
- `-verify_return_error`: makes verification failures fatal rather than allowing the diagnostic client to continue.

Do not remove verification flags to make a failing handshake appear successful.

### 4. Inspect the leaf certificate — 5 minutes

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -showcerts </dev/null 2>/dev/null |
openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

Read:
- **subject:** identity represented by the leaf certificate.
- **issuer:** certificate authority that issued it.
- **notBefore / notAfter:** validity window.
- **X509v3 Subject Alternative Name:** DNS names the certificate is valid for.

Important: `-showcerts` displays certificates sent by the peer. Seeing a certificate in that output does not, by itself, prove that the chain is trusted. The verified handshake in Step 3 performs the trust/hostname checks.

Optional expiry check — does the leaf certificate remain valid for more than 30 days?

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -showcerts </dev/null 2>/dev/null |
openssl x509 -noout -checkend 2592000
```

A successful exit means the leaf certificate does not expire within the next 2,592,000 seconds (30 days). This is a point-in-time check, not a renewal monitor.

### 5. Prove hostname validation rejects a mismatch — 5 minutes

This is an intentional negative test. The connection still goes to example.com and sends the correct SNI, but the client is told to verify against a deliberately incorrect expected hostname:

```bash
openssl s_client \
  -connect example.com:443 \
  -servername example.com \
  -verify_hostname deliberately-wrong.invalid \
  -verify_return_error \
  -brief </dev/null
```

Expected: certificate verification fails with a hostname mismatch. A non-zero exit status is expected for this negative test.

This demonstrates why SNI and hostname verification are separate:
- SNI tells the server which virtual host/certificate the client wants.
- Hostname verification tells the client which name the received certificate must cover.

### 6. Separate TLS from HTTP — 5 minutes

```bash
curl --head \
  --connect-timeout 5 \
  --max-time 15 \
  --output /dev/null \
  --write-out '\nHTTP=%{http_code} remote=%{remote_ip} verify=%{ssl_verify_result}\n' \
  https://example.com/
```

Expected: an HTTP status, remote IP and certificate verification result of `0`. The exact HTTP status and remote IP can vary with the remote service. Do not request `%{ssl_version}` from curl's `--write-out` here: the user's Debian 12 curl 7.88.1 build reports it as an unknown variable. The TLS protocol and cipher are already reported by the verified `openssl s_client -brief` check in Step 3. Keep TLS negotiation evidence separate from curl's HTTP and certificate-verification summary.

Interpretation:
- DNS succeeds, but TCP times out: investigate routing, firewall/security policy or remote reachability.
- TCP connects, but TLS fails before a certificate is presented: investigate protocol mismatch, proxy interference, SNI or the remote TLS listener.
- TLS reaches certificate verification but fails: inspect trust chain, validity dates, system clock and hostname/SAN.
- TLS verifies but HTTP returns 4xx/5xx: the TLS layer completed; investigate the HTTP application, reverse proxy, routing or backend.
- HTTP 200 is not proof that every application workflow is healthy; it only proves this request received that response.

On this Debian 12 host, curl reports version `7.88.1` with OpenSSL `3.0.22`. Its `--write-out` parser rejects `%{ssl_version}` (`curl: unknown --write-out variable: 'ssl_version'`), even though TLS itself works. This is a write-out-variable compatibility issue, not a TLS failure. Use Step 3's OpenSSL output for the negotiated TLS protocol.

## Failure-classification quick reference

| Symptom | First investigation |
|---|---|
| `getent` returns no address | Resolver configuration, DNS response and network path to DNS |
| Connection refused | Destination reachable, but no listener or an active reject at that address/port |
| Connection timeout | Routing, firewall/security group, packet loss or remote endpoint availability |
| `wrong version number` | Wrong service/port, plaintext endpoint, proxy or protocol mismatch |
| Hostname mismatch | SNI target, certificate SAN and the hostname the client actually verifies |
| Unable to get local issuer / self-signed in chain | Missing intermediate, wrong trust store or private CA not trusted by this client |
| Certificate expired / not yet valid | Certificate lifecycle and system clock |
| TLS verifies; HTTP 5xx | Application, reverse proxy or upstream backend—not the certificate handshake |

Do not infer a root cause from one error string alone. Correlate the client output, server-side logs and the exact endpoint/port involved.

## Expected outcome

You should be able to state separately whether:
- DNS resolved the test hostname.
- TCP connected to port 443.
- TLS negotiated and which protocol/cipher were selected.
- The certificate chain and hostname verified.
- The leaf certificate's SAN and validity window were appropriate.
- The HTTP request received a response.

No local service, firewall, certificate store, trust bundle or application configuration is changed.

## Quick self-check

1. What does SNI tell the server, and what does hostname verification tell the client?
2. Why is `-showcerts` not proof that the certificate chain is trusted?
3. Which OpenSSL option makes verification errors abort the handshake?
4. If TLS verification succeeds but curl reports HTTP 502, which layer should you investigate next?
5. Why is `curl -k` not an acceptable production workaround?

## Production / interview relevance

For MFT, API, reverse-proxy and service-to-service incidents, separate DNS, TCP, TLS and application layers. Check SNI, SAN, trust chain, validity and system time before changing certificates or trust stores. Never solve a certificate incident by disabling verification; establish which side presents the wrong identity or incomplete chain.

A concise interview answer is: **“I test resolution, connect to the intended endpoint, negotiate TLS with explicit SNI and hostname/chain verification, inspect the leaf and chain, then make a separate HTTP request. That tells me whether the failure is DNS, transport, TLS identity/trust, or the application.”**

## Completion checkpoint

Report only:
- **OpenSSL version:** major/minor.
- **DNS:** resolved / did not resolve.
- **TLS:** negotiated protocol and whether verification passed.
- **Certificate:** SAN matched the test hostname? yes/no; more than 30 days remaining? yes/no.
- **Negative test:** did the deliberately wrong hostname fail verification?
- **HTTP:** status code and verification result.
- **One takeaway:** why a successful TLS handshake and a successful HTTP response are different checks.

Do not paste private endpoints, internal IPs, certificate PEM data, keys, tokens or production logs.

## 🧸 For Dummies

Think of HTTPS like visiting an office:

1. **DNS** finds the office address.
2. **TCP** checks that you can reach the front door.
3. **SNI** tells the receptionist which company you want.
4. **The certificate** is the company's ID card.
5. **Hostname verification** checks that the ID card really belongs to the company you intended to visit.
6. **HTTP** is the conversation after the receptionist lets you in.

If the ID card is wrong, do not tell the receptionist to stop checking IDs. Find out why the wrong ID was presented. And even if you get inside, that does not mean the person you need is available to help. 🧸
