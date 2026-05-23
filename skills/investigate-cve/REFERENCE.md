# CVE Investigation Reference

## Search Upstream Git

When APIs don't return fix commits, clone and search the upstream repo.

```bash
git log --all --oneline --grep="<cve_id>"
```

If no results, try searching for the fix description keywords:

```bash
git log --all --oneline --grep="<vulnerability_keyword>" --since="<disclosure_date>"
```

## Check Backport Status

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
