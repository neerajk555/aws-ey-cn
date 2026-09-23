# Cleanup Checklist: Exercise 6 - Expose via ALB

Mandatory - do not start the next exercise until every box below is checked.

- [ ] Ingress `web` deleted from your namespace (`kubectl get ingress` shows nothing)
- [ ] **Confirmed in the AWS Console** (EC2 > Load Balancers) that NO load balancer tagged with your username remains - this is the real proof, not just the Ingress object disappearing
- [ ] `kubectl get svc,ingress` in your namespace shows nothing left from this exercise
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
