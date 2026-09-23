# Exercise 11: Jobs on EKS

## What This Exercise Does

A Deployment expects its Pods to run FOREVER, restarting them if they exit. A **Job** expects its Pod to run ONCE and complete successfully - the right tool for batch tasks, migrations, or any one-off script, as opposed to a continuously-running service. This exercise runs a Job on your real namespace and confirms it behaves correctly: completes once, and does NOT get restarted the way a Deployment's Pod would.

## Time and Cost

- **Time:** ~10 min
- **Cost:** $0.00-0.01 (the Job completes in seconds on already-provisioned shared nodes)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete

## Steps

## Step 1: Create a Job

```
kubectl create job hello-job --image=busybox -- echo 'batch task complete on EKS'
```

The `--` separates kubectl's own flags from the command that should actually run INSIDE the container - everything after it (`echo 'batch task complete on EKS'`) is what the busybox container executes once, not a kubectl instruction.

## Step 2: Watch it run to completion

```
kubectl get jobs -w
```

Press Ctrl+C once COMPLETIONS shows 1/1.

## Step 3: See its output

```
kubectl logs job/hello-job
```

`kubectl logs job/<name>` is a convenience form that finds the Job's Pod for you and shows its logs, without needing to look up the exact Pod name first - useful since a Job's Pod name always has a random suffix.

## Step 4: Confirm the completed Pod is NOT restarted, unlike a Deployment's Pod would be

```
kubectl get pods
```

Status should show `Completed`, and it stays that way - compare this to Exercise 5, where deleting a Deployment-managed Pod triggered an automatic replacement. A Job's completed Pod is left alone on purpose.

## Verify It Worked

- The Job completes and its logs show your custom message
- The Job's Pod shows `Completed` status and is never replaced or restarted

## Common Mistakes

- **Using a Deployment for a one-off script instead** - A Deployment would keep restarting the script forever, since it expects continuous running - that's exactly the wrong tool for a task meant to run once.
- **Expecting the completed Pod to disappear on its own** - It doesn't, by default - completed Job Pods stick around until you delete the Job (`kubectl delete job hello-job`) or configure a TTL on it.

## Cleanup (do this now, not later)

```
kubectl delete job hello-job
```

Then complete the mandatory checklist: **`cleanup-checklists/11-cleanup-checklist.md`**.
