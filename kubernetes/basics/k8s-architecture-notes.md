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

========================================================================================
Day3
========================================================================================

Deployment
  └── ReplicaSet v1 (old version)
        ├── Pod 1 (nginx:1.19)
        └── Pod 2 (nginx:1.19)
  └── ReplicaSet v2 (new version) ← created during update
        ├── Pod 3 (nginx:1.21)
        └── Pod 4 (nginx:1.21)

# See deployment history
kubectl rollout history deployment/nginx-app
# REVISION  CHANGE-CAUSE
# 1         Initial deployment - nginx v1.19
# 2         <none>  (we used patch, no annotation)

# Check which image is running now
kubectl describe deployment nginx-app | grep Image

# Rollback to previous version
kubectl rollout undo deployment/nginx-app

# Watch rollback happen
kubectl rollout status deployment/nginx-app

# Verify old image is back
kubectl describe deployment nginx-app | grep Image

# Check history again
kubectl rollout history deployment/nginx-app
# Revision 3 is now the rolled-back version

# Start an update
kubectl set image deployment/nginx-app nginx=nginx:1.23-alpine

# Immediately pause it
kubectl rollout pause deployment/nginx-app

# Check — some pods updated, some not
kubectl get pods
kubectl describe deployment nginx-app | grep Image

# Verify the new version is working on updated pods
# If happy — resume
kubectl rollout resume deployment/nginx-app

# Watch rest of update complete
kubectl rollout status deployment/nginx-app

# Scale up
kubectl scale deployment nginx-app --replicas=5
kubectl get pods

# Scale down
kubectl scale deployment nginx-app --replicas=2
kubectl get pods

# Scale to zero (stops all pods but keeps deployment)
kubectl scale deployment nginx-app --replicas=0
kubectl get pods
# No pods running but deployment still exists

# Scale back up
kubectl scale deployment nginx-app --replicas=3

==================================================================================
Day 4 
============================================================================

Pods are ephemeral — they die and get replaced with new IPs constantly.
Service gives pods a stable DNS name and IP that never changes even when pods behind it change.

The 4 Service Types
ClusterIP (default)
Only accessible inside the cluster. Other pods can reach it. Nothing outside can.
Internet → ✗ Cannot reach
Pod inside cluster → ✓ Can reach via service name
Use for: internal microservices, databases, anything that should not be public.

NodePort
Opens a port on every node (30000-32767). Accessible from outside the cluster via node IP.
Internet → NodeIP:30080 → Service → Pod
Use for: development, Minikube testing. Not recommended for production.

LoadBalancer
Creates a cloud load balancer (AWS ELB, GCP LB). Gets a public IP automatically.
Internet → Public IP → Load Balancer → Service → Pod
se for: production internet-facing services on cloud providers.

How Service Finds Pods — Selector
Service uses label selector to find pods — exactly like Deployment:
# Service
spec:
  selector:
    app: payment-api    # route traffic to pods with THIS label

# Pod (in Deployment template)
metadata:
  labels:
    app: payment-api    # this pod matches the service selector
If labels match → Service routes traffic to that pod.
If pod dies and new one starts with same label → Service automatically includes it.

Kubernetes DNS — How Pods Find Services
Every Service gets a DNS name automatically:

Q1. "What is a Kubernetes Service and why do we need it?"
Pods are ephemeral — they die and get new IP addresses constantly. A Service provides a stable network endpoint with a fixed ClusterIP and DNS name that never changes. It uses label selectors to find healthy pods and automatically load balances traffic across them. When pods are replaced their new IPs are automatically added to the Service endpoints.

Q2. "What are the different Service types and when do you use each?"
ClusterIP is the default — only accessible inside the cluster, used for internal microservices. NodePort opens a fixed port on every node, accessible from outside — used for development and Minikube testing. LoadBalancer creates a cloud load balancer with a public IP — used for production internet-facing services on AWS or GCP. ExternalName maps to an external DNS name — used to abstract external services like RDS so pods do not hardcode external URLs.

Q3. "What is the difference between port, targetPort and nodePort in a Service?"
port is what the Service itself listens on inside the cluster — how other pods reach this service. targetPort is the port the container is actually listening on inside the pod. nodePort is only for NodePort type — the port opened on every node for external access. Traffic flows: nodePort on node → port on service → targetPort on pod.
Q4. "How do microservices discover and communicate with each other in Kubernetes?"
Through Kubernetes DNS. Every Service gets a DNS name automatically in the format service-name.namespace.svc.cluster.local. Within the same namespace pods can use just the service name. The CoreDNS component handles all DNS resolution in the cluster. This means a payment service pod can call http://auth-service/validate without knowing any IP addresses — Kubernetes DNS resolves the name to the stable ClusterIP of the auth-service Service.

Scenario Q5. "Your service is not routing traffic to pods. How do you debug?"
Step 1: Check service exists and has correct selector
  kubectl describe service service-name
  Look at Selector field — does it match pod labels?

Step 2: Check endpoints
  kubectl get endpoints service-name
  If endpoints are empty → selector does not match pod labels
  Fix: align service selector with pod labels

Step 3: Check pod labels
  kubectl get pods --show-labels
  Do pod labels match service selector exactly?

Step 4: Check pods are Ready
  kubectl get pods
  If pods are Running but not Ready → readiness probe failing
  Pods not Ready are removed from endpoints automatically
  Fix: check readiness probe configuration

Step 5: Test from inside cluster
  kubectl run test --image=busybox --rm -it -- wget service-name
  If this works → problem is external access not internal routing

  ClusterIP Flow
  User Browser → ✗ CANNOT reach directly
               ClusterIP only exists inside cluster

Internal pod → Service ClusterIP (10.96.0.1:80)
                      ↓
               kube-proxy routing rules
                      ↓
               One of the healthy pods (10.244.0.5:80)

NodePort Flow
User Browser → Node IP : 30080
(192.168.49.2:30080)
                      ↓
               kube-proxy on that node
               sees traffic on port 30080
                      ↓
               Service (port 80)
                      ↓
               One of the healthy pods (port 80)

LoadBalancer Flow
User Browser → www.payment.com
                      ↓
               DNS resolves to Public IP
               (e.g. 54.123.45.67)
                      ↓
               AWS Elastic Load Balancer
               (created automatically by K8s)
                      ↓
               NodePort on one of the nodes
                      ↓
               Service
                      ↓
               One of the healthy pods

The Role of kube-proxy in All of This
kube-proxy runs on EVERY node.
It watches the API server for Service changes.
It writes iptables rules on each node.

When traffic hits a node:
  iptables rule → "port 30080? forward to service IP"
  iptables rule → "service IP? pick one of these pod IPs"

This is how routing works — not a real proxy process
but iptables rules doing the routing at kernel level.

if tyhe pod is not in ready state the end points gets ddleted automatically and readiness probe fails

==================================================================
DAY 5  configmaps and secrets
============================================================

ConfigMap — 3 Ways to Create
Way 1 — From YAML file (what you did before)
Way 2 — From literal values on command line
Way 3 — From an existing file
3 Ways to Inject Into a Pod
Way 1 — As environment variables (individual keys)
Way 2 — As environment variables (all keys at once)
Way 3 — As a mounted file

You will mostly use Opaque.
Important — Secret Encoding vs Encryption

Secret values are BASE64 ENCODED — not encrypted.
Anyone with kubectl access can decode them:

echo "c2VjcmV0" | base64 --decode
→ secret

In production: enable encryption at rest on API server
Or use: AWS Secrets Manager, HashiCorp Vault

Q1. "What is the difference between a ConfigMap and a Secret?"
ConfigMap stores non-sensitive configuration — log levels, URLs, feature flags — as plain text. Secret stores sensitive data — passwords, API keys, certificates — as base64 encoded values. Both can be injected as environment variables or mounted as files. The key difference is intent and access control — Secrets can have stricter RBAC policies, can be encrypted at rest, and their values are hidden in kubectl describe output. Never put sensitive data in a ConfigMap even though technically possible.

Q2. "What are the ways to inject a ConfigMap into a pod?"
Three ways. First — individual env var using configMapKeyRef, which maps one specific key to one env var name. Second — all keys at once using envFrom with configMapRef, which creates env vars for every key in the ConfigMap. Third — mounted as files using a volume and volumeMount, which creates a file for each key inside the mounted directory. The mounted file approach has one advantage over env vars — when the ConfigMap is updated the files update automatically within about 60 seconds without restarting the pod. Env vars require a pod restart to pick up changes.
Q3. "Are Kubernetes Secrets secure?"
By default Secrets are only base64 encoded — not encrypted. Anyone with kubectl access and permission to read secrets can decode them with a simple base64 decode command. For real security you need additional measures: enable encryption at rest on the Kubernetes API server so secrets are encrypted in etcd, use strict RBAC to limit who can read secrets, and ideally integrate with an external secrets manager like AWS Secrets Manager or HashiCorp Vault using the External Secrets Operator. At JPMC secrets would go through an enterprise secrets management solution rather than relying solely on Kubernetes Secrets.

Q4. "What happens to a running pod when you update a ConfigMap?"
It depends on how the ConfigMap was injected. If injected as environment variables — nothing changes in the running pod. Environment variables are set at container startup and cannot be updated without restarting the pod. If injected as a mounted volume — the files inside the pod update automatically within approximately 60 seconds as Kubernetes syncs the volume. This is why for configuration that changes frequently, mounting as files is preferred over env vars — you can reload config without pod restarts.

Scenario Q5. "A developer committed a database password to a ConfigMap YAML file in Git. What do you do?"

Step 1: IMMEDIATE — rotate the password
  Change the database password right away
  Assume it is already compromised

Step 2: Remove from Git history
  git filter-branch or BFG Repo Cleaner
  Force push to rewrite history
  If repo is public — assume already scraped

Step 3: Update the actual Kubernetes Secret
  kubectl create secret generic db-secret \
    --from-literal=password=NEW_PASSWORD \
    --dry-run=client -o yaml | kubectl apply -f -

Step 4: Restart pods to pick up new secret
  kubectl rollout restart deployment/app-name

Step 5: Fix the process
  Add pre-commit hook to scan for secrets
  Use tools like git-secrets or detect-secrets
  Move secret values to pipeline variables
  Never put real values in YAML files in Git

====================================================================
dAY 6 - request, limits qos classes
====================================================================
requests = what you are GUARANTEED
limits   = the maximum you can use

Node has 4GB memory total

Pod A: requests 512Mi, limits 1GB
Pod B: requests 512Mi, limits 1GB
Pod C: requests 512Mi, limits 1GB
Pod D: requests 512Mi, limits 1GB

Total requested: 2GB → fits on node → all 4 pods scheduled
At runtime Pod A uses 900Mi → still under 1GB limit → fine
At runtime Pod A tries 1.1GB → exceeds limit → OOMKilled

CPU vs Memory — Different Behaviour at Limit
CPU limit exceeded:
  Pod gets THROTTLED — slowed down
  NOT killed
  Just runs slower

Memory limit exceeded:
  Pod gets KILLED — OOMKilled
  Exit code 137
  Kubernetes restarts it

This is a critical difference. CPU is compressible — you can slow it down. Memory is not compressible — once used you cannot take it back without killing the process.

Cpu units

1 CPU core = 1000 millicores

cpu: "1"      = 1 full CPU core
cpu: "500m"   = 0.5 CPU core = half a core
cpu: "100m"   = 0.1 CPU core = 10% of one core
cpu: "2"      = 2 full CPU cores

Rule of thumb for small services:
  requests: 100m   (needs 10% of a core normally)
  limits: 500m     (can burst to 50% of a core)

memory units

Mi = Mebibytes (use this)
Gi = Gibibytes

memory: "128Mi"  = 128 Mebibytes
memory: "256Mi"  = 256 Mebibytes
memory: "1Gi"    = 1 Gibibyte

Rule of thumb for small services:
  requests: 128Mi  (needs 128MB normally)
  limits: 256Mi    (can burst to 256MB)

QoS Classes — Kubernetes Assigns These Automatically
When node runs out of resources Kubernetes must evict pods. It uses QoS class to decide ORDER of eviction.
Guaranteed — safest, evicted last
# requests EQUALS limits for ALL containers
resources:
  requests:
    memory: "256Mi"
    cpu: "500m"
  limits:
    memory: "256Mi"   # same as request
    cpu: "500m"       # same as request
  
  Burstable — middle priority
  # requests set but less than limits
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"   # higher than request
    cpu: "500m"       # higher than request
  
  BestEffort — evicted first, most dangerous

What Happens When Node Runs Out of Memory
Step 1: Node memory pressure detected
Step 2: Kubernetes tries to free memory
Step 3: Eviction order:
  First  → BestEffort pods (no limits set)
  Second → Burstable pods (limits higher than requests)
  Last   → Guaranteed pods (requests equal limits)
Step 4: Evicted pods get rescheduled on another node
Step 5: If no other node has capacity → pod stays Pending

Q1. "What is the difference between resource requests and limits?"
Requests are the guaranteed minimum — Kubernetes uses them to decide which node to schedule a pod on. If a node does not have enough capacity to satisfy requests, the pod is not scheduled there. Limits are the hard ceiling — a pod cannot exceed its memory limit without being OOMKilled, and cannot exceed its CPU limit without being throttled. Setting requests too high wastes capacity. Setting limits too low causes OOMKills. The best practice is to set requests equal to typical usage and limits at around twice the typical usage based on profiling.

Q2. "What are QoS classes in Kubernetes and why do they matter?"
QoS classes determine the order of pod eviction when a node runs out of resources. Kubernetes assigns them automatically based on how resources are configured. Guaranteed means requests equal limits for all containers — these pods are evicted last. Burstable means requests are set but lower than limits — middle priority. BestEffort means no resources set at all — evicted first. In production all pods should be at least Burstable. BestEffort pods should never run in production because they are the first casualties when any node experiences memory pressure.

Q3. "What is the difference between CPU throttling and OOMKill?"
CPU is compressible — when a pod exceeds its CPU limit Kubernetes throttles it, meaning it runs slower but keeps running. Memory is not compressible — when a pod exceeds its memory limit the Linux kernel OOMKills it with exit code 137. The pod then restarts. This is why memory limits need more careful tuning than CPU limits. A memory limit set too low causes constant OOMKill restart cycles which you will see as CrashLoopBackOff with exit code 137 in kubectl describe.

Q4. "What is a LimitRange and when do you use it?"
A LimitRange sets default resource requests and limits for all pods in a namespace that do not specify their own. It also sets minimum and maximum boundaries — if a pod requests more than the max or less than the min, it is rejected. Use LimitRange to enforce resource policies across a team without requiring every developer to manually set resources on every pod. Combined with ResourceQuota which limits total consumption across the namespace, LimitRange gives namespace-level resource governance.

Scenario Q5. "Your pod keeps restarting with exit code 137. What is the cause and fix?"

Exit code 137 = OOMKill = pod exceeded memory limit.

Diagnose:
  kubectl describe pod pod-name
  Look for: Last State → Reason: OOMKilled
  Look for: Exit Code: 137

Check memory usage pattern:
  kubectl top pod pod-name
  Is it near the limit? What is the trend?

Fix options:
  Option 1: Increase memory limit (quick fix)
    kubectl patch deployment name -p \
    '{"spec":{"template":{"spec":{"containers":
    [{"name":"app","resources":{"limits":
    {"memory":"512Mi"}}}]}}}}'

  Option 2: Find and fix memory leak (proper fix)
    Profile the application
    Check for objects not being garbage collected
    Check for connection pools not being closed

  Option 3: Add memory-based HPA
    Scale horizontally instead of giving one pod more memory
    Better for stateless services

QoS Classes — How Eviction Actually Works
Let me explain this with a real scenario.

Your node has 4GB memory total
Running pods consuming 3.8GB
Suddenly memory spikes to 4.1GB
Node is out of memory — must evict something

ROUND 1 — Kill BestEffort pods first
  These have NO resource requests or limits
  Kubernetes thinks: "these pods never told me
  how much they need — they get nothing guaranteed"
  All BestEffort pods evicted immediately

Still not enough memory? →

ROUND 2 — Kill Burstable pods
  These have requests set but are using MORE
  than their requested amount
  Kubernetes picks the pod using the most memory
  above its request value first
  "You asked for 128Mi but are using 400Mi — you go first"

Still not enough memory? →

ROUND 3 — Kill Guaranteed pods LAST
  These have requests = limits
  They told Kubernetes exactly what they need
  and are not exceeding it
  Kubernetes respects this and evicts them last

LimitRange vs ResourceQuota — The Difference
People always confuse these two. They solve different problems.

LimitRange says:
"Every single container in this namespace
 MUST follow these rules:

 - Default CPU if not set: 200m
 - Default memory if not set: 256Mi
 - Maximum CPU any one container can request: 1 core
 - Minimum CPU any one container must have: 50m"

Without LimitRange:
  Developer forgets to set resources → BestEffort pod
  → first to be evicted on memory pressure

With LimitRange:
  Developer forgets → LimitRange applies defaults automatically
  → pod gets Burstable QoS at minimum
  → protected from immediate eviction

Use LimitRange to: protect the cluster from poorly configured individual pods.

ResourceQuota — Controls the Whole Namespace
Think of it as a budget for the entire namespace.

ResourceQuota says:
"The ENTIRE production namespace
 cannot use more than:

 - Total CPU requests: 4 cores combined
 - Total memory requests: 8GB combined
 - Total pods: 20 pods maximum"

Without ResourceQuota:
  One team deploys 50 pods and uses all cluster memory
  Other teams' pods cannot be scheduled
  One namespace starves the entire cluster

With ResourceQuota:
  Each namespace has a hard budget
  Team A gets 4 cores and 8GB
  Team B gets 4 cores and 8GB
  Neither can exceed their allocation

LimitRange:
  Each pod must be between 50m-1 CPU
  Each pod defaults to 100m if not specified
  ↓
ResourceQuota:
  All pods in namespace combined cannot exceed 4 CPU
  Maximum 20 pods total

Why Different API Versions Exist
Kubernetes started with core objects. Over time new objects were added. They organised them into API groups.

Core group (no name) → apiVersion: v1
  These were the ORIGINAL Kubernetes objects
  Pod, Service, ConfigMap, Secret,
  Namespace, PersistentVolume, Node

apps group → apiVersion: apps/v1
  Added LATER for application management
  Deployment, ReplicaSet, StatefulSet, DaemonSet

autoscaling group → apiVersion: autoscaling/v2
  For scaling objects
  HorizontalPodAutoscaler

batch group → apiVersion: batch/v1
  For job-like objects
  Job, CronJob

networking group → apiVersion: networking.k8s.io/v1
  For network objects
  Ingress, NetworkPolicy

rbac group → apiVersion: rbac.authorization.k8s.io/v1
  For access control
  Role, ClusterRole, RoleBinding

apiVersion: v1
  → Pod
  → Service
  → ConfigMap
  → Secret
  → Namespace
  → PersistentVolume
  → PersistentVolumeClaim

apiVersion: apps/v1
  → Deployment
  → ReplicaSet
  → StatefulSet
  → DaemonSet

apiVersion: autoscaling/v2
  → HorizontalPodAutoscaler

apiVersion: batch/v1
  → Job
  → CronJob

apiVersion: networking.k8s.io/v1
  → Ingress

apiVersion: rbac.authorization.k8s.io/v1
  → Role
  → ClusterRole
  → RoleBinding

kubectl api-resources
# Shows EVERY object with its correct apiVersion

===========================================================
Day 7: Storage — Persistent Volumes and PVCs
============================================================

The Problem — Pods Lose Data on Restart
Pod running MySQL → stores data in /var/lib/mysql
Pod crashes → Kubernetes restarts it
New pod starts → /var/lib/mysql is EMPTY
All data lost

Solution: store data OUTSIDE the pod on persistent storage.

The 3 Storage Objects

PersistentVolume (PV)
The actual storage resource. Created by admin or dynamically.
Think of it as: a disk/drive that exists in the cluster.

PV = the actual storage
     "There is a 10GB disk available"

PersistentVolumeClaim (PVC)
A request for storage by a pod. Pod claims a PV.
Think of it as: a pod saying "I need 5GB of storage".

PVC = request for storage
      "I need 5GB, give me a PV that matches"

StorageClass
Defines how storage is provisioned dynamically.
Think of it as: a template for creating storage on demand.
StorageClass = storage template
               "When someone needs storage,
                automatically create an EBS volume on AWS"

Admin creates PV (or StorageClass does it dynamically)
        ↓
Pod creates PVC saying "I need 5GB"
        ↓
Kubernetes matches PVC to a suitable PV
        ↓
PVC is Bound to PV
        ↓
Pod mounts PVC as a volume
        ↓
Data written to pod persists even after pod restarts

Volume Types You Must Know
emptyDir     → temporary, lives as long as pod
               deleted when pod is deleted
               use for: scratch space, sidecar sharing

hostPath     → mounts a directory from the NODE
               data survives pod restart
               dangerous in production (ties pod to node)
               use for: single-node clusters, Minikube testing

persistentVolumeClaim → proper persistent storage
               survives pod restart AND pod deletion
               use for: databases, stateful apps

configMap/secret → inject config/secrets as files
               already covered in Day 5

PV Reclaim Policy
What happens to the PV when the PVC is deleted:
Retain   → PV stays, data preserved, manual cleanup needed
Delete   → PV and data deleted automatically
Recycle  → data wiped, PV made available again (deprecated)

PV Access Modes

ReadWriteOnce (RWO)
  → One node can mount it read-write
  → Most common for databases
  → AWS EBS uses this

ReadOnlyMany (ROX)
  → Many nodes can mount it read-only
  → Use for shared read-only data

ReadWriteMany (RWX)
  → Many nodes can mount it read-write
  → AWS EFS uses this
  → Use for shared file storage

