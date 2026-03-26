---
name: audit
description: Audit npm packages for vulnerabilities, apply fixes, and verify tests still pass
disable-model-invocation: true
allowed-tools: Bash(npm audit), Bash(npm audit fix), Bash(npm test), Bash(npx vitest *)
---

## When to use

Run `/audit` before production deploys, after installing new packages, or on a periodic security schedule.

## Usage

```
/audit
```

## Steps

### Step 1 — Scan for vulnerabilities

Run `npm audit` and report what was found (severity levels, affected packages, total count).

### Step 2 — Apply fixes

Run `npm audit fix` to apply safe, non-breaking updates.

If vulnerabilities remain that require breaking changes, report them clearly but do NOT run `npm audit fix --force` without explicit user approval.

### Step 3 — Verify nothing broke

Run `npx vitest run` to confirm the fixes didn't break any tests.

### Step 4 — Summarize

Output a markdown table:

| Package | Severity | Fixed? | Notes |
|---------|----------|--------|-------|
| example | high     | ✅     | updated 1.0 → 1.1 |

Then report:
- Total vulnerabilities found
- Total fixed
- Any remaining vulnerabilities and why they couldn't be auto-fixed (e.g. require `--force`)
- Test result: passed / failed
