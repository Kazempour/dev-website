---
title: "Understanding Kubernetes"
date: 2026-01-10
description: "A practical deep dive into Kubernetes — what the control plane actually does, the objects you'll touch daily, and how to think about workloads."
---

Kubernetes gets a bad reputation for complexity, and some of it is earned. But most of the
intimidation comes from meeting it at the wrong layer. This post is a mental model: what
Kubernetes *is*, what each major piece *does*, and how to reason about your workloads without
drowning in YAML.

## What problem does it solve?

At its core, Kubernetes is a **scheduler and reconciler for containers**. You tell it the
desired state ("run 3 copies of this image, expose port 8080"), and a control loop keeps
reality matching that state. Node dies? It reschedules. Traffic spikes? It scales (if you ask).
Config changes? It rolls out without downtime.

That reconciliation loop — *observe current state, compare to desired state, act to close the
gap* — is the single idea underneath everything else.

## The control plane

The control plane is the brain. Its components:

- **API server** — the front door. Every `kubectl` command is an HTTP call to it. It's the only
  component that talks to etcd.
- **etcd** — the strongly-consistent datastore. The entire cluster state lives here. Back it up.
- **Scheduler** — watches for unscheduled pods and picks a node for each, based on resources,
  affinity, and taints/tolerations.
- **Controller managers** — the reconcilers. Deployment controller, node controller, endpoint
  controller, and friends. Each owns one resource type's "make reality match spec" loop.
- **kubelet** — the agent on every node. Talks to the API server, runs pods via the container
  runtime (containerd/runtime shim).
- **kube-proxy** — maintains node networking rules so Services route to the right pods.

## The objects you'll actually use

You don't need to memorize the full API. Daily drivers:

- **Pod** — one or more containers that share network + storage. The smallest deployable unit.
  (You rarely create pods directly — you create something that creates pods.)
- **Deployment** — declares *how many* replicas of a pod and *which image*. Gives you rolling
  updates and rollback for free.
- **Service** — stable networking + load balancing to a set of pods (by label selector). ClusterIP
  (internal), NodePort, or LoadBalancer (external).
- **ConfigMap / Secret** — config and credentials, mounted as files or env vars. Don't bake
  secrets into images.
- **Ingress** — HTTP routing rules: "path `/api` → Service X, host `app.example` → Service Y."
  Sits in front of Services.
- **Namespace** — a scoping boundary for names + RBAC. Not a network boundary (that's NetworkPolicy).

## A useful mental model

```
Ingress  ──►  Service  ──►  (Pods from a) Deployment
                                  │
                          controlled by the
                       control plane's reconcilers
```

Requests flow *down* (Ingress → Service → Pod). Desired state flows *up* (you write a
Deployment; the controller makes pods; the scheduler places them; kubelet runs them).

## Where people get stuck

1. **Confusing Deployment and Pod.** Edit the Deployment's image; don't `kubectl edit pod` (it'll
   be replaced).
2. **Forgetting probes.** Liveness + readiness probes are how Kubernetes knows a pod is *actually*
   serving, not just *started*. Without them, traffic lands on broken pods.
3. **Secrets hygiene.** Enable encryption-at-rest for etcd, use an external secret store (Sealed
   Secrets, External Secrets Operator, or your cloud's KMS) rather than committing YAML.
4. **Resource requests/limits.** Set requests so the scheduler places pods sanely; set limits so
   one bad pod can't eat the node.

## When *not* to use it

Kubernetes is a platform, not a product. If you have one small service with stable traffic, a
single container on a managed service (or even a PaaS) is simpler and cheaper. Reaching for
Kubernetes because it's fashionable is how teams end up operating a distributed system they don't
need. Adopt it when you have *multiple* services, *multiple* teams, or real scaling/orchestration
needs.

## Closing thought

Learn it from the reconciliation loop outward. Once "observe → compare → act" clicks, every
object — HorizontalPodAutoscaler, CronJob, Operator — is just another specialized reconciler. The
rest is YAML.
