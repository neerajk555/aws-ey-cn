# Exercise 15: Capstone on EKS

## What This Exercise Does

This exercise combines nearly everything from Exercises 1-14 into one deployment: a Helm chart (Exercise 12) packaging a Deployment with a readiness probe (Exercise 10) and ConfigMap-injected configuration (Exercise 7), plus a Job (Exercise 11) that reads the same configuration - deployed to your real namespace on the shared cluster, then deliberately scaled, rolled back, and probed for failure the same way Exercise 8 and 10 did individually. Minimal hand-holding here is intentional - treat this as a genuine test of what you've retained, referring back to earlier exercises whenever you get stuck rather than skipping ahead.

## Time and Cost

- **Time:** ~30-45 min - budget a full session, this is intentionally longer than earlier exercises
- **Cost:** ~$0.01-0.03 (a handful of small Pods and one completed Job, running briefly)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercises 1-14 all complete
- A working Helm chart from Exercise 12 (reuse `~/course/mychart`, or rebuild it if you cleaned it up entirely)

## Steps

## Step 1: The spec

Extend your Exercise 12 chart (or build a fresh one) so that it defines THREE things:

1. **`api`** - a Deployment whose Pod reads at least one setting from a ConfigMap-injected environment variable (Exercise 7's pattern) and has a working readiness probe (Exercise 10's pattern)
2. **`worker`** - a Job (Exercise 11, not a long-running service) that runs once and prints a message referencing the same ConfigMap value
3. A `values.yaml` exposing at minimum: the ConfigMap's value, and `api`'s replica count


## Step 2: Deploy it to your namespace

```
cd ~/course/mychart
helm install capstone .
kubectl get pods
kubectl get jobs
```

## Step 3: Verify the ConfigMap value reached both api and worker

```
kubectl logs job/$(kubectl get jobs -o jsonpath='{.items[0].metadata.name}')
kubectl exec $(kubectl get pods -l app.kubernetes.io/instance=capstone -o jsonpath='{.items[0].metadata.name}') -- env | grep -i config
```

## Step 4: Scale api via Helm, then roll back (Exercise 8's pattern, through Helm instead of kubectl directly)

```
helm upgrade capstone . --set api.replicaCount=3
kubectl get pods
helm rollback capstone 1
kubectl get pods
```

Notice this achieves the same rolling-update/rollback behavior as Exercise 8, but driven through Helm's own revision history instead of `kubectl rollout undo` directly.

## Step 5: Break api's readiness probe on purpose and confirm the Exercise 10 behavior still holds

Edit your chart's readiness probe path to something that 404s, `helm upgrade` again, and confirm with `kubectl get endpoints` that api is removed from Service traffic without being restarted - exactly Exercise 10's behavior, now inside a real multi-object Helm release instead of a single bare Deployment.


## Verify It Worked

- `helm install capstone .` succeeds with api and web Pods Running and worker's Job Completed
- The Job's logs show it correctly read the ConfigMap value
- `helm upgrade --set api.replicaCount=3` scales only api, and `helm rollback capstone 1` correctly reverts it
- Breaking api's readiness probe removes it from Service traffic without a restart, provable via `kubectl get endpoints`

## Common Mistakes

- **Starting from a blank chart instead of extending Exercise 12's working one** - Reuse what already works - this capstone is about COMBINING known-good pieces, not re-solving Helm chart authoring from scratch.
- **Skipping the deliberate probe-breaking step** - It's the part that actually proves understanding, not just successful copy-pasting - don't skip it even though it takes a few extra minutes.
- **Leaving this running after finishing** - This is the last exercise - clean it up completely, the same discipline as every exercise before it.

## Cleanup (do this now, not later)

```
helm uninstall capstone
```

Then complete the mandatory checklist: **`cleanup-checklists/15-cleanup-checklist.md`**.
