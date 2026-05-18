---
name: aws-cfn-deploy
description: Manage CloudFormation stacks — deploy, preview changesets, check status, tail events, and roll back. Use when the user says "deploy the stack", "update CloudFormation", "preview CFN changes", "check stack status", "roll back the stack", or runs /aws-cfn-deploy.
---

# AWS CloudFormation Deploy Skill

Manage CloudFormation stacks with runtime discovery — no hardcoded stack or template names.

## Step 0 — Discover stacks (if the user did not specify one)

```bash
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE UPDATE_ROLLBACK_COMPLETE ROLLBACK_COMPLETE \
  --query 'StackSummaries[*].{Name:StackName,Status:StackStatus,Updated:LastUpdatedTime}' \
  --output table \
  --no-cli-pager
```

Present the table and ask which stack to operate on. Set `STACK_NAME=<chosen-name>`.

---

## Stack status reference

| Status | Meaning |
|---|---|
| `CREATE_COMPLETE` | Stack created successfully, never updated |
| `UPDATE_COMPLETE` | Last update succeeded |
| `UPDATE_ROLLBACK_COMPLETE` | Last update failed and rolled back cleanly |
| `UPDATE_IN_PROGRESS` | Update is running — wait before taking action |
| `ROLLBACK_IN_PROGRESS` | Update failed, rollback in progress — wait |
| `UPDATE_ROLLBACK_FAILED` | Rollback itself failed — manual intervention required |
| `CREATE_FAILED` | Initial creation failed |
| `ROLLBACK_COMPLETE` | Initial creation failed and rolled back |
| `DELETE_IN_PROGRESS` | Stack deletion in progress |

---

## Operation 1 — Check stack status

```bash
aws cloudformation describe-stacks \
  --stack-name "$STACK_NAME" \
  --query 'Stacks[0].{Status:StackStatus,Reason:StackStatusReason,Created:CreationTime,Updated:LastUpdatedTime}' \
  --output table \
  --no-cli-pager
```

If the status is `UPDATE_ROLLBACK_COMPLETE` or `UPDATE_ROLLBACK_FAILED`, show the reason and suggest tailing events for details.

---

## Operation 2 — Find the template

Before deploying, locate the template file. Check these paths in order:

```bash
ls template.yaml template.yml cloudformation.yaml cloudformation.yml \
   infrastructure/cloudformation.yaml infrastructure/cloudformation.yml \
   main.yaml main.yml cfn/template.yaml 2>/dev/null
```

Use the first match as `TEMPLATE_FILE`. If none are found, ask the user for the template path.

---

## Operation 3 — Deploy / update a stack

Validate the template first:

```bash
aws cloudformation validate-template \
  --template-body file://"$TEMPLATE_FILE" \
  --no-cli-pager
```

Deploy using `deploy` (creates stack if it does not exist, updates if it does):

```bash
aws cloudformation deploy \
  --stack-name "$STACK_NAME" \
  --template-file "$TEMPLATE_FILE" \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --no-cli-pager
```

Pass parameters if needed:

```bash
aws cloudformation deploy \
  --stack-name "$STACK_NAME" \
  --template-file "$TEMPLATE_FILE" \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --parameter-overrides Key1=Value1 Key2=Value2 \
  --no-cli-pager
```

---

## Operation 4 — Changeset preview (recommended before any production update)

### Create the changeset

```bash
CHANGESET_NAME="preview-$(date +%Y%m%d-%H%M%S)"

aws cloudformation create-change-set \
  --stack-name "$STACK_NAME" \
  --change-set-name "$CHANGESET_NAME" \
  --template-body file://"$TEMPLATE_FILE" \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND \
  --no-cli-pager
```

### Wait for the changeset to finish analyzing

```bash
aws cloudformation wait change-set-create-complete \
  --stack-name "$STACK_NAME" \
  --change-set-name "$CHANGESET_NAME" \
  --no-cli-pager
```

### Describe the changes

```bash
aws cloudformation describe-change-set \
  --stack-name "$STACK_NAME" \
  --change-set-name "$CHANGESET_NAME" \
  --query 'Changes[*].ResourceChange.{Action:Action,LogicalId:LogicalResourceId,Type:ResourceType,Replacement:Replacement,Scope:Scope}' \
  --output table \
  --no-cli-pager
```

### Safety rule — flag replacements before proceeding

Scan the output for `Replacement: True`. If any resource of the following types is being replaced, **stop and warn the user before proceeding**:

- DynamoDB tables (`AWS::DynamoDB::Table`)
- RDS instances or clusters (`AWS::RDS::*`)
- ElastiCache clusters (`AWS::ElastiCache::*`)
- CloudFront distributions (`AWS::CloudFront::Distribution`)
- S3 buckets (`AWS::S3::Bucket`)
- Cognito User Pools (`AWS::Cognito::UserPool`)

Example warning:

> **Warning: This changeset will REPLACE the following resources, causing data loss or downtime:**
> - `MyDynamoTable` (AWS::DynamoDB::Table) — replacement destroys all table data
>
> Do you want to continue? This cannot be undone.

Only execute the changeset after explicit user confirmation.

### Execute the changeset

```bash
aws cloudformation execute-change-set \
  --stack-name "$STACK_NAME" \
  --change-set-name "$CHANGESET_NAME" \
  --no-cli-pager
```

### Delete the changeset without executing (if user declines)

```bash
aws cloudformation delete-change-set \
  --stack-name "$STACK_NAME" \
  --change-set-name "$CHANGESET_NAME" \
  --no-cli-pager
```

---

## Operation 5 — Tail stack events

Shows the last 50 events, highlighting failures:

```bash
aws cloudformation describe-stack-events \
  --stack-name "$STACK_NAME" \
  --query 'StackEvents[:50].{Time:Timestamp,LogicalId:LogicalResourceId,Type:ResourceType,Status:ResourceStatus,Reason:ResourceStatusReason}' \
  --output table \
  --no-cli-pager
```

Filter for failures only:

```bash
aws cloudformation describe-stack-events \
  --stack-name "$STACK_NAME" \
  --query 'StackEvents[?contains(ResourceStatus, `FAILED`)].{Time:Timestamp,LogicalId:LogicalResourceId,Status:ResourceStatus,Reason:ResourceStatusReason}' \
  --output table \
  --no-cli-pager
```

---

## Operation 6 — Roll back a failed update

If the stack is in `UPDATE_ROLLBACK_FAILED`, use `continue-update-rollback`. First, identify any resources blocking the rollback:

```bash
aws cloudformation describe-stack-events \
  --stack-name "$STACK_NAME" \
  --query 'StackEvents[?contains(ResourceStatus, `FAILED`)].{LogicalId:LogicalResourceId,Reason:ResourceStatusReason}' \
  --output table \
  --no-cli-pager
```

Skip resources that cannot be rolled back automatically (e.g., manually modified resources):

```bash
aws cloudformation continue-update-rollback \
  --stack-name "$STACK_NAME" \
  --resources-to-skip LogicalResourceId1 LogicalResourceId2 \
  --no-cli-pager
```

Or rollback without skipping:

```bash
aws cloudformation continue-update-rollback \
  --stack-name "$STACK_NAME" \
  --no-cli-pager
```

---

## Operation 7 — Read stack outputs

```bash
aws cloudformation describe-stacks \
  --stack-name "$STACK_NAME" \
  --query 'Stacks[0].Outputs[*].{Key:OutputKey,Value:OutputValue,Description:Description}' \
  --output table \
  --no-cli-pager
```

---

## Nested stacks

Find nested stack ARNs from the parent stack's resources:

```bash
aws cloudformation list-stack-resources \
  --stack-name "$STACK_NAME" \
  --query 'StackResourceSummaries[?ResourceType==`AWS::CloudFormation::Stack`].{LogicalId:LogicalResourceId,PhysicalId:PhysicalResourceId,Status:ResourceStatus}' \
  --output table \
  --no-cli-pager
```

Drill into a nested stack's events using its physical resource ID (the nested stack ARN):

```bash
NESTED_STACK_ARN="<arn-from-above>"

aws cloudformation describe-stack-events \
  --stack-name "$NESTED_STACK_ARN" \
  --query 'StackEvents[?contains(ResourceStatus, `FAILED`)].{Time:Timestamp,LogicalId:LogicalResourceId,Status:ResourceStatus,Reason:ResourceStatusReason}' \
  --output table \
  --no-cli-pager
```

---

## Output Format

- Always show the stack name and current status before and after any operation.
- For changesets, present the changes as a clear table and explicitly call out any replacements.
- After a deploy completes, confirm the new status and show any stack outputs.
- If a deploy fails, tail the events automatically and surface the first `FAILED` event with its reason.
