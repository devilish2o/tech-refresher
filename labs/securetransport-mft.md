# Lab: Axway SecureTransport / managed file transfer

## Goal

Design a secure, supportable file-transfer flow and practise diagnosing common failures. Product screens and exact configuration names vary by SecureTransport version; use the documentation matching the installed release.

## Scenario

A partner sends a daily encrypted file. The platform receives it, validates naming and integrity, routes it to an internal landing zone, and returns an acknowledgement.

## Design checklist

- Partner identity and least-privilege access.
- Supported protocol and endpoint configuration.
- SSH host keys or TLS certificate validation as appropriate.
- Encryption in transit and, where required, at rest.
- Landing-zone permissions, retention and quarantine.
- File naming, size limits, checksum and duplicate detection.
- Acknowledgement semantics and correlation IDs.
- Retry policy, alerting, audit trail and reconciliation.

## Failure exercises

Explain how you would investigate: authentication failure, expired certificate, host-key change, network timeout, partial transfer, duplicate file and successful transfer with missing acknowledgement.

## Safety

Use test partners, dummy keys and synthetic files only. Never disable certificate or host-key verification to make a test pass.

## References

- [Axway documentation portal](https://docs.axway.com/)
