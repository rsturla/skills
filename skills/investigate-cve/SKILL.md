---
name: investigate-cve
description: >
  Investigate a CVE to find upstream fix commits, affected versions, severity, and backport status. Queries OSV.dev,
  NVD, and Red Hat Security APIs, then checks upstream git history. Use when the user mentions a CVE ID, asks about
  a vulnerability fix, or wants to know if a patch has been backported — even if they just say "is this CVE fixed?"
compatibility: Requires curl and jq. Git required for upstream repo searches.
allowed-tools:
  - Bash(curl *)
  - Bash(jq *)
  - Bash(git *)
metadata:
  author: rsturla
  version: 1.0.0
---

# CVE Investigation

Given a CVE ID, trace it to upstream fix commits and check backport status.

## Workflow

1. **OSV.dev** — structured fix data (try first)
1. **NVD** — references and patch links
1. **Debian Security Tracker** — frequently lists fix commits in notes
1. **Red Hat Security API** — advisory context
1. **Upstream git search + backport check** — see [REFERENCE.md](REFERENCE.md)

## Step 1: Query OSV.dev

Best structured data. Maps CVEs to fix commits for Go, Rust, Python, npm, Linux.

```bash
curl -s "https://api.osv.dev/v1/vulns/<cve_id>" | jq '{
  id: .id,
  summary: .summary,
  severity: .severity,
  affected: [.affected[] | {package: .package, versions: .versions, fix_events: [.ranges[]?.events[]? | select(.fixed)]}],
  fix_refs: [.references[]? | select(.type == "FIX")]
}'
```

Key fields:

- `affected[].ranges[].events[].fixed` — version that fixes the CVE
- `references[]` where `type == "FIX"` — direct links to fix commits

## Step 2: Query NVD

Comprehensive CVE details and reference links.

```bash
curl -s "https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=<cve_id>" | jq '{
  id: .vulnerabilities[0].cve.id,
  description: .vulnerabilities[0].cve.descriptions[0].value,
  severity: .vulnerabilities[0].cve.metrics,
  references: [.vulnerabilities[0].cve.references[] | {url, tags}]
}'
```

Look for references tagged `Patch`, `Third Party Advisory`, or URLs containing `commit`, `pull`, `patch`.

NVD rate-limits unauthenticated requests to 5 per 30 seconds. If hitting limits, request a free API key at
<https://nvd.nist.gov/developers/request-an-api-key> and pass it as `?apiKey=<key>` query parameter.

## Step 3: Check Debian Security Tracker

Debian's CVE list frequently contains upstream fix commit URLs in `NOTE:` lines. Clone the tracker repo and grep locally:

```bash
# Clone once (reuse for subsequent queries)
git clone --depth 1 https://salsa.debian.org/security-tracker-team/security-tracker.git ~/.cache/debian-security-tracker

# Search for CVE
grep -A 15 "^<cve_id>" ~/.cache/debian-security-tracker/data/CVE/list | head -20
```

Look for:

- `NOTE: https://...commit/...` — upstream fix commit (often includes version tag in parentheses)
- `NOTE: https://...advisories/...` — security advisory with fix details
- Fixed package versions per Debian release (e.g. `- node-postcss 8.5.12+~cs9.3.32-1`)
- Status per release: `<no-dsa>`, `<postponed>`, `<not-affected>`

If the clone already exists, `git -C ~/.cache/debian-security-tracker pull` to update.

## Step 4: Query Red Hat Security API

```bash
curl -s "https://access.redhat.com/hydra/rest/securitydata/cve/<cve_id>.json" | jq '{
  name: .name,
  severity: .threat_severity,
  cvss3_score: .cvss3.cvss3_scoring_vector,
  bugzilla: .bugzilla,
  affected_packages: [.affected_release[]?.package],
  fix_state: [.package_state[]? | {product: .product_name, package: .package_name, fix_state: .fix_state}]
}'
```

Key fields:

- `bugzilla.url` — links to Bugzilla with patch details
- `affected_release[].package` — packages with fixes shipped
- `package_state[].fix_state` — "Fixed", "Not affected", "Will not fix"

See [REFERENCE.md](REFERENCE.md) for upstream git search and backport detection.

## Output Format

Present findings as:

```markdown
## <cve_id>

- **Summary**: <one-line description>
- **Severity**: <CVSS score and vector>
- **Affected**: <package name> versions <introduced> to <fixed>
- **Fix commit**: <sha> (<link>)
- **Fixed in version**: <version>
- **Backport status**:
  - `main`: included
  - `release/1.x`: included (cherry-pick <sha>)
  - `release/0.9.x`: NOT backported
```

## Gotchas

- OSV.dev is best for open-source ecosystems but may lack data for proprietary or niche packages.
- Red Hat API returns "Will not fix" for CVEs in EOL products — don't confuse with "not affected".
- `git branch --contains` requires a full clone, not shallow. Use `--unshallow` if needed.
- Some fixes span multiple commits — check for related commits near the fix date.
- Cherry-pick detection checks the `(cherry picked from commit ...)` trailer first. For manual backports, fall back to
  searching the target branch for similar diffs or commit message keywords from the original fix.
