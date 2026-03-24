# 文件鮮度保證機制（Doc Freshness SLA）

## 背景

claw-info 中的文件（`docs/`、`usecases/`）具有時效性。例如某功能在 v3.11 新增，可能在 v4.x 已被修改或移除。若文件長期無人維護，agent 讀取後可能產生錯誤行為。

## Frontmatter 標準欄位

每份文件須加入以下欄位：

```yaml
---
last_validated: YYYY-MM-DD
validated_by: <github-username>
freshness: ok   # ok | stale | unreviewed
---
```

### 初始狀態

新文件合併時應預設 `freshness: ok`（剛撰寫即為剛驗證），而非 `unreviewed`。

### 最小範本

```yaml
---
last_validated: 2026-03-24
validated_by: thepagent
freshness: ok
---
```

### 反例（錯誤寫法）

```yaml
# ❌ 缺少欄位
---
title: My Doc
---

# ❌ 日期格式錯誤
last_validated: 24/03/2026

# ❌ 新文件用 unreviewed
freshness: unreviewed
```

## Review 週期（依文件類型分級）

| 路徑 | 週期 |
|------|------|
| `usecases/` | 2 週 |
| `docs/` | 4 週 |
| 架構圖、穩定參考文件 | 8 週 |

## 自動化流程（GHA）

### Workflow 設計

```yaml
# .github/workflows/doc-freshness-check.yml
name: Doc Freshness Check
on:
  schedule:
    - cron: '0 2 * * 1'  # 每週一 UTC 02:00
  workflow_dispatch:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check stale docs and open issues
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          MAX_ISSUES: 50
          GRACE_DAYS: 7
        run: bash .github/scripts/check_freshness.sh
```

> 現行執行版見 [#349](https://github.com/thepagent/claw-info/pull/349)（MVP workflow，issue-based，不自動改檔）。

### 核心腳本（`.github/scripts/check_freshness.sh`）

```bash
#!/usr/bin/env bash
set -euo pipefail

TODAY=$(date +%s)

# 跨平台 date 解析（GNU / macOS BSD）
to_epoch() {
  if date -d "2000-01-01" +%s >/dev/null 2>&1; then
    date -d "$1" +%s  # GNU coreutils (Linux)
  else
    date -jf '%Y-%m-%d' "$1" +%s  # BSD date (macOS)
  fi
}

threshold_for() {
  case "$1" in
    usecases/*) echo 14 ;;
    docs/*)     echo 28 ;;
    *)          echo 56 ;;
  esac
}

# 使用 yq 解析 frontmatter（需安裝 yq v4+）
# 若無 yq，fallback 至 awk
parse_field() {
  local file="$1" field="$2"
  if command -v yq >/dev/null 2>&1; then
    yq e ".${field}" "$file" 2>/dev/null | grep -v '^null$' || true
  else
    awk -F': ' "/^${field}:/{print \$2; exit}" "$file" | tr -d '\r'
  fi
}

while IFS= read -r -d '' f; do
  last=$(parse_field "$f" last_validated)
  owner=$(parse_field "$f" validated_by)
  [ -z "$last" ] && continue

  last_epoch=$(to_epoch "$last") || continue
  age=$(( (TODAY - last_epoch) / 86400 ))
  threshold=$(threshold_for "$f")

  if [ "$age" -gt "$threshold" ]; then
    title="[Doc Review] $f 需要驗證"
    existing=$(gh issue list --label doc-review --search "$title" --state open --json number --jq length)
    if [ "$existing" -eq 0 ]; then
      gh issue create \
        --title "$title" \
        --body "上次驗證：$last（${age} 天前）。請於 7 天內更新 \`last_validated\` 並送 PR。" \
        --assignee "$owner" \
        --label doc-review
    fi
  fi
done < <(find docs usecases -type f -name "*.md" -print0 2>/dev/null)
```

### Issue 格式

```
標題：[Doc Review] docs/xxx.md 需要驗證
Body：
  - 文件路徑
  - 上次驗證：YYYY-MM-DD（N 天前）
  - 請於 7 天內更新 last_validated 並送 PR
Assignee：validated_by 欄位的 GitHub username
Label：doc-review
```

## 不回應的後果

- 超過 deadline（+7 天）未處理：文件標記為 `freshness: stale`
- Agent 讀取 stale 文件時，自動附加警告：`⚠️ 此文件已超過 review 週期，內容可能過時`
- 其他 agent 或貢獻者可接手更新
- **連續 2 次 review cycle（約 30 天）未回應**：原作者從信任名單中移除，文件開放 `help-wanted` 認領

## Agent 驗證流程

Agent 執行 review 時：

1. 讀取文件內容
2. 用 `gh search code` 查對應 source code
3. 比對是否有 breaking change 或 API 變更
4. 若有差異，自動送 PR 修正
5. 更新 frontmatter `last_validated`

## 相關 Issue

- [#346 提案：文件鮮度保證機制](https://github.com/thepagent/claw-info/issues/346)
- [#349 MVP workflow 實作](https://github.com/thepagent/claw-info/pull/349)
