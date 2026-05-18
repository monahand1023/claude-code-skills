---
name: aws-billing-trace
description: Triage a broken Stripe → Lambda → Cognito/DynamoDB subscription pipeline. Walks each layer systematically to find where a payment succeeded but access was not granted. Trigger on "user paid but can't access", "subscription not working", "Stripe webhook failed", "billing issue", or /aws-billing-trace.
---

# AWS Billing Pipeline Trace

Systematically trace a broken Stripe → Lambda → Cognito (or DynamoDB) subscription pipeline. A user paid, Stripe shows the subscription as active, but the user cannot access the product. The break is almost always in one of these layers.

This skill discovers AWS resources at runtime. No hardcoded account IDs, function names, or pool IDs.

---

## Step 1 — Collect identifiers

Ask the user for as many of these as they have:

- **User email** (the address they signed up with)
- **Stripe customer ID** — format `cus_xxxxxxxxxxxx`
- **Stripe subscription ID** — format `sub_xxxxxxxxxxxx`
- **Stripe event ID** (from a failed webhook attempt) — format `evt_xxxxxxxxxxxx`

You need at least one of these to proceed. Start with whatever is available and collect others as you find them in the pipeline.

---

## Step 2 — Verify Stripe subscription state

If the user has Stripe CLI access or dashboard access, ask them to confirm:

1. What is `subscription.status`? Expected: `active` or `trialing`. If it is `past_due`, `canceled`, or `incomplete`, this is a billing issue — not a pipeline bug. Stop here and explain that the payment method needs attention.
2. Did the most recent relevant webhook fire? (Check Stripe Dashboard → Developers → Webhooks → endpoint → recent deliveries)
3. What is the latest event type? Common relevant events: `customer.subscription.created`, `customer.subscription.updated`, `invoice.payment_succeeded`, `checkout.session.completed`.
4. Did any delivery show a non-2xx response? If so, note the HTTP status and response body — that is the Lambda's error.

**If Stripe subscription is not active — stop here.** The problem is on the Stripe/billing side, not in AWS. Do not attempt to manually grant access for a genuinely unpaid subscription.

---

## Step 3 — Discover the webhook Lambda

Find the Lambda function that handles Stripe webhooks:

```bash
aws lambda list-functions \
  --query 'Functions[*].{Name:FunctionName,Runtime:Runtime,LastModified:LastModified}' \
  --output table --no-cli-pager
```

Look for functions with names containing `webhook`, `stripe`, `billing`, or `subscription`. If ambiguous, ask the user which function handles Stripe webhooks.

Store the function name as `$WEBHOOK_FUNCTION`.

---

## Step 4 — Search CloudWatch logs for the customer or event

**macOS date syntax** — use `-v-24H` for 24 hours ago:

```bash
aws logs filter-log-events \
  --log-group-name "/aws/lambda/$WEBHOOK_FUNCTION" \
  --filter-pattern '"$CUSTOMER_ID"' \
  --start-time $(date -v-24H +%s000) \
  --no-cli-pager \
  | jq -r '.events[].message'
```

If the customer ID returns nothing, try the subscription ID or event ID:

```bash
aws logs filter-log-events \
  --log-group-name "/aws/lambda/$WEBHOOK_FUNCTION" \
  --filter-pattern '"$SUBSCRIPTION_ID"' \
  --start-time $(date -v-72H +%s000) \
  --no-cli-pager \
  | jq -r '.events[].message'
```

To broaden the search window to 7 days, replace `72H` with `168H`.

Look for:
- Any ERROR or exception lines
- A log entry showing the event was received
- A log entry showing the Cognito/DynamoDB update succeeded or failed
- Absence of any log entries — means the webhook never reached this Lambda

If you see no logs at all for a known-fired webhook, check whether the log group exists and whether the Lambda has the right CloudWatch permissions.

---

## Step 5 — Check for a stuck idempotency record in DynamoDB (if applicable)

If the architecture uses DynamoDB to track idempotency (common pattern: write a record keyed on `evt_xxxxxxxxxxxx` before processing, delete or update it after):

```bash
# First, discover tables
aws dynamodb list-tables --query 'TableNames' --output json --no-cli-pager
```

Look for a table named something like `idempotency`, `webhook-events`, `stripe-events`, or `payments`.

Then check for the event ID:

```bash
aws dynamodb get-item \
  --table-name "$IDEMPOTENCY_TABLE" \
  --key '{"pk": {"S": "$EVENT_ID"}}' \
  --no-cli-pager \
  | jq '.Item'
```

A record stuck in `status = "processing"` means the Lambda wrote the idempotency key, then crashed before completing. The event cannot be replayed until this record is removed.

**Fix:** Delete the stuck record, then replay the webhook from Stripe dashboard.

```bash
aws dynamodb delete-item \
  --table-name "$IDEMPOTENCY_TABLE" \
  --key '{"pk": {"S": "$EVENT_ID"}}' \
  --no-cli-pager
```

---

## Step 6 — Check Cognito for the user's current subscription attributes

Discover user pools if needed:

```bash
aws cognito-idp list-user-pools --max-results 60 \
  --query 'UserPools[*].{Name:Name,Id:Id}' \
  --output table --no-cli-pager
```

Then fetch the user:

```bash
aws cognito-idp admin-get-user \
  --user-pool-id "$POOL_ID" \
  --username "$EMAIL" \
  --no-cli-pager \
  | jq '.UserAttributes | map({(.Name): .Value}) | add'
```

Check:
- Is `custom:subscription_status` set to `active`? (or whatever the expected active value is)
- Is `custom:stripe_customer_id` set and matching the Stripe customer ID from Step 1?
- Is `custom:plan` or `custom:tier` set correctly?
- Is the account `Enabled: true` and `UserStatus: CONFIRMED`?

---

## Step 7 — Manual fix recipes

Apply the appropriate fix based on what you found:

### Stripe is active but Cognito attribute is wrong

```bash
aws cognito-idp admin-update-user-attributes \
  --user-pool-id "$POOL_ID" \
  --username "$EMAIL" \
  --user-attributes Name="custom:subscription_status",Value="active" \
  --no-cli-pager
```

Add other attributes as needed (`custom:plan`, `custom:stripe_customer_id`, etc.). After updating, re-fetch the user to confirm.

### Stuck idempotency record blocking replay

1. Delete the stuck DynamoDB record (Step 5).
2. Go to Stripe Dashboard → Developers → Webhooks → your endpoint → find the failed event → click "Resend".
3. Watch CloudWatch logs (Step 4) to confirm the Lambda processes it successfully this time.

### Webhook never fired

1. Check that the Stripe webhook endpoint URL is correct and points to the Lambda's API Gateway URL.
2. Verify the endpoint is subscribed to the right event types.
3. Use Stripe Dashboard → "Send test webhook" to trigger a test event and confirm end-to-end delivery.
4. Alternatively, use Stripe CLI: `stripe trigger invoice.payment_succeeded`

### Lambda threw an error on the event

1. Read the full error from CloudWatch (Step 4).
2. Fix the Lambda code bug.
3. Deploy the fix.
4. Replay the original webhook from Stripe Dashboard.

---

## Diagnosis decision tree

Work through this tree top-to-bottom and stop at the first matching branch:

```
Is the Stripe subscription status "active" or "trialing"?
  ├── No → billing issue (past_due, canceled, incomplete). Not a pipeline bug.
  │         Ask the user to fix their payment method in Stripe.
  └── Yes — continue
      │
      Did the webhook fire? (CloudWatch logs show it was received)
        ├── No → webhook never reached Lambda
        │         Check: endpoint URL, event subscriptions, Stripe delivery logs
        │         Fix: resend from Stripe dashboard, or check API Gateway config
        └── Yes — continue
            │
            Did the Lambda log an error?
              ├── Yes → Lambda bug or downstream service error
              │         Read the full error, fix the code, redeploy, replay webhook
              └── No — continue
                  │
                  Is there a stuck idempotency record? (status = "processing")
                    ├── Yes → Lambda crashed mid-execution
                    │         Delete the idempotency record, replay the webhook
                    └── No — continue
                        │
                        Is the Cognito attribute correct?
                          ├── Yes → client-side issue (stale token, cache)
                          │         Ask user to sign out and sign back in to get a fresh JWT
                          └── No → pipeline wrote nothing (silent failure or different Lambda)
                                    Update Cognito attribute manually (Step 7)
                                    Then investigate why the Lambda did not write it
```

---

## Notes

- Never manually grant subscription access when Stripe shows the subscription is not active. Always verify Stripe first.
- After any manual Cognito fix, ask the user to sign out and sign back in — their existing JWT will not reflect the updated attributes until they get a new token.
- If replaying a webhook, watch CloudWatch logs in near-real-time to confirm success before closing the investigation.
