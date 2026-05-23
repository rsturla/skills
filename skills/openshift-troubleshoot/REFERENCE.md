# OpenShift Troubleshooting — Reference

## Node Issues

- **NotReady after upgrade**: Often pending CSRs. Approve them:

```bash
oc get csr
oc adm certificate approve <csr-name>
```

Multiple CSRs per node — repeat until all nodes Ready.

- **Drain**: `oc adm drain <node> --ignore-daemonsets --delete-emptydir-data`
- **Resource pressure**: `oc adm top nodes`
- **Node shell**: `oc debug node/<node>` then `chroot /host`

## Storage

- **PVC stuck Pending**: Check StorageClass exists, provisioner is healthy
- **RWO multi-attach**: Common when a node goes NotReady — old pod still "owns" the PV. Force-delete the pod on the
  dead node: `oc delete pod <pod> --grace-period=0 --force`
- **Check PV/PVC binding**: `oc get pv,pvc -n <ns>`

## Operators

```bash
oc get clusteroperators                    # Available | Degraded | Progressing columns
oc logs -n <operator-ns> <operator-pod>    # Operator logs
oc get pods -n openshift-*                 # Check all operator namespace pods
```

## Build / ImageStream Failures

- Builds fail silently if referenced ImageStream is missing — only visible in events
- Manually trigger import to diagnose registry auth: `oc import-image <imagestream>`
- **DeploymentConfig is legacy** — use standard `Deployment`

## must-gather / sosreport

Fallback chain (most to least comprehensive):

1. `oc adm must-gather` — cluster-wide, requires `cluster-admin`
2. `oc adm inspect clusteroperator/<name>` — single operator diagnostics
3. `sosreport` via `oc debug node/` + toolbox — node-level system logs
4. SSH to node — last resort

On restricted networks, import the must-gather image first.

## Log Collection

```bash
# Pod logs (current + previous crash)
oc logs <pod> -c <container> -f --previous

# Node journal (kubelet)
oc debug node/<node> -- chroot /host journalctl -u kubelet --since "1 hour ago"

# Audit logs (control plane)
oc debug node/<cp-node> -- chroot /host cat /var/log/kube-apiserver/audit.log
```

Centralized logging: OpenShift Logging stack (Loki + Vector, or legacy EFK).

## OpenShift vs Vanilla Kubernetes Gotchas

| OpenShift | Kubernetes | Notes |
| --------- | ---------- | ----- |
| SCC (restricted-v2) | PodSecurityStandards | SCC is stricter, assigns random UID |
| Route | Ingress | Ingress objects auto-convert to Routes |
| ImageStream | (no equivalent) | Virtual tags, server-side management |
| DeploymentConfig | Deployment | DC is legacy, use Deployment |
| Project | Namespace | Projects add admin controls |
| `oc` | `kubectl` | `oc` is a superset of `kubectl` |
