# Exercise 22: Configuration with SSM Parameter Store

## What This Exercise Does

Exercise 7 covered Kubernetes-native ConfigMaps and Secrets - **AWS Systems Manager Parameter Store** is the AWS-native equivalent, useful when configuration needs to be shared BEYOND a single cluster (a Lambda function and an EKS Pod can both read the same parameter, for instance). Unlike Kubernetes Secrets, Parameter Store supports real encryption via KMS for SecureString parameters. This exercise stores both a plain and an encrypted value, scoped entirely to your own path so no other participant can read or overwrite it.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.00 (Standard parameters are free; this exercise uses only a couple of them)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 1 complete (confirms AWS CLI and credentials work)

## Steps

## Step 1: Store a plain configuration value under your own path

```
aws ssm put-parameter --name "/course/$PARTICIPANT/greeting" --value "hello from Parameter Store" --type String --region us-east-1
```

The path `/course/$PARTICIPANT/...` isn't just a naming convention - your IAM grant is specifically scoped to this exact path prefix. A parameter created outside `/course/$PARTICIPANT/*` would be denied outright, the same isolation principle as your EKS namespace, applied here through a path prefix instead of a Kubernetes object.

## Step 2: Store a SECRET value, encrypted with KMS

```
aws ssm put-parameter --name "/course/$PARTICIPANT/api-key" --value "super-secret-value" --type SecureString --region us-east-1
```

`SecureString` (vs. plain `String` in Step 1) means this value is genuinely encrypted at rest using AWS's default KMS key - a real security improvement over a plain Kubernetes Secret, which is only base64-encoded (you proved this yourself in Exercise 7).

## Step 3: Read the plain value back

```
aws ssm get-parameter --name "/course/$PARTICIPANT/greeting" --region us-east-1 --query Parameter.Value --output text
```

This should return exactly what you stored in Step 1, in plain text - a plain `String` parameter has no encryption at all, the SSM equivalent of a Kubernetes ConfigMap rather than a Secret.

## Step 4: Read the SecureString back - notice you must explicitly ask for decryption

```
aws ssm get-parameter --name "/course/$PARTICIPANT/api-key" --region us-east-1 --query Parameter.Value --output text
echo '--- now with decryption ---'
aws ssm get-parameter --name "/course/$PARTICIPANT/api-key" --with-decryption --region us-east-1 --query Parameter.Value --output text
```

The first call returns the ENCRYPTED ciphertext, not your original value - a meaningful difference from Exercise 7's Kubernetes Secret, where anyone with read access sees the real (base64-decoded) value immediately. Here, decryption is a separate, explicit step (`--with-decryption`), which is closer to genuine security than the Kubernetes Secret's 'security by obscurity' base64 encoding.

## Step 5: Try (and fail) to read outside your own path - proves the isolation is real

```
aws ssm get-parameter --name "/course/participant-99/greeting" --region us-east-1
```

Expect an AccessDenied error here, assuming participant-99 isn't actually you - this is your IAM grant's path-based scoping working exactly as intended, the SSM equivalent of Exercise 4's cross-namespace kubectl test.

## Verify It Worked

- get-parameter without --with-decryption on the SecureString returns ciphertext, not your original text
- The same call WITH --with-decryption returns your original value
- Attempting to read another participant's path is denied

## Common Mistakes

- **Forgetting --with-decryption and assuming SecureString is broken** - Getting back ciphertext instead of your value is CORRECT behavior for a SecureString without that flag - it's not an error, it's the encryption working.
- **Storing something outside your own /course/$PARTICIPANT/ path** - Your grant is scoped exactly to this prefix - anything outside it is denied, by design, matching the same isolation model used everywhere else in this course.

## Cleanup (do this now, not later)

```
aws ssm delete-parameter --name "/course/$PARTICIPANT/greeting" --region us-east-1
aws ssm delete-parameter --name "/course/$PARTICIPANT/api-key" --region us-east-1
```

Then complete the mandatory checklist: **`cleanup-checklists/22-cleanup-checklist.md`**.
