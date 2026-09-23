# Cleanup Checklist: Exercise 14 - CloudWatch Logs and Alarm

Mandatory - do not start the next exercise until every box below is checked.

- [ ] CloudWatch alarm `errors-$PARTICIPANT` deleted
- [ ] Lambda function `msg-$PARTICIPANT` deleted, if recreated in this exercise
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
