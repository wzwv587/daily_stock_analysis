# Daily 22:30 Scheduled Analysis Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Configure GitHub Actions to run market review and stock-list analysis sequentially every day at 22:30 Beijing time, with independent PushPlus pushes and failure isolation.

**Architecture:** Keep the existing `.github/workflows/00-daily-analysis.yml` and single `analyze` job. Add a schedule-only Bash branch that runs `--market-review --force-run` and `--no-market-review --force-run` sequentially, captures both exit codes, and fails only after both commands have been attempted; preserve the existing manual-dispatch branch.

**Tech Stack:** GitHub Actions YAML, Bash, Python 3.11 CLI, PushPlus notification integration

## Global Constraints

- Cron must be `30 14 * * *`, which is daily at 22:30 Asia/Shanghai.
- Scheduled runs must execute market review first and stock-list analysis second.
- Both scheduled commands must include `--force-run`, including weekends and non-trading days.
- Failure of the first command must not prevent the second command from running.
- The job must fail after both attempts if either command fails.
- Default timeout must be 60 minutes while `ANALYSIS_TIMEOUT_MINUTES` remains an override.
- Existing `full`, `market-only`, `stocks-only`, and manual `force_run` behavior must remain unchanged.
- Do not change GLM settings, API URLs, keys, PushPlus token, stock list, report generation, or PushPlus batching.
- Commit messages must be English and must not include `Co-Authored-By`.

---

## File Structure

- Modify `.github/workflows/00-daily-analysis.yml`: schedule, timeout, and schedule-only execution branch.
- Modify `docs/CHANGELOG.md`: one flat `[Unreleased]` entry describing the scheduling improvement.
- Reference `docs/superpowers/specs/2026-07-23-daily-scheduled-analysis-design.md`: approved design; no further content changes required.

### Task 1: Implement the daily sequential schedule

**Files:**
- Modify: `.github/workflows/00-daily-analysis.yml:4-6`
- Modify: `.github/workflows/00-daily-analysis.yml:32-35`
- Modify: `.github/workflows/00-daily-analysis.yml:503-521`

**Interfaces:**
- Consumes: GitHub event name `${{ github.event_name }}`, existing workflow-dispatch inputs, existing environment variables and secrets.
- Produces: one schedule-only branch that attempts both CLI commands and returns a combined exit status.

- [ ] **Step 1: Run the workflow-contract check before editing and verify it fails**

```bash
python - <<'PY'
from pathlib import Path
text = Path('.github/workflows/00-daily-analysis.yml').read_text(encoding='utf-8')
assert "cron: '30 14 * * *'" in text, 'daily 22:30 cron missing'
assert "vars.ANALYSIS_TIMEOUT_MINUTES || '60'" in text, '60-minute timeout missing'
assert "github.event_name }}' = 'schedule'" in text, 'schedule branch missing'
assert 'python main.py --market-review --force-run' in text
assert 'python main.py --no-market-review --force-run' in text
assert 'market_status=0' in text and 'stocks_status=0' in text
print('workflow contract OK')
PY
```

Expected: FAIL with `AssertionError: daily 22:30 cron missing` against the current workflow.

- [ ] **Step 2: Replace the schedule and timeout values**

Use this schedule block:

```yaml
on:
  # 定时触发 - 每天北京时间 22:30 (UTC 14:30)
  schedule:
    - cron: '30 14 * * *'     # 每天 UTC 14:30 = 北京时间 22:30
```

Use this timeout line:

```yaml
timeout-minutes: ${{ fromJSON(vars.ANALYSIS_TIMEOUT_MINUTES || '60') }}
```

- [ ] **Step 3: Replace only the execution branch after `# 执行分析`**

Keep the existing `MODE` and `FORCE_RUN_ARG` construction. Replace the current final mode conditional with:

```bash
# 定时任务：大盘与自选股分开分析、分开推送；任一失败不阻断另一项
if [ '${{ github.event_name }}' = 'schedule' ]; then
  echo '🕥 定时任务：先执行大盘分析，再执行股票列表分析'

  market_status=0
  python main.py --market-review --force-run || market_status=$?

  stocks_status=0
  python main.py --no-market-review --force-run || stocks_status=$?

  if [ "$market_status" -ne 0 ] || [ "$stocks_status" -ne 0 ]; then
    echo "❌ 定时分析存在失败：market=$market_status, stocks=$stocks_status"
    exit 1
  fi
elif [ "$MODE" = "market-only" ]; then
  python main.py --market-review $FORCE_RUN_ARG
elif [ "$MODE" = "stocks-only" ]; then
  python main.py --no-market-review $FORCE_RUN_ARG
else
  python main.py $FORCE_RUN_ARG
fi
```

The `cmd || status=$?` form is required because GitHub Actions invokes Bash with fail-fast behavior; the `||` guard lets the second command run.

- [ ] **Step 4: Run deterministic syntax and contract checks**

```bash
ruby -e "require 'yaml'; YAML.load_file('.github/workflows/00-daily-analysis.yml'); puts 'YAML OK'"
python - <<'PY'
from pathlib import Path
text = Path('.github/workflows/00-daily-analysis.yml').read_text(encoding='utf-8')
assert "cron: '30 14 * * *'" in text
assert "cron: '0 10 * * 1-5'" not in text
assert "vars.ANALYSIS_TIMEOUT_MINUTES || '60'" in text
assert text.count('python main.py --market-review --force-run') == 1
assert text.count('python main.py --no-market-review --force-run') == 1
assert 'market_status=0' in text and 'stocks_status=0' in text
assert "elif [ \"$MODE\" = \"market-only\" ]" in text
assert "elif [ \"$MODE\" = \"stocks-only\" ]" in text
print('workflow contract OK')
PY
```

Expected: `YAML OK` followed by `workflow contract OK`.

- [ ] **Step 5: Commit the workflow change**

```bash
git add .github/workflows/00-daily-analysis.yml
git commit -m "ci: schedule separated daily analyses at 22:30"
```

### Task 2: Record the deployment behavior change

**Files:**
- Modify: `docs/CHANGELOG.md:10-24`

**Interfaces:**
- Consumes: the implemented workflow behavior from Task 1.
- Produces: one flat `[Unreleased]` changelog entry; no headings or configuration changes.

- [ ] **Step 1: Verify the changelog entry is absent**

```bash
python - <<'PY'
from pathlib import Path
text = Path('docs/CHANGELOG.md').read_text(encoding='utf-8')
entry = '- [改进] GitHub Actions 每天北京时间 22:30 依次执行大盘分析和股票列表分析，并分别推送；两项失败隔离，默认超时提高到 60 分钟。'
assert entry not in text
print('entry absent as expected')
PY
```

Expected: `entry absent as expected`.

- [ ] **Step 2: Append this exact flat entry under `## [Unreleased]`**

```markdown
- [改进] GitHub Actions 每天北京时间 22:30 依次执行大盘分析和股票列表分析，并分别推送；两项失败隔离，默认超时提高到 60 分钟。
```

Place it before the first released-version heading and do not add a category subheading.

- [ ] **Step 3: Verify the entry appears exactly once and remains in `[Unreleased]`**

```bash
python - <<'PY'
from pathlib import Path
text = Path('docs/CHANGELOG.md').read_text(encoding='utf-8')
entry = '- [改进] GitHub Actions 每天北京时间 22:30 依次执行大盘分析和股票列表分析，并分别推送；两项失败隔离，默认超时提高到 60 分钟。'
unreleased = text.split('## [Unreleased]', 1)[1].split('## [', 1)[0]
assert text.count(entry) == 1
assert entry in unreleased
print('changelog contract OK')
PY
```

Expected: `changelog contract OK`.

- [ ] **Step 4: Commit the changelog change**

```bash
git add docs/CHANGELOG.md
git commit -m "docs: record daily analysis schedule"
```

### Task 3: Verify the deployed workflow

**Files:**
- Verify: `.github/workflows/00-daily-analysis.yml`
- Verify: `docs/CHANGELOG.md`

**Interfaces:**
- Consumes: the commits from Tasks 1 and 2 plus existing GitHub Actions secrets and variables.
- Produces: static validation evidence and two manual-run results without changing secrets or repository variables.

- [ ] **Step 1: Inspect the final diff for scope and secret safety**

```bash
git diff HEAD~2..HEAD -- .github/workflows/00-daily-analysis.yml docs/CHANGELOG.md
git diff HEAD~2..HEAD --check
```

Expected: only cron/comment, timeout, schedule execution branch, and one changelog line change; `git diff --check` exits 0. No secret values appear.

- [ ] **Step 2: Trigger a forced market-only smoke run**

```bash
gh workflow run 00-daily-analysis.yml -f mode=market-only -f force_run=true
gh run list --workflow=00-daily-analysis.yml --event workflow_dispatch --limit 1
```

Expected: a new run appears. Watch its displayed run ID with `gh run watch RUN_ID --exit-status`; it completes successfully and produces a market-review artifact and PushPlus message.

- [ ] **Step 3: Trigger a forced stocks-only smoke run**

```bash
gh workflow run 00-daily-analysis.yml -f mode=stocks-only -f force_run=true
gh run list --workflow=00-daily-analysis.yml --event workflow_dispatch --limit 1
```

Expected: a new run appears. Watch its displayed run ID with `gh run watch RUN_ID --exit-status`; it completes successfully and produces a stock-report artifact and PushPlus message or records any existing PushPlus batch failure in the logs.

- [ ] **Step 4: Verify the next scheduled trigger contract**

```bash
gh api repos/wzwv587/daily_stock_analysis/contents/.github/workflows/00-daily-analysis.yml -H 'Accept: application/vnd.github.raw+json' | python -c "import sys; t=sys.stdin.read(); assert \"cron: '30 14 * * *'\" in t; print('deployed cron OK')"
```

Expected: `deployed cron OK`. The next cron execution is daily at 14:30 UTC / 22:30 Asia/Shanghai; GitHub may start scheduled workflows a few minutes late under load.

- [ ] **Step 5: Prepare delivery and rollback notes**

Delivery must state the changed trigger, execution order, failure behavior, timeout, smoke-run URLs/results, known PushPlus batching limitation, and that secrets were untouched. Rollback is `git revert` of the workflow and changelog commits; do not reset or rewrite history.

## Self-Review

- Spec coverage: cron, daily/weekend execution, command order, force-run, independent failure capture, combined failure status, 60-minute default timeout, manual-mode compatibility, secret boundary, and PushPlus batching boundary are all mapped to Tasks 1-3.
- Repository-rule coverage: deployment change stays in `.github/workflows/`, uses English commit messages, adds the required flat `[Unreleased]` changelog entry, and avoids unrelated refactors.
- Placeholder scan: every implementation step contains exact commands, expected results, and concrete content. `RUN_ID` is an explicit CLI output consumed by the immediately following watch command.
- Consistency: both static checks and implementation use `30 14 * * *`, 60 minutes, `market_status`, `stocks_status`, and the same two CLI commands.
