# Cleanup Checklist: Exercise 18 - X-Ray Tracing

Mandatory - do not start the next exercise until every box below is checked.

- [ ] Lambda tracing mode reverted to PassThrough (`aws lambda get-function-configuration --function-name msg-$PARTICIPANT --region us-east-1 --query TracingConfig`)
- [ ] Note: X-Ray trace data itself ages out automatically after 30 days at no cost - no manual trace deletion needed
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
