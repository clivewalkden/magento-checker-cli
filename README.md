# Magento System Checker

A command-line tool that verifies a Magento 2 project meets required infrastructure and security criteria. Checks are modular and easy to extend — adding a new check is a matter of implementing one interface and calling `Register()` in an `init()` function.

## Features

| Check | Target | What it verifies |
|-------|--------|-----------------|
| `htauth` | Staging domain | HTTP Basic Auth is enabled (site returns 401) |
| `mailhog` | `mailhog.{staging}` | A MailHog instance is reachable at the expected URL |
| `cloudflare` | Production domain | The site is proxied through Cloudflare (CF-Ray / Server headers) |
| `smtp` | Local repository | Staging `env.php` has SMTP enabled, a host configured, and developer mode checked |

---

## Installation

### Pre-built binary

Download the latest release from the [Releases](https://github.com/clivewalkden/magentoSystemChecker/releases) page, then move the binary onto your `PATH`:

```bash
# macOS (Apple Silicon)
curl -L https://github.com/clivewalkden/magentoSystemChecker/releases/latest/download/magento-checker_<version>_darwin_arm64.tar.gz | tar xz
mv magento-checker /usr/local/bin/

# macOS (Intel)
curl -L https://github.com/clivewalkden/magentoSystemChecker/releases/latest/download/magento-checker_<version>_darwin_amd64.tar.gz | tar xz
mv magento-checker /usr/local/bin/

# Linux (amd64)
curl -L https://github.com/clivewalkden/magentoSystemChecker/releases/latest/download/magento-checker_<version>_linux_amd64.tar.gz | tar xz
mv magento-checker /usr/local/bin/
```

### Build from source

Requires Go 1.26+ and [svu](https://github.com/caarlos0/svu) (for version injection via `make build`).

```bash
git clone https://github.com/clivewalkden/magentoSystemChecker
cd magentoSystemChecker
make build          # → bin/magento-checker
```

---

## Configuration

The tool reads configuration from a YAML file. By default it looks for `magento-checker.yaml` in the current directory, then `$HOME/.config/magento-checker/magento-checker.yaml`. Pass a custom path with `--config`.

Copy the example config and fill in your values:

```bash
cp config.example.yaml magento-checker.yaml
```

**`magento-checker.yaml`**

```yaml
# Production domain (without scheme)
production_domain: example.com

# Staging domain (without scheme)
staging_domain: staging.example.com

# Absolute or relative path to the locally cloned Magento 2 repository
repo_path: /path/to/magento-repo
```

CLI flags override config file values:

| Flag | Config key | Description |
|------|-----------|-------------|
| `--production` | `production_domain` | Production domain (no scheme) |
| `--staging` | `staging_domain` | Staging domain (no scheme) |
| `--repo` | `repo_path` | Path to local Magento 2 repository |

All config keys are also readable from environment variables with the prefix `MAGENTO_CHECKER_`:

```bash
export MAGENTO_CHECKER_PRODUCTION_DOMAIN=example.com
export MAGENTO_CHECKER_STAGING_DOMAIN=staging.example.com
export MAGENTO_CHECKER_REPO_PATH=/path/to/repo
magento-checker check
```

---

## Usage

### Run all checks

```bash
magento-checker check
```

Reads `magento-checker.yaml` from the current directory.

```bash
magento-checker check --config /path/to/project.yaml
```

### Run selected checks

```bash
magento-checker check --checks htauth,cloudflare
```

Available check names: `htauth`, `mailhog`, `cloudflare`, `smtp`

### Override config values via flags

```bash
magento-checker check --production example.com --staging staging.example.com --repo ./repo
```

### JSON output

```bash
magento-checker check --output json
```

```json
{
  "production_domain": "example.com",
  "staging_domain": "staging.example.com",
  "checks": ["htauth", "mailhog", "cloudflare", "smtp"],
  "results": {
    "htauth":     { "status": "PASS", "detail": "HTTP 401 received — WWW-Authenticate: Basic realm=\"Restricted\"" },
    "mailhog":    { "status": "PASS", "detail": "MailHog detected at http://mailhog.staging.example.com" },
    "cloudflare": { "status": "PASS", "detail": "CF-Ray: 7a1b2c3d4e5f-LHR" },
    "smtp":       { "status": "PASS", "detail": "host: smtp.mailhog.staging.example.com | developer mode: off" }
  }
}
```

### Table output (default)

```
  Production : example.com
  Staging    : staging.example.com

  CHECK                        STATUS   DETAIL
  ─────────────────────────   ──────   ──────
  Staging HTAuth (HTTP 401)   ✓ PASS   HTTP 401 received — WWW-Authenticate: Basic realm="Restricted"
  Mailhog instance (staging)  ✓ PASS   MailHog detected at http://mailhog.staging.example.com
  Cloudflare proxy (prod)     ✓ PASS   CF-Ray: 7a1b2c3d4e5f-LHR
  SMTP mail config (staging)  ✓ PASS   host: smtp.mailhog.staging.example.com | developer mode: off

  Result: 4/4 checks passed
```

### Version

```bash
magento-checker --version
```

---

## Check reference

### `htauth` — Staging HTAuth

Performs an HTTP GET to the staging domain and expects an HTTP `401 Unauthorized` response. HTTPS is tried first; if it fails (e.g. self-signed certificate), HTTP is used as a fallback. TLS certificate verification is intentionally skipped on the staging client because self-signed certificates are common.

| Status | Meaning |
|--------|---------|
| `PASS` | HTTP 401 received |
| `FAIL` | Any other status code, or the request failed entirely |
| `SKIP` | No staging domain configured |

---

### `mailhog` — MailHog instance

Performs an HTTP GET to `http://mailhog.{staging_domain}` and checks that the response body contains the string `MailHog` (present in the web UI's HTML title).

| Status | Meaning |
|--------|---------|
| `PASS` | MailHog UI detected at the expected URL |
| `WARN` | URL is reachable but response doesn't look like MailHog |
| `FAIL` | URL unreachable or non-200 response |
| `SKIP` | No staging domain configured |

---

### `cloudflare` — Cloudflare proxy

Performs an HTTPS GET to the production domain and looks for Cloudflare-specific response headers:

- `CF-Ray` — present on all Cloudflare-proxied responses
- `Server: cloudflare` — fallback indicator when `CF-Ray` is absent

| Status | Meaning |
|--------|---------|
| `PASS` | At least one Cloudflare header detected |
| `FAIL` | No Cloudflare headers found, or request failed |
| `SKIP` | No production domain configured |

---

### `smtp` — SMTP config in staging env.php

Reads `{repo_path}/environments/staging/app/etc/env.php` and parses it as a PHP array. It then checks the Magento system configuration stored within the file:

1. `system.default.smtp.general.enabled` must equal `"1"` (SMTP module enabled)
2. `system.default.smtp.configuration_option.host` must be non-empty (host configured)
3. `system.default.smtp.developer.developer_mode` is inspected — if `"1"`, a warning is emitted

| Status | Meaning |
|--------|---------|
| `PASS` | SMTP module enabled, host configured, developer mode off (or not set) |
| `WARN` | SMTP is configured correctly but developer mode is ON |
| `FAIL` | Module disabled, no host set, file not found, or parse error |
| `SKIP` | No repository path configured |

---

## Adding a new check

1. Create a new file in `internal/checks/`, e.g. `internal/checks/mycheck.go`
2. Implement the `CheckRunner` interface:

```go
package checks

import "context"

type myChecker struct{}

func (myChecker) Name() string        { return "mycheck" }
func (myChecker) Description() string { return "My custom check" }

func (myChecker) Run(ctx context.Context, input Input) Result {
    // input.ProductionDomain — production domain string
    // input.StagingDomain    — staging domain string
    // input.RepoPath         — local repo path
    // input.HTTPClient       — shared *http.Client (respect the ctx deadline)

    return Result{
        Status: StatusPass,
        Detail: "everything looks good",
    }
}

func init() {
    Register(myChecker{})
}
```

3. Add the check name to `defaultOrder` in `internal/checks/checker.go`:

```go
var defaultOrder = []string{"htauth", "mailhog", "cloudflare", "smtp", "mycheck"}
```

That's it. The check will now appear in `magento-checker check` output automatically.

---

## Development

```bash
make build      # compile → bin/magento-checker
make test       # run tests
make lint       # gofmt + go vet + golangci-lint
make snapshot   # cross-platform build via GoReleaser (no tag required)
make clean      # remove bin/
```

Run `make help` to see all available targets.

### Versioning

Versions follow [Semantic Versioning](https://semver.org). Releases use [git-flow](https://nvie.com/posts/a-successful-git-branching-model/), [git-cliff](https://git-cliff.org) for changelog generation, and [GoReleaser](https://goreleaser.com) for cross-platform binaries.

```bash
# Start a release (VERSION is computed by svu from conventional commits)
make release

# Finish the release, push tags, and return to develop
make finish-release
```

Build-time version information is injected via ldflags:

```
-X 'main.Version=<semver>'
-X 'main.Commit=<short-sha>'
-X 'main.Date=<iso8601>'
```

---

## License

MIT
