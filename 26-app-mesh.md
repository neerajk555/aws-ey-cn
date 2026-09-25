# Exercise 26: Service Mesh with AWS App Mesh

## What This Exercise Does

Every Service you've created so far talks to other Pods directly - **AWS App Mesh** inserts an Envoy proxy sidecar into each Pod, so all traffic between your services flows through these proxies instead, giving you fine-grained traffic control (like weighted routing between two versions) without changing any application code. This exercise creates a real Mesh with two versions of the same backend, then shifts traffic between them using weighted routing - all configuration, zero code changes to the app itself.

## Time and Cost

- **Time:** ~30 min
- **Cost:** $0.00 - App Mesh itself has no additional charge; you're only paying for the same small Pods you've used throughout this course
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 6 complete (App Mesh concepts build naturally on the Ingress/ALB concepts from that exercise)
- Your instructor has confirmed the App Mesh controller is installed cluster-wide

## Steps

## Step 1: Enable sidecar injection for your own namespace only

```
kubectl label namespace ns-$PARTICIPANT mesh=mesh-$PARTICIPANT appmesh.k8s.aws/sidecarInjectorWebhook=enabled
```

This label is what tells the App Mesh controller to automatically inject an Envoy sidecar into every NEW Pod created in your namespace from this point on - existing Pods aren't retroactively affected, only ones created after this label is applied.

## Step 2: Create your own Mesh - notice the name matches your IAM grant's scoping pattern

**In VS Code:** Create a new file named `~/course/mesh.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: mesh-PARTICIPANT_PLACEHOLDER
spec:
  namespaceSelector:
    matchLabels:
      mesh: mesh-PARTICIPANT_PLACEHOLDER
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/mesh.yaml <<'EOF'
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: mesh-PARTICIPANT_PLACEHOLDER
spec:
  namespaceSelector:
    matchLabels:
      mesh: mesh-PARTICIPANT_PLACEHOLDER
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
sed -i "s/PARTICIPANT_PLACEHOLDER/$PARTICIPANT/g" ~/course/mesh.yaml
kubectl apply -f ~/course/mesh.yaml
```


Your IAM grant specifically scopes App Mesh access to mesh names matching `mesh-$PARTICIPANT*` - a differently-named mesh would be denied at the AWS API level, the same per-participant isolation as SSM's path prefix and S3's bucket-naming pattern in earlier exercises.

## Step 3: Deploy TWO versions of a backend, each as its own VirtualNode

```
kubectl create deployment backend-v1 --image=nginx:1.26 -n ns-$PARTICIPANT
kubectl create deployment backend-v2 --image=nginx:1.27 -n ns-$PARTICIPANT
kubectl expose deployment backend-v1 --port=80 -n ns-$PARTICIPANT
kubectl expose deployment backend-v2 --port=80 -n ns-$PARTICIPANT
```

Using two different nginx versions here purely so you can visibly tell which one served a given request later, via the response headers - not because the version number itself matters for App Mesh.

## Step 4: Define a VirtualRouter that splits traffic 50/50 between the two versions

**In VS Code:** Create a new file named `~/course/virtual-router.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: backend-v1
  namespace: ns-PARTICIPANT_PLACEHOLDER
spec:
  podSelector:
    matchLabels:
      app: backend-v1
  listeners:
    - portMapping: { port: 80, protocol: http }
  serviceDiscovery:
    dns: { hostname: backend-v1.ns-PARTICIPANT_PLACEHOLDER.svc.cluster.local }
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: backend-v2
  namespace: ns-PARTICIPANT_PLACEHOLDER
spec:
  podSelector:
    matchLabels:
      app: backend-v2
  listeners:
    - portMapping: { port: 80, protocol: http }
  serviceDiscovery:
    dns: { hostname: backend-v2.ns-PARTICIPANT_PLACEHOLDER.svc.cluster.local }
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualRouter
metadata:
  name: backend-router
  namespace: ns-PARTICIPANT_PLACEHOLDER
spec:
  listeners:
    - portMapping: { port: 80, protocol: http }
  routes:
    - name: split-route
      httpRoute:
        match: { prefix: / }
        action:
          weightedTargets:
            - virtualNodeRef: { name: backend-v1 }
              weight: 50
            - virtualNodeRef: { name: backend-v2 }
              weight: 50
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/virtual-router.yaml <<'EOF'
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: backend-v1
  namespace: ns-PARTICIPANT_PLACEHOLDER
spec:
  podSelector:
    matchLabels:
      app: backend-v1
  listeners:
    - portMapping: { port: 80, protocol: http }
  serviceDiscovery:
    dns: { hostname: backend-v1.ns-PARTICIPANT_PLACEHOLDER.svc.cluster.local }
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: backend-v2
  namespace: ns-PARTICIPANT_PLACEHOLDER
spec:
  podSelector:
    matchLabels:
      app: backend-v2
  listeners:
    - portMapping: { port: 80, protocol: http }
  serviceDiscovery:
    dns: { hostname: backend-v2.ns-PARTICIPANT_PLACEHOLDER.svc.cluster.local }
---
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualRouter
metadata:
  name: backend-router
  namespace: ns-PARTICIPANT_PLACEHOLDER
spec:
  listeners:
    - portMapping: { port: 80, protocol: http }
  routes:
    - name: split-route
      httpRoute:
        match: { prefix: / }
        action:
          weightedTargets:
            - virtualNodeRef: { name: backend-v1 }
              weight: 50
            - virtualNodeRef: { name: backend-v2 }
              weight: 50
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
sed -i "s/PARTICIPANT_PLACEHOLDER/$PARTICIPANT/g" ~/course/virtual-router.yaml
kubectl apply -f ~/course/virtual-router.yaml
```


The `weight: 50` / `weight: 50` split is the actual point of this exercise - this is real, live traffic splitting configured entirely through YAML, not through any code change in nginx itself. Open this file in VS Code and try changing the weights to 90/10 later, to see the effect for yourself.

## Step 5: Create a VirtualService as the stable name clients actually call

**In VS Code:** Create a new file named `~/course/virtual-service.yaml` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualService
metadata:
  name: backend.ns-PARTICIPANT_PLACEHOLDER.svc.cluster.local
  namespace: ns-PARTICIPANT_PLACEHOLDER
spec:
  provider:
    virtualRouter:
      virtualRouterRef:
        name: backend-router
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/virtual-service.yaml <<'EOF'
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualService
metadata:
  name: backend.ns-PARTICIPANT_PLACEHOLDER.svc.cluster.local
  namespace: ns-PARTICIPANT_PLACEHOLDER
spec:
  provider:
    virtualRouter:
      virtualRouterRef:
        name: backend-router
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
sed -i "s/PARTICIPANT_PLACEHOLDER/$PARTICIPANT/g" ~/course/virtual-service.yaml
kubectl apply -f ~/course/virtual-service.yaml
```


This is the piece that ties everything together: callers reach the VirtualService by name, the VirtualService forwards to the VirtualRouter from Step 4, and the VirtualRouter is what actually splits traffic between your two VirtualNodes - three separate objects, each with one clear job.

## Step 6: Send several requests through the mesh and see the 50/50 split for yourself

```
kubectl run test-client -n ns-$PARTICIPANT --image=curlimages/curl --rm -it --restart=Never -- sh -c 'for i in $(seq 1 10); do curl -s -I http://backend-router.ns-$PARTICIPANT.svc.cluster.local | grep Server; done'
```

You should see a mix of nginx version headers across your 10 requests, roughly (not necessarily exactly) split between the two - proving the VirtualRouter's weighted routing is genuinely load-balancing between backend-v1 and backend-v2.

## Verify It Worked

- Repeated requests through the VirtualService show responses from BOTH backend-v1 and backend-v2
- You can explain the difference between a VirtualNode (one specific backend version) and a VirtualRouter (the traffic-splitting logic between them)

## Common Mistakes

- **Forgetting to label the namespace for sidecar injection BEFORE creating Pods** - Sidecar injection only happens at Pod creation time - a Pod created before the namespace label was applied will never get a sidecar retroactively; delete and recreate it if you did this step out of order.
- **Calling the backend Services directly instead of through the VirtualService** - Bypassing the mesh's own service name means you're talking directly to one specific Kubernetes Service, not through App Mesh's routing logic at all - always call the VirtualService's name to actually exercise the mesh.
- **Using a mesh name that doesn't match your IAM grant's pattern** - Your grant specifically allows `mesh-$PARTICIPANT*` - anything else is denied at the AWS API level, not just a Kubernetes-level restriction.

## Cleanup (do this now, not later)

```
kubectl delete -f ~/course/virtual-service.yaml
kubectl delete -f ~/course/virtual-router.yaml
kubectl delete -f ~/course/mesh.yaml
kubectl delete deployment backend-v1 backend-v2 -n ns-$PARTICIPANT
kubectl delete service backend-v1 backend-v2 -n ns-$PARTICIPANT
kubectl label namespace ns-$PARTICIPANT mesh- appmesh.k8s.aws/sidecarInjectorWebhook-
```

Then complete the mandatory checklist: **`cleanup-checklists/26-cleanup-checklist.md`**.
