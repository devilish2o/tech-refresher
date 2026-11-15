# Lesson 004 — Build a resilient Python HTTP health checker

**Domain:** Python automation / HTTP / operational reliability  
**Duration:** 30 minutes  
**Level:** Experienced DevOps / infrastructure engineer  
**Environment:** PEACEKEEPER, Debian 13, Bash, Vim  
**Version assumption:** Python 3.11 or later. The exercise uses only the Python standard library; no pip installation or virtual environment is required.

> **Safety:** Run this only against the local test server created below. The script performs HTTP GET requests only. Do not add credentials, tokens or customer URLs. It does not disable TLS certificate validation.

## Objective

Build a small command-line health checker that behaves predictably in automation:

1. Accept an endpoint and bounded retry/timeout settings.
2. Distinguish a successful 2xx response, a permanent client error such as 404, a retryable server error, and a connection failure.
3. Apply bounded exponential backoff only to transient failures.
4. Write timestamped diagnostics to stderr and a machine-readable JSON result to stdout.
5. Return exit code 0 for healthy and 1 for unhealthy.

The operational point: **a retry is a policy, not a reflex**. Repeating a 404 will not fix it; retrying a temporary 503 or connection failure may help, but retries must have a strict upper bound.

## 1. Check the local Python environment (3 minutes)

Open a terminal on PEACEKEEPER and run:

\`\`\`bash
python3 --version
command -v python3
command -v vim
\`\`\`

Expected: Python 3.11+ and executable paths for Python and Vim. No extra packages are needed.

Create the lab directories:

\`\`\`bash
mkdir -p ~/labs/tech-refresher/lesson-004-python-healthcheck/www
cd ~/labs/tech-refresher/lesson-004-python-healthcheck
pwd
\`\`\`

The final command should show the lesson directory under your home folder. This is a disposable directory; nothing is installed as a service and no system configuration is changed.

## 2. Create a local test endpoint (3 minutes)

Create a harmless page for the local HTTP server:

\`\`\`bash
printf '%s\n' '<html><body>Disposable health-check lab</body></html>' > www/index.html
cat www/index.html
\`\`\`

Keep this terminal open. Start the test server, bound to loopback only:

\`\`\`bash
python3 -m http.server 8765 --bind 127.0.0.1 --directory "$HOME/labs/tech-refresher/lesson-004-python-healthcheck/www"
\`\`\`

Expected: a message that the server is serving on 127.0.0.1 port 8765.

This server is deliberately simple. Its root path returns HTTP 200, while a path that does not exist returns HTTP 404. It is not a production application or a real application-health endpoint.

Open a **second terminal** for the remaining steps. Enter the lab directory:

\`\`\`bash
cd ~/labs/tech-refresher/lesson-004-python-healthcheck
\`\`\`

## 3. Create the health-check script (11 minutes)

Create the file:

\`\`\`bash
vim healthcheck.py
\`\`\`

Paste this complete script:

\`\`\`python
#!/usr/bin/env python3
import argparse
import json
import logging
import sys
import time
from urllib.error import HTTPError, URLError
from urllib.parse import urlsplit
from urllib.request import Request, urlopen

RETRYABLE_HTTP = {429, 500, 502, 503, 504}


def parse_args():
    parser = argparse.ArgumentParser(description="Bounded HTTP(S) health check")
    parser.add_argument(
        "--url", required=True,
        help="HTTP(S) endpoint; do not include credentials",
    )
    parser.add_argument(
        "--attempts", type=int, default=3,
        help="Maximum attempts (1-5; default: 3)",
    )
    parser.add_argument(
        "--timeout", type=float, default=3.0,
        help="Per-request timeout in seconds (default: 3)",
    )
    parser.add_argument(
        "--delay", type=float, default=1.0,
        help="Initial retry delay in seconds (default: 1)",
    )
    args = parser.parse_args()

    try:
        parsed = urlsplit(args.url)
        _ = parsed.port
    except ValueError as exc:
        parser.error(f"invalid URL: {exc}")

    if parsed.scheme not in {"http", "https"} or not parsed.hostname:
        parser.error("--url must be an absolute http:// or https:// URL")
    if parsed.username is not None or parsed.password is not None:
        parser.error("do not put credentials in the URL")
    if not 1 <= args.attempts <= 5:
        parser.error("--attempts must be between 1 and 5")
    if not 0 < args.timeout <= 30:
        parser.error("--timeout must be greater than 0 and no more than 30 seconds")
    if not 0 <= args.delay <= 10:
        parser.error("--delay must be between 0 and 10 seconds")

    return args


def main():
    args = parse_args()
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(levelname)s %(message)s",
        datefmt="%Y-%m-%dT%H:%M:%S",
    )

    started = time.monotonic()
    status = None
    last_error = None
    ok = False
    attempts_used = 0

    for attempt in range(1, args.attempts + 1):
        attempts_used = attempt
        retryable = False

        try:
            request = Request(
                args.url,
                headers={
                    "User-Agent": "devops-healthcheck/1.0",
                    "Accept": "*/*",
                },
                method="GET",
            )
            with urlopen(request, timeout=args.timeout) as response:
                status = response.status

            if 200 <= status < 300:
                ok = True
                logging.info(
                    "healthy: HTTP %s (attempt %d/%d)",
                    status, attempt, args.attempts,
                )
                break

            last_error = f"HTTP {status}"
            retryable = status in RETRYABLE_HTTP

        except HTTPError as exc:
            status = exc.code
            last_error = f"HTTP {exc.code}"
            retryable = exc.code in RETRYABLE_HTTP
            exc.close()

        except (URLError, OSError) as exc:
            status = None
            reason = getattr(exc, "reason", exc)
            last_error = f"{type(reason).__name__}: {reason}"
            retryable = True

        if retryable and attempt < args.attempts:
            wait_seconds = args.delay * (2 ** (attempt - 1))
            logging.warning(
                "attempt %d/%d failed (%s); retrying in %.1f seconds",
                attempt, args.attempts, last_error, wait_seconds,
            )
            time.sleep(wait_seconds)
        else:
            logging.error(
                "unhealthy after %d attempt(s): %s",
                attempt, last_error,
            )
            break

    result = {
        "ok": ok,
        "url": args.url,
        "status": status,
        "attempts": attempts_used,
        "elapsed_ms": round((time.monotonic() - started) * 1000, 1),
        "error": last_error,
    }
    print(json.dumps(result, indent=2))
    return 0 if ok else 1


if __name__ == "__main__":
    sys.exit(main())
\`\`\`

Save and quit Vim with :wq.

Make it executable and run Python's syntax check:

\`\`\`bash
chmod 750 healthcheck.py
python3 -m py_compile healthcheck.py
\`\`\`

Expected: no output from py_compile and exit code 0. It creates a local __pycache__ directory; that is normal.

### What to notice in the implementation

- **argparse** defines a predictable CLI and validates the input ranges.
- **urlopen(..., timeout=...)** prevents a single request from waiting indefinitely.
- **HTTPError** is caught before **URLError**, because HTTPError is a subclass of URLError.
- Only 429 and selected 5xx responses are retryable. A 404 is treated as a permanent failure.
- Connection-level URLError/OSError failures are retried, but only up to the configured attempt limit.
- **time.monotonic()** measures elapsed time without depending on wall-clock adjustments.
- **logging** writes diagnostics to stderr; JSON is written to stdout. This separation lets a pipeline capture JSON without mixing it with log messages.
- The process exit code is suitable for Jenkins, shell scripts and schedulers.

## 4. Test healthy and unhealthy HTTP responses (6 minutes)

First, test the local root endpoint:

\`\`\`bash
python3 healthcheck.py --url http://127.0.0.1:8765/
echo "exit=$?"
\`\`\`

Expected:
- JSON contains "ok": true, "status": 200 and "attempts": 1.
- The final line reports exit=0.

Now test a path that does not exist:

\`\`\`bash
python3 healthcheck.py --url http://127.0.0.1:8765/not-found
echo "exit=$?"
\`\`\`

Expected:
- JSON contains "ok": false, "status": 404, "attempts": 1 and "error": "HTTP 404".
- The final line reports exit=1.
- There is no retry: repeating a 404 is not useful.

## 5. Test a connection failure and bounded retries (4 minutes)

Port 8766 should not have a listener. Test it with two attempts and a short delay:

\`\`\`bash
python3 healthcheck.py \
  --url http://127.0.0.1:8766/ \
  --attempts 2 \
  --timeout 1 \
  --delay 0.2
echo "exit=$?"
\`\`\`

Expected:
- A warning for attempt 1, followed by a retry after 0.2 seconds.
- A final error after attempt 2.
- JSON contains "ok": false, "status": null and "attempts": 2.
- Exit code is 1.

The retry delay is exponential: with an initial delay of 0.2 seconds, the next delay would be 0.4 seconds. The attempt cap prevents an endless retry loop.

Stop the local HTTP server in the first terminal with Ctrl+C. This stops only the disposable test server.

## 6. Quick self-check (3 minutes)

1. Why is HTTPError handled before URLError?
2. Why should a 404 not normally be retried?
3. What prevents a slow endpoint from hanging the check indefinitely?
4. Why use time.monotonic() for elapsed time?
5. Why send logs to stderr and JSON to stdout?
6. Why must retry count and delay be bounded in a CI/CD or scheduler job?

**Completion checkpoint:** report:
- Python version;
- whether py_compile passed;
- observed result and exit code for HTTP 200;
- observed result and exit code for HTTP 404;
- observed result and exit code for the refused connection;
- one improvement you would make before using this in a production monitoring framework;
- confidence from 1–5.

Do not paste private endpoints, credentials, tokens or sensitive logs. Do not mark the lesson complete until you have run the checks.

## Production / interview relevance

This is the foundation of a useful operational probe or deployment gate. In a real environment, define health semantics with the application team, use HTTPS with normal certificate validation, set timeouts against the service's latency objective, and keep the total retry budget within the caller's job/SLA timeout. Avoid retry storms: retries across many parallel jobs can amplify an outage. For a production-grade monitor, also consider response-body validation, structured metrics, correlation IDs, proxy behavior, rate limits and an explicit distinction between liveness and readiness.

In an interview, be ready to explain why you retry selected transient failures but fail fast on permanent client errors, and how the exit code makes the script usable by Jenkins or Control-M.

## Official documentation

- [Python 3.13 — urllib.request.urlopen()](https://docs.python.org/3.13/library/urllib.request.html#urllib.request.urlopen) — request handling and the per-request timeout.
- [Python 3.13 — urllib.error](https://docs.python.org/3.13/library/urllib.error.html) — the HTTPError and URLError exception hierarchy.

## 30-minute plan

| Time | Activity |
|---:|---|
| 0–3 min | Check Python and create the disposable lab directory |
| 3–6 min | Start the loopback-only test HTTP server |
| 6–17 min | Create the CLI health checker and syntax-check it |
| 17–23 min | Test HTTP 200 and 404, including exit codes |
| 23–27 min | Test connection refusal and bounded retries |
| 27–30 min | Review retry policy and record the checkpoint |
| **Total** | **30 minutes** |
