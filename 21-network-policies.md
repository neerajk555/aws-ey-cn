# Exercise 21: Network Policies

## What This Exercise Does

By default, every Pod in a Kubernetes namespace can reach every other Pod - there's no network isolation between them out of the box, only the API-level RBAC isolation you've relied on so far. A **NetworkPolicy** restricts actual network traffic at the Pod level: which Pods can talk to which other Pods, over which ports. This exercise proves two Pods CAN reach each other by default, then applies a policy that blocks it, proving the restriction is real by testing the connection both before and after.

## Time and Cost

- **Time:** ~20 min
- **Cost:** $0.00 - purely Kubernetes-native, no billed AWS resources involved
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete
- Your instructor has confirmed Network Policy enforcement is enabled on the cluster's VPC CNI - without this specific setting, this exercise's policy would apply with no error, but silently do nothing

## Steps

## Step 1: Deploy two separate apps in your namespace

```
kubectl create deployment frontend --image=busybox -- sleep 3600
kubectl create deployment backend --image=nginx:1.27
kubectl expose deployment backend --port=80
```

`frontend` is a plain busybox container kept alive with `sleep 3600` purely so you have a shell to exec into and test connectivity FROM - it's not a real application, just a convenient test client for this exercise.

## Step 2: Confirm frontend can reach backend BEFORE any policy exists

```
FRONTEND_POD=$(kubectl get pods -l app=frontend -o jsonpath='{.items[0].metadata.name}')
kubectl exec $FRONTEND_POD -- wget -qO- --timeout=5 http://backend
```

This should succeed and return nginx's welcome page HTML - proving there's no isolation at all by default, exactly as expected before any NetworkPolicy exists.

## Step 3: Apply a policy that only allows traffic from Pods labeled app=allowed-client (frontend is NOT labeled this way)

**In VS Code:** Create a new file named `~/course/netpol.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-restrict
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: allowed-client
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/netpol.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-restrict
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: allowed-client
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
kubectl apply -f ~/course/netpol.yaml
```


`podSelector` (top-level) picks WHICH Pods this policy applies to - here, anything labeled `app: backend`. The `ingress.from.podSelector` picks WHICH Pods are still allowed in - only ones labeled `app: allowed-client`. Since `frontend`'s Pods are labeled `app: frontend`, not `app: allowed-client`, they should now be blocked.

## Step 4: Confirm frontend can NO LONGER reach backend

```
kubectl exec $FRONTEND_POD -- wget -qO- --timeout=5 http://backend
```

This should now time out or fail - a real, enforced change in behavior from Step 2, proving the NetworkPolicy is genuinely being enforced by the CNI, not just accepted as a Kubernetes object with no real effect.

## Verify It Worked

- Step 2's request succeeds (no policy yet)
- Step 4's identical request fails after the policy is applied - same command, different outcome

## Common Mistakes

- **Seeing Step 4 still succeed** - This means Network Policy enforcement isn't actually enabled on the cluster's CNI - flag this to your instructor rather than assuming your YAML is wrong, since the addon setting (not your policy) is the more likely cause.
- **Forgetting NetworkPolicy is deny-by-specificity, not deny-by-default across the whole namespace** - Only Pods actually SELECTED by a NetworkPolicy's podSelector become restricted - any Pod not matched by any policy remains fully open, exactly like `frontend` still is in this exercise.

## Cleanup (do this now, not later)

```
kubectl delete -f ~/course/netpol.yaml
kubectl delete deployment frontend backend
kubectl delete service backend
```

Then complete the mandatory checklist: **`cleanup-checklists/21-cleanup-checklist.md`**.
