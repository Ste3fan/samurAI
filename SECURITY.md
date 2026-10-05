# Security policy

## Supported versions

| Version | Supported |
|---|---|
| 1.0 | yes |

## Reporting a vulnerability

Please **do not** open a public issue for security problems. Report them privately instead:

- through GitHub: **Security → Report a vulnerability** (private advisory) on this repository, or
- by e-mail to the maintainer, via [www.softway.ro](https://www.softway.ro).

Please include the affected component (core, XML, studio, web server), the version, steps to reproduce and the impact.
You will receive an answer as soon as possible. Once a fix is available, the advisory will be published with credit to
the reporter, unless you prefer otherwise.

## Scope and secure-by-default behaviour

- **Secrets** are referenced as `${VAR}` and never stored in configuration. Secrets are masked in logs, audit records
  and error messages.
- **Servers:**
  - REST, the dashboard and MCP listen on `127.0.0.1` by default;
  - REST requires API keys, the dashboard a token generated at each start;
  - a port already in use is refused.
- **XML:** XXE-safe parsing, file paths restricted to the application directory, classes only from allowed packages.
- **OS commands:** argument lists without a shell, an allow-list, a restricted environment and timeouts.
  A sub-orchestrator cannot allow more than its parent.
- **HTTP:** `HttpAgent` blocks private networks (SSRF) unless explicitly allowed.

Exposing the REST API or the dashboard publicly (`--host 0.0.0.0`) is your responsibility. Put them behind an HTTPS
reverse proxy and configure your own API keys.
