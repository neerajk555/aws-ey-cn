# Exercise 17: Protect Your API with a Cognito Authorizer

## What This Exercise Does

Exercise 16's API is wide open - anyone with the URL can call it. **Amazon Cognito** provides user identity for exactly this problem: a User Pool holds real user accounts, and API Gateway can require a valid Cognito-issued token before it will route a request through at all. This exercise creates a User Pool, registers one test user, gets a real authentication token, and proves the API now rejects requests without one.

## Time and Cost

- **Time:** ~25 min
- **Cost:** ~$0.00 (Cognito's free tier covers a handful of users easily)
- This cost estimate is only accurate if you complete the Cleanup section below immediately after
  finishing - don't leave AWS resources running "to come back to later."

## Prerequisites

- Exercise 16 complete - you need a working API Gateway HTTP API to attach the authorizer to

## Steps

## Step 1: Create a Cognito User Pool

```
POOL_ID=$(aws cognito-idp create-user-pool --pool-name pool-$PARTICIPANT --region us-east-1 --query UserPool.Id --output text)
echo $POOL_ID
```

A User Pool is Cognito's term for a directory of user accounts - think of it as a managed, purpose-built database of users with built-in sign-up, sign-in, and password reset flows, so you never have to build that yourself.

## Step 2: Create an App Client (this is what your API/app uses to talk to the pool)

```
CLIENT_ID=$(aws cognito-idp create-user-pool-client --user-pool-id $POOL_ID --client-name app-client-$PARTICIPANT --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH --region us-east-1 --query UserPoolClient.ClientId --output text)
echo $CLIENT_ID
```

`ALLOW_USER_PASSWORD_AUTH` is what lets you authenticate directly with a username/password from the CLI in the next step - production apps typically use a more secure flow (hosted UI, OAuth), but this is the simplest way to prove the mechanism works for this exercise.

## Step 3: Create a test user with a permanent password

```
aws cognito-idp admin-create-user --user-pool-id $POOL_ID --username testuser --temporary-password 'TempPass123!' --message-action SUPPRESS --region us-east-1
aws cognito-idp admin-set-user-password --user-pool-id $POOL_ID --username testuser --password 'TempPass123!' --permanent --region us-east-1
```

`--message-action SUPPRESS` skips sending a real welcome email (no email service is configured for this test pool). The second command sets the password as PERMANENT rather than temporary, skipping the forced-reset flow entirely - appropriate for a throwaway test user, not for real accounts.

## Step 4: Authenticate as that user and get a real token

```
TOKEN=$(aws cognito-idp initiate-auth --auth-flow USER_PASSWORD_AUTH --client-id $CLIENT_ID --auth-parameters USERNAME=testuser,PASSWORD='TempPass123!' --region us-east-1 --query 'AuthenticationResult.IdToken' --output text)
echo $TOKEN | cut -c1-50
```

This ID token is a real, signed JWT proving Cognito authenticated this exact user - it's what you'll attach to your API request next. Only printing the first 50 characters since the full token is long and not meaningful to read.

## Step 5: Attach a Cognito authorizer to your Exercise 16 API

```
AUTHORIZER_ID=$(aws apigatewayv2 create-authorizer --api-id $API_ID --authorizer-type JWT --identity-source '$request.header.Authorization' --name cognito-auth-$PARTICIPANT --jwt-configuration Audience=$CLIENT_ID,Issuer=https://cognito-idp.us-east-1.amazonaws.com/$POOL_ID --region us-east-1 --query AuthorizerId --output text)
aws apigatewayv2 get-routes --api-id $API_ID --region us-east-1 --query 'Items[0].RouteId' --output text
```

You'll need the RouteId from this command's output for the next step.

## Step 6: Require the authorizer on your route (paste the RouteId from the previous step)

```
aws apigatewayv2 update-route --api-id $API_ID --route-id <paste-route-id-here> --authorization-type JWT --authorizer-id $AUTHORIZER_ID --region us-east-1
```

Before this command, your route had no authorization at all (exactly like Exercise 16). This one change is what actually flips the route from public to protected - everything before it in this exercise was just setting up the User Pool and authorizer objects, not yet attaching them to anything.

## Step 7: Prove it - call WITHOUT a token (should fail), then WITH one (should succeed)

```
curl -i $API_URL
curl -i -H "Authorization: Bearer $TOKEN" $API_URL
```

The first call should return 401 Unauthorized. The second, with the real Cognito token attached, should succeed exactly like Exercise 16 did before the authorizer existed.

## Verify It Worked

- The unauthenticated curl returns 401
- The authenticated curl (with the Bearer token) succeeds and returns your Lambda's response

## Common Mistakes

- **Confusing the User Pool ID with the App Client ID** - They're different things used in different places - the Issuer URL needs the Pool ID, the Audience needs the Client ID. Mixing them up produces a confusing 'invalid token' error rather than an obviously-wrong one.
- **Forgetting tokens expire** - Cognito ID tokens are short-lived (typically 1 hour) - if your curl test fails unexpectedly after a break, get a fresh token by re-running Step 4.
- **Not noticing the authorization type must be exactly 'JWT'** - API Gateway also supports Lambda-based custom authorizers with a different type string - using the wrong one silently fails to attach the authorizer you intended.

## Cleanup (do this now, not later)

```
aws apigatewayv2 delete-api --api-id $API_ID --region us-east-1
aws cognito-idp delete-user-pool --user-pool-id $POOL_ID --region us-east-1
```

Then complete the mandatory checklist: **`cleanup-checklists/17-cleanup-checklist.md`**.
