---
description: Полный workflow для сложных фич с планированием, реализацией, проверками, review и аудитом.
---

# workflow-feature

**Полный workflow** — задействует planner, worker, refactor, test-runner, debugger, reviewer, security-auditor, senior-reviewer, documenter. Для сложных фич с планированием, реализацией, проверками, review и аудитом.

**Использование:** `/workflow-feature <описание фичи>` — например: `/workflow-feature Добавь систему аутентификации с email/password и OAuth`

**Пути артефактов фичи:** как в `prd-assessment`, возьми корневой каталог из `.agent/config.json` → `documentation.paths.prdAssessment` (сейчас `docs/`). Используй существующий `<assessment-dir>/<feature-slug>/`, если фича уже проходила assessment (в том числе при вызове `@docs/<feature-slug>/prd.md`); иначе создай папку с нормализованным slug латиницей и дефисами. PRD находится в этой папке под именем `prd.md`. План реализации и итоговая документация также находятся в этой папке. Не создавай вторую папку или копию PRD для той же фичи.

> **Поддержка систем:**
> - **Cursor:** Инструкции выполняются через встроенный Orchestrator. Субагенты вызываются через инструмент `internal_context`. Правка файлов — `StrReplace`, Linear — `plugin-linear-linear`.
> - **Antigravity:** Агент работает в режиме `sequential_thinking` как единственный исполнитель, "переключая" внутренние роли (planner, worker и т.д.) согласно этапам. Правка файлов — `replace_file_content` / `multi_replace_file_content`. Доступ к Linear — через сервер `linear-mcp-server` (`mcp_linear-mcp-server_*`). GitHub CLI доступен через `run_command`.
> - **Claude Code:** Субагенты вызываются через `Task` tool с `subagent_type="general-purpose"` — в `prompt` передаётся содержимое `.agent/instructions/<role>.md` + контекст. Параллельные роли — несколько `Task` в одном сообщении. Правка файлов — `Edit` / `Write`. Команды — `Bash`. Трекинг — `TodoWrite`.

🛑 **КРИТИЧЕСКОЕ ПРАВИЛО:** Тебе строго **ЗАПРЕЩЕНО** пропускать шаги пайплайна. Ты обязан последовательно пройти через все этапы: планирование (Step 1), реализация каждой задачи (Step 2a), тестирование (Step 2b) и все виды ревью (Step 2c, 3, 4). Пропуск этапов тестирования или ревью «потому что всё очевидно» — грубое нарушение. Каждый этап должен быть зафиксирован в твоих размышлениях (thought).

## Шаги

**Как работать с ролями (субагентами):**
- **Cursor:** для каждого шага вызывай инструмент **internal_context** с параметром `subagent_type`.
- **Antigravity:** выполняй задачи роли самостоятельно в текущей сессии, явно указывая в `thought` (sequential_thinking), какую роль ты сейчас "берешь" (например: "Роль: Planner. Начинаю декомпозицию...").
- **Claude Code:** для каждого шага вызывай `Task(subagent_type="general-purpose", prompt="<содержимое .agent/instructions/<role>.md>\n\nЗадача: ...", description="<role>: ...")`. Параллельные роли — несколько `Task` в одном сообщении.

**Роли:** `planner` | `worker` | `refactor` | `test-runner` | `debugger` | `reviewer` | `senior-reviewer` | `security-auditor` | `documenter`

Не выполняй шаги planner/worker/test-runner/reviewer/documenter/security-auditor/senior-reviewer "от себя" — только через `internal_context` (Cursor), ролевое разделение (Antigravity) или `Task` (Claude Code).

Выполни последовательно:

0. **Инициализация репо (если новый проект)**
   - **Опционально:** если пользователь явно не просит инициализировать репо (например, «быстрый прототип», «без git») — можно пропустить шаг.
   - Иначе: если папка не git-репозиторий (или нет remote) — выполни `git init`, при наличии `gh` и прав — `gh repo create` (или выведи команду пользователю). Задай origin. Первый коммит — после создания структуры worker'ом.

1. **Planner — декомпозиция**
   - **Cursor:** вызови `internal_context(subagent_type="planner", ...)` с описанием фичи.
   - **Antigravity:** используя `sequential_thinking`, декомпозируй задачу.
   - **Claude Code:** `Task(subagent_type="general-purpose", prompt="<.agent/instructions/planner.md>\n\nЗадача: ...")`. Зарегистрируй все подзадачи через `TodoWrite`.
   - **Результат:** план с ID задач (например AUTH-001, AUTH-002), порядком, зависимостями и **рекомендуемой ролью** (worker, refactor, documenter и т.д.). Формат — см. [task-management SKILL](task-management.md).
   - Перед сохранением плана создай папку фичи, если её ещё нет.
   - Если существует `<assessment-dir>/<feature-slug>/prd.md`, используй его `## Change Plan` как исходную декомпозицию и сохрани план реализации в `<assessment-dir>/<feature-slug>/implementation-plan.md`. Если PRD нет, сохрани план по тому же пути.

   **Если есть GitHub (origin, gh):** для каждой подзадачи из плана создай GitHub issue через `gh issue create`. Сохрани соответствие task ID ↔ issue number (например, в плане или в ai_docs). В промптах для worker/test-runner/thought передавай ссылку на issue (например, «Closes #N»). Используй [gh-commands.md](gh-commands.md) для кириллицы.

   **Перед циклом реализации:** создай ветку `feature/<short-name>`.

2. **Для каждой задачи из плана — цикл реализации**
   Выполняй по порядку с учётом зависимостей:

   a. **Реализация (worker или refactor)**
      - **Cursor:** вызови `internal_context` с типом роли, рекомендованным planner'ом.
      - **Antigravity:** прими на себя роль `worker` (новые фичи) или `refactor` (улучшение структуры) и реализуй подзадачу. Используй `replace_file_content` / `multi_replace_file_content`.
      - **Claude Code:** `Task(subagent_type="general-purpose", ...)` с `.agent/instructions/worker.md` или `.agent/instructions/refactor.md`. Правка файлов — `Edit` / `Write`.

   b. **Test-Runner — тесты**
      - Выполни тесты (через `internal_context` в Cursor, `run_command` в Antigravity, `Task`+`Bash` в Claude Code).
      - Если тесты падают:
        - Перейди к роли `debugger` (максимум 3 попытки на задачу).
        - После каждой попытки — снова тесты.
      - Продолжай только при успешных тестах или явном решении пользователя.

   c. **Reviewer — code review**
      - **Вариант A (per-task):** делай review после каждой подзадачи — раннее обнаружение проблем.
      - **Вариант B (batch):** для проектов с множеством подзадач допустимо отложить review до конца (шаги 3–4).
      - Если reviewer нашёл проблемы:
        - **Простые правки** (одно-два точечных изменения) — примени сам (`StrReplace` / `replace_file_content` / `Edit`).
        - **Сложные исправления** (неочевидные баги, рефакторинг) — роль `debugger`.
      - Повтори reviewer при необходимости.

   d. После каждой подзадачи (или группы): коммит с `Closes #N` или `Fixes #N` при наличии issue.

3. **Senior-Reviewer — архитектурный обзор**
   - После завершения всех подзадач перейди к роли `senior-reviewer`.
   - Оцени архитектуру, граничные случаи, производительность, maintainability по SKILL [architecture-principles](architecture-principles.md).
   - При серьёзных замечаниях — исправь (роль `debugger` или `refactor`).

4. **Security-Auditor — финальный аудит (если фича security-sensitive)**
   - Если фича касается auth, payments, sensitive data — выполни аудит (роль `security-auditor`) по SKILL [security-guidelines](security-guidelines.md).

**Параллельность:** Шаги reviewer (вариант B), senior-reviewer и security-auditor можно запускать параллельно через `internal_context` (Cursor) или несколько `Task` в одном сообщении (Claude Code). В **Antigravity** выполняй их строго последовательно.

5. **Documenter — итоговый отчёт**
   - Роль `documenter`: создай итоговый отчет/документацию (что реализовано, структура, использование) в `<assessment-dir>/<feature-slug>/implementation.md`. Пользуйся SKILL [docs](docs.md). Если в папке уже есть документация реализации, обнови её вместо создания дубликата.

6. **PR (если есть GitHub)**
   - Выполни `gh pr create` с подробным описанием, привязывающим PR к issues.

## Результат

Перед возвратом результата:

1. **Session report:** сохрани отчёт в `<metrics.sessionsPath>/session-<YYYYMMDD-HHmm>.json` (путь из `.agent/config.json`; сейчас `ai_docs/develop/reports`) со структурой: command (workflow-feature), workflow (feature), subagentsCalled (или rolesAssumed), debuggerCalls, testsPassed, reviewerFindings, documentationCreated, taskSummary.
2. **Запусти скрипт метрик:** `node .agent/scripts/metrics-report.js`.
3. Включи итоговый скор в ответ (блок «Метрики»).

Верни пользователю:
- План (список выполненных задач)
- Статус тестов по каждой задаче
- Резюме reviewer и senior-reviewer
- Результаты security-auditor (если вызывался)
- Ссылку на итоговую документацию
- Блок «Метрики» со скором

## Заметки

- **Antigravity vs Cursor vs Claude Code:** Antigravity выполняет все этапы в рамках одного агента (MCP). Cursor и Claude Code делегируют роли субагентам (`internal_context` и `Task` соответственно).
- Security-auditor вызывается один раз в конце (шаг 4), не на каждую подзадачу.
- Соблюдай порядок задач из плана (учти зависимости).
- Ограничение: максимум 3 попытки debugger на одну подзадачу при падении тестов.
- При зацикливании или неразрешимых ошибках — остановись и сообщи пользователю.
- Можно пропустить тесты, review или security-auditor для конкретной задачи, если пользователь явно попросит.
