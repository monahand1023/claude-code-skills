---
name: aws-cloudwatch
description: Query and analyze CloudWatch logs for Lambda functions and other AWS services. Use when the user says "check the logs", "CloudWatch", "why did this fail", "show me errors", "Lambda logs", or runs /aws-cloudwatch.
---

# AWS CloudWatch Logs Skill

Query CloudWatch logs with runtime discovery — no hardcoded log group names required.

## Step 1 — Identify the log group

**If the user did not specify a function name**, discover available Lambda log groups:

```bash
aws logs describe-log-groups \
  --log-group-name-prefix /aws/lambda \
  --query 'logGroups[*].logGroupName' \
  --output text \
  --no-cli-pager
```

Present the list and ask the user which function(s) to query. Set `LOG_GROUP=/aws/lambda/<chosen-name>`.

**If the user specified a function name**, use it directly:

```bash
LOG_GROUP=/aws/lambda/<function-name>
```

**For non-Lambda log groups** (ECS, API Gateway, etc.), discover with:

```bash
aws logs describe-log-groups \
  --query 'logGroups[*].{Name:logGroupName,RetentionDays:retentionInDays,StoredMB:storedBytes}' \
  --output table \
  --no-cli-pager
```

---

## Step 2 — Choose the time range

Default when not specified: **last 1 hour**.

**macOS time helpers:**
```bash
START=$(date -v-1H +%s)   # 1 hour ago
START=$(date -v-6H +%s)   # 6 hours ago
START=$(date -v-24H +%s)  # 24 hours ago
END=$(date +%s)
```

**Linux time helpers:**
```bash
START=$(date -d '1 hour ago' +%s)
START=$(date -d '6 hours ago' +%s)
START=$(date -d '24 hours ago' +%s)
END=$(date +%s)
```

---

## Search Patterns

### Pattern 1 — Errors in the last N hours

Default filter when not specified: `ERROR`, `error`, `panic`, `FATAL`.

```bash
aws logs filter-log-events \
  --log-group-name "$LOG_GROUP" \
  --start-time $(( START * 1000 )) \
  --end-time $(( END * 1000 )) \
  --filter-pattern "?ERROR ?error ?panic ?FATAL" \
  --query 'events[*].{Time:timestamp,Message:message}' \
  --output table \
  --no-cli-pager
```

### Pattern 2 — Search by keyword

Use for request IDs, customer IDs, email addresses, Stripe event IDs, trace IDs, or any correlation token.

```bash
KEYWORD="<value-to-search>"

aws logs filter-log-events \
  --log-group-name "$LOG_GROUP" \
  --start-time $(( START * 1000 )) \
  --end-time $(( END * 1000 )) \
  --filter-pattern "\"$KEYWORD\"" \
  --query 'events[*].{Time:timestamp,Message:message}' \
  --output table \
  --no-cli-pager
```

### Pattern 3 — Cold starts

Finds REPORT lines that include `Init Duration`, which only appear on cold starts.

```bash
aws logs filter-log-events \
  --log-group-name "$LOG_GROUP" \
  --start-time $(( START * 1000 )) \
  --end-time $(( END * 1000 )) \
  --filter-pattern '"Init Duration"' \
  --query 'events[*].message' \
  --output text \
  --no-cli-pager
```

### Pattern 4 — Panics, timeouts, and OOM

```bash
aws logs filter-log-events \
  --log-group-name "$LOG_GROUP" \
  --start-time $(( START * 1000 )) \
  --end-time $(( END * 1000 )) \
  --filter-pattern "?\"Task timed out\" ?\"Runtime.ExitError\" ?\"Runtime.OOMKilled\" ?\"memory\" ?panic" \
  --query 'events[*].{Time:timestamp,Message:message}' \
  --output table \
  --no-cli-pager
```

### Pattern 5 — Log Insights aggregates (error counts, P95 latency)

Log Insights queries are async. Start the query, then poll for results.

**Error count by 5-minute bucket:**

```bash
# Start query
QUERY_ID=$(aws logs start-query \
  --log-group-name "$LOG_GROUP" \
  --start-time $START \
  --end-time $END \
  --query-string 'fields @timestamp, @message
    | filter @message like /ERROR/
    | stats count() as errorCount by bin(5m)
    | sort @timestamp desc' \
  --query 'queryId' --output text --no-cli-pager)

echo "Query ID: $QUERY_ID"

# Poll until complete (usually 2–5 seconds)
aws logs get-query-results --query-id "$QUERY_ID" --no-cli-pager
```

**P50 / P95 / P99 duration from REPORT lines:**

```bash
QUERY_ID=$(aws logs start-query \
  --log-group-name "$LOG_GROUP" \
  --start-time $START \
  --end-time $END \
  --query-string 'fields @timestamp, @message
    | filter @message like /REPORT/
    | parse @message "Duration: * ms" as durationMs
    | stats
        count() as invocations,
        pct(durationMs, 50) as p50ms,
        pct(durationMs, 95) as p95ms,
        pct(durationMs, 99) as p99ms,
        max(durationMs) as maxMs' \
  --query 'queryId' --output text --no-cli-pager)

aws logs get-query-results --query-id "$QUERY_ID" --no-cli-pager
```

**Top 20 most recent errors with full message:**

```bash
QUERY_ID=$(aws logs start-query \
  --log-group-name "$LOG_GROUP" \
  --start-time $START \
  --end-time $END \
  --query-string 'fields @timestamp, @message, @logStream
    | filter @message like /ERROR/
    | sort @timestamp desc
    | limit 20' \
  --query 'queryId' --output text --no-cli-pager)

aws logs get-query-results --query-id "$QUERY_ID" --no-cli-pager
```

---

## Output Format

- Show log lines with human-readable timestamps where possible.
- After fetching results, summarize: **"X errors found in the last Y hours in function Z."**
- If errors are found, group them by type or message pattern and highlight the most frequent one.
- If no errors are found, confirm: **"No errors matching [pattern] in the last Y hours."**
- If the log group does not exist, say so clearly and suggest checking the function name or whether the function has ever been invoked.
