# Exercise 23: Build a CI/CD Pipeline with CodePipeline

## What This Exercise Does

Exercise 13 ran a single CodeBuild project by hand - a real CI/CD pipeline chains multiple stages together automatically: pull source code, build it, deploy it, with no manual step in between once it's triggered. **AWS CodePipeline** is that orchestration layer. This exercise builds a minimal two-stage pipeline (Source from S3, then Build via CodeBuild) and triggers it by uploading a new source file, watching the pipeline pick it up automatically.

## Time and Cost

- **Time:** ~25 min
- **Cost:** ~$1.00 (CodePipeline charges a flat rate per active pipeline per month, prorated - the only exercise in this course with a genuinely non-trivial cost; delete it promptly)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 13 complete (reuses the same CodeBuild concepts)
- The shared IAM role `course-codepipeline-execution` already exists (your instructor's `create-shared-roles.ps1` creates this)

## Steps

## Step 1: Create your own S3 bucket for pipeline source artifacts

```
aws s3 mb s3://pipeline-artifacts-$PARTICIPANT --region us-east-1
```

Your IAM grant scopes S3 access specifically to a bucket named exactly `pipeline-artifacts-$PARTICIPANT` - a different bucket name would be denied, the same per-participant isolation pattern as SSM's path prefix in Exercise 22, just applied to S3's bucket-naming instead.

## Step 2: Create a minimal source file and upload it as your pipeline's starting point

```
mkdir -p ~/course/pipeline-demo && cd ~/course/pipeline-demo
echo 'console.log("pipeline build ran");' > index.js
zip source.zip index.js
aws s3 cp source.zip s3://pipeline-artifacts-$PARTICIPANT/source.zip
```

This uploaded zip file is what CodePipeline's Source stage watches - any time a new object lands at this exact S3 key, the pipeline treats it as new source code and starts a fresh run, which is exactly what Step 6 relies on.

## Step 3: Fetch both shared roles you'll need

```
PIPELINE_ROLE_ARN=$(aws iam get-role --role-name course-codepipeline-execution --query 'Role.Arn' --output text)
BUILD_ROLE_ARN=$(aws iam get-role --role-name course-codebuild-execution --query 'Role.Arn' --output text)
```

Two DIFFERENT roles for two different purposes: the pipeline role lets CodePipeline itself orchestrate stages and read/write your S3 artifact bucket, while the build role is the same one Exercise 13 used, letting CodeBuild actually run its build commands.

## Step 4: Create a CodeBuild project as the pipeline's build stage

```
aws codebuild create-project --name pipeline-build-$PARTICIPANT \
  --source type=CODEPIPELINE,buildspec='version: 0.2\nphases:\n  build:\n    commands:\n      - echo Building via pipeline\n      - cat index.js' \
  --artifacts type=CODEPIPELINE \
  --environment type=LINUX_CONTAINER,image=aws/codebuild/standard:7.0,computeType=BUILD_GENERAL1_SMALL \
  --service-role $BUILD_ROLE_ARN \
  --tags key=Owner,value=$PARTICIPANT key=Course,value=cloudnative-course-2026 \
  --region us-east-1
```

Notice `type=CODEPIPELINE` for both source and artifacts here, instead of Exercise 13's `NO_SOURCE`/`NO_ARTIFACTS` - this project is now specifically wired to receive its input from, and hand its output back to, a pipeline rather than running standalone.

## Step 5: Create the pipeline itself, wiring the S3 source to the CodeBuild stage

**In VS Code:** Create a new file named `~/course/pipeline-def.json` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
{
  "pipeline": {
    "name": "pipeline-PARTICIPANT_PLACEHOLDER",
    "roleArn": "PIPELINE_ROLE_PLACEHOLDER",
    "artifactStore": { "type": "S3", "location": "pipeline-artifacts-PARTICIPANT_PLACEHOLDER" },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "S3Source", "actionTypeId": {"category": "Source", "owner": "AWS", "provider": "S3", "version": "1"},
          "outputArtifacts": [{"name": "SourceOutput"}],
          "configuration": {"S3Bucket": "pipeline-artifacts-PARTICIPANT_PLACEHOLDER", "S3ObjectKey": "source.zip"}
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "CodeBuild", "actionTypeId": {"category": "Build", "owner": "AWS", "provider": "CodeBuild", "version": "1"},
          "inputArtifacts": [{"name": "SourceOutput"}],
          "outputArtifacts": [{"name": "BuildOutput"}],
          "configuration": {"ProjectName": "pipeline-build-PARTICIPANT_PLACEHOLDER"}
        }]
      }
    ]
  }
}
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > ~/course/pipeline-def.json <<'EOF'
{
  "pipeline": {
    "name": "pipeline-PARTICIPANT_PLACEHOLDER",
    "roleArn": "PIPELINE_ROLE_PLACEHOLDER",
    "artifactStore": { "type": "S3", "location": "pipeline-artifacts-PARTICIPANT_PLACEHOLDER" },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "S3Source", "actionTypeId": {"category": "Source", "owner": "AWS", "provider": "S3", "version": "1"},
          "outputArtifacts": [{"name": "SourceOutput"}],
          "configuration": {"S3Bucket": "pipeline-artifacts-PARTICIPANT_PLACEHOLDER", "S3ObjectKey": "source.zip"}
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "CodeBuild", "actionTypeId": {"category": "Build", "owner": "AWS", "provider": "CodeBuild", "version": "1"},
          "inputArtifacts": [{"name": "SourceOutput"}],
          "outputArtifacts": [{"name": "BuildOutput"}],
          "configuration": {"ProjectName": "pipeline-build-PARTICIPANT_PLACEHOLDER"}
        }]
      }
    ]
  }
}
EOF
```

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
sed -i "s/PARTICIPANT_PLACEHOLDER/$PARTICIPANT/g; s#PIPELINE_ROLE_PLACEHOLDER#$PIPELINE_ROLE_ARN#" ~/course/pipeline-def.json
aws codepipeline create-pipeline --cli-input-json file://~/course/pipeline-def.json --region us-east-1
```


This JSON definition is genuinely the most complex structure in this course - built as a real file with a heredoc specifically so you can open it in VS Code and read through the Source -> Build stage chain clearly, rather than trying to follow it as one long inline command.

## Step 6: Trigger the pipeline by uploading a changed source file

```
echo 'console.log("pipeline build ran - v2");' > index.js
zip source.zip index.js
aws s3 cp source.zip s3://pipeline-artifacts-$PARTICIPANT/source.zip
aws codepipeline get-pipeline-state --name pipeline-$PARTICIPANT --region us-east-1 --query 'stageStates[].{Stage:stageName,Status:latestExecution.status}'
```

A new object uploaded to the S3 source location automatically starts a new pipeline execution - no manual 'run pipeline' click or command needed, which is the actual point of CI/CD. Re-run the get-pipeline-state command a few times over the next minute to watch both stages move from InProgress to Succeeded.

## Verify It Worked

- Both the Source and Build stages show status Succeeded after uploading the v2 source file
- You never manually triggered a build - the S3 upload alone caused it

## Common Mistakes

- **Leaving the pipeline running** - Unlike everything else in this course, CodePipeline has a real flat monthly charge per active pipeline - this is the one exercise where 'I'll clean it up later today' has a meaningfully higher cost than the others.
- **Forgetting the artifact store must be a bucket you actually have permission to use** - Your grant only covers `pipeline-artifacts-$PARTICIPANT` exactly - a typo in the bucket name here fails with a permissions error that can look confusingly like a CodePipeline configuration problem instead.

## Cleanup (do this now, not later)

```
aws codepipeline delete-pipeline --name pipeline-$PARTICIPANT --region us-east-1
aws codebuild delete-project --name pipeline-build-$PARTICIPANT --region us-east-1
aws s3 rm s3://pipeline-artifacts-$PARTICIPANT --recursive
aws s3 rb s3://pipeline-artifacts-$PARTICIPANT
```

Then complete the mandatory checklist: **`cleanup-checklists/23-cleanup-checklist.md`**.
