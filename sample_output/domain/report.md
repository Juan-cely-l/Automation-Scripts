# Recon Report: scanme.nmap.org

- Mode: domain
- Timestamp: 2026-05-11T18:45:02

## Summary
| Tool | Status | Finding |
| --- | --- | --- |
| whois | ok | MarkMonitor, Inc. |
| dig | ok | 6 DNS records |
| curl | ok | 2 relevant headers |

## DNS Records
### A
- `45.33.32.156`
### MX
No records found.
### NS
- `ns1.linode.com.`
- `ns2.linode.com.`
- `ns3.linode.com.`
- `ns4.linode.com.`
- `ns5.linode.com.`
### TXT
- `"Try Nmap at: https://nmap.org/try/"`

## HTTP Headers
| Header | Value |
| --- | --- |
| Server | `Apache/2.4.7 (Ubuntu)` |
| X-Powered-By | `PHP/5.5.9-1ubuntu4.14` |

## Missing Security Headers
- Content-Security-Policy
- Strict-Transport-Security
- X-Frame-Options

## Tool Errors
No tool errors recorded.

## Ethical Note
Run this tool only against systems you own or are explicitly authorized to test.
