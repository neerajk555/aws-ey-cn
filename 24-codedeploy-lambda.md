# Exercise 24: Blue/Green Lambda Deployment with CodeDeploy

## What This Exercise Does

Exercise 8 did a rolling update on Kubernetes - **AWS CodeDeploy** provides a similar safety mechanism for Lambda specifically: instead of instantly switching 100% of traffic to a new function version, it shifts traffic gradually (or all-at-once, your choice) between an old and new version via a traffic-shifting alias, with automatic rollback if a CloudWatch alarm fires during the shift. This exercise deploys a new version of your Lambda function through CodeDeploy rather than overwriting it directly, and observes the alias move between versions.

## Time and Cost

- **Time:** ~25 min
- **Cost:** ~$0.00 (CodeDeploy itself is free for Lambda deployments; you only pay Lambda's normal per-invocation cost)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 2 complete (or willing to quickly recreate the function)
- The shared IAM role `course-codedeploy-execution` already exists

## Steps

## Step 1: Publish a numbered version of your existing function (not $LATEST)

```
aws lambda publish-version --function-name msg-$PARTICIPANT --region us-east-1 --query Version --output text
```

Note the version number this returns (likely 1, if you haven't published before) - CodeDeploy needs to shift traffic between two specific PUBLISHED versions, not the mutable $LATEST pointer you've been using in every earlier Lambda exercise.

## Step 2: Create an alias pointing at that version - this is what CodeDeploy will actually shift

```
aws lambda create-alias --function-name msg-$PARTICIPANT --name live --function-version 1 --region us-east-1
```

From here on, think of `live` as your function's stable public name - callers should invoke `msg-$PARTICIPANT:live`, not a specific version number, so that CodeDeploy can move the alias underneath them without callers needing to change anything.

## Step 3: Change the function's code and publish a NEW version

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
cd ~/course/lambda-exercise
```

**In VS Code:** Create a new file named `index.js` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
exports.handler = async () => ({ statusCode: 200, body: 'version 2 - deployed via CodeDeploy' });
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > index.js <<'EOF'
exports.handler = async () => ({ statusCode: 200, body: 'version 2 - deployed via CodeDeploy' });
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
zip fn.zip index.js
aws lambda update-function-code --function-name msg-$PARTICIPANT --zip-file fileb://fn.zip --region us-east-1
aws lambda publish-version --function-name msg-$PARTICIPANT --region us-east-1 --query Version --output text
```


Note this new version number too (likely 2) - `live` still points at version 1 at this point; publishing a new version doesn't move the alias by itself.

## Step 4: Create a CodeDeploy application and deployment group for Lambda

```
DEPLOY_ROLE_ARN=$(aws iam get-role --role-name course-codedeploy-execution --query 'Role.Arn' --output text)
aws deploy create-application --application-name app-$PARTICIPANT --compute-platform Lambda --region us-east-1
aws deploy create-deployment-group --application-name app-$PARTICIPANT --deployment-group-name dg-$PARTICIPANT --service-role-arn $DEPLOY_ROLE_ARN --deployment-config-name CodeDeployDefault.LambdaAllAtOnce --region us-east-1
```

`CodeDeployDefault.LambdaAllAtOnce` shifts 100% of traffic immediately - CodeDeploy also offers gradual options (`LambdaLinear10PercentEvery1Minute`, `LambdaCanary10Percent5Minutes`) for safer production rollouts; all-at-once is used here to keep this exercise's timing simple and predictable.

## Step 5: Trigger the deployment, shifting the alias from version 1 to version 2

**In VS Code:** Create a new file named `~/course/appspec.json` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
{
  "version": 0.0,
  "Resources": [{
    "myLambdaFunction": {
      "Type": "AWS::Lambda::Function",
      "Properties": {
        "Name": "FUNCTION_PLACEHOLDER",
        "Alias": "live",
        "CurrentVersion": "1",
        "TargetVersion": "2"
      }
    }
  }]
}
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/appspec.json <<'EOF'
{
  "version": 0.0,
  "Resources": [{
    "myLambdaFunction": {
      "Type": "AWS::Lambda::Function",
      "Properties": {
        "Name": "FUNCTION_PLACEHOLDER",
        "Alias": "live",
        "CurrentVersion": "1",
        "TargetVersion": "2"
      }
    }
  }]
}
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
sed -i "s/FUNCTION_PLACEHOLDER/msg-$PARTICIPANT/" ~/course/appspec.json
aws deploy create-deployment --application-name app-$PARTICIPANT --deployment-group-name dg-$PARTICIPANT --revision "revisionType=AppSpecContent,appSpecContent={content='$(cat ~/course/appspec.json)'}" --region us-east-1 --query deploymentId --output text
```


The AppSpec file is CodeDeploy's own format for describing exactly what to change - `CurrentVersion`/`TargetVersion` tells it precisely which two published versions to shift the `live` alias between, and since this deployment config is `LambdaAllAtOnce`, the shift happens immediately rather than gradually.

## Step 6: Confirm the alias now points at version 2

```
aws lambda get-alias --function-name msg-$PARTICIPANT --name live --region us-east-1 --query FunctionVersion --output text
aws lambda invoke --function-name msg-$PARTICIPANT:live --region us-east-1 out.json
cat out.json
```

Invoking `msg-$PARTICIPANT:live` (with the alias suffix) rather than the bare function name is what actually exercises the alias you've been manipulating this whole exercise - invoking the bare name would always hit $LATEST regardless of anything CodeDeploy did.

## Verify It Worked

- get-alias shows FunctionVersion is now 2, not 1
- Invoking msg-$PARTICIPANT:live returns your v2 message, without you ever manually pointing the alias yourself

## Common Mistakes

- **Invoking the function by name alone instead of by alias** - Once you introduce an alias, `msg-$PARTICIPANT` and `msg-$PARTICIPANT:live` can diverge - always invoke through the alias to see what CodeDeploy actually shifted.
- **Confusing a published version with $LATEST** - $LATEST is mutable and changes every time you update the function's code - CodeDeploy specifically needs immutable, numbered versions to shift between, which is why Steps 1 and 3 explicitly publish rather than just update.

## Cleanup (do this now, not later)

```
aws deploy delete-deployment-group --application-name app-$PARTICIPANT --deployment-group-name dg-$PARTICIPANT --region us-east-1
aws deploy delete-application --application-name app-$PARTICIPANT --region us-east-1
aws lambda delete-alias --function-name msg-$PARTICIPANT --name live --region us-east-1
aws lambda delete-function --function-name msg-$PARTICIPANT --region us-east-1
```

Then complete the mandatory checklist: **`cleanup-checklists/24-cleanup-checklist.md`**.
