# Cleanup Checklist: Exercise 26 - App Mesh

Mandatory - do not start the next exercise until every box below is checked.

- [ ] Mesh, VirtualRouter, VirtualService, and both VirtualNodes deleted
- [ ] Both backend Deployments and Services deleted
- [ ] Namespace labels for mesh/sidecar injection removed (`kubectl get namespace ns-$PARTICIPANT --show-labels` no longer shows them) - otherwise every FUTURE Pod you create in later exercises would unexpectedly get an Envoy sidecar injected
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
