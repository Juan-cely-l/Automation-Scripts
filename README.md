# Automation-Scripts

## Sebastian Buitrago & Juan Esteban Cely

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

uv run python auth_analysis.py --input auth.log --output auth_results.json --threshold 10

uv run python log_analysis.py --input access.log --output web_results.json --report report.md

uv run python recon.py 127.0.0.1 --mode ip --output sample_output --verbose
```

`sample_output/` contains two real runs:

- **IP mode** (`sample_output/`): run against `127.0.0.1`. `nmap` and `dig` succeeded; `whois` was not installed and that error is preserved in `audit.log` and `results.json`.
- **Domain mode** (`sample_output/domain/`): run against `scanme.nmap.org` (Nmap's official scan-me host, publicly listed for this purpose). All tools succeeded; three missing security headers were found and reported.

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

### Part 1 — False negatives at high concurrency

At very high concurrency (e.g. `--rate 2000`), the operating system can exhaust its
per-process file-descriptor limit and its ephemeral port range before all scan tasks
complete. When the kernel cannot allocate a new socket it raises `OSError` immediately,
which the scanner catches and interprets as "port closed." The port was never actually
tested — the connection attempt failed locally, not at the target.

This means "not detected" and "closed" are categorically different claims. A scanner
reports what was observable within its resource and timing constraints. Any tool,
including nmap, can produce false negatives when configured too aggressively, when a
stateful firewall silently drops packets, when the target itself is rate-limiting
incoming SYNs, or when a per-host timeout is shorter than the network round-trip time.
Scan results are evidence with a confidence level attached to them, not ground truth.
In practice this means always interpreting results conservatively: a port that did not
respond should be re-tested at a lower rate before being reported as closed.

### Part 2 — Service version banners and attacker intelligence

A banner like `Apache httpd 2.4.54 (Ubuntu)` immediately maps to a specific row in
public vulnerability databases such as the NVD and Exploit-DB. An attacker can query
those databases in seconds and learn which CVEs are unpatched on that exact version,
whether a public exploit exists, and whether the default configuration is vulnerable.
The version string converts a generic "there is an HTTP server" observation into an
actionable attack plan without sending a single additional probe.

A server that returns only `Server: Apache`, or no `Server` header at all, forces the
attacker to perform active version fingerprinting — sending probe requests and
correlating responses against a signature database. That process is slower, generates
more traffic, and is easier for a defender to detect and block. Hiding the version is
not a fix for the underlying vulnerability, but it raises the cost of exploitation and
reduces the value of passive reconnaissance against that host.

### Part 3 — Limits of a global 3-sigma baseline for web traffic

The 3-sigma rule assumes the data is approximately normally distributed around a single
mean. Web traffic almost never is: servers that handle business users see several
thousand requests per hour during the working day and a few dozen overnight. A global
baseline computed across all 24 hours merges those two very different populations,
which inflates the standard deviation artificially. The resulting threshold is too
permissive during peak hours (a genuine attack spike blends into normal load) and too
sensitive at night (a modest increase over the low overnight baseline triggers a
false positive).

A more robust approach segments the baseline by time stratum before computing
statistics. The simplest version compares each hour only against the same clock-hour
on previous days: the 3:00 AM reading on Tuesday is compared against 3:00 AM on
Monday, Sunday, Saturday, and so on. This normalises the daily cycle before measuring
deviation, so the threshold adapts to what is actually expected at that time rather
than averaging over the entire day. More sophisticated approaches build separate
models for weekdays vs. weekends, or use time-series decomposition to remove the
seasonal component before applying anomaly detection.

### Part 4 — Active vs. passive reconnaissance

Active reconnaissance (this tool, nmap) sends packets directly from your IP to the
target. Every probe leaves a trace: DNS resolvers log the query source, web servers
log the `curl -I` request, and network monitoring infrastructure at the target sees
your IP's SYNs arrive. A defender with a SIEM or IDS can correlate those events,
identify the scan pattern within seconds, and block or alert on your IP. Active recon
produces current, authoritative data — you see what is reachable right now — but it
is inherently visible.

Passive reconnaissance (Shodan) queries a third-party database that was built from
scans run months or years ago by Shodan's own infrastructure. Your IP never touches
the target. There is no packet to log, no connection to detect, no alert to trigger.
From a defender's perspective it is undetectable because nothing happens on their
network. The trade-off is staleness and incompleteness: Shodan may not have scanned
a given host recently, may have missed hosts behind NAT or firewalls, and does not
reflect configuration changes made since the last crawl.

In a real engagement both are used sequentially. Passive recon first: query Shodan
and public DNS records to build a target map without alerting anyone. Active recon
second: confirm reachability and fill gaps where Shodan's data is stale or absent,
only after the engagement scope and rules of engagement are confirmed in writing.
For particularly sensitive targets (ICS/SCADA, production financial systems) passive-
only recon may be the appropriate choice even during an authorised assessment.

## Ethics

Run these scripts only against systems you own or are explicitly authorized to test. Avoid broad scans, public IP ranges, and high scan rates unless they are inside an approved scope.
