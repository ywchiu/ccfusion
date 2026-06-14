# Controlled Coding Fusion Skill(`ccfusion`)

[English](README.md) | **繁體中文**

> 靈感來自 [OpenRouter Fusion](https://openrouter.ai/)。

**三個諸葛亮,勝過一個 Fable 5。**

與其讓一個 AI 憑第一直覺就衝進去改你的程式碼,`ccfusion` 先找一排 AI 來挑這份計畫的
毛病:一個看架構、一個看測試、一個看安全,Codex 專門唱反調,需要查最新資料時再叫
Gemini 出馬。接著一個「裁判」把所有意見綜合起來,等嚴重問題都解掉了,才核准這份計畫。
**到這一步,才讓一個 AI 真的動手寫程式。**

(這跟 OpenRouter Fusion 同一個精神 —— 把多個模型湊起來,得到比單一模型更好的結果。)

它不會跟你保證程式碼完美;它只是讓「爛計畫」變得很難發生,還讓你看得到計畫是怎麼決定的。

> 先規劃。大家審。一個人寫。

```text
很多 AI 都能檢視、挑毛病。
只有一個 —— Claude Code 主 session —— 能動手寫。
```

### 為什麼值得用

- **爛計畫早點砍。** 問題在動任何檔案之前就被抓出來,而不是等你花一小時做錯方向才發現。
- **很多人挑毛病,只有一個人動手。** 一堆 AI 可以戳洞,但只有一個能碰你的檔案,不會有
  意外改動。
- **整個過程攤在陽光下。** 每次跑都留下完整、不會被改掉的紀錄,看得到計畫怎麼來的。
- **不會暴走。** 每個重試迴圈都有上限、到頂就交給人;機密永遠不外送;危險指令直接擋。

**狀態:v0.1.0。** 它刻意只做一件事:在有人動手寫之前,先幫寫程式的計畫把關。先不搞更花俏的。

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

## 一句話設計

> Controlled Coding Fusion v0.1 不是框架。
> 它是一道很窄的 Plan Gate,讓 Claude Code 在動筆寫之前先「找人幫忙一起想」。

## 授權

Apache-2.0,詳見 `LICENSE`。
