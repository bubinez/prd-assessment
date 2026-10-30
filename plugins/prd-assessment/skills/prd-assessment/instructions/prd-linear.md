---
name: prd-linear
description: Step 13 of /prd-assessment. Get team from Linear MCP — list_teams, list_users, list_cycles. Use when executing Step 13 of prd-assessment workflow.
---

# PRD Assessment — Step 13: Get Team (Linear)

При выполнении Step 13 пайплайна /prd-assessment — следуй этой инструкции.

## MCP Server
- **Cursor:** `plugin-linear-linear`
- **Antigravity:** `linear-mcp-server` (вызовы `mcp_linear-mcp-server_toolname`)

---

## Действия

### 1. Team Selection
- **If `--team` в команде** — use it.
- **Else** — вызови инструмент `list_teams` (или аналогичный для твоего MCP-сервера, например `mcp_linear-mcp-server_list_teams`) — аргументы опциональны.
- Output: «Команды Linear: [имя1], [имя2], … Укажите team name/key (например, Toprent) или «пропустить» для оценки без velocity.» — имена из массива ответов.

### 2. After User Picks Team
- **get_team** (для teamId): `get_team` (или `mcp_linear-mcp-server_get_team`) с аргументами `{query: "Team key"}` — получить id (UUID) команды.
- **list_users**: `list_users` (или `mcp_linear-mcp-server_list_users`) с аргументами `{team: "Team key"}` — Responsible для Change Plan. Параметр `team`: name или ID.
- **list_cycles**: `list_cycles` (или `mcp_linear-mcp-server_list_cycles`) с аргументами `{teamId: "<uuid>"}` — **teamId обязателен** (UUID из get_team). Velocity: completedScopeHistory, scopeHistory.

### 3. Optional Tools
- `get_team` — team details
- `list_issues` — per-assignee velocity (optional)
- `list_issue_statuses` — resolve "Done" state type

### 4. Output
- Team members (displayName) → для Responsible
- Velocity baseline (points/cycle) → для калибровки оценок

### 5. Degraded
- **Linear MCP fails / not connected** → proceed без team/velocity.
- Estimate = expert only; Responsible = «TBD».

---

## Перед вызовом MCP
Проверь схему инструмента (если mcps folder доступен) для корректных аргументов.
