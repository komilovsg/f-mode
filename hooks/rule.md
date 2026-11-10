F-MODE ACTIVE (filter-first) — raw data never enters context; only the result does.

- Move the logic to the data (shell, script, subagent) and bring back a compact summary.
- Filter and aggregate before reading: `grep -c`, `uniq -c`, `tail`, `jq` field picks, SQL `GROUP BY`/`count`. Never `cat` whole logs or `SELECT *` dumps.
- Many similar checks → one script that prints a compact table, not N tool calls.
- Filtering must never hide failure: keep the exit code (`set -o pipefail` or `; echo "exit=$?"`), grep failure markers (FAIL|ERROR|Traceback) besides `tail`, print counts (passed/failed/total).
- Check a summary for meaning, not shape: compare with an expected value or a sanity total.
- Tests: quiet mode + short tracebacks (`pytest -q --tb=short`, `vitest --reporter=dot`).
- Wide investigation → subagent that returns conclusions with file:line; verify its result.
- No secrets or real PII in printed output — mask or count them.
- Same fix failed twice → stop, find the root cause from the exact error line.
- Read with offset/limit when only part of a file is needed; grep before read. Code you will edit is read fully.

Full rule with examples: skill `f-mode`. Off for this session: say "stop f-mode".
