---
name: f-mode
description: Token-economy working mode for coding agents — raw data never enters the context, only the result does. Use when running tests, reading logs, querying databases, inspecting large files or JSON, checking many similar things, doing browser checks, or when the user says "f-mode", "filter first", "save tokens", "context is filling up", "too much output". Covers filtering without hiding failures, compact summaries, verification of summaries, and when to delegate to a subagent.
---

# F-mode — filter-first mode

**Principle: raw data never enters the context; only the result does.** Move the logic to the data (a shell pipeline, a short script, a subagent) and bring back a compact summary. Every token of a log dump you read is a token you can't spend on reasoning — and long noisy context makes reasoning worse, not just more expensive.

## 1. Filter and aggregate before reading
| Instead of | Do |
|---|---|
| `cat app.log` | `grep -c ERROR app.log; grep -E 'ERROR|Traceback' app.log \| tail -20` |
| reading a 5 MB JSON | `jq '.items \| length'`, `jq -r '.items[] \| [.id,.status] \| @tsv' \| sort \| uniq -c` |
| `SELECT * FROM orders` | `SELECT status, count(*) FROM orders GROUP BY 1` |
| `git log` | `git log --oneline -20`, `git log --format=%s origin/main..HEAD` |
| scrolling a file | `grep -n 'def handle' file.py`, then read with offset/limit |

## 2. Filtering must never hide failure
A truncated view that looks green is worse than no view.
- Keep the exit code: `set -o pipefail`, or `cmd > out.txt 2>&1; echo "exit=$?"; tail -20 out.txt`. Piping straight into `tail` loses the command's exit code.
- Grep failure markers in addition to `tail`: `grep -cE 'FAIL|ERROR|Traceback|panic' out.txt`.
- Print counts with denominators: `passed 41/42`, never just "OK".

```bash
pytest -q --tb=short > /tmp/t.out 2>&1; echo "exit=$?"
grep -cE 'FAILED|ERROR' /tmp/t.out; tail -15 /tmp/t.out
```

## 3. Many similar checks → one script, one table
Ten endpoints, twenty files, thirty rows to validate: write one script that loops and prints a compact table (`name | status | detail`) plus totals. Not ten tool calls with ten full outputs.

```bash
for u in / /api/health /api/items; do
  printf "%-14s %s\n" "$u" "$(curl -s -o /dev/null -w '%{http_code}' "localhost:8000$u")"
done
```

## 4. Check the summary for meaning, not shape
Valid-looking output can still be wrong. Compare against an expected value or a sanity total: counts add up to the input size, a known row is present, the number of files matches `ls | wc -l`. If a summary can't be cross-checked, say so.

## 5. Tests
Quiet mode, short tracebacks, tail of the failure section:
- `pytest -q --tb=short`
- `vitest run --reporter=dot`, `jest --silent`
- `go test ./... 2>&1 | grep -E '^(FAIL|ok|---)'`
- `cargo test -q`

## 6. Browser checks
Evaluate a script that returns compact JSON (`{title, errors: n, buttons: n, overflowX: bool}`) instead of full DOM snapshots. Take a snapshot only when the DOM structure itself is the question. Screenshots for visual checks, not for reading text.

## 7. Wide investigation → subagent
Big searches, log forensics, database dumps: delegate to a subagent that returns conclusions with `file:line` references. Its result is still a claim — verify the key points before acting on them.

## 8. Secrets and personal data
Never print secrets or real PII in summaries or diagnostics. Mask (`postgres://***`) or count them (`emails found: 12`). Mask connection strings and keys inside caught exception messages too.

## 9. Reading files
- `grep -n` first, then read only the relevant range with offset/limit.
- Source code you are about to edit is read fully — partial reads cause broken edits.
- Don't re-read a file right after editing it to "verify"; the edit tool already errors on failure.

## 10. Stop rule
The same fix failed twice → stop. Find the root cause from the exact error line, or ask. No third blind retry.

## 11. Context hygiene
- Clear context between unrelated tasks; compact long sessions.
- Disable plugins, skills and MCP servers you don't use in this project: each one costs prompt tokens in every request.
- Judge savings by real usage numbers (e.g. `/cost`), not by claimed percentages.
