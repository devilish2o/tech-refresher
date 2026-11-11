# Lab: systems, networking and security

## Linux

Practise inspecting processes, systemd services, sockets, disk and memory pressure, permissions and logs. Useful commands include `ps`, `ss`, `systemctl`, `journalctl`, `df`, `du`, `free`, `vmstat` and `lsof`. Capture timestamps and compare before/after measurements.

Documentation: [systemd](https://www.freedesktop.org/software/systemd/man/latest/) · [Debian Administrator's Handbook](https://debian-handbook.info/)

## Windows / PowerShell

Inspect services, processes, scheduled tasks, event logs, TCP connections and DNS. Practise exporting a concise diagnostic bundle. Prefer PowerShell cmdlets and structured objects over fragile text parsing.

Documentation: [PowerShell documentation](https://learn.microsoft.com/powershell/) · [Windows troubleshooting](https://learn.microsoft.com/windows/)

## Networking

For a failed connection, work methodically: name resolution → route → TCP reachability → listener → TLS handshake → application response. Understand DNS records, ports, routing, NAT, proxies, firewall rules, certificates and SNI. Use packet capture only on systems and traffic you are authorised to inspect.

Documentation: [Cloudflare learning centre](https://www.cloudflare.com/learning/) · [Wireshark documentation](https://www.wireshark.org/docs/)

## Security

Refresh least privilege, identity, patching, secrets management, certificate lifecycle, secure defaults, audit logging and incident response. Practise threat modelling a simple service and identify trust boundaries.

Documentation: [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) · [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
