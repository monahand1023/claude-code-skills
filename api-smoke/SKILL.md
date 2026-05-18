---
name: api-smoke
description: Run smoke tests against deployed API endpoints. Reads test definitions from smoke-tests.yaml if present, otherwise guides the user through creating one. Produces a clean pass/fail report. Trigger on "smoke test the API", "are the endpoints up", "test after deploy", "verify deployment", or /api-smoke.
---

# API Smoke Test

Run a quick suite of smoke tests against deployed API endpoints. Works from a `smoke-tests.yaml` config file, or helps you create one if it does not exist yet.

---

## Step 1 — Find the test config

Look for a config file in the project:

```bash
find . -maxdepth 3 \( -name "smoke-tests.yaml" -o -name "smoke-tests.yml" -o -name "smoke-tests.json" \) 2>/dev/null
```

- If found: read it and proceed to Step 3.
- If not found: proceed to Step 2 to create one.

---

## Step 2 — Create smoke-tests.yaml (only if it does not exist)

Ask the user:
1. What is the base URL of the deployed API? (e.g. `https://api.example.com` or `https://abc123.execute-api.us-east-1.amazonaws.com/prod`)
2. Does the API require authentication? If so, what type — Bearer token, API key header, basic auth, or none?
3. What environment variable holds the token or key? (Never hardcode credentials)
4. What are the 3–5 most important endpoints to verify after a deploy? For each: HTTP method, path, expected status code, and optionally a string that should appear in the response body.

Then create the file:

```yaml
# smoke-tests.yaml
# Run with: /api-smoke
base_url: https://api.example.com
auth:
  type: bearer          # options: bearer, api_key, basic, none
  env_var: API_TOKEN    # token is read from this environment variable at runtime
  header: Authorization # for api_key type, set this to the header name (e.g. X-Api-Key)

endpoints:
  - name: health check
    method: GET
    path: /health
    expect_status: 200

  - name: unauthenticated rejection
    method: GET
    path: /api/v1/protected
    auth: none          # override: send this request with no auth
    expect_status: 401

  - name: list items
    method: GET
    path: /api/v1/items
    expect_status: 200
    expect_body_contains: '"items"'

  - name: create item
    method: POST
    path: /api/v1/items
    body: '{"name": "smoke-test-item-delete-me"}'
    expect_status: 201
    expect_body_contains: '"id"'
```

Place it in the project root (or a `tests/` subdirectory). Then proceed to Step 3.

---

## Step 3 — Read the config

Parse the YAML to extract:
- `base_url`
- `auth.type`, `auth.env_var`, `auth.header` (if present)
- The list of endpoint test definitions

Resolve the auth token from the environment:

```bash
echo "${API_TOKEN:-(not set)}"
```

If the token variable is not set and auth type is not `none`, warn the user and ask how to proceed — do not run authenticated tests without a token.

---

## Step 4 — Run each test

For each endpoint definition, run:

```bash
STATUS=$(curl -s -o /tmp/smoke_body -w "%{http_code}" \
  -X GET \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  --max-time 10 \
  "https://api.example.com/health")

BODY=$(cat /tmp/smoke_body)

if [ "$STATUS" = "200" ]; then
  echo "PASS  health check ($STATUS)"
else
  echo "FAIL  health check — expected 200, got $STATUS"
  echo "      Response: $BODY"
fi
```

Adapt the command for each test:
- Omit `Authorization` header for `auth: none` endpoints or when `auth.type` is `none`
- Include `-d "$BODY"` for POST/PUT/PATCH requests
- For `api_key` auth type, use the configured header name instead of `Authorization`
- For `basic` auth type, use `-u "$USERNAME:$PASSWORD"`

If `expect_body_contains` is set, also check:

```bash
if echo "$BODY" | grep -q '"items"'; then
  echo "      Body check: PASS (contains expected string)"
else
  echo "      Body check: FAIL (expected string not found)"
  echo "      Response: $BODY"
fi
```

---

## Step 5 — Print the final report

After all tests complete, print a summary:

```
Smoke Test Results — https://api.example.com
─────────────────────────────────────────────
PASS  health check                  (200)
PASS  unauthenticated rejection     (401)
PASS  list items                    (200)  body contains "items"
FAIL  create item                   — expected 201, got 500

─────────────────────────────────────────────
3 passed, 1 failed

Failed test details:
  create item — Response body:
  {"error": "Internal Server Error", "requestId": "abc-123"}
```

- If all tests pass: state clearly that the deployment looks healthy.
- If any test fails: show the full response body for each failure, and suggest next steps (check CloudWatch logs, verify the endpoint exists, check auth config).
- Do not truncate response bodies for failed tests — full output helps diagnose the problem.

---

## Notes

- Never hardcode credentials in `smoke-tests.yaml`. Always use environment variable references.
- Add `smoke-tests.yaml` to `.gitignore` only if it contains environment-specific URLs. If the base URL is a placeholder, it is safe to commit.
- Smoke tests should be fast (all complete under 30 seconds) and not create lasting side-effects. If a `create` test creates a resource, note that it should be cleaned up — or add a corresponding delete test.
- The `--max-time 10` flag prevents a hung endpoint from blocking the entire suite indefinitely.
