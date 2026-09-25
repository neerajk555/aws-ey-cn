# Cleanup Checklist: Exercise 24 - CodeDeploy for Lambda

Mandatory - do not start the next exercise until every box below is checked.

- [ ] CodeDeploy application `app-$PARTICIPANT` and deployment group deleted
- [ ] Lambda alias `live` deleted
- [ ] Lambda function `msg-$PARTICIPANT` deleted (this also removes all its published versions)
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
