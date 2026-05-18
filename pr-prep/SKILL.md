---
name: pr-prep
description: Full pre-push workflow — runs quality gates for the detected stack, drafts a conventional commit message, and reviews the diff. Trigger when user says "prep my PR", "ready to push", "get this ready to commit", "run pr-prep", or runs /pr-prep. Also trigger when user asks to "commit and push" and wants everything checked first.
---

# pr-prep

Full pre-push workflow. Detects the project stack, runs the appropriate quality gates, writes a conventional commit message, and reviews the diff. Use this when you're ready to commit or open a PR — not just for a quick look at the code (use `/diff-review` for that).

## When this skill triggers

- "prep my PR", "get this ready to push", "ready to commit"
- "run pr-prep", "run the quality gates"
- "commit and push" (when the user wants everything checked before it goes out)
- User runs `/pr-prep`

Use `/diff-review` instead when the user just wants a code review without running tests or drafting a commit message.

## Step 1 — Capture the diff

```bash
git diff HEAD --stat
git diff HEAD
```

If the working tree is clean, check for unpushed commits:

```bash
git log origin/$(git rev-parse --abbrev-ref HEAD)..HEAD --oneline 2>/dev/null
git diff origin/$(git rev-parse --abbrev-ref HEAD)..HEAD
```

If both are empty, tell the user there's nothing to prep and stop.

## Step 2 — Detect the project stack

Find the project root (the directory containing the relevant config file) and identify which stacks are in play:

| Indicator | Stack |
|---|---|
| `*.go` files changed, `go.mod` present | Go |
| `*.py` files changed, `pyproject.toml` or `pytest.ini` present | Python |
| `*.ts` / `*.tsx` files, `package.json` with `typescript` dependency | TypeScript |
| `*.js` files, `package.json` | JavaScript / Node.js |
| `*.rb` files, `Gemfile` present | Ruby |
| Mix of types | Multi-stack — run all relevant suites |

```bash
# Locate config files to find project roots
find . -maxdepth 3 -name "go.mod" -o -name "pyproject.toml" -o -name "pytest.ini" -o -name "package.json" | grep -v node_modules
```

Note the directory of each config file — that is the root to `cd` into when running that stack's commands.

## Step 3 — Review the diff

Before running gates, read the diff critically. Look for:

- **Correctness**: logic errors, off-by-one, wrong operator
- **Error handling**: errors discarded, panics without recovery, unhandled promise rejections
- **Security**: credentials or secrets in code, injection vectors (SQL, shell, HTML), missing auth checks
- **Performance**: N+1 queries, unbounded loops, missing indexes mentioned in schema changes
- **Test coverage**: changed logic with no corresponding test update

Flag any issues with `[bug]`, `[security]`, `[perf]`, or `[nit]` tags and file:line references.

## Step 4 — Run quality gates

Run gates for every detected stack. Stop and report on first gate failure — don't proceed to commit message drafting if gates are red.

### Go

```bash
# From the directory containing go.mod
cd <go-module-root>

# Format check
gofmt -l .
# If any files listed, run: gofmt -w .

# Vet
go vet ./...

# Lint (if golangci-lint is installed)
golangci-lint run ./...

# Tests
go test ./... -race -count=1
```

### Python

```bash
# From the directory containing pyproject.toml or pytest.ini
cd <python-project-root>

# Tests
pytest

# Linting (run whichever are configured)
ruff check .
ruff format --check .

# Type checking (if mypy is configured)
mypy .
```

Detect whether `ruff`, `mypy`, and `pytest` are available before running them. If a tool isn't installed or isn't configured (no `[tool.mypy]` / `[tool.ruff]` section in `pyproject.toml`), skip it silently.

### TypeScript / JavaScript

```bash
# From the directory containing package.json
cd <js-project-root>

# Determine package manager
# Use yarn if yarn.lock present, pnpm if pnpm-lock.yaml present, otherwise npm

# Type check (TypeScript only)
npx tsc --noEmit

# Tests
npm test    # or: yarn test / pnpm test

# Lint (if eslint is configured)
npx eslint .
```

### Ruby

```bash
# From the directory containing Gemfile
cd <ruby-project-root>

bundle exec rspec          # or: bundle exec rake test
bundle exec rubocop        # if .rubocop.yml is present
```

### Gate failure

If any gate fails, output:

```
Gate failed: <gate name>
<error output>

Fix the above before committing. Re-run /pr-prep when ready.
```

Do not draft a commit message or proceed further.

## Step 5 — Draft the commit message

Only reach this step if all gates pass.

### Format

```
<type>(<scope>): <description>

<body — optional>

<footer — optional>
```

### Types

| Type | When to use |
|---|---|
| `feat` | New user-visible feature |
| `fix` | Bug fix |
| `refactor` | Restructuring with no behavior change |
| `test` | Adding or fixing tests only |
| `perf` | Performance improvement |
| `security` | Security fix or hardening |
| `chore` | Dependency updates, config, tooling |
| `ci` | CI/CD pipeline changes |
| `docs` | Documentation only |

### Scope

Scope is the package or area changed — e.g., `auth`, `api`, `cli`, `db`, `handler`, `docs`. Use the most specific name that describes the changed area without being redundant with the type. Omit scope if the change is truly cross-cutting.

### Description rules

- Imperative mood: "add retry logic" not "added retry logic"
- No period at the end
- 72 characters or fewer on the subject line
- Lowercase after the colon

### Body (include when the subject line isn't enough)

- Explain *why*, not *what* — the diff already shows what changed
- Wrap at 72 characters
- Separate from subject with a blank line

### Breaking changes

Add `BREAKING CHANGE: <explanation>` in the footer, and append `!` after the type/scope: `feat(api)!: rename endpoint`.

### Examples

```
feat(auth): add token refresh on 401 response

Tokens were expiring mid-session with no recovery path. The client
now automatically retries the original request after a successful
refresh, so users aren't logged out unexpectedly.
```

```
fix(db): close rows after query in ListUsers

rows.Close() was missing, causing connection pool exhaustion under
load. Added defer immediately after the rows.Err() check.
```

```
chore: upgrade dependencies to latest patch versions
```

```
perf(api): replace sequential fetches with concurrent fan-out

List endpoint was making N serial calls to the upstream service.
Replaced with a bounded worker pool; p99 latency dropped from 4s to
~400ms at 50 concurrent users.
```

### Output

Present the proposed commit message in a fenced code block. Ask the user to confirm or suggest changes before running `git commit`.

## Step 6 — Confirm and commit

After the user confirms the message:

```bash
git add -p    # or specific files if the user specified them
git commit -m "<confirmed message>"
```

Do not `git push` unless the user explicitly asks.

## Flags

- `/pr-prep --no-commit` — run gates and draft message but stop before `git commit`
- `/pr-prep --skip-tests` — run format/lint gates only, skip the test suite
- `/pr-prep --since-push` — diff against `origin/<branch>` instead of `HEAD`

## What this skill does NOT do

- Push to remote (unless the user explicitly asks after committing)
- Open or update a pull request (use `gh pr create` for that)
- Review code that hasn't changed
- Bypass failing gates — fix them first
