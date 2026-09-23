# Exercise 2: Lambda with Environment Config

## What This Exercise Does

AWS Lambda runs your code without you managing any server at all - AWS invokes your function only when needed and you pay only for the time it actually runs. This is architecturally different from the image you just pushed to ECR in Exercise 1: there's no persistent container listening on a port, and no `docker run` equivalent. Instead, you write a **handler function**, AWS invokes it per-request, and configuration is passed in through **environment variables** set on the function itself - the serverless equivalent of a `.env` file. This exercise deploys a small function, configures it via an environment variable, and proves the same function can behave differently just by changing that variable, without touching any code.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.00 (Lambda's free tier covers this easily)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 1 complete (confirms your AWS CLI and IAM credentials work)
- The shared IAM role `course-lambda-basic-execution` already exists in your account - your instructor sets this up once for the whole cohort; you never create your own Lambda execution role
- `zip` installed on your Ubuntu VM (`sudo apt install zip` if not)
- VS Code open on a working folder

## Steps

## Step 1: Fetch the shared execution role's ARN

```
ROLE_ARN=$(aws iam get-role --role-name course-lambda-basic-execution --query 'Role.Arn' --output text)
echo $ROLE_ARN
```

You cannot create your own IAM role for Lambda to assume - your account's permissions specifically block this. Every Lambda exercise in this course fetches this one pre-created role by name instead.

## Step 2: Write the handler function - notice it reads its message from an environment variable, not hardcoded

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
mkdir -p ~/course/lambda-exercise && cd ~/course/lambda-exercise
```

**In VS Code:** Create a new file named `index.js` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
exports.handler = async () => {
  const message = process.env.MESSAGE || 'default message';
  return { statusCode: 200, body: message };
};
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > index.js <<'EOF'
exports.handler = async () => {
  const message = process.env.MESSAGE || 'default message';
  return { statusCode: 200, body: message };
};
EOF
```


Open this file in VS Code and look closely at `process.env.MESSAGE` - this one line is what lets the SAME deployed code produce different output depending on configuration, exactly like an environment variable in a Docker container.

## Step 3: Package the function

```
zip fn.zip index.js
```

AWS Lambda deploys code as a .zip archive, not as raw source files - this command bundles your single `index.js` file into `fn.zip`, which the next step uploads. For a function with dependencies, you'd also zip up a `node_modules` folder here, but this function has none.

## Step 4: Create the function with a custom MESSAGE value

```
aws lambda create-function --function-name msg-$PARTICIPANT --runtime nodejs20.x --handler index.handler --role $ROLE_ARN --zip-file fileb://fn.zip --environment 'Variables={MESSAGE=hello from Lambda env var}' --tags Owner=$PARTICIPANT,Course=cloudnative-course-2026 --region us-east-1
```

Breaking down the key flags: `--runtime nodejs20.x` tells Lambda which language/version to run your code with. `--handler index.handler` tells it WHICH function to call inside your code - the format is `<filename-without-extension>.<exported-function-name>`, matching `exports.handler` in index.js. `--zip-file fileb://fn.zip` uploads the package from Step 3 (the `fileb://` prefix means 'read this as binary from a local file'). `--environment 'Variables={...}'` is where your MESSAGE value actually gets set.

## Step 5: Invoke it and see the configured message come back

```
aws lambda invoke --function-name msg-$PARTICIPANT --region us-east-1 out.json
cat out.json
```

`aws lambda invoke` triggers a real, one-time execution of your function and writes whatever it returns into the local file `out.json` - there's no separate 'start the server' step like Docker; the function only runs for the duration of this single invocation, then stops.

## Step 6: Change ONLY the configuration, not the code, and invoke again

```
aws lambda update-function-configuration --function-name msg-$PARTICIPANT --environment 'Variables={MESSAGE=a completely different message, same code}' --region us-east-1
aws lambda invoke --function-name msg-$PARTICIPANT --region us-east-1 out2.json
cat out2.json
```

This is the actual point of the exercise: the deployed handler code never changed, but its behavior did, purely through configuration - the same principle as Docker environment variables, applied to serverless.

## Verify It Worked

- `out.json` contains your first custom message in the `body` field
- `out2.json` contains the second, different message - proving the same code produced different output

## Common Mistakes

- **Expecting to use `-p` port mapping like Docker** - Lambda doesn't work that way at all - there's no persistent container listening on a port; AWS invokes your handler function per-request, and there's nothing to 'map' a port to.
- **Trying to pass your own IAM role** - You don't have `iam:CreateRole` - you must fetch `course-lambda-basic-execution` by name, exactly as shown in Step 1.
- **Forgetting to update BOTH the function code and the config independently** - This exercise deliberately changes ONLY the environment variable in Step 5, without touching the code at all - that's the point being demonstrated, not an oversight.

## Cleanup (do this now, not later)

```
aws lambda delete-function --function-name msg-$PARTICIPANT --region us-east-1
```

Then complete the mandatory checklist: **`cleanup-checklists/02-cleanup-checklist.md`**.
