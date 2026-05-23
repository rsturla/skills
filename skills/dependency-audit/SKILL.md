---
name: dependency-audit
description: >
  Audit project dependencies for known vulnerabilities, license issues, and supply chain risks. Supports Go, Rust,
  and Python. Use when the user changes dependencies, asks about vulnerability scanning, mentions CVEs in deps,
  or wants a security check — even if they just say "are my deps safe?" or "check for vulnerabilities."
compatibility: Requires one or more of govulncheck, cargo-audit, pip-audit, osv-scanner.
allowed-tools:
  - Bash(govulncheck *)
  - Bash(cargo audit*)
  - Bash(cargo deny*)
  - Bash(pip-audit*)
  - Bash(osv-scanner*)
  - Bash(go-licenses*)
  - Bash(pip-licenses*)
  - Bash(git diff*)
metadata:
  author: rsturla
  version: 1.0.0
---

# Dependency Audit

Scan project dependencies for vulnerabilities, license compliance, and supply chain risks.

## Process

1. Detect project type from lockfiles (`go.sum`, `Cargo.lock`, `requirements.txt`, `pyproject.toml`)
2. Run ecosystem-specific vulnerability scanner
3. Run license check if applicable
4. Check recent dependency changes for supply chain indicators
5. Report findings with severity, fix version, and action items

## Vulnerability Scanning

### Go

```bash
govulncheck ./...
```

Key: `govulncheck` does **reachability analysis** — it tells you if your code actually calls the vulnerable function,
not just that you imported the package. "Called" findings are critical; "imported but not called" are lower priority.

### Rust

```bash
cargo audit
```

Uses the RustSec Advisory Database. `cargo audit fix` auto-updates to patched versions.

For comprehensive checks (advisories + licenses + bans):

```bash
cargo deny check
```

### Python

```bash
pip-audit
```

For `uv`-managed projects:

```bash
pip-audit -r <(uv pip compile pyproject.toml)
```

### Cross-Ecosystem (OSV-Scanner)

When multiple ecosystems are present or ecosystem-specific tools aren't installed:

```bash
osv-scanner -r .
```

Auto-detects `go.sum`, `Cargo.lock`, `requirements.txt`, `poetry.lock`, `pnpm-lock.yaml`. Uses the OSV.dev database.

## License Checking

```bash
# Go
go-licenses check ./... --allowed_licenses=Apache-2.0,MIT,BSD-3-Clause

# Rust
cargo deny check licenses

# Python
pip-licenses --format=json
```

Flag any dependency with a license not in: `Apache-2.0`, `MIT`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `MPL-2.0`.

## Supply Chain Risk Check

After any dependency change, review the diff for red flags:

```bash
git diff HEAD~1 -- go.sum Cargo.lock requirements.txt pyproject.toml
```

Red flags to look for:

- **Typosquatting**: package name off by one char (`requets`, `crytpo`, `coloura`)
- **New transitive deps**: a small update pulling in many new packages
- **Version anomalies**: jump from v1.2.3 to v9.0.0, or new major version with far fewer downloads
- **Maintainer takeover**: check if the package recently changed ownership

For Python, inspect `setup.py` for suspicious install hooks:

```bash
grep -rn 'subprocess\|os\.system\|exec(' setup.py
```

## Output Format

```markdown
## Dependency Audit Results

### Vulnerabilities
- **CRITICAL**: <package> <version> — <CVE-ID> — fix: upgrade to <fixed_version>
- **HIGH**: <package> <version> — <advisory> — fix: <action>

### License Issues
- <package>: <license> — not in allowed list

### Supply Chain
- No red flags detected / <specific concerns>
```

## Gotchas

- `govulncheck` reachability analysis is Go-only — other tools just check version ranges
- `cargo audit` reads `Cargo.lock`, not `Cargo.toml`. If no lockfile exists, run `cargo generate-lockfile` first.
- `pip-audit` scans the active environment by default. Use `-r requirements.txt` for file-based scanning.
- `osv-scanner` is the best cross-ecosystem fallback but lacks reachability analysis (except experimental Go support)
- License checking finds the declared license — it doesn't verify the license text matches the declaration
- A clean scan doesn't mean safe. Zero-days and undisclosed vulnerabilities won't appear in any database.
