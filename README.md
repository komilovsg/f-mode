# F-mode — filter-first mode

**Raw data never enters the context — only the result does.**

A small working mode for Claude Code (and other coding agents). It makes the agent filter, aggregate and summarize *before* reading, so test runs, logs, JSON and database queries stop flooding the context window. And it makes sure filtering never hides a failure.

## Why

Agents burn most of their context on output nobody needed: full test logs, `SELECT *` dumps, 5 MB JSON files, repeated tool calls for similar checks. That costs money, and long noisy context makes reasoning worse.

The obvious fix (`| tail -20`) creates a new problem: the pipe swallows the exit code, and a truncated view can look green while tests fail. f-mode handles both.

## Before / after

```bash
# before: 3,000 lines of output in context, exit code lost
pytest | tail -20

# after: ~20 lines, failure can't hide
pytest -q --tb=short > /tmp/t.out 2>&1; echo "exit=$?"
grep -cE 'FAILED|ERROR' /tmp/t.out; tail -15 /tmp/t.out
```

```sql
-- before
SELECT * FROM orders;
-- after
SELECT status, count(*) FROM orders GROUP BY 1;
```

## What's inside

| Part | What it does |
|---|---|
| `hooks/rule.md` | A short rule (~15 lines) injected at the start of every session by a `SessionStart` hook — always on. |
| `skills/f-mode/SKILL.md` | The full rule with examples: tests, logs, JSON, SQL, browser checks, subagents, secrets, stop rule. Loaded on demand. |

The rules:
1. Filter and aggregate before reading (`grep -c`, `jq`, `GROUP BY`, `uniq -c`).
2. Filtering never hides failure: keep exit codes, grep failure markers, print counts with denominators.
3. Many similar checks → one script, one compact table.
4. Check a summary for meaning, not shape (sanity totals, known rows).
5. Quiet tests with short tracebacks.
6. Browser checks return compact JSON, not DOM dumps.
7. Wide investigation → subagent with `file:line` conclusions, then verify.
8. No secrets or PII in output — mask or count.
9. Grep before read; code you will edit is read fully.
10. Same fix failed twice → stop and find the root cause.

## Install

### Claude Code plugin (recommended)
```
/plugin marketplace add komilovsg/f-mode
/plugin install f-mode@f-mode
```
Restart the session. Disable any time with `/plugin disable f-mode`.

### Skill only (no always-on hook)
```bash
git clone https://github.com/komilovsg/f-mode
cp -r f-mode/skills/f-mode ~/.claude/skills/
```

### Other agents (Codex, Cursor, Gemini CLI, …)
`skills/f-mode/SKILL.md` follows the open Agent Skills format. Or paste `hooks/rule.md` into your `AGENTS.md` / rules file to make it always on.

## Cost

The always-on part is ~15 lines (~300 tokens) per session. It pays for itself the first time it keeps a test log out of the context.

## Requirements

`cat` in the hook shell (macOS, Linux, Git Bash on Windows).

## License

MIT

---

## По-русски

**Сырые данные не попадают в контекст, попадает только результат.**

Режим работы для Claude Code и других агентов. Агент сначала фильтрует и считает (`grep -c`, `jq`, `GROUP BY`) и только потом читает. Тесты, логи, JSON и SQL больше не забивают контекст.

При этом фильтр никогда не прячет ошибку: агент сохраняет код выхода, ищет FAIL/ERROR и выводит счётчики «прошло 41/42».

**Установка:**
```
/plugin marketplace add komilovsg/f-mode
/plugin install f-mode@f-mode
```

Короткое правило (~15 строк) включается в каждой сессии. Полная версия с примерами лежит в скилле `f-mode` и загружается по необходимости.
