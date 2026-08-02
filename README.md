# Kubernetes Self-Learn

A self-paced path for learning Kubernetes from zero, one hands-on step at a
time.

- **[`TUTORIAL.md`](./TUTORIAL.md)** — start here. A step-by-step,
  copy-paste-able walkthrough for absolute beginners: install Minikube, run
  your first Pod, learn Deployments and Services, externalize config, and
  finish by deploying a two-tier web app + MongoDB system. Each section ends
  with a checkpoint so you can pick back up later without re-reading
  everything above it.
- **[`docs/`](./docs/)** — reference notes for going deeper once a concept
  from the tutorial clicks:
  - [`kubernetes-fundamentals-notes.md`](./docs/kubernetes-fundamentals-notes.md) —
    broader CKA/CKAD-style reference: namespaces, resource limits,
    probes, storage, network policies.
  - [`kubernetes-crash-course-study-guide.md`](./docs/kubernetes-crash-course-study-guide.md) —
    the same web app + MongoDB capstone summarized as study notes, plus a
    kubectl cheat sheet.

## Prerequisites

Comfortable with Docker and the command line. No prior Kubernetes knowledge
assumed — `TUTORIAL.md` explains every concept before using it.

## Quick start

```bash
brew install minikube
minikube start --driver=docker
kubectl get nodes
```

Then open [`TUTORIAL.md`](./TUTORIAL.md) and start at Module 0.
