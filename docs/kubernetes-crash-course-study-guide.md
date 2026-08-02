# Kubernetes Crash Course — Study Guide

A structured breakdown of the "Kubernetes Crash Course" video, organized for self-study. Work through it top to bottom — each section builds on the last.

---

## 1. What Is Kubernetes & Why It Exists

**Definition:** Kubernetes (K8s) is an open-source **container orchestration framework**, originally built by Google. It manages containers (Docker or otherwise) across physical machines, VMs, cloud, or hybrid environments.

**The problem it solves:**
- Microservices architecture → apps split into hundreds/thousands of small containers
- Managing that many containers manually (scripts, homemade tools) becomes unmanageable
- Container orchestration tools emerged to solve this at scale

**What orchestration tools like Kubernetes guarantee:**
| Feature | What it means |
|---|---|
| **High availability** | App has no downtime, always accessible |
| **Scalability** | Scale up/down fast based on load |
| **Disaster recovery** | Backup + restore mechanisms if infrastructure fails |

---

## 2. Kubernetes Architecture

A cluster = **1+ master nodes** + **multiple worker nodes**.

### Worker Nodes
- Run your actual application containers (the real workload)
- Each has a **kubelet** process — lets the node communicate with the cluster and execute tasks
- Usually need more resources (CPU/memory) than master nodes since they run many containers

### Master Node Processes
| Process | Role |
|---|---|
| **API Server** | Entry point to the cluster — all clients (UI, API, CLI) talk to it |
| **Controller Manager** | Watches cluster state; detects and fixes problems (e.g., restarts dead containers) |
| **Scheduler** | Decides which worker node a new container should run on, based on available resources |
| **etcd** | Key-value store holding the current state/config of the whole cluster — the "cluster brain." Backups are restored from etcd snapshots |

### Virtual Network
Connects all nodes so the cluster behaves like one large machine with pooled resources.

> **Production tip:** Losing your only master = losing cluster access. Production clusters should have **at least 2 masters**.

---

## 3. Core Kubernetes Components

Build understanding using this mental model: a simple **web app + database** setup.

### 🔹 Pod
- Smallest deployable unit — an abstraction layer on top of a container
- Usually **1 main container per pod** (plus optional helper/sidecar containers)
- Each pod gets its own **internal IP address**
- **Pods are ephemeral** — they can die and get recreated, and they get a **new IP** each time

### 🔹 Service
- A **permanent/static IP** attached to a pod (or group of pods)
- Solves the "pod IP keeps changing" problem — service lifecycle ≠ pod lifecycle
- Two types:
  - **Internal service** — only reachable inside the cluster (e.g., database)
  - **External service** — reachable from outside (e.g., web app)
- Also acts as a **load balancer**, routing traffic to the least-busy replica

### 🔹 Ingress
- Sits in front of services; routes external traffic to the right service
- Enables clean URLs (domain name + HTTPS) instead of `http://<node-ip>:<port>`

### 🔹 ConfigMap
- Stores **external configuration** (e.g., a database URL) outside the app image
- Change config without rebuilding/redeploying your application image

### 🔹 Secret
- Like ConfigMap, but for **sensitive data** (passwords, credentials, certs)
- Stored in **base64**, not plaintext — ⚠️ base64 is *encoding*, not encryption. Use a third-party tool to actually encrypt secrets in production.

### 🔹 Volumes
- Attaches **persistent storage** (local disk or remote/cloud storage) to a pod
- Ensures data survives pod restarts
- **Kubernetes does NOT manage backup/replication of this data** — that's on you

### 🔹 Deployment
- A **blueprint** for pods — defines the container image + how many replicas to run
- In practice, you work with Deployments, not raw Pods
- Enables replication (multiple copies across nodes) → no downtime if one pod/node dies

### 🔹 StatefulSet
- Like Deployment, but for **stateful apps** (databases: MySQL, MongoDB, Elasticsearch)
- Manages synchronized reads/writes to shared storage across replicas
- More complex to run than Deployments — many teams host databases **outside** the K8s cluster and only run stateless apps (via Deployments) inside it

---

## 4. YAML Configuration Files

All cluster configuration is sent to the **API server** as YAML or JSON — this is how you talk to Kubernetes.

**Declarative model:** You declare the *desired state*; Kubernetes continuously works to match the *actual state* to it (this is the basis of self-healing — e.g., if a replica dies, the Controller Manager notices and recreates it).

### Every config file has 3 parts:
1. **Metadata** — name, labels, etc.
2. **Specification (spec)** — the configuration specific to that component type
3. **Status** — auto-generated & continuously updated by Kubernetes (sourced from etcd)

### Key concepts inside a Deployment YAML:
- `template` — defines the pod blueprint (its own metadata + spec) nested inside the Deployment
- `labels` — key/value tags on components (required for pods, optional but recommended elsewhere); convention is to use `app: <name>`
- `selector.matchLabels` — tells the Deployment (or Service) which pods belong to it
- `replicas` — how many pod copies to run

### Connecting Secret/ConfigMap values into a Deployment:
```yaml
env:
  - name: MONGO_INITDB_ROOT_USERNAME
    valueFrom:
      secretKeyRef:
        name: mongodb-secret
        key: mongo-user
  - name: DB_URL
    valueFrom:
      configMapKeyRef:
        name: mongodb-configmap
        key: mongo-url
```

### Service types:
| Type | Use case |
|---|---|
| `ClusterIP` (default) | Internal only |
| `NodePort` | Exposes app externally on a port in range **30000–32767**, via `<node-ip>:<nodePort>` |

> **Practice tip:** Store YAML config files alongside your application code (infrastructure-as-code), or in a dedicated repo.

---

## 5. Local Setup: Minikube + kubectl

- **Minikube** = a one-node cluster (master + worker processes on a single machine) — for local testing, not production
- Minikube needs a driver (VM or container) to run on — **Docker** is the recommended driver on all OSes
- **kubectl** = the command-line tool used to interact with *any* Kubernetes cluster (local or cloud) — installed automatically alongside Minikube

### Install & start (macOS example):
```bash
brew install minikube
minikube start --driver=docker
minikube status
kubectl get node
```

---

## 6. Hands-On Demo: Deploying a Web App + MongoDB

**Goal:** Deploy a Node.js web app connected to MongoDB, using ConfigMap + Secret for external config, exposed via NodePort.

### Files to create:
1. **`config.yaml`** — ConfigMap with the MongoDB service URL
2. **`secret.yaml`** — Secret with base64-encoded MongoDB username/password
   ```bash
   echo -n 'username' | base64
   echo -n 'password' | base64
   ```
3. **`mongo.yaml`** — MongoDB Deployment (1 replica) + internal Service
4. **`webapp.yaml`** — Web app Deployment + external Service (`NodePort`)

### Deployment order (dependencies matter):
```bash
kubectl apply -f config.yaml
kubectl apply -f secret.yaml
kubectl apply -f mongo.yaml     # DB must be up before the app needs it
kubectl apply -f webapp.yaml
```

### Access the app:
```bash
minikube ip                     # get node IP
# visit http://<minikube-ip>:<nodePort>  e.g. :30100
```

---

## 7. kubectl Command Cheat Sheet

| Command | Purpose |
|---|---|
| `kubectl apply -f <file>.yaml` | Create/update a component from a config file |
| `kubectl get all` | List deployments, pods, services in the cluster |
| `kubectl get pod` / `get service` / `get configmap` / `get secret` | List a specific component type |
| `kubectl get node -o wide` | List nodes with extra detail (IPs, etc.) |
| `kubectl describe <component> <name>` | Detailed info about one instance (e.g. `kubectl describe pod webapp-xxxx`) |
| `kubectl logs <pod-name>` | View container logs |
| `kubectl logs -f <pod-name>` | Stream logs live |
| `kubectl help` | Full command reference |

---

## 8. Big-Picture Summary

```
Client (UI / API / kubectl)
        │
        ▼
   API Server (master)
        │
 ┌──────┴───────┐
 │ Controller Mgr│  → watches desired vs actual state
 │ Scheduler      │  → places pods on nodes
 │ etcd           │  → stores cluster state
 └──────┬────────┘
        ▼
   Worker Nodes (kubelet)
        │
   ┌────┴─────┐
   │  Pods     │ ← created via Deployments/StatefulSets
   │ (containers)
   └────┬─────┘
        │
  Services (stable IP + load balancing)
        │
     Ingress (routing + domain/HTTPS)
        │
   External Users
```

**Config & Secrets** plug into pods via env vars.
**Volumes** attach persistent storage to pods (independent of the cluster's own lifecycle).

---

## Suggested Next Steps for Self-Study
1. Install Minikube + kubectl and replicate the hands-on demo yourself from scratch (don't copy-paste — type it out).
2. Intentionally kill a pod (`kubectl delete pod <name>`) and watch Kubernetes self-heal it.
3. Practice scaling a Deployment: `kubectl scale deployment <name> --replicas=3`.
4. Read the official Kubernetes docs for Deployments, Services, and StatefulSets to see full config options beyond what's covered here.
5. Once comfortable, look into Helm (package manager for K8s) and readiness/liveness probes as next-level topics.
