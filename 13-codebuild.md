# Exercise 13: CodeBuild

## What This Exercise Does

CI/CD automates the build-test-deploy cycle so it happens identically every time, without manual steps. **AWS CodeBuild** runs the build stage of that cycle as a managed service - you define the commands to run in a **buildspec**, CodeBuild provisions a temporary build environment, runs them, and tears the environment down when finished. This exercise runs a minimal real build using your account's shared CodeBuild execution role, the same 'fetch a pre-created role by name, never create your own' pattern used for Lambda in Exercise 2.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.01-0.02 (CodeBuild bills per build-minute; a small build is fractions of a cent)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 1 complete (confirms AWS CLI and IAM credentials work)
- The shared IAM role `course-codebuild-execution` already exists in your account

## Steps

## Step 1: Fetch the shared CodeBuild role's ARN

```
ROLE_ARN=$(aws iam get-role --role-name course-codebuild-execution --query 'Role.Arn' --output text)
echo $ROLE_ARN
```

Same pattern as Exercise 2's Lambda role - you cannot create your own CodeBuild service role, so every CodeBuild exercise fetches this pre-created one by name.

## Step 2: Create a minimal CodeBuild project

```
aws codebuild create-project --name build-$PARTICIPANT \
  --source type=NO_SOURCE,buildspec='version: 0.2\nphases:\n  build:\n    commands:\n      - echo Build stage running on CodeBuild' \
  --artifacts type=NO_ARTIFACTS \
  --environment type=LINUX_CONTAINER,image=aws/codebuild/standard:7.0,computeType=BUILD_GENERAL1_SMALL \
  --service-role $ROLE_ARN \
  --tags key=Owner,value=$PARTICIPANT key=Course,value=cloudnative-course-2026 \
  --region us-east-1
```

`NO_SOURCE` means this build doesn't pull code from anywhere - the buildspec's commands are defined directly on the project, which is enough to prove the build mechanism works without needing a real source repository.

## Step 3: Start a build

```
BUILD_ID=$(aws codebuild start-build --project-name build-$PARTICIPANT --region us-east-1 --query 'build.id' --output text)
echo $BUILD_ID
```

`start-build` kicks off the build asynchronously and returns immediately with an ID - it doesn't wait for the build to finish. That ID is what the next step uses to check on progress, similar to how `docker build` runs synchronously in your terminal but a cloud build service hands you a reference to check back on instead.

## Step 4: Watch it complete

```
aws codebuild batch-get-builds --ids $BUILD_ID --region us-east-1 --query 'builds[0].buildStatus'
```

Re-run this command every 15-20 seconds until it shows `SUCCEEDED` - usually well under a minute for a build this small.

## Verify It Worked

- `buildStatus` shows `SUCCEEDED`

## Common Mistakes

- **Trying to pass your own IAM role instead of the shared one** - You don't have `iam:CreateRole` - always fetch `course-codebuild-execution` by name, exactly as shown in Step 1.
- **Leaving the CodeBuild project around after finishing** - It doesn't bill when idle, but delete it anyway to keep the shared account tidy for the next exercise or cohort.

## Cleanup (do this now, not later)

```
aws codebuild delete-project --name build-$PARTICIPANT --region us-east-1
```

Then complete the mandatory checklist: **`cleanup-checklists/13-cleanup-checklist.md`**.
