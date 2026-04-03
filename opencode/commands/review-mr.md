---
description: Perform a thorough read-only review of a GitLab MR and print findings.
agent: build
---

# Review MR

Perform a thorough code review of GitLab Merge Request $ARGUMENTS and print the review to the console. No changes are made anywhere — no comments are posted, no files are modified.

If $ARGUMENTS is empty, use the MR for the current branch: `glab mr view`.

## Step 1: Gather context

```bash
glab mr view $ARGUMENTS
glab mr diff $ARGUMENTS
glab mr note list $ARGUMENTS
```

If the description contains a Notion task URL or TASK-ID, fetch the task to understand intent:
```bash
notion task view <id>
```

Read all review comments from `glab mr note list`. Note which threads are open vs resolved and what concerns were already raised — avoid duplicating feedback that was already given.

## Step 2: Read changed files in full

For every file touched by the MR, read the full file from the codebase — not just the diff lines — to understand the surrounding context, existing patterns, and how the change fits in.

Spawn an Explore agent if the MR spans many modules or requires deep traversal.

## Step 3: Review against these criteria

Check each change against the following, in order of importance:

**Correctness**
- Does the implementation match the task requirements?
- Are there logic errors, off-by-one issues, incorrect conditions?
- Are all code paths and error cases handled?

**Project conventions** (from CLAUDE.md and codebase patterns)
- No `anyhow`, no `HttpServerError` in new code — typed errors only.
- No legacy HTTP request structs (`HttpRequestMsg`, etc.).
- Bus requests: always `let (req, responder) = req.into_parts();`, no unanswered requests silently ignored.
- Route return types: always `Result<T, E>` with project error types; no `HttpResponse` as `T`; no `Result<Json<()>, E>`.
- Newtype wrappers: inner type private, `#[derive(AsMut, AsRef, Deref, DerefMut, From, Into)]`.
- No `Timestamp` from `co-time` — prefer `DateTime<Utc>` / `DateTime<FixedOffset>`.
- Dependency versions: left-most non-zero rule.
- Doc comments: `///` not `//`.
- Tuples/ambiguous values must have comments describing fields.
- Migration: both `up.sql` and `down.sql` must be filled.

**Security**
- Path traversal, injection, unvalidated input from outside the trust boundary.
- Auth/permission checks present where required.

**Error handling quality**
- Errors logged with sufficient context (plate, object_id, camera_id, UUID, etc.).
- No unanswered Bus requests.
- Appropriate HTTP status codes.

**Code quality**
- Unnecessary complexity or premature abstraction?
- Dead code, unused imports, leftover debug output?
- Obvious performance issues?

## Step 4: Output the review

Print the full review to console in this structure:

---

### Summary
One paragraph: what the MR does, overall impression (approve / needs changes / blocking issues).

### Issues

List each issue with:
- **Severity**: `blocking` | `major` | `minor` | `nit`
- **File**: `path/to/file.rs:line`
- **Description**: what's wrong and why
- **Suggestion**: concrete fix or alternative (code snippet if helpful)

If there are no issues: explicitly state "No issues found."

### Positive observations
Briefly note anything done especially well.

---

Do not post any comments to GitLab. Do not modify any files. Do not create branches. Do not commit anything.
