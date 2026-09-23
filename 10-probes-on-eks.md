# Exercise 10: Probes on EKS

## What This Exercise Does

A **liveness probe** answers 'is this container still alive, or should it be restarted?' A **readiness probe** answers 'is this container ready to receive traffic right now?' The difference matters: a failing readiness probe removes a Pod from Service traffic WITHOUT restarting it (useful for temporary overload or startup delays); a failing liveness probe triggers an actual restart. This exercise configures both on your real namespace and proves the readiness-specific behavior by breaking it on purpose.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.00-0.01 (a single small Pod running briefly)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete

## Steps

## Step 1: Deploy with both probes configured

**In VS Code:** Create a new file named `~/course/probes.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/probes.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          readinessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            periodSeconds: 10
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
kubectl apply -f ~/course/probes.yaml
kubectl expose deployment web --port=80
```


Review this file in VS Code before applying - notice readinessProbe and livenessProbe both point at the same path here, but that's not required; in a real app they're often different endpoints.

## Step 2: Confirm it's ready and reachable

```
kubectl get pods
kubectl get endpoints web
```

READY should show 1/1, and the endpoints list should show a real Pod IP - both prove the readiness probe is currently passing.

## Step 3: Break ONLY readiness (point it at a path that 404s) and observe

The YAML has TWO `path: /` lines - one under `readinessProbe`, one under `livenessProbe`. Changing both by accident (a plain find-and-replace would do exactly this, since the lines are identical text) would break the wrong thing: it would make BOTH probes fail, causing an actual restart - the opposite of what this step is meant to demonstrate. Do this in VS Code instead of the terminal, specifically so you can see and target the right line:

1. Open `~/course/probes.yaml` in VS Code
2. Find the `readinessProbe:` block (NOT `livenessProbe:`, a few lines below it)
3. Change only that block's `path: /` to `path: /does-not-exist`
4. Leave `livenessProbe`'s `path: /` untouched
5. Save (`Ctrl+S`)

```
kubectl apply -f ~/course/probes.yaml
kubectl get pods
kubectl get endpoints web
```

`kubectl get pods` should still show the Pod as Running - it hasn't crashed. But `kubectl get endpoints web` should now be EMPTY. This is the actual distinction this exercise exists to prove: the Pod is alive but no longer receiving traffic, because only the READINESS probe is failing. If you accidentally changed both probes, the Pod would start restarting instead - if that happens, revert livenessProbe's path back to `/` and reapply.

## Verify It Worked

- Before Step 3: endpoints list shows a real IP. After Step 3: the Pod stays Running, but endpoints becomes empty
- You can state, without looking it up, which of the two probe types was responsible for removing the Pod from traffic

## Common Mistakes

- **Only ever configuring a liveness probe** - A liveness-only setup means a temporarily overloaded (but not crashed) Pod keeps receiving traffic it can't handle, or gets needlessly restarted - configure both for any real app.
- **Expecting the broken readiness probe to restart the Pod** - It won't - that's specifically what a LIVENESS probe failure does. A readiness failure only removes the Pod from Service traffic, which is the whole point of this exercise.

## Cleanup (do this now, not later)

```
kubectl delete -f ~/course/probes.yaml
kubectl delete service web
```

Then complete the mandatory checklist: **`cleanup-checklists/10-cleanup-checklist.md`**.
