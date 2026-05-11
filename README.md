# automation-scripts

## By: Sebastian Buitrago & Juan Esteban Cely

Python 3.12 scripts for SPTI workshop 15 - Automation. The project automates TCP scanning, nmap XML parsing, log analysis, anomaly detection, and integrated reconnaissance with auditable output.

## Requirements

- Python 3.12
- uv
- nmap
- whois
- dig
- curl
- ssh-keyscan

## Installation

```bash
uv sync
```

## Validation

```bash
uv run ruff check .
uv run pytest
```

## Usage

```bash
uv run python scanner.py 127.0.0.1 --ports 1-1024 --rate 200 --timeout 0.5
uv run python scanner.py 127.0.0.1 --ports 22,80,443 --output scanner_results.json

uv run python parse_scan.py --input scan.xml --output hosts.json --ssh-timeout 5

uv run python auth_analysis.py --input auth.log --output auth_results.json

uv run python log_analysis.py --input access.log --output web_results.json --report report.md

uv run python recon.py 127.0.0.1 --mode ip --output sample_output --verbose
```

`sample_output/` was generated safely against `127.0.0.1`. In this environment `nmap` and `dig` succeeded, while `whois` was not installed; that real tool error is preserved in `sample_output/audit.log` and `sample_output/results.json`.

## Scripts

- `scanner.py`: concurrent TCP connect scanner using `asyncio` and `asyncio.Semaphore`. It accepts port ranges, lists, and mixed inputs such as `22,80,100-110`, then writes structured JSON.
- `parse_scan.py`: reads nmap XML with `xml.etree.ElementTree`, extracts live hosts and open services, and enriches SSH hosts with `ssh-keyscan`. If `ssh-keyscan` is missing, times out, or returns no key, `ssh_host_key_type` is set to `null`.
- `auth_analysis.py`: parses Linux/sshd authentication logs, reports brute-force source IPs, targeted users, and the failed-to-successful login ratio. If there are no successful logins, the ratio is `null`.
- `log_analysis.py`: parses Apache/Nginx access logs, flags SQL injection, path traversal, XSS, command injection, WordPress probing, top IPs, status codes, and 3-sigma hourly anomalies.
- `recon.py`: runs domain or IP reconnaissance and always creates `audit.log`, `results.json`, and `report.md`. Tool failures are isolated and recorded instead of stopping the full run.

## Design Decisions

- `asyncio` plus `Semaphore` limits scan concurrency without creating hundreds of OS threads.
- nmap output is parsed as XML, not human text, so host, port, state, service, and version fields remain structured.
- External tools run through `subprocess` with timeouts, captured stdout/stderr, and independent error handling.
- `audit.log` is mandatory in `recon.py` because security automation needs a timestamped record of actions.
- JSON is used for machine-readable output, while Markdown reports are generated for human review.

## Concept Questions

- High concurrency can cause false negatives because local socket limits, packet loss, target rate limits, or short timeouts can make open ports fail to complete a connection. "Not detected" is not the same as "closed"; scanner output should be treated as evidence with limits, including nmap output.
- Service banners and versions help attackers map software to known CVEs and exploit paths. `Apache httpd 2.4.54` gives more actionable intelligence than a server that hides its version, although hidden banners do not prove the service is safe.
- A single global 3-sigma baseline can be noisy when traffic has daily cycles. A better approach is to compare each hour against the same hour on previous days or maintain separate baselines for business hours, nights, and weekends.
- Active reconnaissance sends packets to the target and is visible to network monitoring. Passive reconnaissance, such as querying Shodan, uses existing third-party observations and is harder for the target defender to detect, but it may be stale or incomplete.

## Ethics

Run these scripts only against systems you own or are explicitly authorized to test. Avoid broad scans, public IP ranges, and high scan rates unless they are inside an approved scope.
