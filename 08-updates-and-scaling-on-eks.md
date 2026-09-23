# Exercise 8: Updates and Scaling on EKS

## What This Exercise Does

A **rolling update** replaces old Pods with new ones gradually, keeping the app available throughout - Kubernetes' declarative model reconciling toward a new desired state step by step rather than all at once. Scaling up or down is just changing the desired replica count and letting Kubernetes reconcile toward it. This exercise does both on your real namespace, then deliberately tries to scale PAST what your namespace's ResourceQuota allows - something your own free local cluster (if you've used one) never enforced - so you can see exactly what that failure looks like and why it's there.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.00-0.02 (a handful of small Pods running briefly on already-provisioned shared nodes)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete
- Exercise 4's `kubectl describe resourcequota` check, so you know your namespace's actual limits

## Steps

## Step 1: Deploy version 1

```
kubectl create deployment web --image=nginx:1.26 --replicas=2
```

Deliberately starting on `nginx:1.26` (not the latest) so the next step's update to `1.27` has something real to change - watch the exact version number as you go through this exercise.

## Step 2: Perform a rolling update to version 2

```
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
```

Watch how it replaces Pods a few at a time, not all at once - the app stays available to any in-cluster caller throughout.

## Step 3: Roll back

```
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
kubectl get deployment web -o jsonpath='{.spec.template.spec.containers[0].image}'
```

The final command should print `nginx:1.26` again - confirming the rollback actually reverted the image, not just the replica count.

## Step 4: Try to scale beyond what your quota allows

```
kubectl scale deployment web --replicas=10
kubectl get pods
kubectl describe resourcequota
```

If some Pods are stuck Pending, run `kubectl describe pod <pending-pod-name>` and look for a message about exceeding your namespace's quota. This is expected and intentional, not a bug - your namespace's ResourceQuota is a hard, shared-account cost/capacity control, not a suggestion.

## Step 5: Scale back to something your quota comfortably allows

```
kubectl scale deployment web --replicas=2
```

`kubectl scale` changes ONLY the replica count - it doesn't touch the image, environment variables, or anything else about the Deployment. This is the cleanup step for the quota experiment above, returning you to a safe baseline before moving on.

## Verify It Worked

- After the rollback, the deployment's image is back to `nginx:1.26`
- You can explain, in your own words, why scaling to 10 didn't fully succeed, referencing the specific quota message from `kubectl describe pod`

## Common Mistakes

- **Assuming a quota-blocked scale-up means something is broken** - It's a deliberate, shared-account capacity control - simply scale to a value within your quota instead of troubleshooting it as an error.
- **Leaving replicas at a high count after this exercise** - Scale back down as part of cleanup - other participants share this cluster's total node capacity, and leftover Pods reduce headroom for everyone else.

## Cleanup (do this now, not later)

```
kubectl delete deployment web
```

Then complete the mandatory checklist: **`cleanup-checklists/08-cleanup-checklist.md`**.
