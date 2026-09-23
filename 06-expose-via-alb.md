# Exercise 6: Expose via ALB

## What This Exercise Does

A ClusterIP Service (Exercise 5) is only reachable inside the cluster. To expose something to the outside world on AWS, you use an **Ingress** object with AWS's own ingress class - the AWS Load Balancer Controller (already installed cluster-wide by your instructor, running under its own dedicated permissions, not yours) watches for Ingress objects and provisions a real **Application Load Balancer (ALB)** to route traffic to them. This is genuinely the most expensive resource, per-minute, in this entire set of exercises - it bills hourly regardless of traffic - so cleanup here is not optional.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.02-0.03 for a few minutes of use, BUT bills hourly regardless of traffic if left running - the most expensive resource in these exercises
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete
- Your instructor has confirmed the AWS Load Balancer Controller is installed on the shared cluster

## Steps

## Step 1: Create a Deployment and a ClusterIP Service (same pattern as Exercise 5)

```
kubectl create deployment web --image=nginx:1.27
kubectl expose deployment web --port=80
```

Notice this stays a plain ClusterIP Service - the ALB is created by an Ingress routing TO this Service, not by changing the Service's own type.

## Step 2: Write an Ingress using AWS's ALB ingress class, WITH required tags

```
cat > ~/course/web-ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/tags: Owner=$PARTICIPANT,Course=cloudnative-course-2026
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
EOF
```


**Using VS Code instead:** rather than the heredoc above, create this directly in the editor.

In VS Code's Explorer panel, create a new file named `~/course/web-ingress.yaml` in your working folder, paste this, and save (`Ctrl+S`):

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/tags: Owner=$PARTICIPANT,Course=cloudnative-course-2026
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

Open this file in VS Code and check the `alb.ingress.kubernetes.io/tags` annotation carefully before applying - without it, the ALB is created UNTAGGED and your account's permission boundary denies untagged load balancer creation outright. `ingressClassName: alb` is what tells the AWS Load Balancer Controller (not any other ingress controller) to handle this specific object.

## Step 3: Apply it

```
kubectl apply -f ~/course/web-ingress.yaml
```

## Step 4: Wait for the ALB address to appear

```
kubectl get ingress web -w
```

This can take 1-2 minutes. Press Ctrl+C once the ADDRESS column shows a real hostname instead of being empty.

## Step 5: Test it from your own machine

```
curl http://<paste-the-ADDRESS-value-here>
```

Unlike the ClusterIP Service in Exercise 5, this genuinely works from outside the cluster - a real, internet-facing ALB is now routing to your Pod.

## Verify It Worked

- curl against the ALB's address returns nginx's welcome page
- The load balancer is visible in the AWS Console under EC2 > Load Balancers, tagged with your Owner value

## Common Mistakes

- **Using a Service annotation instead of an Ingress** - A Service of type LoadBalancer with an `aws-load-balancer-type` annotation provisions a Network Load Balancer (Layer 4), not an ALB (Layer 7) - a genuine ALB always comes from an Ingress resource, exactly as this exercise does it.
- **Forgetting the `alb.ingress.kubernetes.io/tags` annotation** - The permission boundary denies untagged load balancer creation outright, not just leaves it untracked - this isn't optional.
- **Leaving the ALB running after finishing** - It bills hourly regardless of traffic - delete it the moment you're done verifying it works, not at the end of your session.

## Cleanup (do this now, not later)

```
kubectl delete -f ~/course/web-ingress.yaml
kubectl delete deployment web
kubectl delete service web
```

Then complete the mandatory checklist: **`cleanup-checklists/06-cleanup-checklist.md`**.
