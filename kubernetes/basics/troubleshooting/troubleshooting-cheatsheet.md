# Kubernetes Troubleshooting Cheat Sheet

## Pod Status → Root Cause → Fix

| Status | Root Cause | First Command | Fix |
|---|---|---|---|
| ImagePullBackOff | Wrong image/tag or registry issue | kubectl describe pod | Fix image name or tag |
| CrashLoopBackOff | App crashing on startup | kubectl logs -p | Fix app config or code |
| Pending | Not enough node resources | kubectl describe pod | Reduce resource requests |
| OOMKilled (137) | Memory limit exceeded | kubectl describe pod | Increase memory limit |
| CreateContainerConfigError | Missing ConfigMap/Secret key | kubectl describe pod | Add missing key |
| Error | Various — check events | kubectl describe pod | Depends on event |

## Debug Commands in Order
1. kubectl get pods                          → what is status?
2. kubectl describe pod NAME                 → what do events say?
3. kubectl logs NAME                         → what did app print?
4. kubectl logs NAME -p                      → what before crash?
5. kubectl get events --sort-by=lastTimestamp → cluster events
6. kubectl exec -it NAME -- /bin/sh          → get inside pod
7. kubectl get endpoints SERVICE-NAME        → is service routing?
8. kubectl auth can-i list pods --as=SA      → RBAC issue?

## Service Not Working
kubectl get endpoints SERVICE   → empty = selector mismatch
kubectl get pods --show-labels  → check pod labels
kubectl describe service NAME   → check selector

## Node Issues
kubectl get nodes               → node status
kubectl describe node NAME      → node events and pressure
kubectl top nodes               → resource usage