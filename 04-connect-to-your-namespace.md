# Exercise 4: Connect to Your Namespace

## What This Exercise Does

A **namespace** divides one Kubernetes cluster into logical sections - your instructor has already created one namespace per participant on the shared cluster (yours is named `ns-$PARTICIPANT`), each with its own **ResourceQuota** (a cap on how much CPU/memory/Pods you can use) and **RBAC** binding (so your IAM identity can only act inside your own namespace, never anyone else's). This exercise confirms your access is scoped correctly - both that you CAN work in your own namespace, and that you genuinely CANNOT see or touch anyone else's, which is the real security property this whole design exists to guarantee.

## Time and Cost

- **Time:** ~10 min
- **Cost:** $0.00 - connecting and verifying access has no cost
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 3 complete
- Your instructor has confirmed your namespace `ns-$PARTICIPANT` exists with a ResourceQuota and RBAC binding already set up

## Steps

## Step 1: Confirm kubectl is still pointed at the shared cluster

```
kubectl config current-context
```

Should show something referencing `course-shared-cluster`. If not, repeat Exercise 3's `aws eks update-kubeconfig` command.

## Step 2: Set your own namespace as the default for this session

```
kubectl config set-context --current --namespace=ns-$PARTICIPANT
```

Without this, every kubectl command would need an explicit `-n ns-$PARTICIPANT` flag - setting it once here saves you from repeating that in every later exercise.

## Step 3: Verify you can see your own namespace's resources

```
kubectl get all
```

Likely empty right now (you haven't deployed anything yet) - that's expected. An empty-but-successful result means your access works.

## Step 4: Verify you genuinely CANNOT see another namespace - this should fail

```
kubectl get pods -n ns-someone-else-fake-name
```

Expect a `Forbidden` or `NotFound` error here. This is the actual security guarantee this exercise proves - if this command somehow succeeded, that would mean something is misconfigured, and you should tell your instructor immediately rather than continue.

## Step 5: Check your namespace's resource quota

```
kubectl describe resourcequota
```

This shows the hard ceiling on CPU/memory/Pod count you have to work with for the rest of these exercises - worth knowing now, before Exercise 8 has you deliberately test what happens when you exceed it.

## Verify It Worked

- `kubectl get all` in your own namespace succeeds (even if it shows nothing)
- Attempting another namespace is denied with `Forbidden` or `NotFound`
- `kubectl describe resourcequota` shows real limits, not an empty/missing quota

## Common Mistakes

- **Trying to create a cluster or namespace yourself** - You don't have `eks:CreateCluster` or the ability to create namespaces on the shared cluster - by design, for cost and isolation reasons.
- **Forgetting to set your namespace context and wondering why kubectl shows nothing later** - Re-run the `set-context` command from Step 2 if you ever lose this - it's easy to lose after restarting your terminal or switching between clusters.

## Cleanup (do this now, not later)

```
# Nothing was created here - connecting and verifying access has no cleanup.
```

Then complete the mandatory checklist: **`cleanup-checklists/04-cleanup-checklist.md`**.
