# Cleanup Checklist: Exercise 16 - API Gateway + Lambda

Mandatory - do not start the next exercise until every box below is checked.

- [ ] API Gateway `api-$PARTICIPANT` deleted (`aws apigatewayv2 get-apis --region us-east-1` no longer shows it)
- [ ] The `curl` URL from this exercise no longer responds (confirms deletion took effect, not just the API object)
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
