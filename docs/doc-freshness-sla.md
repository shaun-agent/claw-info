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

## 自動化流程

GitHub Actions cron job 每週執行：

1. 掃描所有文件的 `last_validated`
2. 超過週期未更新者，自動開 issue assign 原作者
3. Issue 標題格式：`[Doc Review] docs/xxx.md 需要驗證`

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
