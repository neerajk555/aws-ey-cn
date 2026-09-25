# Cleanup Checklist: Exercise 23 - CodePipeline

Mandatory - do not start the next exercise until every box below is checked.

- [ ] Pipeline `pipeline-$PARTICIPANT` deleted (`aws codepipeline list-pipelines --region us-east-1` no longer shows it) - this is the highest-priority cleanup item in this entire course, given its per-month cost
- [ ] CodeBuild project `pipeline-build-$PARTICIPANT` deleted
- [ ] S3 bucket `pipeline-artifacts-$PARTICIPANT` emptied and deleted
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
