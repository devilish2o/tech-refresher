# Lab: resilient Python operations automation

## Goal

Build a small command-line tool that checks a list of endpoints or files and produces a useful report.

## 30-minute starter

- Accept input path and timeout as command-line arguments.
- Validate inputs and return a meaningful non-zero exit code on failure.
- Use the standard `logging` module; send diagnostics to stderr.
- Handle expected exceptions explicitly.
- Write structured JSON output to a separate file.

## Extension

Add bounded retries with backoff, request timeouts, idempotency, configuration precedence, unit tests and a dry-run mode. Never retry non-idempotent operations blindly.

## Self-check

Can an operator distinguish a target failure from a script failure? Are errors actionable? Does the tool behave predictably when run twice?

## Documentation

- [Python tutorial](https://docs.python.org/3/tutorial/)
- [argparse](https://docs.python.org/3/library/argparse.html)
- [Logging](https://docs.python.org/3/library/logging.html)
- [unittest](https://docs.python.org/3/library/unittest.html)
