---
name: diff-review
description: Code review of git changes — either uncommitted (diff HEAD) or commits since last push (diff origin..HEAD). No quality gates, no commit message drafting. Just a focused review of what changed. Trigger when user says "review my diff", "look at my changes", "what do you think of this", "review since my last push", or runs /diff-review.
---

# diff-review

Lightweight code review. Reads the diff and gives an honest assessment. No test running, no commit message drafting — just review the code.

## When this skill triggers

- "review my changes", "look at what I changed", "what do you think of this"
- "review since my last push", "review my commits"
- "any issues with these changes?", "does this look right?"
- User runs `/diff-review`
- User pastes a diff directly

Use `/pr-prep` instead when the user is about to commit/push and wants the full workflow (quality gates + commit message + review).

## Step 1 — Determine the diff scope

### Default: uncommitted changes

```bash
git diff HEAD --stat
git diff HEAD
```

If the working tree is clean (no staged or unstaged changes), fall through to the "since last push" scope automatically.

### Since last push (explicit or fallback)

```bash
# Commits not yet pushed
git log origin/$(git rev-parse --abbrev-ref HEAD)..HEAD --oneline 2>/dev/null

# Full diff of those commits
git diff origin/$(git rev-parse --abbrev-ref HEAD)..HEAD
```

If the user says "since my last push" or "my recent commits", always use this scope even if there are also uncommitted changes.

If both scopes are empty, tell the user: "Nothing to review — working tree is clean and no unpushed commits."

## Step 2 — Establish context

Before reviewing, read the files that provide project context. These shape what "correct" looks like:

- `CLAUDE.md` in the repo root (if present) — project brief, architecture notes, constraints
- `go.mod` — module name and Go version
- `pyproject.toml` or `setup.cfg` — package name, dependencies, Python version
- `package.json` — project name, scripts, key dependencies (framework, test runner)

Don't read everything — just enough to understand the shape of the codebase the diff touches.

## Step 3 — Review the diff

### What to look for by language

#### Go

- **Error handling**: errors silently discarded (`_ = err`, `err != nil` branch missing)
- **Nil dereference**: pointer dereference before nil check, returning nil interface without checking
- **Goroutine leaks**: goroutines launched without context cancellation or WaitGroup tracking
- **Context propagation**: `context.Background()` used inside a handler or function that already has a context
- **Mutex correctness**: `Lock()` without paired `Unlock()` (prefer `defer`), holding a lock across a channel send
- **Interface satisfaction**: concrete type assigned to interface without compile-time assertion
- **Resource cleanup**: `defer rows.Close()`, `defer resp.Body.Close()` missing
- **Test quality**: table-driven tests that don't name sub-tests, assertions that don't report the failing case

#### Python

- **Exception swallowing**: bare `except:` or `except Exception: pass`
- **Mutable default arguments**: `def fn(items=[])` — classic footgun
- **Type confusion**: comparing across incompatible types, implicit string/bytes mixing
- **Lambda cold-start state**: module-level variables that accumulate state across invocations (counters, caches without TTLs, open file handles)
- **External API calls**: missing retry logic, no timeout on `requests.get()` / `httpx` calls, credentials hardcoded or read from globals instead of environment
- **Import side effects**: code that runs at import time (DB connections, file I/O) that will re-run every Lambda cold start
- **f-string misuse**: f-strings used to construct SQL or shell commands (injection risk)
- **Missing `__all__`**: public API surface unclear in library modules

#### TypeScript / JavaScript

- **Type safety**: `any` casts that erase guarantees, missing type guards before narrowing (`typeof`, `instanceof`, discriminated unions), non-null assertions (`!`) on values that can genuinely be null
- **Async/await error handling**: `async` functions without `try/catch` where the error would be silently swallowed, unhandled promise rejections (`.then()` without `.catch()`), `await` inside a `forEach` (use `Promise.all` + `map` instead)
- **React (when `.tsx` files are in the diff)**:
  - Missing or incorrect dependency arrays in `useEffect` / `useCallback` / `useMemo`
  - Direct state mutation (`state.items.push(x)` instead of spreading)
  - Keys derived from array index when list order can change
  - Effects that start async work without cleanup (stale closure / memory leak on unmount)
- **Security**:
  - User input passed to `eval()`, `new Function()`, or `setTimeout(string)`
  - `innerHTML` or `dangerouslySetInnerHTML` set from unsanitized data
  - `postMessage` listeners that don't validate `event.origin`
- **Module hygiene**: side-effectful code at module top level, circular imports, barrel files that re-export everything (tree-shaking killer)

### Review output format

Structure the response as:

**Summary** — one sentence on what the change does and whether it looks correct overall.

**Issues** — bulleted list, each item:
- Severity tag: `[bug]`, `[security]`, `[perf]`, `[style]`, `[nit]`
- File and line reference (e.g. `auth/handler.go:42`)
- What's wrong and why it matters
- Suggested fix (inline code snippet when it's short)

If there are no issues, say so explicitly rather than inventing nits.

**Questions** — things that aren't clearly wrong but that you'd want the author to confirm (e.g., "Is the 30s timeout intentional here, or a placeholder?").

Keep the tone direct. Don't pad with praise.

## Flags

- `/diff-review --since-push` — force the "commits since last push" scope even if uncommitted changes exist
- `/diff-review --uncommitted` — force the "uncommitted changes" scope
- `/diff-review --file path/to/file.go` — review only a specific file's changes

## What this skill does NOT do

- Run tests or linters — that's `/pr-prep`
- Draft a commit message — that's `/pr-prep`
- Review code that hasn't changed — use `/code-review` for reviewing existing code
- Approve or merge anything
