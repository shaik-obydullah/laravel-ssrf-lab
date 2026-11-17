# Laravel SSRF Lab (local only)

An intentionally vulnerable Laravel app for learning and demonstrating
Server-Side Request Forgery (SSRF). It is paired with a fake "internal-only"
metadata service so you can see what an attacker reaches when a server fetches
an arbitrary user-supplied URL.

> [!WARNING]
> This app is **deliberately vulnerable**. It accepts any URL and fetches it
> with no allow-list and no protection against loopback, private, or link-local
> addresses. Never deploy it, expose it beyond `127.0.0.1`, or copy the fetcher
> code into a real project.

## Status

| Phase | State | Description |
|---|---|---|
| Phase 1 — attack | **Live** | Vulnerable URL fetcher + fake internal service + [attack walkthrough](ATTACK.md) |
| Phase 2 — fix | **Not implemented** | Allow-list / DNS + IP validation / redirect handling. Patterns are previewed at the end of [ATTACK.md](ATTACK.md). |

The controller in `app/app/Http/Controllers/SsrfLabController.php` is the
intended target and is still vulnerable.

## Requirements

- Linux with `bash` and `curl`.
- No Docker, no system PHP, no database setup. A self-contained PHP 8.3
  runtime and a Composer PHAR are bundled under `runtime/`.
- The Laravel app and its `vendor/` directory are already installed under
  `app/`.

Both servers bind to loopback only: the app on `127.0.0.1:8082` and the fake
internal service on `127.0.0.1:9090`.

## Quick start

```bash
./start.sh
```

This starts two PHP built-in servers and prints their URLs:

| Service | URL |
|---|---|
| Vulnerable Laravel app | <http://127.0.0.1:8082/ssrf-lab> |
| Fake internal metadata service | <http://127.0.0.1:9090/> |

Open the app URL in a browser and submit a URL in the form. Stop everything
with:

```bash
./stop.sh
```

`start.sh` writes the two server PIDs to `.server.pid`; `stop.sh` reads that
file and terminates them. Starting again while the lab is running is a no-op.

## Verify it works

Hit the fake service directly to see the internal-only responses:

```bash
curl http://127.0.0.1:9090/internal/admin
```

```json
{
    "admin": true,
    "flag": "FLAG{ssrf-lab-internal-admin-reached}"
}
```

Then reach the same endpoint *through* the vulnerable app:

```bash
./exploit.sh 'http://127.0.0.1:9090/internal/admin'
```

```
Response
Status: 200
Content-Type: application/json
{
"admin": true,
"flag": "FLAG{ssrf-lab-internal-admin-reached}"
}
```

A second target simulates cloud instance metadata:

```bash
./exploit.sh 'http://127.0.0.1:9090/latest/meta-data/iam/security-credentials/lab-role'
# FLAG{ssrf-lab-metadata-credentials-read}
```

`exploit.sh` is just a convenience wrapper that URL-encodes the first argument
into `?url=` on the lab endpoint and strips the HTML from the response. You can
also paste the URL into the form in the browser. See [ATTACK.md](ATTACK.md) for
the full walkthrough and further ideas.

## Layout

| Path | Purpose |
|---|---|
| `README.md` | This file: overview, setup, verification |
| `ATTACK.md` | Phase 1 walkthrough and Phase 2 fix preview |
| `start.sh` / `stop.sh` | Start / stop both servers (PID file: `.server.pid`) |
| `exploit.sh` | Send a URL to the lab and print the response |
| `app/` | Laravel 13 / PHP 8.3 application (its own stock `README.md` lives here) |
| `app/app/Http/Controllers/SsrfLabController.php` | Vulnerable URL fetcher — the lab target |
| `app/routes/web.php` | Defines the named `ssrf-lab` route at `GET /ssrf-lab` |
| `app/resources/views/ssrf-lab.blade.php` | Form and response display |
| `mock-metadata/router.php` | Fake internal metadata / admin service on port 9090 |
| `runtime/` | Bundled PHP 8.3 runtime, `php.ini`, and Composer PHAR |

The fake service exposes these paths:

| Path | Simulates |
|---|---|
| `/` | Service index listing available endpoints |
| `/latest/meta-data/` | Metadata directory listing |
| `/latest/meta-data/iam/security-credentials/` | Role listing |
| `/latest/meta-data/iam/security-credentials/lab-role` | Fake IAM credentials (IMDSv1-style) |
| `/internal/admin` | Internal admin endpoint (not reachable from the internet) |

## Troubleshooting

- **"Address already in use"** — another process holds port 8082 or 9090, or a
  stale `.server.pid` points at a dead process. Run `./stop.sh`, check with
  `ss -ltnp | grep -E '8082|9090'`, and remove `.server.pid` if needed.
- **Blank page or errors in the browser** — start the lab with `./start.sh`
  from this directory so the bundled PHP runtime, `php.ini`, and paths resolve
  correctly. `display_errors` is enabled in `runtime/php.ini` for debugging.
- **Ports are fixed** — 8082 and 9090 are hard-coded in `start.sh`,
  `exploit.sh`, and the docs. Change them together if you need different ports.

## Development notes

The generated Laravel app's guidelines (`app/AGENTS.md`) were followed:
Artisan `make:` commands for generated files, relevant best-practices rules
read, and Pint run after PHP edits. Fetching an arbitrary user-supplied URL is
a **deliberate, documented deviation** from the HTTP-client/security rules —
it is the lab target, not an oversight.
