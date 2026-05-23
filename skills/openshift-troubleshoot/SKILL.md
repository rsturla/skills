---
name: openshift-troubleshoot
description: >
  Diagnose and troubleshoot OpenShift cluster and workload issues. Pod failures, networking, storage, operators, SCCs,
  and node problems. Use when the user has an OpenShift issue, mentions oc commands, pod crashes, CrashLoopBackOff,
  ImagePullBackOff, routes not working, or cluster problems — even if they just say "my pod won't start" or "the app
  is broken."
compatibility: Requires oc CLI (authenticated to an OpenShift cluster).
allowed-tools:
  - Bash(oc *)
metadata:
  author: rsturla
  version: 1.0.0
---

# OpenShift Troubleshooting

## Quick Start

```bash
# What's happening right now?
oc get events -n <ns> --sort-by=.metadata.creationTimestamp
oc describe pod <pod> -n <ns>     # check Events section at bottom
oc logs <pod> --previous -n <ns>  # logs from last crashed container
```

## Diagnostic Flowchart

1. **Check events**: `oc get events -n <ns> --sort-by=.metadata.creationTimestamp`
2. **Describe the resource**: `oc describe pod/svc/route/pvc <name>`
3. **Check logs**: `oc logs <pod> --previous` (for crashes), `oc logs <pod> -f` (live)
4. **Debug interactively**: `oc debug deployment/<name>` or `oc debug node/<node>`
5. **Check operators**: `oc get clusteroperators` — look for Degraded=True
6. **Collect diagnostics**: `oc adm must-gather` for support cases

## Pod Failure Patterns

| Status | Likely Cause | Diagnostic Command |
| ------ | ------------ | ------------------ |
| CrashLoopBackOff | App error, SCC, bad probes | `oc logs <pod> --previous` |
| ImagePullBackOff | Bad image ref, missing pull secret | `oc describe pod <pod>` Events |
| Pending | No capacity or PVC unbound | `oc describe pod <pod>` Events; `oc get pvc` |
| OOMKilled | Memory limit too low (exit 137) | `oc adm top pod -n <ns>` vs resource limits |
| CreateContainerConfigError | Missing ConfigMap/Secret ref | `oc describe pod <pod>` Events |

## SCC (Security Context Constraints)

Rule of thumb: "If it doesn't work, it's RBAC. If not, it's SCC. If not, it's DNS."

- Default `restricted-v2` drops ALL capabilities, assigns random UID from namespace range
- Most Docker Hub images fail because they assume root
- Grant `anyuid` to a **service account**, not a user:

```bash
oc adm policy add-scc-to-user anyuid -z <sa-name> -n <ns>
```

- Check which SCC a pod is using: `oc get pod <pod> -o jsonpath='{.metadata.annotations.openshift\.io/scc}'`

## Networking

- **Routes return 503**: Default-deny NetworkPolicy blocks the ingress controller. Allow traffic from `openshift-ingress`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
spec:
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              policy-group.network.openshift.io/ingress: ""
  podSelector: {}
```

- **DNS issues**: Allow both UDP and TCP on port 53. TCP needed for responses >512 bytes.
- Use `oc debug` + `curl`/`dig` to test connectivity from inside the cluster.

See [REFERENCE.md](REFERENCE.md) for node troubleshooting, storage, operators, builds, must-gather, and log collection.
