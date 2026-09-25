# Exercise 18: Trace a Lambda Function with X-Ray

## What This Exercise Does

CloudWatch Logs (Exercise 14) tells you WHAT happened - X-Ray tells you WHERE TIME WAS SPENT. **AWS X-Ray** traces a request as it flows through your system, breaking down exactly how long each piece took - critical once a request touches more than one service, which yours now does (API Gateway, then Lambda). This exercise turns on tracing for the Exercise 16 API and reads back a real trace showing that breakdown.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.00 (X-Ray's free tier covers this easily - 100,000 traces/month free)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 16 complete - needs a working API Gateway + Lambda integration to trace

## Steps

## Step 1: Enable Active Tracing on your Lambda function

```
aws lambda update-function-configuration --function-name msg-$PARTICIPANT --tracing-config Mode=Active --region us-east-1
```

This single flag tells Lambda to automatically emit trace segments for every invocation - no code changes needed for this basic level of tracing, since Lambda's own runtime handles it.

## Step 2: Enable tracing on the API Gateway stage too

```
echo 'HTTP APIs propagate X-Ray tracing automatically once the Lambda side is enabled in Step 1 - unlike REST APIs, there is no separate stage-level tracing flag to set for this simple case, so this step is confirming that rather than configuring anything new.'
```

This is a real difference worth knowing: API Gateway REST APIs (the older, more feature-rich type) DO require an explicit --tracing-enabled flag on the stage; HTTP APIs (what Exercise 16 built) don't need this extra step.

## Step 3: Generate some real traced traffic

```
for i in 1 2 3 4 5; do curl -s $API_URL > /dev/null; done
```

Five quick, silent requests (`-s` suppresses curl's own output, `> /dev/null` discards the response body) - you don't care about the response content here, only that X-Ray has something real to capture and index for the next step.

## Step 4: Wait a moment, then look at the trace summary

```
sleep 20
aws xray get-trace-summaries --start-time $(date -d '5 minutes ago' +%s) --end-time $(date +%s) --region us-east-1 --query 'TraceSummaries[].{Id:Id,Duration:Duration,ResponseTime:ResponseTime}'
```

X-Ray takes a short time to process and index traces after they happen - if this comes back empty, wait another 15-20 seconds and re-run it.

## Step 5: Pull the full detail of one trace (paste an Id from the previous step)

```
aws xray batch-get-traces --trace-ids <paste-a-trace-id-here> --region us-east-1 --query 'Traces[0].Segments[].Document' --output text
```

This is dense, but look for the overall Duration value and compare it across your 5 requests - real, request-by-request timing data you didn't have to instrument by hand.

## Verify It Worked

- get-trace-summaries returns at least one trace from your 5 test requests
- batch-get-traces shows real segment-level timing data for a specific request

## Common Mistakes

- **Expecting traces to appear instantly** - X-Ray has a short indexing delay - a trace from a request you just made may not be queryable for 10-20 seconds.
- **Enabling tracing but never generating traffic** - Tracing only captures requests that actually happen - Step 3's test traffic is what gives you something to look at in Steps 4-5.

## Cleanup (do this now, not later)

```
aws lambda update-function-configuration --function-name msg-$PARTICIPANT --tracing-config Mode=PassThrough --region us-east-1
```

Then complete the mandatory checklist: **`cleanup-checklists/18-cleanup-checklist.md`**.
