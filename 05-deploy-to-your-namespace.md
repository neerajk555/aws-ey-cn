# Exercise 5: Deploy to Your Namespace

## What This Exercise Does

A **Deployment** manages a set of identical Pods, giving you self-healing (a deleted Pod gets automatically replaced) and controlled scaling - properties a bare Pod doesn't have on its own. A **Service** gives those Pods a single, stable network address, since individual Pod IPs change every time a Pod is replaced. This exercise creates both inside your own namespace on the real shared cluster, then proves self-healing actually works by deliberately deleting a Pod and watching Kubernetes replace it - the same behavior you'd see on any Kubernetes cluster, including a free local one, now happening on real, billed infrastructure.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.00-0.02 (a couple of small Pods running briefly on already-provisioned shared nodes)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 4 complete - your namespace context is set and verified

## Steps

## Step 1: Create a Deployment

```
kubectl create deployment web --image=nginx:1.27 --replicas=2
```

This creates a Deployment, which in turn creates a ReplicaSet, which in turn creates the actual Pods - a three-layer chain you can inspect with `kubectl get deployments,replicasets,pods`.

## Step 2: Expose it with a Service

```
kubectl expose deployment web --port=80
```

This creates a ClusterIP Service - reachable from inside the cluster only, by the stable name `web`, regardless of which specific Pods are currently backing it.

## Step 3: Confirm both are running

```
kubectl get deployments,pods,svc
```

## Step 4: Prove self-healing: delete a Pod directly and watch it get replaced

```
kubectl get pods
# copy one pod name from the output above, then:
kubectl delete pod <paste-one-pod-name-here>
kubectl get pods -w
```

Press Ctrl+C once you see a replacement Pod reach Running. A brand new Pod (with a different auto-generated name) should appear within seconds - this is the ReplicaSet noticing actual Pod count (1) no longer matches desired count (2), and fixing it automatically.

## Verify It Worked

- `kubectl get pods` shows 2 Pods Running, one of which has a different name than the one you deleted
- `kubectl get svc web` shows a ClusterIP address

## Common Mistakes

- **Deleting a Pod and expecting it to just be gone** - That's the whole point of a Deployment - a bare Pod would stay deleted, but a Deployment-managed Pod gets automatically replaced. If you want it gone for good, delete the Deployment itself, not an individual Pod.
- **Trying to reach the Service from outside the cluster with curl on your own machine** - A ClusterIP Service is only reachable from INSIDE the cluster - Exercise 6 covers how to expose something externally.

## Cleanup (do this now, not later)

```
kubectl delete deployment web
kubectl delete service web
```

Then complete the mandatory checklist: **`cleanup-checklists/05-cleanup-checklist.md`**.
