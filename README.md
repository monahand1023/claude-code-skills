# Claude Code Skills

A collection of Claude Code skills for everyday development workflows and AWS production operations.

## What are Claude Code skills?

Claude Code skills are markdown files that extend what Claude knows about your workflow. Drop a `SKILL.md` into `~/.claude/skills/<name>/SKILL.md` and Claude will automatically use it when your message matches the skill's trigger phrases — or invoke any skill explicitly with `/skill-name`.

## Skills

### Drop-in — zero configuration required

| Skill | What it does | Trigger phrases |
|---|---|---|
| [`diff-review`](diff-review/SKILL.md) | Code review of git changes — uncommitted or since last push. Go, Python, TypeScript/JavaScript. | `"review my changes"`, `"any issues with this?"`, `/diff-review` |
| [`pr-prep`](pr-prep/SKILL.md) | Pre-push quality gates + code review + conventional commit message draft. Detects stack from CWD. | `"ready to push"`, `"am I good to commit?"`, `/pr-prep` |

### AWS Ops — works with any AWS project

These skills discover your resources at runtime (`aws logs describe-log-groups`, `aws cloudformation list-stacks`, etc.) instead of requiring a config file. Point your AWS CLI at the right account and region and they just work.

| Skill | What it does | Trigger phrases |
|---|---|---|
| [`aws-cloudwatch`](aws-cloudwatch/SKILL.md) | Search Lambda logs by keyword, time range, or error pattern. Log Insights for aggregates. | `"check the logs"`, `"why did this fail"`, `/aws-cloudwatch` |
| [`aws-lambda-deploy`](aws-lambda-deploy/SKILL.md) | Deploy Lambda functions. Detects SAM, CDK, Serverless Framework, Makefile, or falls back to zip-and-update. | `"deploy the Lambda"`, `"push to AWS"`, `/aws-lambda-deploy` |
| [`aws-cfn-deploy`](aws-cfn-deploy/SKILL.md) | CloudFormation stack management — deploy, changeset preview, event tail, rollback. | `"deploy the stack"`, `"preview CFN changes"`, `/aws-cfn-deploy` |
| [`aws-dynamodb-debug`](aws-dynamodb-debug/SKILL.md) | Inspect and repair DynamoDB records. Detects stuck idempotency, webhook, and TTL patterns. | `"check the DynamoDB table"`, `"fix a stuck record"`, `/aws-dynamodb-debug` |
| [`aws-cognito-admin`](aws-cognito-admin/SKILL.md) | User pool administration — look up users, reset passwords, fix subscription attributes. | `"look up a user"`, `"reset password"`, `/aws-cognito-admin` |
| [`aws-billing-trace`](aws-billing-trace/SKILL.md) | Triage a broken Stripe → Cognito subscription pipeline layer by layer. | `"user paid but can't access"`, `"webhook failed"`, `/aws-billing-trace` |
| [`api-smoke`](api-smoke/SKILL.md) | Run curl smoke tests against your deployed API. Reads `smoke-tests.yaml` or creates one. | `"smoke test the API"`, `"are the endpoints up"`, `/api-smoke` |
| [`scaffold-lambda-go`](scaffold-lambda-go/SKILL.md) | Scaffold a new Go Lambda handler or standalone project, matching your existing conventions. | `"new Lambda handler"`, `"scaffold a Lambda"`, `/scaffold-lambda-go` |

## Installation

```bash
# Clone directly into your skills directory
git clone https://github.com/monahand1023/claude-code-skills ~/.claude/skills

# Or add individual skills to an existing skills directory
cp -r diff-review ~/.claude/skills/
cp -r pr-prep ~/.claude/skills/
```

Claude Code reads every `SKILL.md` in `~/.claude/skills/` at the start of each session.

## Example: diff-review in action

```
you:  review my changes

claude: [reads git diff HEAD]

## Diff review
**Scope:** uncommitted changes
**Changed:** 3 files, +47 −12 lines

### Issues

**[MUST FIX]**
- `handler/checkout.go:84` — Nil dereference: `session.User` is used without
  a nil guard after `GetSession`, which returns nil on cache miss.

**[SHOULD FIX]**
- `handler/checkout.go:91` — Error from `json.Unmarshal` is discarded.
  Malformed input will silently produce a zero-value struct.

### Overall
One blocking issue. Fix the nil dereference, then this is ready to ship.
```

## Example: aws-cloudwatch with runtime discovery

```
you:  check the logs

claude: [runs aws logs describe-log-groups --log-group-name-prefix /aws/lambda]
Found 4 Lambda log groups:
  1. /aws/lambda/checkout-api
  2. /aws/lambda/product-service
  3. /aws/lambda/auth-service
  4. /aws/lambda/stripe-webhook

Which function? (or "all" to search errors across all of them in the last hour)

you:  checkout-api, last 30 minutes

claude: [runs filter-log-events with --filter-pattern ERROR/panic/FATAL]
Found 3 errors in /aws/lambda/checkout-api (last 30m):

[2026-05-18T14:23:11Z] ERROR checkout handler: nil dereference at checkout.go:84
[2026-05-18T14:23:44Z] ERROR checkout handler: nil dereference at checkout.go:84
[2026-05-18T14:31:02Z] ERROR stripe webhook: ConditionalCheckFailedException on event evt_abc123
```

## Building your own skills

The pattern is simple: a frontmatter `description` that Claude reads to decide when to activate, and step-by-step instructions Claude follows when it does.

```markdown
---
name: my-skill
description: What this skill does and the exact phrases that trigger it.
  Include phrases like "deploy to staging", "run smoke tests", or /my-skill.
---

# my-skill

Brief description of the skill.

## When this skill triggers
- "phrase one"
- "phrase two"
- User runs `/my-skill`

## Step 1 — Do something
...

## Step 2 — Do something else
...
```

The `description` field is the trigger. Make it specific — include the exact phrases you say when you need this workflow.

## License

MIT
