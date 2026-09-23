# Exercise 14: CloudWatch Logs and Alarm

## What This Exercise Does

You can't fix what you can't see. On AWS, container and function logs flow automatically into **CloudWatch Logs** - no separate logging agent to install for Lambda specifically. **CloudWatch Alarms** can then watch a metric (like a function's error count) and notify you when it crosses a threshold, the cloud-scale equivalent of watching a terminal yourself. This exercise generates real log data from a Lambda function, reads it back through CloudWatch, and sets up one basic alarm.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.00-0.01 (a handful of log events and one alarm, both far under free-tier limits)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 2 complete, OR willing to quickly recreate a small Lambda function for this exercise

## Steps

## Step 1: Recreate a small Lambda function if you already cleaned up Exercise 2's

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
ROLE_ARN=$(aws iam get-role --role-name course-lambda-basic-execution --query 'Role.Arn' --output text)
mkdir -p ~/course/logs-exercise && cd ~/course/logs-exercise
```

**In VS Code:** Create a new file named `index.js` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
exports.handler = async () => ({ statusCode: 200, body: 'logging test' });
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > index.js <<'EOF'
exports.handler = async () => ({ statusCode: 200, body: 'logging test' });
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
zip fn.zip index.js
aws lambda create-function --function-name msg-$PARTICIPANT --runtime nodejs20.x --handler index.handler --role $ROLE_ARN --zip-file fileb://fn.zip --tags Owner=$PARTICIPANT,Course=cloudnative-course-2026 --region us-east-1
```


Skip this step entirely if Exercise 2's function still exists (`aws lambda get-function --function-name msg-$PARTICIPANT --region us-east-1` to check).

## Step 2: Invoke it a few times to generate real log data

```
for i in 1 2 3; do aws lambda invoke --function-name msg-$PARTICIPANT --region us-east-1 out.json; done
```

A `for` loop running the same invoke command 3 times - each invocation writes its own log entry to CloudWatch behind the scenes, which is what you'll read back in the next step. A single invocation would work too, but a few gives you more realistic log output to look at.

## Step 3: Read the logs back through CloudWatch

```
aws logs tail /aws/lambda/msg-$PARTICIPANT --region us-east-1
```

Every Lambda function automatically gets a log group named `/aws/lambda/<function-name>` - you never had to create or configure this yourself.

## Step 4: Create a basic alarm on the function's error count

```
aws cloudwatch put-metric-alarm --alarm-name errors-$PARTICIPANT --metric-name Errors --namespace AWS/Lambda --dimensions Name=FunctionName,Value=msg-$PARTICIPANT --statistic Sum --period 300 --threshold 1 --comparison-operator GreaterThanOrEqualToThreshold --evaluation-periods 1 --region us-east-1
```

Reading this as a sentence: alarm when the SUM of the Errors metric, for this specific function, over a 300-second (5-minute) period, is GREATER THAN OR EQUAL TO a threshold of 1, checked over 1 evaluation period. In plain terms - 'tell me if this function throws even one error in a given 5-minute window.'

## Step 5: Confirm the alarm exists

```
aws cloudwatch describe-alarms --alarm-names errors-$PARTICIPANT --region us-east-1 --query 'MetricAlarms[0].StateValue'
```

A state of `OK` or `INSUFFICIENT_DATA` is expected and fine - this exercise is about SETTING UP the monitoring correctly, not necessarily forcing it to fire. It only evaluates every 5 minutes (the period you set) and needs an actual error to cross the threshold.

## Verify It Worked

- `aws logs tail` shows your invocations' output
- The alarm shows a valid state (`OK` or `INSUFFICIENT_DATA`)

## Common Mistakes

- **Expecting the alarm to fire immediately** - It evaluates on a 5-minute period and needs a real error to trigger - this exercise proves the monitoring is correctly WIRED UP, which is the actual point.
- **Forgetting the log group's exact naming convention** - It's always `/aws/lambda/<function-name>` - AWS creates and names this automatically, you never configure it yourself.

## Cleanup (do this now, not later)

```
aws cloudwatch delete-alarms --alarm-names errors-$PARTICIPANT --region us-east-1
aws lambda delete-function --function-name msg-$PARTICIPANT --region us-east-1
```

Then complete the mandatory checklist: **`cleanup-checklists/14-cleanup-checklist.md`**.
