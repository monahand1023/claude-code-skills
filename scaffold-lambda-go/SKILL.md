---
name: scaffold-lambda-go
description: Scaffold Go Lambda functions. Two modes — add a new handler to an existing project (default), or scaffold a complete new standalone project (--project). Discovers existing conventions before writing any code. Trigger on "scaffold a Lambda", "new Lambda handler", "add a Go handler", "new Lambda function", or /scaffold-lambda-go.
---

# Scaffold Go Lambda Function

Add a new handler to an existing Go Lambda project, or scaffold a complete new project from scratch. Reads existing code before writing anything so the generated files match the project's conventions.

---

## Step 0 — Determine mode and collect inputs

**Mode: --handler (default)**
Add a single new handler to an existing project.

**Mode: --project**
Scaffold a complete standalone Go Lambda project.

Ask the user:
1. Which mode? (If they said "add a handler" or similar, default to `--handler`. If they said "new project", use `--project`.)
2. What is the handler name? (e.g. `CreateOrder`, `GetUser`, `ProcessWebhook`) — use PascalCase.
3. (--project only) What is the Go module path? (e.g. `github.com/yourorg/service-name`)

---

## Step 1 — Detect existing project context

```bash
find . -maxdepth 3 -name "go.mod" 2>/dev/null
```

If `go.mod` exists, read it to get the module path:

```bash
head -3 go.mod
```

Then look for existing handler patterns:

```bash
find . -maxdepth 4 \( -path "*/handler/*.go" -o -path "*/handlers/*.go" -o -path "*/cmd/lambda/*.go" \) \
  ! -name "*_test.go" 2>/dev/null | head -10
```

Read one existing handler file to understand the project's conventions — look for:
- How the handler function signature is declared (APIGatewayProxyRequest, APIGatewayV2HTTPRequest, custom event type, etc.)
- How request bodies are parsed (json.Unmarshal, a custom decoder, etc.)
- How errors are returned (nil error with status 400/500, or non-nil error, or a custom error wrapper)
- Any middleware or wrapper functions the handler is passed through
- The import alias style for `github.com/aws/aws-lambda-go/events`
- The package name used in the handler directory

Use these conventions in the generated code. Do not impose a different style if the project already has one.

---

## Step 2a — Generate handler file (--handler mode)

Determine the correct directory for the new handler based on what Step 1 found:
- If `internal/handler/` exists, place it there
- If `handler/` exists, place it there
- If `cmd/lambda/*/handler.go` is the pattern, ask the user for the subcommand name
- Otherwise, create `internal/handler/`

Generate `$HANDLER_DIR/$snake_name.go` (convert PascalCase to snake_case for the filename, e.g. `CreateOrder` → `create_order.go`):

```go
package handler

import (
	"context"
	"encoding/json"
	"fmt"

	"github.com/aws/aws-lambda-go/events"
)

// $NameRequest defines the expected request body for the $Name handler.
type $NameRequest struct {
	// TODO: define request fields
}

// $NameResponse defines the response body for the $Name handler.
type $NameResponse struct {
	Message string `json:"message"`
}

// Handle$Name processes an API Gateway request for the $Name operation.
func Handle$Name(ctx context.Context, req events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	var input $NameRequest
	if err := json.Unmarshal([]byte(req.Body), &input); err != nil {
		return events.APIGatewayProxyResponse{
			StatusCode: 400,
			Body:       `{"error":"invalid request body"}`,
		}, nil
	}

	// TODO: implement $Name logic

	resp := $NameResponse{Message: "ok"}
	body, err := json.Marshal(resp)
	if err != nil {
		return events.APIGatewayProxyResponse{StatusCode: 500}, fmt.Errorf("marshal response: %w", err)
	}
	return events.APIGatewayProxyResponse{
		StatusCode: 200,
		Headers:    map[string]string{"Content-Type": "application/json"},
		Body:       string(body),
	}, nil
}
```

Substitute `$Name` with the PascalCase handler name throughout. Substitute `$snake_name` with the snake_case equivalent.

If the existing project uses `APIGatewayV2HTTPRequest` instead of `APIGatewayProxyRequest`, use that type consistently.

---

## Step 2b — Generate test file (--handler mode)

Generate `$HANDLER_DIR/${snake_name}_test.go`:

```go
package handler_test

import (
	"context"
	"testing"

	"github.com/aws/aws-lambda-go/events"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
)

func TestHandle$Name(t *testing.T) {
	t.Run("returns 200 for valid input", func(t *testing.T) {
		req := events.APIGatewayProxyRequest{
			Body: `{}`,
		}
		resp, err := Handle$Name(context.Background(), req)
		require.NoError(t, err)
		assert.Equal(t, 200, resp.StatusCode)
	})

	t.Run("returns 400 for invalid JSON", func(t *testing.T) {
		req := events.APIGatewayProxyRequest{
			Body: `{invalid json}`,
		}
		resp, err := Handle$Name(context.Background(), req)
		require.NoError(t, err)
		assert.Equal(t, 400, resp.StatusCode)
	})
}
```

---

## Step 3 — Project scaffold (--project mode only)

If `--project` mode: create all of the following files in the current directory (which should be empty or a new directory).

### go.mod

```
module $MODULE_PATH

go 1.22

require (
	github.com/aws/aws-lambda-go v1.47.0
	github.com/stretchr/testify v1.9.0
)
```

Use the module path provided in Step 0. Run `go mod tidy` after creating all files to resolve exact versions.

### cmd/lambda/main.go

```go
package main

import (
	"github.com/aws/aws-lambda-go/lambda"
	"$MODULE_PATH/internal/handler"
)

func main() {
	lambda.Start(handler.Handle$Name)
}
```

### internal/handler/$snake_name.go

Use the handler template from Step 2a.

### internal/handler/${snake_name}_test.go

Use the test template from Step 2b.

### Makefile

```makefile
BINARY    := bootstrap
BUILD_DIR := dist

.PHONY: build test clean deploy

build:
	GOOS=linux GOARCH=arm64 go build -tags lambda.norpc -o $(BUILD_DIR)/$(BINARY) ./cmd/lambda/

test:
	go test ./... -v -race

clean:
	rm -rf $(BUILD_DIR)

deploy: build
	# Replace with your deployment command:
	# sam deploy --guided
	# aws lambda update-function-code --function-name $FUNCTION_NAME --zip-file fileb://$(BUILD_DIR)/$(BINARY).zip
	@echo "Add your deployment command here"
```

### template.yaml (SAM template stub)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Lambda function scaffolded by /scaffold-lambda-go

Globals:
  Function:
    Runtime: provided.al2023
    Architectures: [arm64]
    Timeout: 30
    MemorySize: 128

Resources:
  $NameFunction:
    Type: AWS::Serverless::Function
    Metadata:
      BuildMethod: go1.x
    Properties:
      CodeUri: ./
      Handler: bootstrap
      Events:
        Api:
          Type: Api
          Properties:
            Path: /TODO
            Method: GET
```

---

## Step 4 — Verify it compiles

After creating all files, run:

```bash
go build ./...
```

If this fails:
- Check that the module path in `go.mod` matches the import paths in the generated files.
- Run `go mod tidy` to fetch missing dependencies.
- Show the error and fix it before reporting completion.

If tests are present, also run:

```bash
go test ./... -v
```

---

## Step 5 — Report what was created

List every file created, with a one-line description of each. Note any TODOs the user needs to fill in (request fields, business logic, deployment command). If in `--handler` mode, remind the user to register the new handler in `main.go` or their router if that is not done automatically.
