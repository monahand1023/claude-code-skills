---
name: aws-dynamodb-debug
description: Inspect and repair DynamoDB records. Discovers tables at runtime. Handles stuck idempotency records, orphaned items, stale TTLs, and safe conditional updates. Trigger on "check the DynamoDB table", "inspect a record", "fix a stuck record", "DynamoDB", or /aws-dynamodb-debug.
---

# AWS DynamoDB Debug

Inspect and repair DynamoDB records without hardcoded table names. All resources are discovered at runtime.

---

## Step 0 — Discover tables (skip if user already named a table)

```bash
aws dynamodb list-tables --query 'TableNames' --output json --no-cli-pager
```

Present the list and ask the user which table to inspect (or infer from context). If there is only one table, proceed with it automatically.

---

## Step 1 — Describe the table

Before touching any data, understand the table structure:

```bash
aws dynamodb describe-table \
  --table-name "$TABLE_NAME" \
  --no-cli-pager \
  | jq '{
      KeySchema: .Table.KeySchema,
      GSIs: [.Table.GlobalSecondaryIndexes[]? | {Name: .IndexName, Keys: .KeySchema}],
      TTL: .Table.TimeToLiveDescription,
      ItemCount: .Table.ItemCount,
      TableSizeBytes: .Table.TableSizeBytes,
      BillingMode: .Table.BillingModeSummary.BillingMode
    }'
```

Summarize: partition key name and type, sort key (if any), active GSIs, TTL attribute name (if enabled), approximate item count, and billing mode.

---

## Step 2 — Retrieve a single item

Use this when the user provides an exact key.

```bash
aws dynamodb get-item \
  --table-name "$TABLE_NAME" \
  --key '{"pk": {"S": "$PK_VALUE"}}' \
  --no-cli-pager \
  | jq '.Item'
```

If the table has a sort key, include it:

```bash
aws dynamodb get-item \
  --table-name "$TABLE_NAME" \
  --key '{"pk": {"S": "$PK_VALUE"}, "sk": {"S": "$SK_VALUE"}}' \
  --no-cli-pager \
  | jq '.Item'
```

Adapt key attribute names to match what Step 1 revealed.

---

## Step 3 — Query by partition key (all records for an entity)

Retrieves every item sharing a partition key — useful for viewing an entire user's or entity's records:

```bash
aws dynamodb query \
  --table-name "$TABLE_NAME" \
  --key-condition-expression "pk = :pk_val" \
  --expression-attribute-values '{":pk_val": {"S": "$PK_VALUE"}}' \
  --no-cli-pager \
  | jq '.Items'
```

To query a GSI instead of the main table, add `--index-name "$GSI_NAME"` and use the GSI's key attributes.

---

## Step 4 — Scan with a filter (find by attribute value)

**Warn the user before running this:** a scan reads every item in the table and consumes read capacity proportional to table size. On large tables, prefer a GSI query.

```bash
aws dynamodb scan \
  --table-name "$TABLE_NAME" \
  --filter-expression "#attr = :val" \
  --expression-attribute-names '{"#attr": "$ATTRIBUTE_NAME"}' \
  --expression-attribute-values '{":val": {"S": "$VALUE"}}' \
  --no-cli-pager \
  | jq '{Count: .Count, Items: .Items}'
```

---

## Step 5 — Analyze retrieved items for common problems

After fetching an item, check for these stuck-record patterns and explain what you find:

**Idempotency record stuck in "processing"**
- Symptom: a record with `status = "processing"` and a `created_at` or `ttl` timestamp that is more than 10 minutes in the past.
- Cause: the Lambda that wrote the record crashed before completing; the idempotency key is now blocking retries.
- Fix: delete the record (Step 7) so the operation can be retried, or manually advance the status (Step 6).

**Webhook delivery record stuck in "pending"**
- Symptom: a record with `status = "pending"` or `delivery_status = "pending"` and a timestamp hours old.
- Cause: the downstream service was unreachable, or the worker never picked up the job.
- Fix: reset status to allow reprocessing, or trigger a manual replay from the originating system.

**Expired TTL item still present**
- Symptom: a record with a `ttl` attribute whose Unix timestamp value is in the past.
- Cause: DynamoDB TTL deletion is eventually consistent and can lag up to 48 hours.
- Behavior: these items will not be returned by Query/GetItem in most cases, but can appear in Scans. Safe to delete manually if causing issues.

For every item retrieved, display attributes clearly and flag:
- Any `status` / `state` attribute that looks stuck
- Any TTL attribute value — convert it to a human-readable date and note whether it is in the past
- Any timestamp that is unexpectedly old (more than 10 minutes for processing states, more than 24 hours for pending states)

---

## Step 6 — Safe conditional update

Use a condition expression to prevent overwriting a value that has already changed:

```bash
aws dynamodb update-item \
  --table-name "$TABLE_NAME" \
  --key '{"pk": {"S": "$PK_VALUE"}}' \
  --update-expression "SET #status = :new_status" \
  --condition-expression "#status = :expected_status" \
  --expression-attribute-names '{"#status": "status"}' \
  --expression-attribute-values '{
    ":new_status": {"S": "resolved"},
    ":expected_status": {"S": "stuck"}
  }' \
  --no-cli-pager
```

If the condition fails, the update is rejected with `ConditionalCheckFailedException` — this is the safe behavior. Re-fetch the item to see its current state before trying again.

To update multiple attributes at once, extend `--update-expression`:

```
SET #status = :new_status, updated_at = :now
```

And add corresponding entries to `--expression-attribute-values`.

---

## Step 7 — Delete a stuck or orphaned record

Always confirm with the user before deleting. Show the full item content first (Step 2), then ask for explicit confirmation.

```bash
aws dynamodb delete-item \
  --table-name "$TABLE_NAME" \
  --key '{"pk": {"S": "$PK_VALUE"}}' \
  --no-cli-pager
```

To delete only if the item is still in the expected state (safer):

```bash
aws dynamodb delete-item \
  --table-name "$TABLE_NAME" \
  --key '{"pk": {"S": "$PK_VALUE"}}' \
  --condition-expression "#status = :expected" \
  --expression-attribute-names '{"#status": "status"}' \
  --expression-attribute-values '{":expected": {"S": "processing"}}' \
  --no-cli-pager
```

---

## Output format

- Pretty-print all items as JSON.
- For each attribute, briefly explain what it likely represents based on its name.
- Highlight any attribute that looks wrong (see Step 5 patterns).
- After any write operation, re-fetch the item and show its updated state.
