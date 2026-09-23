# Exercise 3: Explore the Shared EKS Cluster (Read-Only)

## What This Exercise Does

Every exercise from here on connects to **one Amazon EKS cluster your instructor already created and manages for the entire cohort** - you never create, modify, or delete this cluster, and your IAM permissions specifically prevent it. Amazon EKS is a **managed Kubernetes control plane**: AWS runs and patches the control plane components (the API server, scheduler, etcd) for you - all you and your instructor manage are the worker nodes and the workloads running on them. This exercise is entirely read-only - you're looking at real, managed infrastructure and comparing it to Kubernetes concepts, without creating anything yet.

## Time and Cost

- **Time:** ~10 min
- **Cost:** $0.00 - this is entirely read-only; nothing is created
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 2 complete
- Your instructor has confirmed the shared EKS cluster is up and running for this cohort
- `kubectl` installed on your Ubuntu VM

## Steps

## Step 1: Point kubectl at the shared cluster

```
aws eks update-kubeconfig --name course-shared-cluster --region us-east-1
```

This command writes connection details (the cluster's API endpoint and a way to authenticate) into `~/.kube/config` - it does not create or change anything about the cluster itself.

## Step 2: See the cluster's control plane details

```
aws eks describe-cluster --name course-shared-cluster --region us-east-1 --query 'cluster.{Status:status,Endpoint:endpoint,Version:version}'
```

The `--query` flag uses JMESPath to pull out just three fields from a much larger JSON response - `Status` should read `ACTIVE`, `Endpoint` is the URL kubectl actually talks to, and `Version` is the Kubernetes version your instructor's cluster is running. Try the same command without `--query` once to see the full response and appreciate how much this flag is filtering out.

## Step 3: See the worker nodes

```
kubectl get nodes
```

These are real EC2 instances your instructor provisioned - but you'll never see them listed under your own EC2 permissions, since node management is an instructor-only responsibility.

## Step 4: See that a control plane you never provisioned is running this for you

```
kubectl cluster-info
```

Notice you never ran anything like `eksctl create cluster` yourself - that's exactly what 'managed control plane' means in Amazon EKS. Compare this to any local Kubernetes setup (like `kind`) where YOU are responsible for the entire control plane.

## Verify It Worked

- `kubectl get nodes` returns at least one node in `Ready` status - real infrastructure your instructor set up, not something on your own machine
- `describe-cluster`'s Status field shows `ACTIVE`

## Common Mistakes

- **Trying to create or delete anything here** - This exercise is intentionally read-only - your IAM permissions don't include `eks:CreateCluster`, and this is enforced deliberately, not a limitation to work around.
- **Confusing this shared cluster with a personal one** - There is exactly ONE EKS cluster in this account, shared by the whole cohort - every later exercise uses this same cluster, scoped to your own namespace.

## Cleanup (do this now, not later)

```
# Nothing was created - nothing to delete. This exercise has zero cleanup.
```

Then complete the mandatory checklist: **`cleanup-checklists/03-cleanup-checklist.md`**.
