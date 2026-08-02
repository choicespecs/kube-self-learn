# Kubernetes Fundamentals — Study Notes
*Organized from "Zero to Hero Kubernetes Fundamentals" course (CKA/CKAD prep)*

---

## 1. Why Kubernetes?

- Apps today are built as **microservices**: code + dependencies packaged into an isolated **container**.
- Containers are great because they're: easy to spin up, easy to replace/update, portable ("works on my machine" → ship the whole environment), and scalable (make many copies).
- **The problem without Kubernetes:** containers run on top of a host (e.g., a VM), and manual container management breaks down in 3 ways:
  1. **Host runs out of resources** — containers can't scale further.
  2. **Containers die and get ignored** — nothing restarts them automatically.
  3. **Host machine dies** — every container on it dies too, with nothing to bring them back.
- **Kubernetes solves all three** by automating container lifecycle, scaling, and recovery across many machines.

---

## 2. Core Architecture

| Concept | What it is |
|---|---|
| **Container** | Same as always (Docker/containerd) — holds your app + dependencies |
| **Pod** | A Kubernetes wrapper *around* one or more containers. **Kubernetes manages pods, not containers directly.** |
| **Node** | A VM (or machine) that hosts pods. Nodes provide the CPU/memory resources pods consume. |
| **Cluster** | A group of nodes working together = combined pool of resources |

### Two types of nodes:
- **Worker nodes** — where the majority of your pods actually run.
- **Controller (control plane) node** — runs the cluster's "brain." Avoid scheduling regular workload pods here; it needs its resources to manage the cluster.

### Control Plane Components (what lives on the controller node)
| Component | Role |
|---|---|
| **kubectl** | CLI tool installed on *your* machine, not the cluster. Talks to the cluster via the API server. Requires a **kubeconfig** file (cluster location + auth cert/key) to work. |
| **API server** | The front door — **all** communication in Kubernetes goes through it. Validates requests, talks to every node, and is the *only* component that talks directly to etcd. |
| **etcd** | A strongly-consistent distributed key-value store. Stores **all cluster state**: nodes, pods, running status, everything. Losing etcd = losing your cluster's memory. Treat with extreme care. |
| **Scheduler** | Decides *which node* a new pod goes on (tries to balance load + obey any placement rules). |
| **Controller manager** | The "sheriff" — runs many background controllers that manage namespaces, RBAC, service accounts, replicas, etc. |
| **kubelet** | Runs on **every node** (including the controller). The "eyes and ears" of the node — starts/stops/restarts containers per instructions from the API server, and reports status back. If kubelet fails, that node is cut off from the cluster. |
| **Container runtime** (Docker/containerd/etc.) | Actually builds/runs containers. Kubelet issues the `docker run`-style commands — you never do this yourself. |

### How "kubectl create pod" actually flows
1. You run a `kubectl` command → sent to **API server**.
2. API server validates it, saves the plan to **etcd**.
3. API server asks the **scheduler** which node to use (unless you specified one).
4. Scheduler picks a node → API server records that decision in etcd.
5. API server tells **kubelet** on the chosen node to build it.
6. Kubelet runs the container engine commands → pod + container are created.
7. Kubelet continuously reports status back to the API server → etcd.

> **Note:** Pod is created first, container is created inside it second — never the other way around.

---

## 3. YAML Basics (manifests are written in YAML)

Two core data types:

- **Lists/sequences** — items prefixed with a dash `-`
  ```yaml
  - milk
  - eggs
  - bread
  ```
- **Dictionaries/mappings** — key:value pairs (order doesn't matter)
  ```yaml
  flavor: chocolate
  expires: June 30
  organic: true
  ```

**Indentation = ownership.** Anything indented under a key belongs to that key. Nesting lists inside dictionaries (and vice versa) is how complex manifests are built.

> 💡 **Tip:** Never write manifests from scratch — search kubernetes.io docs for the object type and copy/adapt the example manifest. Saves you from indentation and typo disasters.

---

## 4. Anatomy of a Manifest

Every manifest starts with:

```yaml
apiVersion: v1     # which API handles this object type — must match or you get errors
kind: Pod          # case-sensitive! "pod" ≠ "Pod"
metadata:
  name: nginx
spec:               # the actual specification/order
  containers:
    - name: nginx
      image: nginx  # must be spelled correctly or it won't be found
```

- `apiVersion` changes as Kubernetes versions change — upgrading a cluster can deprecate old API versions and break old manifests (check the **deprecated API migration guide** when upgrading).
- `spec.containers` is a **list** (note the dash) — a pod *can* hold multiple containers.

---

## 5. Essential kubectl Commands

| Command | Purpose |
|---|---|
| `kubectl get pods` | List objects (in current namespace only) |
| `kubectl get pods -n <namespace>` | List objects in a specific namespace |
| `kubectl get pods -A` | List across **all** namespaces |
| `kubectl get pods -L <label-key>` | Add a column showing a specific label's value |
| `kubectl apply -f file.yaml` | Create/update based on a manifest (declarative) |
| `kubectl create -f file.yaml` | Create from manifest |
| `kubectl describe pod <name>` | Deep-dive info + an **Events** section — first place to check when something's broken |
| `kubectl delete -f file.yaml` / `kubectl delete pod <name>` | Delete a resource |
| `kubectl logs <pod>` | View stdout/stderr of a container |
| `kubectl exec -it <pod> -- sh` | Open an interactive shell inside a pod |
| `kubectl cp <local> <pod>:<path>` | Copy files into/out of a pod |
| `kubectl port-forward <pod> <local-port>:<container-port>` | Tunnel a local port to a pod's port |
| `kubectl top nodes` / `kubectl top pods -A` | Resource usage (needs metrics-server installed first) |
| `kubectl label pod <name> key=value` | Add a label (use `--overwrite` to change, `key-` to remove) |
| `kubectl scale deployment <name> --replicas=3` | Change replica count |
| `kubectl rollout history deployment <name>` | View revision history |
| `kubectl rollout undo deployment <name>` | Roll back to previous revision |
| `kubectl expose <object>` | Quickly create a Service for a pod/deployment |

**Reading `kubectl get pods` output:**
- `READY 0/1` → container isn't up yet (or crashed)
- **Restarts > 0 repeatedly** → red flag, something's crash-looping

---

## 6. Namespaces

- Logical partitions inside a cluster (`default`, `kube-system`, custom ones like `demo`).
- `kube-system` holds cluster-internal pods (some control-plane components may literally run as pods here, depending on setup).
- Create imperatively: `kubectl create namespace demo`
- Force an object into a namespace by adding it under `metadata`:
  ```yaml
  metadata:
    name: mypod
    namespace: demo
  ```
- If you don't specify a namespace, objects go to whatever your **context**'s default is.
- **Secrets and ConfigMaps must live in the same namespace as the pods that use them.**

### Resource Quotas
Attach to a namespace to hard-cap total resource consumption:
```yaml
# example concept
spec:
  hard:
    cpu: "1"
    memory: 1Gi
```
Think of it as a nightclub bouncer — once the namespace hits its cap, new pods requesting more get **denied**.

---

## 7. Resource Requests & Limits (set per container)

```yaml
resources:
  requests:
    memory: "65Mi"
    cpu: "250m"      # "m" = millicore; 1000m = 1 full CPU core
  limits:
    memory: "130Mi"
    cpu: "500m"
```
- **Requests** = guaranteed minimum (so the container doesn't starve).
- **Limits** = hard cap (so the container doesn't hog everything). Limits must be ≥ requests.
- These interact with **ResourceQuotas** — if a new pod's requested resources would exceed the namespace's quota, pod creation is denied.

---

## 8. Probes (health checks)

Two types, set at the **container** level:

| Probe | What happens on failure |
|---|---|
| **Liveness probe** | After N consecutive failures → Kubernetes **kills and restarts** the container |
| **Readiness probe** | After N consecutive failures → pod is **removed from service traffic** (not killed) until it passes again |

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 2   # wait before first probe
  periodSeconds: 5         # how often to probe
  timeoutSeconds: 1        # how long to wait for a response
  failureThreshold: 3      # consecutive failures before action
```
Readiness probes use the exact same syntax — just placed as `readinessProbe`.

---

## 9. ConfigMaps & Secrets

**Problem:** containers are stateless/ephemeral — anything written inside a container disappears when it dies. Configuration needs to live *outside* the container.

### ConfigMaps
Store config data (files, env vars) as cluster objects, decoupled from containers.

```bash
kubectl create configmap dem-heroes --from-file=heroes.txt
```

To attach to a pod — **2 steps**:
1. Add the ConfigMap as a **volume** on the pod (`spec.volumes`).
2. **Mount** that volume into the container (`spec.containers[].volumeMounts`) at a chosen `mountPath`.

```yaml
spec:
  volumes:
    - name: dc-heroes
      configMap:
        name: dem-heroes     # note capital M — camelCase matters!
  containers:
    - name: nginx
      volumeMounts:
        - name: dc-heroes
          mountPath: /heroes
```

> ⚠️ **Gotcha:** mounting a volume to a directory that already has files in it will **overwrite/wipe that directory**. If you need to add just one file into an existing directory (e.g. `/etc/nginx`), use a **`subPath`**:
> ```yaml
> volumeMounts:
>   - name: dc-heroes
>     mountPath: /etc/nginx/heroes.txt
>     subPath: heroes.txt
> ```

### Secrets
Same mounting mechanics as ConfigMaps, but for sensitive data (passwords, tokens, keys).

- ⚠️ **Secrets are NOT encrypted by default** — anyone with API/etcd access can read them. You must add your own encryption solution for real security (not covered/needed for the exam).
- Common types: `Opaque` (generic), `kubernetes.io/basic-auth`, `kubernetes.io/ssh-auth`, etc.
- Typical use: inject as an **environment variable**:
  ```yaml
  env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: password
  ```

---

## 10. Logs

- Every container has its own stdout/stderr logs.
- `kubectl logs <pod>` — defaults to the **first container listed** in a multi-container pod.
- `kubectl logs <pod> -c <container>` — logs for a specific container.
- `kubectl logs <pod> --all-containers` — all containers at once.
- `kubectl logs <pod> -f` — stream/follow logs live.
- `kubectl logs <pod> --since=10s` — recent logs only.
- **Logs are impermanent** — when a pod is deleted, its logs are deleted too (kubelet cleans up `/var/log/pods/...` on the node). Use a third-party logging solution for persistence in production.

---

## 11. Labels

- Simple **key:value pairs** attached to almost any object (pods, nodes, etc.) — think of them like cattle ear tags.
- Used heavily by Deployments, Services, and Network Policies to identify *which* pods they should manage/target.
- Add/edit in the manifest under `metadata.labels`, or imperatively:
  ```bash
  kubectl label pod demo-pod env=prod
  kubectl label pod demo-pod env=prod --overwrite   # update existing key
  kubectl label pod demo-pod env-                   # remove a label
  ```
- Filter by label:
  ```bash
  kubectl get pods --selector app=nginx
  ```

---

## 12. Deployments & ReplicaSets

**Problem:** a standalone pod that dies doesn't come back. Pods are never meant to be created directly in production.

### The hierarchy
**Deployment → creates ReplicaSet → ReplicaSet creates Pods**

- The Deployment doesn't create pods directly — it manages a **ReplicaSet**, which is the thing that actually maintains the pod count.
- ReplicaSets alone were hard to use for rolling updates in production — **Deployments exist to layer version control on top.**

### Key behaviors
- **Scaling:** `kubectl scale deployment demo-deploy --replicas=3` (or edit `spec.replicas` in the manifest + `apply`).
- **Self-healing:** delete a pod that belongs to a Deployment → it's immediately recreated. Only way to truly remove them: delete the Deployment or scale to 0.
- **Selector matching:** Deployment's `spec.selector` must match the label(s) on the pod template — this is how it knows which pods belong to it. Remove that label from a pod manually → the Deployment thinks it's "missing" and creates a replacement (old pod becomes orphaned, no longer managed).

### Rolling updates (zero downtime)
When you change the pod template (e.g., new image), the Deployment:
1. Creates a **new** ReplicaSet.
2. **Surges** one new pod, then removes one old pod, repeating until fully swapped — the total pod count never dips below the desired count during the process.

```bash
kubectl rollout history deployment demo-deploy
kubectl rollout undo deployment demo-deploy   # roll back to previous ReplicaSet/revision
```

### Deployment manifest — what's different from a plain pod manifest
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deploy          # deployment-level metadata
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx              # must match template labels below
  template:                   # this whole block = pod manifest
    metadata:
      labels:
        app: nginx             # required so the deployment recognizes its pods
    spec:
      containers:
        - name: nginx
          image: nginx
```

---

## 13. Storage: StorageClass → PersistentVolume → PersistentVolumeClaim

**Problem:** containers are stateless and shouldn't hold real data. Storage must live outside the container/pod.

### The chain
```
StorageClass  →  PersistentVolume (PV)  →  PersistentVolumeClaim (PVC)  →  Volume on Pod  →  volumeMount on Container
```

| Object | Role |
|---|---|
| **StorageClass** | Teaches the cluster *how* to provision a type of storage (AWS EBS, Azure Disk, local/manual, etc.) — Kubernetes itself is storage-agnostic. |
| **PersistentVolume (PV)** | Represents an actual chunk of storage carved out per the StorageClass. |
| **PersistentVolumeClaim (PVC)** | The "key" that binds to a PV (the "locker") — **1:1 relationship**, one PVC binds to one PV. |
| **Volume + volumeMount** | Same 2-step process as ConfigMaps/Secrets — attach the PVC as a pod volume, then mount it into the container at a path. |

```yaml
# Simplified example using "manual" storage class (local node directory)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx-pv
spec:
  storageClassName: manual
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce      # only one node can mount it read-write at a time
  hostPath:
    path: /mnt/data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  volumes:
    - name: nginx-pv-storage
      persistentVolumeClaim:
        claimName: nginx-pvc
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: nginx-pv-storage
          mountPath: /data
  nodeSelector:
    kubernetes.io/hostname: node1   # force pod onto a specific node
```

- Data written to `/data` inside the container → persists at the PV's `hostPath` on the node → survives pod deletion/recreation, as long as the pod lands on the same node (or the storage is genuinely shared/networked).
- If a PVC requests less than what a PV offers, it may bind and just report the PV's larger size (nothing is "wasted"/split).

---

## 14. Networking: Services

**Problem:** every pod gets its own IP, but pod IPs constantly change (pods die, get recreated, get rescheduled). You can't rely on pod IPs directly.

**Solution:** a **Service** gives you a stable IP address / DNS name that always points to the current healthy set of matching pods (matched via label selector — same mechanism as Deployments).

```bash
kubectl expose deployment demo-deploy
```

Even if the deployment scales up/down or pods get replaced, the **Service's IP never changes** — it just automatically tracks the pods' current IPs.

### 3 Service Types (each builds on the previous)

| Type | Access | Notes |
|---|---|---|
| **ClusterIP** (default) | Internal only, pod-to-pod | Only reachable from inside the cluster |
| **NodePort** | External, via `<NodeIP>:<NodePort>` | Opens a port (30000–32767 range) on **every** node; works no matter which node you hit, thanks to kube-proxy |
| **LoadBalancer** | External, via a provisioned load balancer | Requires an external load balancer (cloud providers auto-provision one); makes smarter routing decisions than plain round-robin |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  type: NodePort            # or ClusterIP / LoadBalancer
  selector:
    app: nginx               # matches pod labels
  ports:
    - port: 3423              # port ON the service
      targetPort: 80          # port on the pod/container (nginx's real port)
      # nodePort: 30337        # auto-assigned if omitted, only valid for NodePort type
```

- `port` = the Service's own port.
- `targetPort` = the container's actual listening port.
- A Service can define multiple port mappings if a pod has multiple containers/ports.

### kube-proxy — the traffic router
- Runs on **every node**, maintains IP routing tables for all Services.
- When traffic targets a Service, kube-proxy picks one of the healthy matching pod IPs (**round-robin by default**) and routes there — it doesn't matter which node the client or target pod is on.
- Unhealthy pods (still starting/erroring) are skipped.

---

## 15. Network Policies

- Control **ingress** (incoming) and/or **egress** (outgoing) traffic between pods — like a strict "helicopter parent" for network access.
- Scoped to a **namespace**.
- Uses label selectors (`podSelector`) to decide which pods the policy applies to, and further selectors/IP ranges to decide who they can talk to.
- **Everything not explicitly allowed is denied** once a policy applies to a pod.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}       # empty = applies to ALL pods in the namespace
  policyTypes:
    - Ingress
    - Egress
  # no rules defined = deny everything (default-deny policy)
```

Example — only allow ingress from pods labeled `run: client`:
```yaml
spec:
  podSelector:
    matchLabels:
      app: demo-deploy
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              run: client
      ports:
        - port: 80
```

Other selector options inside `from`/`to` blocks: `ipBlock` (CIDR range), `namespaceSelector` (pods in namespaces with a certain label), `podSelector` (pods with a certain label) — these are combined with **OR** logic, so be precise.

---

## Quick-Reference: Object Relationship Map

```
Namespace
 ├── ResourceQuota (caps total resource use)
 ├── Deployment
 │     └── ReplicaSet
 │           └── Pod(s)
 │                 ├── Container(s)
 │                 │     ├── resources.requests / limits
 │                 │     ├── livenessProbe / readinessProbe
 │                 │     └── volumeMounts
 │                 └── volumes → ConfigMap / Secret / PersistentVolumeClaim
 ├── Service (ClusterIP / NodePort / LoadBalancer) → selects Pods by label
 └── NetworkPolicy → controls ingress/egress between Pods by label
```

---

## Study/Exam Tips Mentioned in the Video
- **CKA/CKAD exams** allow you to reference the official Kubernetes docs — always pull manifest examples from there rather than writing from scratch.
- **tmux** is available in the exam environment — useful for split terminals (e.g., watching `kubectl get pods` while editing a manifest).
- NodePort-related questions are considered highly likely to appear on the exam.
- When something breaks: **`kubectl describe`** first (check the Events section), then **`kubectl logs`** for the real error detail.
- Watch out for the classic YAML gotchas: indentation errors, case-sensitivity (`Pod` vs `pod`), and camelCase keys (`configMap` not `Configmap`).
