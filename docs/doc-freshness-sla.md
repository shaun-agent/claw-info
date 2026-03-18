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
        run: python3 .github/scripts/check_freshness.py
```

### 核心腳本邏輯（`.github/scripts/check_freshness.py`）

```python
# 週期設定（天）
THRESHOLDS = {
    "usecases/": 14,
    "docs/":     28,
}
DEFAULT_THRESHOLD = 56

# 流程：
# 1. 掃描所有 .md，解析 frontmatter 取得 last_validated / validated_by
# 2. 計算距今天數，超過 threshold 者列為 stale
# 3. 檢查是否已有同名 open issue（避免重複開）
# 4. 若無，用 gh issue create assign validated_by
# 5. 若超過 threshold + 7 天仍 open，更新文件 freshness: stale
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

### 防重複機制

開 issue 前先執行：
```bash
gh issue list --label doc-review --search "[Doc Review] docs/xxx.md" --state open
```
若已有 open issue 則跳過。

原作者收到 issue 後須：

1. 對照 source code 確認內容仍正確
2. 更新 `last_validated` 與 `validated_by`
3. 若有過時內容，一併修正並送 PR

## 不回應的後果

- 超過 deadline（+7 天）未處理：文件標記為 `freshness: stale`
- Agent 讀取 stale 文件時，自動附加警告：`⚠️ 此文件已超過 review 週期，內容可能過時`
- 其他 agent 或貢獻者可接手更新
- 長期不回應的原作者，可能從信任名單中移除

## Agent 驗證流程

Agent 執行 review 時：

1. 讀取文件內容
2. 用 `gh search code` 查對應 source code
3. 比對是否有 breaking change 或 API 變更
4. 若有差異，自動送 PR 修正
5. 更新 frontmatter `last_validated`

## 相關 Issue

- [#346 提案：文件鮮度保證機制](https://github.com/thepagent/claw-info/issues/346)
