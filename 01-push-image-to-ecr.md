# Exercise 1: Push an Image to ECR

## What This Exercise Does

A container image is a portable, versioned package - the exact same one that runs on your laptop can run anywhere else, including AWS. **Amazon ECR** (Elastic Container Registry) is AWS's private registry for these images - conceptually identical to Docker Hub, just private to your account and authenticated via IAM instead of a Docker Hub login. This exercise builds a small image entirely from one self-contained Dockerfile and pushes it to your own private ECR repository.

## Time and Cost

- **Time:** ~15 min
- **Cost:** ~$0.00-0.01 (a few MB of ECR storage for a few minutes)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- AWS CLI v2 installed and configured with your issued IAM credentials
- Docker installed and running on your Ubuntu VM (`docker version` shows both a Client and Server section)
- VS Code open on a working folder, e.g. `~/course/ecr-exercise`
- `$PARTICIPANT` exported for this session (see 00-START-HERE.md)

## Steps

## Step 1: Create your working folder and app source

In a terminal (VS Code's integrated terminal works well here - `` Ctrl+` ``):

```
mkdir -p ~/course/ecr-exercise && cd ~/course/ecr-exercise
```

**In VS Code:** Create a new file named `index.js` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
require('http').createServer((req, res) => res.end('Hello from ECR!')).listen(3000);
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > index.js <<'EOF'
require('http').createServer((req, res) => res.end('Hello from ECR!')).listen(3000);
EOF
```


`mkdir -p` creates the folder (and any missing parent folders) without erroring if it already exists - safe to re-run. `index.js` is a minimal Node.js web server: it starts an HTTP server that responds to every request with the same fixed text, listening on port 3000. This is deliberately trivial - the point of this exercise is the ECR push mechanics, not the application code.

## Step 2: Write the Dockerfile - this ONE file defines everything you're about to push

**In VS Code:** Create a new file named `Dockerfile` in your current working folder (Explorer panel, right-click your folder > New File), paste this in, and save (`Ctrl+S`):

```
FROM node:20 AS build
WORKDIR /app
COPY index.js .

FROM node:20-slim
WORKDIR /app
COPY --from=build /app/index.js .
CMD ["node", "index.js"]
```

*(If you'd rather use the terminal instead of VS Code, this does the same thing:)*

```
cat > Dockerfile <<'EOF'
FROM node:20 AS build
WORKDIR /app
COPY index.js .

FROM node:20-slim
WORKDIR /app
COPY --from=build /app/index.js .
CMD ["node", "index.js"]
EOF
```


This is a multi-stage build: the `build` stage has the full Node.js toolchain, but the final image (`node:20-slim`) only contains what's actually needed to run the app - a smaller, cheaper image to store and push. Open this file in VS Code now and read through it before continuing, so you know exactly what you're about to build.

## Step 3: Build the image locally first, and confirm it works, before touching AWS at all

```
docker build -t myapp:v1 .
docker run -d --name test-container -p 3000:3000 myapp:v1
curl http://localhost:3000
docker rm -f test-container
```

Always confirm an image works locally before pushing it anywhere - debugging a broken image is much easier on your own machine than after it's in a registry.

## Step 4: Create your own ECR repository

```
aws ecr create-repository --repository-name myapp-$PARTICIPANT --region us-east-1 --tags Key=Owner,Value=$PARTICIPANT Key=Course,Value=cloudnative-course-2026
```

The repository name includes your `$PARTICIPANT` value specifically so 25 people sharing one AWS account never collide on a name.

## Step 5: Authenticate Docker to ECR

```
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
```

`aws ecr get-login-password` generates a temporary authentication token (valid for 12 hours) and pipes it straight into `docker login` - you never handle a long-lived password for this.

## Step 6: Tag and push the image

```
docker tag myapp:v1 $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/myapp-$PARTICIPANT:v1
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/myapp-$PARTICIPANT:v1
```

Docker tags identify WHICH registry an image belongs to - `docker tag` doesn't move or copy anything, it just gives your existing local image an additional name that happens to point at your ECR repository.

## Verify It Worked

- `aws ecr describe-images --repository-name myapp-$PARTICIPANT --region us-east-1` shows one image with tag `v1`
- The image's size in the AWS Console (ECR > Repositories > myapp-$PARTICIPANT) is a few tens of MB, consistent with the slim final stage

## Common Mistakes

- **Forgetting the region flag** - ECR is region-scoped - always confirm you're working in us-east-1, matching your account's permission boundary.
- **Skipping the Owner/Course tags on the repository** - Your account's permission boundary requires these on several resource types - get in the habit now, since later exercises assume it.
- **Testing the image AFTER pushing instead of before** - Always confirm an image runs correctly with `docker run` locally first - this exercise's Step 3 does this deliberately, in order.
- **Not understanding what `docker tag` actually does** - It doesn't move or copy the image - it adds an additional name/reference to the SAME image, one that happens to point at your ECR repository's address.

## Cleanup (do this now, not later)

```
aws ecr delete-repository --repository-name myapp-$PARTICIPANT --region us-east-1 --force
docker rmi myapp:v1
```

Then complete the mandatory checklist: **`cleanup-checklists/01-cleanup-checklist.md`**.
