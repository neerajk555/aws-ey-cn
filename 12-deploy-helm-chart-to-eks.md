# Exercise 12: Deploy a Helm Chart to EKS

## What This Exercise Does

Helm packages a set of related Kubernetes YAML files into one reusable, versioned unit (a **chart**), parameterized by a `values.yaml` file - so the same chart can deploy to different environments just by changing values, instead of maintaining near-duplicate YAML per environment. This exercise builds a small chart from scratch and deploys it to your real EKS namespace, proving the actual promise of Helm: one chart, genuinely portable across environments, with zero modification required to make it work here.

## Time and Cost

- **Time:** ~20 min
- **Cost:** ~$0.00-0.02 (a few small Pods running briefly on already-provisioned shared nodes)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 5 complete
- Helm installed on your Ubuntu VM (`helm version` to confirm)

## Steps

## Step 1: Scaffold a new chart

```
cd ~/course && helm create mychart
find mychart -type f | sort
```

This generates a fully working, generic chart - open `mychart/values.yaml` and `mychart/templates/deployment.yaml` in VS Code now to see the structure before changing anything.

## Step 2: Add a custom, parameterized value

```
cd mychart
echo 'greeting: hello-from-helm-on-eks' >> values.yaml
```

`>>` appends this line to the end of `values.yaml` without disturbing anything already in the file - equivalent to opening the file in VS Code and typing a new line at the bottom, just faster for a single line. Every value in this file becomes available inside your chart's templates as `.Values.<key>`.

## Step 3: Reference it in the Deployment template

Open `templates/deployment.yaml` in VS Code and add this under the container spec (watch your indentation - it must line up with the other keys at the same level, like `image:`):

```yaml
          env:
            - name: GREETING
              value: {{ .Values.greeting | quote }}
```


## Step 4: Render locally to check your work, before touching the cluster at all

```
helm template . | grep -A2 GREETING
```

This is the safest way to catch a YAML/templating mistake - it renders the chart to plain text on your own machine, without touching AWS or the cluster.

## Step 5: Deploy to your real EKS namespace

```
helm install eks-chart-demo .
```

No cluster-specific changes were needed - the same chart genuinely works here, on real shared infrastructure, exactly as written.

## Step 6: Verify it deployed correctly

```
kubectl get pods -l app.kubernetes.io/instance=eks-chart-demo
kubectl exec $(kubectl get pods -l app.kubernetes.io/instance=eks-chart-demo -o jsonpath='{.items[0].metadata.name}') -- env | grep GREETING
```

`app.kubernetes.io/instance=eks-chart-demo` is a label Helm automatically adds to every object it creates, tagged with your release name - using it as a selector is a reliable way to find 'everything this specific Helm release created' without knowing exact Pod names in advance. The `$(...)` part looks up the Pod's name automatically so you don't have to copy-paste it by hand.

## Verify It Worked

- The Pod's environment shows `GREETING=hello-from-helm-on-eks`
- You made zero modifications to the chart specifically 'for AWS' - it's the same chart, unchanged

## Common Mistakes

- **Modifying the chart to work around something on EKS specifically** - That defeats the actual point of this exercise - the whole value of Helm is that the SAME chart works across environments without modification. If something doesn't work, that's worth investigating, not routing around.
- **Skipping `helm template` and going straight to `helm install`** - Catching a templating mistake locally is much faster than debugging a failed cluster deployment - always render first when you're not sure a change is correct.

## Cleanup (do this now, not later)

```
helm uninstall eks-chart-demo
```

Then complete the mandatory checklist: **`cleanup-checklists/12-cleanup-checklist.md`**.
