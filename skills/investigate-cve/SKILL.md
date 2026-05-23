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
2. **NVD** — references and patch links
3. **Red Hat Security API** — advisory context
4. **Upstream git log** — fallback when APIs lack fix commits
5. **Backport check** — verify fix reached relevant LTS/release branches

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

## Step 3: Query Red Hat Security API

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

## Step 4: Search Upstream Git

When APIs don't return fix commits, clone and search the upstream repo.

```bash
git log --all --oneline --grep="<cve_id>"
```

If no results, try searching for the fix description keywords:

```bash
git log --all --oneline --grep="<vulnerability_keyword>" --since="<disclosure_date>"
```

## Step 5: Check Backport Status

Once you have the fix commit SHA, check which branches include it:

```bash
git branch -r --contains <fix_commit_sha>
```

If the relevant LTS/stable branch is missing, check for backports:

```bash
# Check for cherry-picks (standard trailer)
git log --all --oneline --grep="cherry picked from commit <fix_commit_sha>"

# If no cherry-pick found, check if the fix is already in the code
# Compare the fixed lines against the target branch
git show <fix_commit_sha> -- <affected_files>   # see what changed
git show <target_branch>:<affected_file>         # see current state on target branch

# If the fix is present, find who applied it
git log -1 --format="%H %s" <target_branch> -- <affected_files>
git blame <target_branch> -- <affected_file> | grep "<fixed_line>"

# Fallback: search by commit message keywords
git log --oneline <target_branch> --grep="<fix_summary_keywords>" --since="<fix_date>"
```

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

- NVD rate-limits unauthenticated requests to 5/30s. Add `apiKey` param if hitting limits.
- OSV.dev is best for open-source ecosystems but may lack data for proprietary or niche packages.
- Red Hat API returns "Will not fix" for CVEs in EOL products — don't confuse with "not affected".
- `git branch --contains` requires a full clone, not shallow. Use `--unshallow` if needed.
- Some fixes span multiple commits — check for related commits near the fix date.
- Cherry-pick detection checks the `(cherry picked from commit ...)` trailer first. For manual backports, fall back to
  searching the target branch for similar diffs or commit message keywords from the original fix.
