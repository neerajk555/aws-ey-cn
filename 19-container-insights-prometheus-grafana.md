# Exercise 19: View Cluster Metrics with Container Insights and Managed Prometheus

## What This Exercise Does

CloudWatch Logs (Exercise 14) and X-Ray (Exercise 18) cover logs and traces - **metrics** are the third pillar. Your instructor has already enabled **Container Insights** (CPU/memory per Pod, automatically, no code changes) and stood up a **shared Amazon Managed Prometheus workspace** on the cluster. This exercise is entirely READ-ONLY: you generate a small load on your own Deployment, then find and read the resulting metrics through both systems, without creating any infrastructure yourself - the workspace is shared cohort-wide since these have a real ongoing cost per-workspace, unlike the free-when-idle services you've used so far.

## Time and Cost

- **Time:** ~15 min
- **Cost:** $0.00 - this exercise creates a Deployment (already free at this scale) and only READS from shared, instructor-owned monitoring infrastructure
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete (or willing to quickly redo its Deployment)
- Your instructor has confirmed Container Insights and the shared Prometheus workspace are active

## Steps

## Step 1: Deploy something to generate metrics from

```
kubectl create deployment web --image=nginx:1.27 --replicas=2
```

Same simple Deployment pattern as Exercise 5 - the point here isn't the app itself, it's having real, running Pods for Container Insights and Prometheus to actually collect data about in the next steps.

## Step 2: Find your Pods' CPU/memory metrics via Container Insights

```
aws cloudwatch get-metric-statistics --namespace ContainerInsights --metric-name pod_cpu_utilization --dimensions Name=Namespace,Value=ns-$PARTICIPANT --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average --region us-east-1
```

Notice the `Dimensions` filter scopes this query to YOUR namespace specifically - Container Insights collects metrics cluster-wide, but you're only reading the slice that's actually yours, the same isolation principle as everything else in this course, just applied to observability data instead of compute.

## Step 3: Ask your instructor for the shared Prometheus workspace ID, then query it directly

```
aws amp query-metrics --workspace-id <paste-workspace-id-from-instructor> --query 'up{namespace="ns-$PARTICIPANT"}' --region us-east-1
```

This is a real PromQL query (the same query language used in any Prometheus setup, on-prem or cloud) - `up{namespace="..."}` asks 'which targets in my namespace are currently being scraped successfully.' The label-based filtering here does the same isolation job as the CloudWatch dimension filter above.

## Verify It Worked

- The CloudWatch query returns real, non-empty CPU utilization data points for your namespace
- The Prometheus query returns at least one result scoped to your namespace

## Common Mistakes

- **Assuming you need to install or configure anything yourself** - Both Container Insights and the Prometheus workspace are already running cluster-wide - this exercise is purely about reading data that's already being collected, not setting anything up.
- **Querying without a namespace filter** - Without it, you'd see (or try to see) every participant's metrics mixed together - always scope your queries the way Steps 2 and 3 do.

## Cleanup (do this now, not later)

```
kubectl delete deployment web
```

Then complete the mandatory checklist: **`cleanup-checklists/19-cleanup-checklist.md`**.
