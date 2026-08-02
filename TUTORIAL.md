# Kubernetes, Start to Finish — A Self-Paced Hands-On Tutorial

This is a step-by-step, hands-on walkthrough of Kubernetes for absolute
beginners. It assumes **zero prior Kubernetes knowledge** — but it does
assume you're comfortable with Docker and the command line, so container
basics ("what is an image," "what is `docker run`") are not re-explained
here.

**How to use this file:** work through it top to bottom, in order — each
module builds on the last. Every command is written exactly as you'll type
it; copy/paste is fine, nothing to improvise. Each module ends with a
**✅ Checkpoint** — a description of exactly what should be true if
everything worked. If you're coming back to this after a break, skim the
checkpoint of the last module you completed, run its verification command,
and pick up from the next module.

If a command's output doesn't match what's shown, don't skip past it — the
"if this goes wrong" notes are written from real failure modes, not
hypothetical ones.

**Where this is going:** by the end, you'll have manually deployed a Pod, a
self-healing Deployment, a networked Service, externalized config via
ConfigMap/Secret, and finally a full two-tier app (Node.js web app +
MongoDB) — all running on a local cluster on your machine.

**Further reading:** once a module clicks, the companion docs in
[`docs/`](./docs/) go deeper on the same topic —
[`docs/kubernetes-fundamentals-notes.md`](./docs/kubernetes-fundamentals-notes.md)
for CKA/CKAD-style depth, and
[`docs/kubernetes-crash-course-study-guide.md`](./docs/kubernetes-crash-course-study-guide.md)
for the same capstone app from a slightly different angle. Each module below
links to the relevant section.

---

## Timing overview

| Module | Topic | ~Time |
|---|---|---|
| 0 | Setup & Orientation | 10 min |
| 1 | Pods & the Declarative Model | 15 min |
| 2 | Deployments & Self-Healing | 15 min |
| 3 | Services & Networking | 15 min |
| 4 | Config & Secrets | 15 min |
| 5 | Capstone: Web App + MongoDB | 25 min |
| 6 | Cleanup & Where to Go Next | 5 min |

---

## Module 0 — Setup & Orientation

### Why this matters

Kubernetes manages containers across a *cluster* of machines. Running a real
multi-machine cluster just to learn the basics would be overkill — Minikube
gives you a single-machine stand-in that behaves like a real cluster, so
every command you learn here works unchanged against a real production
cluster later.

### Before you start

Confirm Docker is actually running first — Minikube's Docker driver depends
on it, and if Docker is down, `minikube start` will hang or fail with a
confusing error.

```bash
docker ps
```

You should see a table header (`CONTAINER ID   IMAGE   ...`), even if it's
empty. If this errors out, start Docker Desktop and wait for it to say
"Docker is running" before continuing.

### Install Minikube and kubectl

```bash
brew install minikube
```

`kubectl` (the CLI you'll use to talk to *any* Kubernetes cluster, local or
cloud) is installed automatically alongside Minikube. Confirm both:

```bash
minikube version
kubectl version --client
```

### Start your cluster

```bash
minikube start --driver=docker
```

This downloads a Kubernetes node image (first run only — can take a few
minutes) and boots a single-node cluster inside a Docker container on your
machine.

**Expected output (last line):** `Done! kubectl is now configured to use "minikube" cluster...`

### Verify

```bash
minikube status
kubectl get nodes
```

**Expected output:**

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   1m    v1.3x.x
```

`kubectl get nodes` talking to your cluster and getting back a `Ready` node
is the whole point of this module — it confirms `kubectl` is correctly
pointed at Minikube (via a config file at `~/.kube/config` it just wrote for
you) and the cluster is healthy.

### ✅ Checkpoint

- `minikube status` shows `host: Running`, `kubelet: Running`, `apiserver: Running`
- `kubectl get nodes` shows one node with `STATUS: Ready`

If you closed your laptop and came back later and this checkpoint fails,
run `minikube start` again — it's safe to re-run, it just resumes the
existing cluster instead of creating a new one.

*Deeper reading: [fundamentals-notes.md §1–2](./docs/kubernetes-fundamentals-notes.md#1-why-kubernetes) on why Kubernetes exists and what each control-plane piece does.*

---

## Module 1 — Pods & the Declarative Model

### Why this matters

A **Pod** is the smallest thing Kubernetes lets you deploy — a thin wrapper
around one or more containers. You will almost never create a bare Pod in
real usage (Module 2 explains why), but understanding it first is what makes
Deployments make sense later, instead of feeling like unexplained magic.

The other thing this module teaches is **declarative vs. imperative**:
instead of telling Kubernetes *how* to do something step by step, you write
a file describing the *end state you want*, and Kubernetes continuously
works to make reality match it.

### Write your first manifest

Create a working directory for these exercises:

```bash
mkdir -p ~/k8s-tutorial && cd ~/k8s-tutorial
```

Create a file named `pod.yaml` with exactly this content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    app: hello
spec:
  containers:
    - name: hello-container
      image: nginx:alpine
      ports:
        - containerPort: 80
```

**What each part means:**
- `apiVersion` / `kind` — tells Kubernetes which API handles this object type. `Pod` is case-sensitive — `pod` would fail.
- `metadata.name` — how you'll refer to this Pod in every other command.
- `metadata.labels` — a tag (`app: hello`). Doesn't do anything by itself yet, but Module 2 depends on it.
- `spec.containers` — a **list** (note the `-`) — a Pod *can* hold more than one container, though most hold exactly one.

### Apply it

```bash
kubectl apply -f pod.yaml
```

`apply` is the declarative command: "make the cluster match this file." Run
it again right now with no changes — it will say `unchanged`, because
reality already matches the file.

### Inspect it

```bash
kubectl get pods
```

**Expected output:**

```text
NAME        READY   STATUS    RESTARTS   AGE
hello-pod   1/1     Running   0          10s
```

- `READY 1/1` — one container defined, one container up. If you ever see
  `0/1`, the container isn't up yet or crashed — that's your first signal
  something's wrong.

For more detail (and the first place to look when something's broken):

```bash
kubectl describe pod hello-pod
```

Scroll to the **Events** section at the bottom — this is where Kubernetes
narrates exactly what it did to get this Pod running, in order.

### Prove it's really running nginx

```bash
kubectl port-forward pod/hello-pod 8080:80
```

Leave that running, and in a **new terminal tab**:

```bash
curl localhost:8080
```

You should get back nginx's default HTML welcome page. Go back to the first
tab and press `Ctrl+C` to stop the port-forward before continuing.

### The important part: kill it and watch

```bash
kubectl delete pod hello-pod
kubectl get pods
```

**Expected output:** `No resources found in default namespace.`

Notice: it did **not** come back. A bare Pod has no supervisor — once it's
gone, it's gone. Hold onto that observation; it's the exact problem Module 2
solves.

### ✅ Checkpoint

- You understand: `apply` = declarative, "make it look like this file"
- You've seen `kubectl get pods` / `describe pod` / `delete pod`
- You've confirmed a bare Pod does **not** self-heal

*Deeper reading: [fundamentals-notes.md §4](./docs/kubernetes-fundamentals-notes.md#4-anatomy-of-a-manifest) on manifest anatomy; [crash-course §3](./docs/kubernetes-crash-course-study-guide.md#3-core-kubernetes-components) on the Pod concept.*

---

## Module 2 — Deployments & Self-Healing

### Why this matters

Real workloads need to survive a crashed container, a deleted Pod, or a
node going away — without you manually re-running `kubectl apply` every
time. A **Deployment** is a supervisor for Pods: it doesn't run containers
directly, it manages a **ReplicaSet**, which is the thing that actually
keeps a set number of identical Pods alive at all times.

```text
Deployment → creates/manages → ReplicaSet → creates/manages → Pod(s)
```

### Write the manifest

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello-container
          image: nginx:alpine
          ports:
            - containerPort: 80
```

**What's new here vs. `pod.yaml`:** `spec.template` is a full Pod
specification nested inside the Deployment — same shape as Module 1's
`pod.yaml`, just indented one level deeper. `spec.selector.matchLabels` must
match `template.metadata.labels` exactly — that's the mechanism the
Deployment uses to recognize "these are my Pods."

### Apply it and watch it create 3 pods

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

**Expected output:** three pods, names like `hello-deploy-7d8f9c6b5-x7k2p`
— the random suffix means "created by a ReplicaSet," not typed by hand.

```bash
kubectl get all
```

Shows the Deployment, its ReplicaSet, and all 3 Pods in one view — a
literal picture of the hierarchy above.

### Prove self-healing

Pick one of the pod names from `kubectl get pods` output and delete it:

```bash
kubectl delete pod <paste-one-pod-name-here>
kubectl get pods
```

**Expected output:** you'll see the deleted pod disappear and a **brand
new** pod (new random suffix) appear almost immediately, and the total
count stays at 3 the whole time. This is the Deployment noticing "actual
state (2 pods) doesn't match desired state (3 pods)" and fixing it — the
same declarative reconciliation loop from Module 1, just automated
continuously instead of one-shot.

### Scale it

```bash
kubectl scale deployment hello-deploy --replicas=5
kubectl get pods
```

Watch two new pods appear. Scale back down:

```bash
kubectl scale deployment hello-deploy --replicas=3
```

### ✅ Checkpoint

- `kubectl get pods` shows 3 pods with `READY 1/1`, `RESTARTS 0`, all `Running`
- You've deleted a pod by hand and watched it get replaced automatically
- You understand the chain: Deployment → ReplicaSet → Pods

*Deeper reading: [fundamentals-notes.md §12](./docs/kubernetes-fundamentals-notes.md#12-deployments--replicasets) covers rolling updates and rollback, which this tutorial doesn't exercise directly.*

---

## Module 3 — Services & Networking

### Why this matters

Every Pod gets its own internal IP address — but Pods are disposable (you
just watched one get destroyed and replaced with a *different* IP in Module
2). Nothing that depends on a stable address to reach your app can use a
Pod IP directly. A **Service** is a permanent address that automatically
tracks whichever Pods currently match its label selector, no matter how
many times they're replaced.

### Expose the Deployment internally first

```bash
kubectl expose deployment hello-deploy --port=80 --target-port=80 --name=hello-service
kubectl get service hello-service
```

**Expected output:**

```text
NAME            TYPE        CLUSTER-IP     PORT(S)   AGE
hello-service   ClusterIP   10.x.x.x       80/TCP    5s
```

`ClusterIP` is the default type — reachable only from *inside* the cluster.
`port` is the Service's own port; `target-port` is the port the container
actually listens on (nginx listens on 80). Right now these happen to match,
but they don't have to.

Prove the Service survives pod churn — delete a pod again like you did in
Module 2. This grabs one pod's name automatically instead of you having to
copy-paste it:

```bash
POD_NAME=$(kubectl get pods -l app=hello -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD_NAME
kubectl get service hello-service
```

The `CLUSTER-IP` is unchanged, even though the pod behind it just got
replaced. That's the whole point of a Service.

### Now expose it externally

A `ClusterIP` Service can't be reached from your laptop's browser directly.
For that, change the type to `NodePort`, which opens a port in the
`30000–32767` range on the node itself:

```bash
kubectl delete service hello-service
kubectl expose deployment hello-deploy --port=80 --target-port=80 --type=NodePort --name=hello-service
kubectl get service hello-service
```

**Expected output:** a `PORT(S)` column like `80:31xxx/TCP` — that `31xxx`
number is your NodePort.

### Access it from your browser

Minikube runs inside Docker, not directly on your machine, so `localhost`
won't reach it — you need Minikube's own IP:

```bash
minikube service hello-service --url
```

This prints a URL (or opens it in your browser directly, depending on your
Minikube version). Visit it — you should see nginx's welcome page, now
served through a Service instead of a direct port-forward.

**If this doesn't load:** run `minikube ip` and manually visit
`http://<that-ip>:<the-nodeport-from-above>` in your browser.

### ✅ Checkpoint

- `kubectl get service hello-service` shows `TYPE: NodePort`
- You loaded the nginx welcome page in a browser via the Minikube URL, not via `port-forward`
- You understand: Service = stable address, Pod = disposable address behind it

*Deeper reading: [fundamentals-notes.md §14](./docs/kubernetes-fundamentals-notes.md#14-networking-services) covers `LoadBalancer` type and kube-proxy routing, which only matter on a real cloud cluster.*

---

## Module 4 — Config & Secrets

### Why this matters

Containers should be able to run unmodified in dev, staging, and prod — the
only thing that should differ is *configuration* (a database URL, a
feature flag) and *credentials* (a password). Baking either into your
container image means rebuilding the image every time a value changes, or
every time you promote to a new environment. ConfigMaps and Secrets solve
this by injecting values at deploy time instead.

### Create a ConfigMap

```bash
kubectl create configmap hello-config --from-literal=GREETING=hello-from-configmap
kubectl get configmap hello-config -o yaml
```

You'll see your key/value stored as a cluster object, independent of any
Pod.

### Create a Secret

Secrets work the same way, but Kubernetes stores the value **base64
encoded** — this is *encoding*, not encryption. Anyone with API access can
trivially decode it. That's fine for this tutorial; production setups need
a real secrets-encryption solution on top, which is out of scope here.

```bash
kubectl create secret generic hello-secret --from-literal=API_KEY=super-secret-value
kubectl get secret hello-secret -o jsonpath='{.data.API_KEY}' | base64 -d
```

That last command proves the round-trip: `super-secret-value` should print
back out.

### Inject both into a Pod as environment variables

Create `configured-pod.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configured-pod
spec:
  containers:
    - name: hello-container
      image: nginx:alpine
      env:
        - name: GREETING
          valueFrom:
            configMapKeyRef:
              name: hello-config
              key: GREETING
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: hello-secret
              key: API_KEY
```

```bash
kubectl apply -f configured-pod.yaml
kubectl exec configured-pod -- printenv GREETING API_KEY
```

**Expected output:**

```text
hello-from-configmap
super-secret-value
```

The container never had these values baked in — they were injected purely
from cluster objects at startup.

### If this goes wrong

The single most common mistake here: a typo in `key:` (must exactly match
what you passed to `--from-literal`) or in `name:` (must exactly match the
ConfigMap/Secret's own name). The failure mode looks like this:

```bash
kubectl get pods
# STATUS: CreateContainerConfigError
kubectl describe pod configured-pod
# Events section will say something like:
# "couldn't find key GREETING in ConfigMap default/hello-config"
```

`describe pod` → **Events** is always your first move when a pod won't
start. This is deliberately worth breaking once on purpose if you want the
error message to stick — try changing `key: GREETING` to `key: GREETIN` and
re-apply to see it.

### ✅ Checkpoint

- `kubectl exec configured-pod -- printenv GREETING API_KEY` prints both values correctly
- You've seen a `CreateContainerConfigError` and used `describe pod` to diagnose it

*Deeper reading: [fundamentals-notes.md §9](./docs/kubernetes-fundamentals-notes.md#9-configmaps--secrets) also covers mounting config as files via volumes, which this tutorial skips in favor of env vars.*

---

## Module 5 — Capstone: Web App + MongoDB

### Why this matters

Everything so far has been one container at a time. Real applications are
usually *systems* — here, a stateless web app that depends on a stateful
database. This module combines every concept from Modules 1–4 into one
working two-tier deployment: ConfigMap + Secret (Module 4) feeding into a
MongoDB Deployment + Service (Modules 2–3), which a web app Deployment +
Service (Modules 2–3) connects to.

### Clean up the practice objects first

```bash
kubectl delete pod hello-pod configured-pod --ignore-not-found
kubectl delete deployment hello-deploy --ignore-not-found
kubectl delete service hello-service --ignore-not-found
kubectl delete configmap hello-config --ignore-not-found
kubectl delete secret hello-secret --ignore-not-found
```

This isn't strictly required — different names would coexist fine — but it
keeps `kubectl get all` readable while you build the capstone.

### The plan

Four files, applied in dependency order — database config must exist
*before* the app that needs it:

1. `mongo-secret.yaml` — MongoDB root username/password
2. `mongo.yaml` — MongoDB Deployment (1 replica) + internal Service
3. `webapp-config.yaml` — ConfigMap with the Mongo connection URL
4. `webapp.yaml` — web app Deployment + external Service

### 1. MongoDB credentials

Encode a username and password:

```bash
echo -n 'admin' | base64
echo -n 'changeme123' | base64
```

Create `mongo-secret.yaml`, pasting your two base64 values in:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
type: Opaque
data:
  mongo-user: <paste-base64-username-here>
  mongo-password: <paste-base64-password-here>
```

### 2. MongoDB Deployment + internal Service

Create `mongo.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
        - name: mongo
          image: mongo:6
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-user
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongo-secret
                  key: mongo-password
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
spec:
  selector:
    app: mongo
  ports:
    - port: 27017
      targetPort: 27017
```

Note: no `type:` on this Service means it defaults to `ClusterIP` —
correct here, since only the web app (inside the cluster) needs to reach
MongoDB, never your browser directly.

### 3. Web app config

Create `webapp-config.yaml`. The URL embeds the same credentials from
`mongo-secret.yaml` and points at `mongo-service` — Kubernetes gives every
Service a DNS name matching its `metadata.name`, so `mongo-service` resolves
automatically from any Pod in the same namespace:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  MONGO_URL: "mongodb://admin:changeme123@mongo-service:27017"
```

(In a real project, the credentials wouldn't be duplicated in plaintext
here — they'd be composed from the Secret at the app level. Keeping it
inline keeps this capstone's YAML count manageable; the crash-course study
guide's demo does this same simplification.)

### 4. Web app Deployment + external Service

Create `webapp.yaml`. This uses a small pre-built demo image that serves a
page confirming its Mongo connection status — swap `image:` for your own
app later:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: nginx:alpine
          ports:
            - containerPort: 80
          env:
            - name: MONGO_URL
              valueFrom:
                configMapKeyRef:
                  name: webapp-config
                  key: MONGO_URL
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
```

> **Note:** this uses `nginx:alpine` again (not a real Mongo-aware app) so
> the capstone stays runnable with zero extra image-building steps. It
> proves the full wiring — Secret → Mongo → ConfigMap → web app → Service —
> even though nginx itself won't actually query Mongo. If you want to prove
> a *real* app-to-database round trip, swap `webapp.yaml`'s image for any
> small Node/Express + Mongoose app you have, using `MONGO_URL` exactly as
> defined here.

### Deploy in order

```bash
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo.yaml
kubectl apply -f webapp-config.yaml
kubectl apply -f webapp.yaml
```

### Verify the whole system

```bash
kubectl get all
```

**Expected output:** one `mongo-deploy` pod, two `webapp-deploy` pods, all
`Running`, plus `mongo-service` (ClusterIP) and `webapp-service`
(NodePort).

Confirm the web app can actually see its Mongo connection string:

```bash
kubectl exec deploy/webapp-deploy -- printenv MONGO_URL
```

**Expected output:** `mongodb://admin:changeme123@mongo-service:27017`

Load it in a browser:

```bash
minikube service webapp-service --url
```

### If this goes wrong

- Web app pod stuck in `CreateContainerConfigError` → check
  `webapp-config.yaml`'s `key:` matches exactly (Module 4's exact failure
  mode).
- `kubectl exec deploy/mongo-deploy -- printenv MONGO_INITDB_ROOT_USERNAME`
  prints nothing or wrong value → check `mongo-secret.yaml`'s base64 values
  were pasted correctly (no trailing newline — `echo -n` matters).
- Everything shows `Running` but the app can't reach Mongo → confirm
  `mongo-service` exists (`kubectl get service mongo-service`) *before* the
  web app pods started; if you applied out of order, delete and re-apply
  `webapp.yaml` after confirming Mongo is up.

### ✅ Checkpoint

- `kubectl get all` shows 3 pods total (1 mongo + 2 webapp), all `Running`, `0` restarts
- `kubectl exec deploy/webapp-deploy -- printenv MONGO_URL` prints the full connection string
- You've deployed in dependency order and understand *why* order mattered

*Deeper reading: [crash-course-study-guide.md §6](./docs/kubernetes-crash-course-study-guide.md#6-hands-on-demo-deploying-a-web-app--mongodb) walks the same demo from a slightly different narrative angle — useful as a second pass.*

---

## Module 6 — Cleanup & Where to Go Next

### Tear down what you built

To remove just the capstone's resources but keep your cluster running for
next time:

```bash
kubectl delete -f webapp.yaml -f webapp-config.yaml -f mongo.yaml -f mongo-secret.yaml
```

To stop the cluster entirely (keeps it on disk, fast to resume):

```bash
minikube stop
```

To delete the cluster completely (next `minikube start` rebuilds from
scratch, slower):

```bash
minikube delete
```

Use `stop`, not `delete`, if you plan to come back and re-run any part of
this tutorial later.

### You now know

- Pods, and why they're never used bare in practice
- Deployments, ReplicaSets, self-healing, and scaling
- Services, and the difference between `ClusterIP` and `NodePort`
- ConfigMaps and Secrets for externalized configuration
- How to deploy a multi-component system in dependency order

### Suggested next steps

1. Re-run the capstone from memory, without copy-pasting — the parts that
   feel shaky are exactly what to re-read.
2. Read [`docs/kubernetes-fundamentals-notes.md`](./docs/kubernetes-fundamentals-notes.md)
   in full — it covers Namespaces, resource requests/limits, liveness/readiness
   probes, PersistentVolumes, and NetworkPolicies, none of which this
   tutorial touched.
3. Read [`docs/kubernetes-crash-course-study-guide.md`](./docs/kubernetes-crash-course-study-guide.md)
   for the same capstone app summarized as reference notes, plus a
   kubectl cheat sheet.
4. Try rolling updates: change `webapp.yaml`'s image tag and `kubectl apply`
   again — watch `kubectl rollout status deployment/webapp-deploy` and then
   `kubectl rollout undo deployment/webapp-deploy`.
5. Once comfortable, look into Helm (a package manager for Kubernetes
   manifests) as the natural next tool to learn.
