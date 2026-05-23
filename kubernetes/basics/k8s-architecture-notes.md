A Kubernetes cluster has two types of nodes:
CONTROL PLANE (Master Node)
  — the brain — makes decisions
  — runs cluster management components

WORKER NODES
  — the muscle — runs your actual applications
  — runs your pods/containers

Control Plane Components
1. API Server (kube-apiserver)
The front door of Kubernetes. Every single operation goes through it.

kubectl talks to API server
Scheduler talks to API server
Controller manager talks to API server
Nothing talks to anything else directly

kubectl get pods
    ↓
API Server receives request
    ↓
Authenticates you (are you allowed?)
    ↓
Validates request (is YAML correct?)
    ↓
Stores in etcd
    ↓
Returns response

2. etcd
The database of Kubernetes. Stores the entire cluster state.

What pods are running
What deployments exist
What secrets are stored
Every config, every object
If etcd dies — cluster loses all state. This is why etcd backup is critical in production.
3. Scheduler (kube-scheduler)
Decides which node a new pod runs on.

Looks at pod resource requests (CPU/memory)
Looks at available node capacity
Applies rules like node affinity, taints, tolerations
Writes decision to API server — does NOT start the pod itself

4. Controller Manager (kube-controller-manager)
Runs multiple controllers in one process. Each controller watches the cluster and makes it match the desired state.

ReplicaSet controller — ensures correct number of pods are running
Deployment controller — manages rolling updates
Node controller — detects when nodes go down
Service Account controller — creates default service accounts

Think of it as: "Someone said I want 3 replicas. Are there 3 replicas? No? Let me create one."

Worker Node Components
5. kubelet
The agent running on every worker node.

Talks to API server constantly
Receives pod specs — "run this container"
Tells container runtime to start containers
Reports pod status back to API server
Runs your liveness and readiness probes

6. kube-proxy
Handles networking on each node.

Maintains network rules
Routes traffic to correct pods
Makes Services work — when you hit a Service IP it routes to a pod

7. Container Runtime
Actually runs the containers. Docker used to be common. Now containerd is standard.

Pulls images from registry
Starts and stops containers
kubelet tells it what to do

Developer runs: kubectl apply -f deployment.yaml
                        ↓
              API Server receives request
                        ↓
              Stores desired state in etcd
                        ↓
         Controller Manager notices: "deployment exists but no pods"
                        ↓
         Creates ReplicaSet → creates Pod objects in etcd
                        ↓
         Scheduler notices: "pod exists but no node assigned"
                        ↓
         Scheduler picks best node → updates pod in etcd
                        ↓
         kubelet on that node notices: "pod assigned to me"
                        ↓
         kubelet tells container runtime: "start this container"
                        ↓
         Container starts running
                        ↓
         kubelet reports back to API server: "pod is Running"

Your Laptop
  └── Minikube VM/Container
        ├── Control Plane (API server + etcd + scheduler + controller manager)
        └── Worker Node (kubelet + kube-proxy + container runtime)


What Exactly is a Pod?
A pod is the smallest deployable unit in Kubernetes. It wraps one or more containers and gives them:

Shared network — all containers in a pod share the same IP address and can talk to each other via localhost
Shared storage — containers in a pod can share volumes
Shared lifecycle — all containers start and stop together

Pod (one IP address: 10.244.0.5)
  ├── Container 1: nginx (listens on port 80)
  └── Container 2: log-collector (reads nginx logs)

Container 1 talks to Container 2:
  localhost:9000  ← because they share the same network

Single Container vs Multi-Container Pods
99% of pods have one container. That is the normal case.
Multi-container pods are used for specific patterns:
Sidecar pattern — helper container alongside main container:

Pod:
  Main container: payment-api (serves traffic)
  Sidecar: log-shipper (ships logs to Splunk)

Init container pattern — runs before main container starts:
Pod startup sequence:
  Init container 1: wait-for-database (checks DB is ready)
  Init container 2: run-migrations (runs DB migrations)
  Main container: payment-api (starts AFTER both init containers finish)

Pod Lifecycle — States You Must Know (PIRSFU)

Pending      → pod created, waiting for node assignment or image pull
Init         → init containers running
Running      → at least one container is running
Succeeded    → all containers exited with code 0 (batch jobs)
Failed       → all containers exited, at least one with non-zero code
Unknown      → node lost contact with API server

Container states inside a pod: (WRT)

Waiting      → not started yet (pulling image, waiting for init)
Running      → currently executing
Terminated   → finished (exit code 0=success, non-zero=error)

Pod Failure States — Asked in Every Interview
CrashLoopBackOff:
Container starts → crashes → K8s restarts → crashes again → repeat.
Backoff = wait time increases between restarts (10s, 20s, 40s, 80s, 5min).
OOMKilled:
Container exceeded memory limit. Kernel killed it.
Exit code 137 = OOMKill.
ImagePullBackOff:
Cannot pull the container image.
Wrong image name, wrong tag, registry credentials missing.
Evicted:
Node ran out of resources. Pod was evicted to free up space.
Happens when node memory/disk pressure is high.
Pending (forever):
No node has enough resources to schedule the pod.
Or node selector/affinity rules cannot be satisfied.

# Most useful debugging commands — memorise these

# See pod status and basic info
kubectl get pod POD-NAME

# Full details including events — USE THIS FIRST when pod fails
kubectl describe pod POD-NAME

# Current container logs
kubectl logs POD-NAME

# Previous container logs (after crash)
kubectl logs POD-NAME -p

# Specific container in multi-container pod
kubectl logs POD-NAME -c CONTAINER-NAME

# Stream logs in real time
kubectl logs POD-NAME -f

# Run command inside pod
kubectl exec -it POD-NAME -- /bin/sh

# Run command in specific container
kubectl exec -it POD-NAME -c CONTAINER-NAME -- /bin/sh

# Copy file from pod to local
kubectl cp POD-NAME:/path/to/file ./local-file

# Port forward — access pod directly from laptop
kubectl port-forward POD-NAME 8080:80
# Now visit localhost:8080 to reach pod port 80

🎯 Interview Questions — Day 2
Q1. "What is a pod in Kubernetes?"
A pod is the smallest deployable unit. It wraps one or more containers and gives them a shared network namespace — same IP address — and optional shared storage volumes. All containers in a pod start and stop together. In 99% of cases a pod has one container. Multi-container pods are used for sidecar patterns like log shipping or init patterns like waiting for dependencies.
Q2. "What is the difference between a pod and a container?"
A container is a running process packaged with its dependencies using Docker. A pod is a Kubernetes wrapper around one or more containers. The pod gives containers a stable network identity, shared storage, and manages their lifecycle within Kubernetes. You never interact with containers directly in Kubernetes — always through pods.
Q3. "What is CrashLoopBackOff and how do you debug it?"
CrashLoopBackOff means a container is repeatedly starting and crashing. Kubernetes keeps restarting it but with increasing delays — 10 seconds, 20, 40, up to 5 minutes. To debug: first run kubectl describe pod to check Events section for the failure reason. Then run kubectl logs pod-name -p — the -p flag gets logs from the previous crashed container which shows what the app printed before it crashed. Common causes are missing environment variables, wrong startup command, application code error, or insufficient memory.

Q4. "What is a sidecar container and when do you use it?"
A sidecar is a second container in the same pod that enhances the main container without changing it. Common uses: log shipping (sidecar reads app logs and ships to Splunk or CloudWatch), service mesh proxy (Istio injects a proxy sidecar for traffic management), config reloader (watches for config changes and reloads the main app). At JPMC the Dynatrace OneAgent is often deployed as a sidecar to instrument applications for APM without changing the application code.
Q5. "What is an init container and when do you use it?"
An init container runs to completion before any main containers start. If the init container fails the pod restarts and tries again. Use cases: wait for a database to be ready before starting the app, run database migrations before the application starts, download configuration files from a secure location before the app needs them. Init containers run in sequence — each must complete before the next starts. This is different from sidecar containers which run in parallel with the main container.

Scenario Q6. "A pod is in Pending state for 10 minutes. What do you do?"
Step 1: kubectl describe pod pod-name
        Look at Events — what does it say?

Step 2: Common causes of Pending:

  a. Insufficient resources on nodes
     Events show: "0/1 nodes are available:
     1 Insufficient memory"
     Fix: reduce resource requests OR add more nodes

  b. Image being pulled (slow registry)
     Events show: "Pulling image nginx:latest"
     Wait OR check registry connectivity

  c. Node selector not matching any node
     Events show: "0/1 nodes matched node selector"
     Fix: check nodeSelector in pod spec

  d. PVC not bound (storage issue)
     Events show: "persistentvolumeclaim not found"
     Fix: check PVC exists and is bound

Step 3: kubectl get nodes
        Are nodes Ready? If NotReady — node problem not pod problem

