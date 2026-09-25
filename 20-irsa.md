# Exercise 20: IAM Roles for Service Accounts (IRSA)

## What This Exercise Does

Every AWS exercise so far that needed a role (Lambda, CodeBuild) used a role your INSTRUCTOR pre-created, passed in by name. **IRSA** is the mechanism that makes this possible for Pods specifically: a Kubernetes ServiceAccount, annotated with an IAM role's ARN, lets any Pod using that ServiceAccount get real, temporary AWS credentials automatically - no access keys stored anywhere, no Secret to manage. This exercise proves a Pod can call a real AWS API with zero credentials of its own, purely through this ServiceAccount-to-role binding.

## Time and Cost

- **Time:** ~20 min
- **Cost:** ~$0.00 (a single Pod running briefly, and a read-only S3 API call)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete
- Your instructor has created the shared `course-irsa-demo-role` and confirmed its exact trust policy (see the exercise text below for the exact service account name it requires)

## Steps

## Step 1: Create a ServiceAccount with the EXACT name the role's trust policy expects

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

**In VS Code:** Create a new file named `~/course/irsa-sa.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-irsa-sa
  annotations:
    eks.amazonaws.com/role-arn: REPLACE_WITH_ROLE_ARN
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/irsa-sa.yaml <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: demo-irsa-sa
  annotations:
    eks.amazonaws.com/role-arn: REPLACE_WITH_ROLE_ARN
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
sed -i "s#REPLACE_WITH_ROLE_ARN#arn:aws:iam::${ACCOUNT_ID}:role/course-irsa-demo-role#" ~/course/irsa-sa.yaml
kubectl apply -f ~/course/irsa-sa.yaml
```


The name `demo-irsa-sa` is not arbitrary - your instructor's role trust policy specifically only trusts service accounts with this exact name (across any participant namespace). A ServiceAccount with a different name would be correctly REJECTED by the role's trust policy, since IRSA's security model is built entirely around this exact-match (or pattern-match) checking.

## Step 2: Run a Pod using that ServiceAccount, with NO AWS credentials configured anywhere

**In VS Code:** Create a new file named `~/course/irsa-pod.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: v1
kind: Pod
metadata:
  name: irsa-demo
spec:
  serviceAccountName: demo-irsa-sa
  containers:
    - name: aws-cli
      image: amazon/aws-cli
      command: ["sleep", "3600"]
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/irsa-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: irsa-demo
spec:
  serviceAccountName: demo-irsa-sa
  containers:
    - name: aws-cli
      image: amazon/aws-cli
      command: ["sleep", "3600"]
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
kubectl apply -f ~/course/irsa-pod.yaml
kubectl wait --for=condition=Ready pod/irsa-demo --timeout=60s
```


`serviceAccountName: demo-irsa-sa` is the ONLY thing connecting this Pod to AWS credentials at all - the container image (`amazon/aws-cli`) is just a normal image with the AWS CLI pre-installed, with nothing baked in about your account or credentials. `sleep 3600` keeps it running for an hour so you have time to exec into it in the next steps, rather than it exiting immediately.

## Step 3: From inside the Pod, prove it has real, temporary AWS credentials

```
kubectl exec irsa-demo -- aws sts get-caller-identity
```

This should return an assumed-role identity referencing `course-irsa-demo-role` - NOT your own participant IAM user. The Pod authenticated as this role entirely through the ServiceAccount annotation, with zero access keys ever placed in the container, the image, or any Secret.

## Step 4: Prove the credentials actually work with a real AWS call

```
kubectl exec irsa-demo -- aws s3 ls --region us-east-1
```

The role only has S3 read-only access (a deliberately narrow demo permission) - this succeeds because IRSA correctly delivered temporary credentials scoped to exactly what the role allows, not because the Pod has any broader access.

## Verify It Worked

- get-caller-identity from inside the Pod shows the assumed role, not your own IAM user
- The S3 call succeeds despite the Pod never being given an access key anywhere

## Common Mistakes

- **Using the wrong ServiceAccount name** - The trust policy is scoped to `demo-irsa-sa` specifically (across `ns-*` namespaces) - any other name is correctly rejected, which is IRSA working as intended, not a bug to work around.
- **Expecting to grant this role broader permissions yourself** - You don't have `iam:PutRolePolicy` or similar - this role's permissions are fixed by your instructor. In a real environment you'd typically have one dedicated role per specific service account rather than one shared demo role like this - worth knowing this exercise's setup is a simplification for teaching, not the production pattern.
- **Looking for a Secret or ConfigMap holding credentials** - There isn't one - that's the entire point of IRSA over the older pattern of mounting static credentials into Pods.

## Cleanup (do this now, not later)

```
kubectl delete -f ~/course/irsa-pod.yaml
kubectl delete -f ~/course/irsa-sa.yaml
```

Then complete the mandatory checklist: **`cleanup-checklists/20-cleanup-checklist.md`**.
