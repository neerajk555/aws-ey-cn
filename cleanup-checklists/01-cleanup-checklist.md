# Cleanup Checklist: Exercise 1 - Push an Image to ECR

Mandatory - do not start the next exercise until every box below is checked.

- [ ] ECR repository `myapp-$PARTICIPANT` deleted (`aws ecr describe-repositories --region us-east-1` no longer lists it)
- [ ] Local Docker image `myapp:v1` removed (`docker images | grep myapp` shows nothing)
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
