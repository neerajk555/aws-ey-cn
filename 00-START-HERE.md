# Start Here: AWS Exercises

This folder contains **only the AWS-facing exercises** from the full Cloud Native Course - 15 exercises,
each done for real on a shared AWS account using your own assigned IAM user. It assumes you've already
built the underlying skill somewhere else (the full course's local exercises, prior experience, or
another course) - each exercise here explains what state needs to already exist and gives you a way to
verify it, rather than re-teaching the concept from scratch.

If you haven't done any Docker/Kubernetes work before, use the full **`cloud-native-course`** folder
instead - it builds every concept locally, for free, before transferring it here.

## What you need before Exercise 1

- **An Ubuntu VM** (yours, or provided by the course)
- **AWS CLI v2** installed and configured with the IAM credentials your instructor issued you
- **VS Code** installed and open - every exercise uses it for file creation and editing, not just the
  terminal. Recommended extensions: **YAML** (`redhat.vscode-yaml`), **Docker**
  (`ms-azuretools.vscode-docker`), **Kubernetes** (`ms-kubernetes-tools.vscode-kubernetes-tools`)
- **`kubectl`** and **`helm`** installed (needed from Exercise 4 onward)
- Your `$PARTICIPANT` variable exported, every session:
  ```
  export PARTICIPANT=$(aws iam get-user --query "User.UserName" --output text)
  echo "You are: $PARTICIPANT"
  ```
  Add this line to `~/.bashrc` if you want it set automatically in every new terminal.

## How file creation works in these exercises

Every time an exercise needs you to create a file, the instructions lead with **VS Code**: open the
Explorer panel, create the file, paste the content, save. Right below that, a terminal-equivalent
command is also shown in case you prefer typing directly into a terminal for a specific step - but
VS Code is the primary, recommended way throughout, not an afterthought.

## The rule that matters most: cleanup is mandatory, not optional

This is a **shared AWS training account** with real cost implications. Every exercise ends with a
**Cleanup** section and a link to a **mandatory checklist** in `cleanup-checklists/`. Do the cleanup
**immediately** after finishing each exercise, not "later" - two exercises in particular need extra
attention:

- **Exercise 6 (ALB)** creates a real Application Load Balancer, which bills **hourly regardless of
  traffic** - the single most expensive thing per minute in this whole set of exercises.
- **Exercise 9 (EBS storage)** creates a real EBS volume, which keeps billing **every month** even with
  zero Pods using it, until the volume itself is explicitly deleted - deleting the Pod or even the PVC
  isn't always enough to guarantee the underlying volume is gone; that exercise shows you how to verify
  it directly by volume ID.

## The shared EKS cluster

From Exercise 4 onward, you're connecting to **one EKS cluster your instructor already created and
manages for the whole cohort** - you never create, modify, or delete this cluster yourself, and you
don't have permission to. You get your own Kubernetes namespace (`ns-$PARTICIPANT`) inside it, with a
resource quota and RBAC scoping you to only your own namespace.

## The 15 exercises

| # | Exercise | What it covers |
|---|---|---|
| 1 | Push an Image to ECR | Docker image, private registry |
| 2 | Lambda with Environment Config | Serverless compute, env-based config |
| 3 | Explore the Shared EKS Cluster | Read-only - managed Kubernetes control plane |
| 4 | Connect to Your Namespace | RBAC-scoped cluster access |
| 5 | Deploy to Your Namespace | Deployments, Services, self-healing |
| 6 | Expose via ALB | Ingress, Application Load Balancer |
| 7 | Config and Secrets on EKS | ConfigMap, Secret, Secrets Manager comparison |
| 8 | Updates and Scaling on EKS | Rolling updates, ResourceQuota limits |
| 9 | EBS-Backed Storage | PersistentVolumeClaim, real durable storage |
| 10 | Probes on EKS | Readiness/liveness in a real cluster |
| 11 | Jobs on EKS | Run-to-completion workloads |
| 12 | Deploy a Helm Chart to EKS | Packaging, portability across environments |
| 13 | CodeBuild | CI build stage |
| 14 | CloudWatch Logs and Alarm | Observability, alerting |
| 15 | Capstone on EKS | End-to-end: chart, probes, scaling, rollback |

Start with Exercise 1.
