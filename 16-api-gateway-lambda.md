# Exercise 16: Expose a Lambda Function Through API Gateway

## What This Exercise Does

A Lambda function on its own has no public URL - Exercise 2's function could only be invoked via the AWS CLI, never by a browser or another service. **Amazon API Gateway** is what turns a Lambda function into a real HTTP API: it accepts incoming requests, routes them to your function, and returns whatever the function returns. This exercise builds a minimal HTTP API backed by your own Lambda function and calls it with a real HTTP request, not the AWS CLI.

## Time and Cost

- **Time:** ~20 min
- **Cost:** ~$0.00 (API Gateway's free tier covers this easily; you pay per-request beyond it, and this exercise makes only a handful)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 2 complete (or willing to quickly recreate a small Lambda function)
- Participant grant policy updated to include apigateway:* (already done if you're using the current policies/participant-grant.json)

## Steps

## Step 1: Confirm your Lambda function still exists, or recreate it

```
aws lambda get-function --function-name msg-$PARTICIPANT --region us-east-1
```

If this errors, redo Exercise 2's Steps 1-4 first - this exercise builds on that same function rather than duplicating its setup.

## Step 2: Create an HTTP API (the simpler, cheaper API Gateway type - REST APIs exist too but cost more and add complexity this exercise doesn't need)

```
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
API_ID=$(aws apigatewayv2 create-api --name api-$PARTICIPANT --protocol-type HTTP --target arn:aws:lambda:us-east-1:${ACCOUNT_ID}:function:msg-$PARTICIPANT --region us-east-1 --query ApiId --output text)
echo $API_ID
```

The `--target` shorthand does three things in one command: creates the API, creates a Lambda integration pointing at your function, and creates a default catch-all route - normally three separate API calls, collapsed here since this is the simplest possible API shape.

## Step 3: Allow API Gateway to actually invoke your function

```
aws lambda add-permission --function-name msg-$PARTICIPANT --statement-id apigw-$PARTICIPANT --action lambda:InvokeFunction --principal apigateway.amazonaws.com --source-arn "arn:aws:execute-api:us-east-1:${ACCOUNT_ID}:${API_ID}/*/*" --region us-east-1
```

This is a RESOURCE-BASED policy on the Lambda function itself (separate from your own IAM identity policy) - without it, API Gateway can reach your function but gets an authorization error trying to invoke it, since Lambda checks both your IAM permissions AND this resource policy independently.

## Step 4: Get your API's public URL and call it

```
API_URL=$(aws apigatewayv2 get-apis --region us-east-1 --query "Items[?Name=='api-$PARTICIPANT'].ApiEndpoint" --output text)
echo $API_URL
curl $API_URL
```

This is a REAL public HTTPS endpoint now - no VPN, no AWS CLI, no credentials needed to reach it. Anyone with this URL can call your Lambda function, which is worth pausing on: this is very different from every namespace-scoped EKS exercise so far, where access was tightly restricted to you.

## Verify It Worked

- curl against your API's endpoint returns the same message your Lambda function returns when invoked directly
- You can explain why the add-permission step in Step 3 was necessary even though your own IAM user already has full Lambda access

## Common Mistakes

- **Forgetting Step 3 (resource-based permission)** - Your OWN IAM permissions letting you call `aws lambda invoke` are irrelevant here - API Gateway is a DIFFERENT caller, and Lambda checks whether API Gateway specifically is allowed to invoke your function, independent of who created the API.
- **Using a REST API instead of an HTTP API for something this simple** - REST APIs support more features (usage plans, API keys, request validation) but cost more per-request and take longer to configure - HTTP APIs are the right default unless you specifically need a REST-only feature.
- **Leaving this running and forgetting it's a PUBLIC endpoint** - Unlike every EKS exercise, this URL is reachable by anyone on the internet, not just you - all the more reason to clean it up promptly.

## Cleanup (do this now, not later)

```
aws apigatewayv2 delete-api --api-id $API_ID --region us-east-1
```

Then complete the mandatory checklist: **`cleanup-checklists/16-cleanup-checklist.md`**.
