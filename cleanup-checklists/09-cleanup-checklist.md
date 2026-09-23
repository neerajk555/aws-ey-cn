# Cleanup Checklist: Exercise 9 - EBS-Backed Storage

Mandatory - do not start the next exercise until every box below is checked.

- [ ] PVC `my-pvc` deleted from your namespace (`kubectl get pvc` shows nothing)
- [ ] **Explicitly verified** the underlying EBS volume is gone by re-running `aws ec2 describe-volumes --volume-ids $VOLUME_ID --region us-east-1` and confirming it now errors with `InvalidVolume.NotFound`
- [ ] If you no longer have `$VOLUME_ID`, list ALL unattached volumes instead and check for anything unexpected: `aws ec2 describe-volumes --filters Name=status,Values=available --region us-east-1`
- [ ] If any volume DOES still show up, delete it directly: `aws ec2 delete-volume --volume-id <id> --region us-east-1`
- [ ] Ran the check below and confirmed nothing remains

```
aws resourcegroupstaggingapi get-resources --tag-filters Key=Owner,Values=$PARTICIPANT --region us-east-1
```

If that still shows something you created in this exercise, delete it before continuing. Note: this
generic check only sees resources that carry an Owner tag - directly-created resources (ECR, Lambda,
CodeBuild) reliably do, but Kubernetes-provisioned resources (like an EBS volume behind a PVC) aren't
guaranteed to. Where this exercise's own steps gave you a more specific check, trust that one first.
