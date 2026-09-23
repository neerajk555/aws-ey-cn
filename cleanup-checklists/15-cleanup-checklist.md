# Cleanup Checklist: Exercise 15 - Capstone on EKS

Mandatory - do not start the next exercise until every box below is checked.

- [ ] `helm list` in your namespace shows nothing
- [ ] `kubectl get all,pvc,configmap,secret` in your namespace shows nothing left over from this exercise or any earlier one
- [ ] Confirmed with your instructor that your namespace is fully clean
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
