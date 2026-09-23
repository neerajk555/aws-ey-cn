# Exercise 7: Config and Secrets on EKS

## What This Exercise Does

A **ConfigMap** injects non-sensitive configuration into Pods without baking it into your image - the Kubernetes equivalent of an environment variable file. A **Secret** does the same for sensitive values, but is only **base64-ENCODED**, not encrypted, by default - anyone with API read access to the Secret object can trivially decode it. This exercise creates both, injects them into a real Deployment on your namespace, proves the base64 point yourself, and compares this to **AWS Secrets Manager**, which is what you'd actually reach for when a value needs real protection.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.00-0.01 (a small Deployment running briefly)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete

## Steps

## Step 1: Create a ConfigMap and a Secret

```
kubectl create configmap app-config --from-literal=GREETING=hello
kubectl create secret generic app-secret --from-literal=API_KEY=demo123
```

`--from-literal=KEY=value` creates the object with a single key-value pair directly from the command line - the quickest way to create either object for a simple case like this. Both commands create objects scoped to your current namespace, same as everything else you've deployed so far.

## Step 2: Create a Deployment and inject both as environment variables

```
kubectl create deployment web --image=nginx:1.27
kubectl set env deployment/web --from=configmap/app-config
kubectl set env deployment/web --from=secret/app-secret
```

`kubectl set env --from=configmap/X` (or `secret/X`) takes EVERY key in that ConfigMap or Secret and injects it as an environment variable with the same name - you don't have to list each key individually. Each `set env` command triggers a new rollout of the Deployment, since changing the Pod template always does.

## Step 3: Verify both landed inside the Pod

```
kubectl get pods
kubectl exec <paste-pod-name-here> -- env | grep -E 'GREETING|API_KEY'
```

`kubectl exec <pod> -- <command>` runs a command INSIDE the running container, same idea as `docker exec`. Here it runs `env` (which prints every environment variable) and pipes it through `grep` to show only the two you care about.

## Step 4: Prove base64 is not encryption

```
kubectl get secret app-secret -o jsonpath='{.data.API_KEY}' | base64 -d
```

This decodes instantly with a single, standard command - anyone with read access to this Secret object, not just you, could do the same thing. This is the actual, concrete reason Secrets alone aren't 'secure' in the way the name implies.

## Step 5: Compare: look at what AWS Secrets Manager offers instead (read-only, nothing created)

```
aws secretsmanager list-secrets --region us-east-1
```

This will likely come back empty for your account - that's fine, the point here is understanding the OPTION exists, not using it in this exercise. Secrets Manager encrypts values at rest with KMS by default, supports automatic rotation, and logs every read - none of which a plain Kubernetes Secret gives you.

## Verify It Worked

- The Pod's environment shows both `GREETING=hello` and `API_KEY=demo123`
- The base64-decode command in Step 4 returns `demo123` in plain text
- You can explain, in your own words, when you'd reach for Secrets Manager instead of a plain Kubernetes Secret

## Common Mistakes

- **Treating a Kubernetes Secret as automatically secure** - Base64 is an ENCODING, not encryption - real protection needs cluster-level encryption-at-rest AND tight RBAC restricting who can read Secret objects at all.
- **Assuming this exercise requires actually creating a Secrets Manager secret** - It doesn't - the comparison in Step 5 is read-only and conceptual; creating one isn't necessary here and avoids extra cleanup for zero learning benefit.

## Cleanup (do this now, not later)

```
kubectl delete deployment web
kubectl delete configmap app-config
kubectl delete secret app-secret
```

Then complete the mandatory checklist: **`cleanup-checklists/07-cleanup-checklist.md`**.
