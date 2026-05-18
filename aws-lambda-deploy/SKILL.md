---
name: aws-lambda-deploy
description: Deploy Lambda functions by detecting the project's deploy mechanism automatically. Use when the user says "deploy the Lambda", "deploy my function", "push to AWS", "update Lambda code", or runs /aws-lambda-deploy.
---

# AWS Lambda Deploy Skill

Deploy Lambda functions with automatic detection of the project's deployment mechanism — no hardcoded assumptions about how the project is structured.

## Step 1 — Detect the deployment mechanism

Inspect the project root for these files in order:

```bash
ls Makefile template.yaml template.yml serverless.yml serverless.yaml cdk.json deploy.sh 2>/dev/null
```

Apply the first match:

| File found | Deploy command |
|---|---|
| `Makefile` with `deploy` or `deploy-lambda` target | `make deploy` or `make deploy-lambda` |
| `template.yaml` or `template.yml` | AWS SAM |
| `serverless.yml` or `serverless.yaml` | Serverless Framework |
| `cdk.json` | AWS CDK |
| `deploy.sh` | `bash deploy.sh` |
| None of the above | Manual zip + `aws lambda update-function-code` |

**Check for a deploy target in a Makefile:**

```bash
grep -E '^(deploy|deploy-lambda):' Makefile
```

If multiple targets exist, show them and ask the user which to run.

---

## Step 2 — Identify the function name (for fallback zip deploy or post-deploy verification)

If the function name is not already known, discover deployed Lambda functions:

```bash
aws lambda list-functions \
  --query 'Functions[*].{Name:FunctionName,Runtime:Runtime,LastModified:LastModified}' \
  --output table \
  --no-cli-pager
```

Ask the user which function to target if it is ambiguous.

---

## Step 3 — Run the deploy

### Option A: Makefile

```bash
make deploy
# or
make deploy-lambda
```

### Option B: AWS SAM

```bash
sam build && sam deploy
```

If `samconfig.toml` does not exist (first deploy), add `--guided`:

```bash
sam build && sam deploy --guided
```

### Option C: Serverless Framework

```bash
serverless deploy
```

Deploy a single function only (faster):

```bash
serverless deploy function --function <function-name>
```

### Option D: AWS CDK

```bash
cdk deploy
```

Deploy a specific stack:

```bash
cdk deploy <StackName>
```

### Option E: deploy.sh

```bash
bash deploy.sh
```

### Option F: Manual zip + update-function-code (fallback)

Build and zip based on runtime:

**Go (arm64):**
```bash
GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -o bootstrap . \
  && zip lambda.zip bootstrap
```

**Go (x86_64):**
```bash
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o bootstrap . \
  && zip lambda.zip bootstrap
```

**Python:**
```bash
pip install -r requirements.txt -t ./package \
  && cd package && zip -r ../lambda.zip . && cd .. \
  && zip lambda.zip *.py
```

**Node.js:**
```bash
npm ci --production \
  && zip -r lambda.zip . -x '*.test.js' '.git/*' 'node_modules/.bin/*'
```

Upload the zip:

```bash
FUNCTION_NAME=<function-name>

aws lambda update-function-code \
  --function-name "$FUNCTION_NAME" \
  --zip-file fileb://lambda.zip \
  --no-cli-pager
```

---

## Step 4 — Post-deploy verification

### Check function state

Wait for the update to finish propagating (state should be `Active`, last update status should be `Successful`):

```bash
aws lambda get-function \
  --function-name "$FUNCTION_NAME" \
  --query 'Configuration.{State:State,LastStatus:LastUpdateStatus,LastStatusReason:LastUpdateStatusReason}' \
  --output table \
  --no-cli-pager
```

If `LastStatus` is `Failed`, surface `LastStatusReason` to the user immediately.

### Invoke with a test payload (if the user has one)

```bash
aws lambda invoke \
  --function-name "$FUNCTION_NAME" \
  --payload '{"key":"value"}' \
  --cli-binary-format raw-in-base64-out \
  --log-type Tail \
  --query 'LogResult' \
  --output text \
  --no-cli-pager \
  response.json | base64 --decode

cat response.json
```

### Run a smoke test (if /api-smoke is available)

If the project has an `/api-smoke` skill available, offer to run it:

> "Deploy succeeded. Want me to run the API smoke test to confirm the endpoint is healthy?"

---

## Output Format

- Announce which deploy mechanism was detected and why.
- Stream or show the deploy command output so the user can see progress.
- After deploy, confirm the function state.
- If the deploy fails, show the full error and suggest next steps (check IAM permissions, inspect build output, check CloudWatch logs with `/aws-cloudwatch`).
