# Exercise 25: GitOps with ArgoCD

## What This Exercise Does

Every exercise so far, you ran `kubectl apply` yourself, by hand. **GitOps** flips this: your git repository becomes the single source of truth for what SHOULD be running, and a tool continuously reconciles the cluster to match it - no more manually running commands to make changes. **ArgoCD** (already installed cluster-wide by your instructor) is that tool. This exercise points ArgoCD at a public git repo, watches it deploy automatically with zero `kubectl apply` from you, then proves the reconciliation loop is real by manually breaking what it deployed and watching ArgoCD fix it.

## Time and Cost

- **Time:** ~20 min
- **Cost:** $0.00 - purely Kubernetes-native, no billed AWS resources involved
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete
- Your instructor has confirmed ArgoCD is installed and given you the initial admin password
- A public git repo with at least one Kubernetes manifest in it (a public example works fine - you don't need to author your own for this exercise)

## Steps

## Step 1: Log into the ArgoCD UI (read-only exploration first, before creating anything)

Port-forward to reach the UI without needing a public load balancer for this exercise:

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080` in a browser, log in as `admin` with the password your instructor gave you, and look around - notice it's currently empty for you specifically, since you haven't created an Application yet.


## Step 2: Create an Application pointed at a public git repo, targeting YOUR OWN namespace

**In VS Code:** Create a new file named `~/course/argo-app.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: PARTICIPANT_PLACEHOLDER-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: ns-PARTICIPANT_PLACEHOLDER
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/argo-app.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: PARTICIPANT_PLACEHOLDER-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: ns-PARTICIPANT_PLACEHOLDER
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
sed -i "s/PARTICIPANT_PLACEHOLDER/$PARTICIPANT/g" ~/course/argo-app.yaml
kubectl apply -f ~/course/argo-app.yaml
```


`destination.namespace` is set to YOUR namespace specifically - this Application will only ever manage objects there, never anyone else's, the same isolation principle as everything else, just expressed through ArgoCD's own destination field this time. `syncPolicy.automated.selfHeal: true` is the setting that makes Step 4 below actually work.

## Step 3: Watch ArgoCD deploy it - notice you never ran kubectl apply for the actual application

```
kubectl get application $PARTICIPANT-app -n argocd -w
```

Ctrl+C once STATUS shows Synced and HEALTH shows Healthy. Everything that got created in `ns-$PARTICIPANT` from this point came from the git repo, reconciled by ArgoCD - not from a command you typed.

## Step 4: Confirm the actual Kubernetes objects exist in your namespace

```
kubectl get all -n ns-$PARTICIPANT -l app.kubernetes.io/instance=$PARTICIPANT-app
```

The label filter here works the same way it did in Exercise 12's Helm release check - `app.kubernetes.io/instance` identifies everything belonging to this one specific Application, letting you see exactly what ArgoCD created without guessing at object names.

## Step 5: Prove the reconciliation loop is real: manually delete something ArgoCD deployed

```
kubectl delete deployment guestbook-ui -n ns-$PARTICIPANT
sleep 30
kubectl get deployment guestbook-ui -n ns-$PARTICIPANT
```

Because `selfHeal: true` was set in Step 2, ArgoCD should notice the live state no longer matches the git repo's desired state, and recreate the Deployment within about 30-60 seconds - entirely on its own, with no command from you. This is the actual point of GitOps: the desired state lives in git, and the cluster is continuously pulled back toward it.

## Verify It Worked

- The Application shows Synced and Healthy in the ArgoCD UI or via kubectl get application
- After manually deleting a Deployment, it reappears on its own within about a minute, with no kubectl apply from you

## Common Mistakes

- **Manually running kubectl apply on the guestbook manifests yourself** - That defeats the entire point of this exercise - ArgoCD, not you, should be the one applying anything to your namespace from this point forward.
- **Forgetting selfHeal must be explicitly enabled** - Without it, ArgoCD only reports drift as OutOfSync rather than actively correcting it - the self-healing behavior in Step 4 is opt-in, not automatic by default.
- **Pointing destination.namespace at someone else's namespace** - Always double check this field matches your own `ns-$PARTICIPANT` before applying - your RBAC would ultimately block it, but it's worth getting right at the source rather than relying on that as a backstop.

## Cleanup (do this now, not later)

```
kubectl delete -f ~/course/argo-app.yaml
```

Then complete the mandatory checklist: **`cleanup-checklists/25-cleanup-checklist.md`**.
