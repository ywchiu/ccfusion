# Controlled Coding Fusion Skill(`ccfusion`)

[English](README.md) | **繁體中文**

> 靈感來自 [OpenRouter Fusion](https://openrouter.ai/)。

一個讓 AI 輔助寫程式更安全的 Claude Code Skill —— 專注在**規劃階段**。

> 先規劃。並行審查。只用一位寫手執行。

```text
眾多 agent 可以檢視與批判。
只有 Claude Code 主 session 可以執行。
```

`ccfusion` 在**動任何檔案之前**先把規劃品質拉高,並留下「這份計畫是怎麼形成的」
完整可稽核軌跡。它**不保證正確性** —— 它只負責把計畫卡關審過,再交由單一寫手實作。

**狀態:v0.1.0(已凍結)。** 刻意做窄:只有一個 preset(`plan-gate`)、一個執行
設定檔(`strict-single-writer`)。它不是通用 fusion 框架、不是 preset 路由器、也不是
多 agent 執行環境。

---

## 安裝

本儲存庫提供的 skill 位於:

```text
.claude/skills/ccfusion/SKILL.md
```

把 `ccfusion` skill 目錄放到你的 Claude Code skills 路徑底下(專案層
`.claude/skills/` 或個人層 `~/.claude/skills/`),然後請 Claude Code「use controlled
coding fusion」或執行 `/ccfusion`。

## 它做什麼

```text
使用者任務
  -> 建立 run 目錄 + run-state.json
  -> 建立 Context Pack
  -> Pre-gate 主要輸入驗證            (需 PASS 才能繼續)
  -> 選用的 Gemini 外部研究            (需 scoped manifest)
  -> 唯讀的內部 repo 審查(Claude subagents)
  -> Codex 對抗式審查
  -> Plan Judge 綜整裁決(most-severe-wins,最嚴重者勝)
  -> APPROVE_PLAN -> 寫出 Approved Plan -> Claude Code 主 session 才可實作
     REVISE_PLAN          -> 計畫修訂迴圈(有上限)
     NEEDS_MORE_RESEARCH  -> 研究重入迴圈(有上限)
     BLOCKED_NEEDS_HUMAN  -> 停止、寫 final summary、上報人類
```

每個迴圈都有上限;任何超限都以 `BLOCKED_NEEDS_HUMAN` 收尾。Plan Gate 永遠不改檔案;
唯有 Approved Plan artifact 存在後,才開始實作。

## 角色

| 角色 | 能力 |
| ---- | ---- |
| Claude Code 主 session | 協調者、裁決者、規劃者,**也是唯一的執行者** |
| Claude Code 唯讀 subagents | 內部 repo 審查者(不改檔案) |
| Codex plugin | 只做對抗式審查(不改檔案) |
| Gemini | 只做外部研究(不改檔案、不輸出 gating 決策) |
| 測試 / CI | 實作之後的客觀驗證者 |

## Run 目錄

每次執行都建立一個 run 目錄 —— 這是稽核軌跡,**永不**被 stash、搬移或刪除:

```text
.fusion/runs/<run_id>/
```

`run_id` = `YYYYMMDDTHHMMSSZ-<repo_slug>-<git_ref>-<random6>`,其中 `git_ref` 有確定性
fallback(`no-commit`、`no-git`、`detached-or-unknown`)。所有 artifact 都做版本化、
永不覆寫;重新產生的 artifact 是新的 `attempt-NN`。

## 目錄結構

```text
.claude/skills/ccfusion/
  SKILL.md
  templates/   context-pack、input-validation、external-call-manifest、
               gemini-research-pack、reviewer-output、codex-adversarial-review、
               plan-judge、approved-plan、final-summary
  schemas/     context-pack、input-validation、external-call-manifest、
               reviewer-output、plan-judge、approved-plan  (JSON Schema)
  examples/    bug-fix、dependency-upgrade、refactor
```

v0.1 刻意**沒有 `presets/` 目錄**。

## 安全邊界

- 只有 Claude Code 主 session 可以改檔案。
- 外部呼叫以 **(target + data-scope)** manifest 卡關,slug 採可讀名稱(不用 hash);
  機密與客戶資料一律不外送 —— 改送脫敏後的技術摘要。詳見 `SECURITY.md`。
- 破壞性指令與無範圍的 `git stash`,未經明確核准一律禁止。

## Roadmap(v0.1 尚未實作)

以下純屬未來方向。**它們目前並不存在** —— 本儲存庫沒有對應的目錄、preset 或程式碼,
`ccfusion` 也不會表現得好像它們存在。

```text
v0.2 Review Gate
  實作後的 Codex 審查;P0/P1/P2 分級;修補迴圈;補測試迴圈;實作中重規劃並做路徑
  範圍化的 partial-patch 保留;review-gate 總嘗試次數保險絲;獨立的候選計畫目錄
  (與 approved-plan/ 分開),讓「候選 vs 已核准」是結構性區分而非只靠 attempt 編號

v0.3 Research Fusion
  獨立的研究 preset;多來源比對;來源品質評分

v0.4 Decision Fusion
  成本 / 風險 / 時間 取捨面板;選項比較;決策備忘錄

v0.5 Patch Fusion
  以隔離的 git worktree 執行;多個候選 patch;以測試為依據做挑選
```

**Roadmap 規則:** 在 preset 尚未實作前,不得先建立該 preset 的目錄。通用的
*Normalize → Panel → Judge → Decision* 核心,若真的被驗證為實在,也只有在 Plan Gate
與 Review Gate 都在真實 repo 上跑過之後才會抽取出來。

## 一句話設計

> Controlled Coding Fusion v0.1 不是框架。
> 它是一道很窄的 Plan Gate,讓 Claude Code 在動筆寫之前先「找人幫忙一起想」。

## 授權

Apache-2.0,詳見 `LICENSE`。
