# Kubernetes Practice — Complete Summary

## What I Built
A multi-tier application on Kubernetes covering all
core concepts used in production SRE environments.

## Architecture
Frontend (nginx, 2 replicas, NodePort)
    ↓
Backend (nginx API, 2 replicas, HPA 2-8, ClusterIP)
    ↓
Database (PostgreSQL, StatefulSet, PVC 500Mi)

## Concepts Covered
| Day | Topic | Key Learning |
|---|---|---|
| 1 | Architecture | Control plane + worker node components |
| 2 | Pods | Lifecycle, sidecar, init containers, failures |
| 3 | Deployments | Rolling updates, rollback, strategies |
| 4 | Services | ClusterIP, NodePort, DNS, load balancing |
| 5 | ConfigMaps + Secrets | All creation and injection patterns |
| 6 | Resource Management | QoS classes, LimitRange, ResourceQuota |
| 7 | Storage | PV, PVC, StorageClass, StatefulSet |
| 8 | RBAC | Roles, ClusterRoles, ServiceAccounts |
| 9 | Troubleshooting | 6 real failure scenarios debugged |
| 10 | Mini Project | Complete multi-tier app deployment |

## Key Commands
kubectl get all -n mini-app
kubectl describe pod NAME -n mini-app
kubectl logs NAME -n mini-app
kubectl rollout undo deployment/NAME -n mini-app
kubectl auth can-i VERB RESOURCE --as=SA

## Interview Scenarios I Can Answer
- Walk through K8s architecture end to end
- Debug CrashLoopBackOff, OOMKill, Pending, ImagePullBackOff
- Explain zero-downtime deployments
- Explain RBAC and least privilege
- Explain persistent storage for stateful apps
- Explain HPA and when to use it
