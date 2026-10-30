---
description: 
---

# prd-assessment

**Пайплайн декомпозиции и формирования PRD** — уточнение требований, анализ кодовой базы, технические решения и декомпозиция. Это отдельный assessment-пайплайн, не `/workflow-feature` и не оркестрация ролей `planner/worker`. В среде Codex допускается делегирование автономных аналитических шагов отдельным subagent'ам, но пользовательский диалог и критичные решения остаются в основном flow. Переходы между шагами по умолчанию **автономные**: пользователь нужен только там, где действительно требуется выбор, подтверждение изменения scope или ответ на вопрос, который нельзя подтвердить локально.

## Переносимый контекст

Пути `instructions/`, `rules/`, `agents/` и `references/` в этом workflow и runbook относятся к каталогу глобального скилла, а не к рабочему проекту. Проектные правила имеют приоритет над перенесёнными общими рекомендациями. При отсутствии `.agent/config.json` определи команду и каталог тестов по документации и конфигурации проекта; папку PRD выбери по соглашениям проекта, иначе `docs/`. Не создавай конфигурацию ради скилла. При делегировании передавай абсолютные пути к файлам скилла. Используй доступные механизмы запуска агентов с инструкциями из соответствующих TOML; ограничения среды и пользовательские инструкции имеют приоритет.

## Entry Contract For Codex Wrapper

Если этот workflow был вызван через plugin skill / wrapper в Codex, считай этот раздел обязательным контрактом входа. Не полагайся на краткий `SKILL.md` как на полное описание поведения.

1. Сразу работай по этому файлу как по единственному source of truth для порядка шагов, resume-логики, checkpoint-ов и финального результата.
2. Если пользователь вызвал `/create-prd`, обрабатывай его как точный алиас `/prd-assessment`.
3. Если пользователь передал `@draft.md` или другой draft-файл, используй его как вход пайплайна, а не как готовый финальный PRD.
4. Если пользователь передал свободный текст, трактуй его как feature description для Step 1.
5. Если пользователь просит продолжить существующий assessment или передаёт уже созданный `prd.md`, проверь resume path и переиспользуй существующую папку `docs/<feature-slug>/` без дублирования артефактов.
6. Если входа нет или он непригоден, остановись и запроси у пользователя короткое описание фичи или draft-файл.
7. Не создавай параллельный упрощённый flow вне этого workflow; step-specific детали загружай из `instructions/prd-*.md`, когда этот файл на них ссылается.
8. После завершения assessment остановись на результате, описанном в секции guardrail; не переходи к реализации без новой явной команды пользователя.

## Delegation Contract For Explicit `/prd-assessment`

Если пользователь **явно** вызвал `/prd-assessment` или `/create-prd`, для Codex действует обязательный delegation contract.

1. Heavy steps `Step 4`, `Step 11`, `Step 12`, `Step 14` должны выполняться через project-scoped custom subagents из `agents/`.
2. Имена агентов:
   - `Step 4` -> `prd_explorer`
   - `Step 11` -> `prd_self_reviewer`
   - `Step 12` -> `prd_devils_advocate`
   - `Step 14` -> `prd_decomposer`
3. Как запускать этих subagent-ов в Codex: см. локальный runbook `references/subagent-runbook.md` -> `PRD Assessment Custom Subagent Runbook`. Не искать механику запуска в интернете: сначала использовать доступный Desktop `spawn_agent`, затем локальный `codex exec` fallback, иначе остановиться по stop condition из runbook.
4. **Delegation Boundary (hard guard):** только основной flow имеет право запускать обязательные heavy-step subagent-ы (`Step 4/11/12/14`).
   - subagent-у запрещено запускать других subagent-ов (никаких sub-subagent chains);
   - subagent-у запрещено вызывать `codex exec` для запуска другого custom agent;
   - при недоступности runtime/tools subagent возвращает `blocked_by_runtime` в основной flow и останавливается;
   - retry/escalation/stop-condition принимает только основной flow.
5. Для этих шагов локальный shortcut запрещён:
   - основной flow не имеет права заменить subagent-анализ собственной краткой проверкой;
   - отсутствие subagent-result означает, что шаг не завершён;
   - основной flow может только передать контекст, дождаться результата, показать summary и интегрировать решения пользователя.
6. Если custom agent недоступен, основной flow обязан явно остановиться и сообщить, какой именно agent missing, вместо тихого single-agent fallback.
7. Этот контракт сильнее общих эвристик. Для explicit `/prd-assessment` heavy-step delegation считается обязательной частью корректного выполнения workflow.
8. Если heavy-step subagent возвращает `decision_required`, основной flow обязан показать пользователю короткий decision-shaped вопрос и не имеет права подменить выбор своим допущением.
9. Для heavy steps допустимый протокол ранней эскалации включает поле `questions_for_user`. Минимальные поля:
   - `observed`
   - `why_it_matters`
   - `question`
   - `options`
10. Subagent-ам запрещены молчаливые допущения по same-class sibling feature, scope fork, boundary choice или include/exclude решению. Такие случаи должны возвращаться вверх как `decision_required`.

**Использование:** `/prd-assessment [описание или @файл] [--team X] [--path Y] [--lang ru|en] [--linear]`
**Возобновление:** `/prd-assessment --resume <feature-slug> [--from-step N] [--team X] [--linear]`

**Алиас:** `/create-prd` — тот же пайплайн (встроенный алиас скилла).

Примеры:
- `/prd-assessment Добавить экспорт дашборда в PDF`
- `/prd-assessment @draft-tz.md --team Toprent --lang ru`
- `/prd-assessment Интеграция с CRM --linear --estimate` ← полный цикл с Linear и стресс-тестом оценок
- `/prd-assessment --resume integration-crm-tra` ← повтор всего цикла по готовому PRD
- `/prd-assessment --resume integration-crm-tra --from-step 9` ← войти с любого шага

**Флаги (опциональные):**
- `--linear` — включить Step 13 (получение команды и velocity из Linear). По умолчанию: **выключен**.

> **Поддержка систем:**
> - **Cursor:** Инструкции выполняются через встроенный Orchestrator, правка файлов происходит через `StrReplace`, Linear доступен через `plugin-linear-linear`.
> - **Antigravity:** Агент работает в режиме `sequential_thinking` как единственный исполнитель, последовательно проходя шаги. Правка файлов осуществляется встроенными инструментами (e.g., `replace_file_content`, `multi_replace_file_content`), доступ к Linear — через сервер `linear-mcp-server` (`mcp_linear-mcp-server_*`).
> - **Claude Code / Codex:** Основной flow координирует диалог и переходы между шагами. Автономные аналитические шаги `Step 4`, `Step 11`, `Step 12`, `Step 14` выполняются через `Task` / subagent. Steps `1`, `2`, `3`, `5`, `6`, `7`, `8`, `9`, `10`, checkpoint-вопросы и ответы пользователя остаются в основном flow. Правка файлов — `Edit` / `Write`. Чтение файлов — `Read`. Команды — `Bash`. Доступ к Linear — через `mcp_linear-mcp-server_*` (если настроен MCP).
>
> 🛑 **КРИТИЧЕСКОЕ ПРАВИЛО:** Тебе строго **ЗАПРЕЩЕНО** пропускать шаги пайплайна. Если пользователь пишет «да» или «продолжай», ты должен четко определить текущий шаг, перейти к **строго следующему** шагу по номерам (например, после Step 11 ОДНОЗНАЧНО идёт Step 12, затем Step 13, а без `--linear` сразу Step 14), прочитать его `.md` инструкцию и выполнить ЕЕ ДО КОНЦА. Никогда не перепрыгивай через шаги (например, сразу к декомпозиции), даже если кажется, что всё готово.
>
> 🛑 **SEQUENTIAL EXECUTION HARD GUARD (для всех шагов 1-14):**
> - Разрешён только один активный шаг пайплайна в каждый момент времени.
> - Параллельное выполнение разных шагов запрещено (включая `Step 1-3`, checkpoint-шаги и resume-режим).
> - Следующий шаг можно начинать только после явного завершения текущего по его выходному контракту.
> - Нельзя запускать `Step N+1`, пока `Step N` не закрыт; нельзя запускать `Step N+K` в обход промежуточных шагов.
> - В resume-режиме правило идентично: старт только с указанного `--from-step`, затем строго последовательно без параллельных веток.
> - Единственное допустимое "пропускание" — только явно описанные условные переходы workflow (например, `Step 13` при отсутствии `--linear`).

> ✅ **КРИТИЧЕСКОЕ ПРАВИЛО КАЧЕСТВА (код/тесты):** если в рамках пайплайна ты доходишь до частей, где формируется Change Plan / Testing Strategy или предполагаются изменения кода (в этом repo) — ты обязан соблюдать проектные правила программирования и тестирования:
> - **Правила программирования:** следуй `instructions/worker.md` (минимальные правки, следование существующим конвенциям, без лишнего рефакторинга, и т.д.).
> - **Правила работы с тестами:** следуй `instructions/test-runner.md` и настройкам проекта из `.agent/config.json` (секция `testing`: `command` и `path`).
> - Любой план работ (Change Plan) должен явно включать задачи на тестирование и шаг “запуск тестов” указанной командой проекта.
> - Любое утверждение, которое можно проверить по коду, тестам, docstring или проектной документации **сейчас**, должно быть либо подтверждено в рамках assessment, либо переписано как **целевое состояние реализации**. Нельзя оставлять в PRD «проверим потом», если проверка доступна локально.
> - Нельзя описывать **target-state** языком уже подтверждённого baseline. Confirmed baseline и целевое поведение должны быть явно разделены.
> - Для парных или зеркальных действий (например, like/dislike, create/delete, push/pull) assessment обязан проверить **симметрию** по осям: payload, auth, response, side-effects, tests, docs.
> - Корректность документации — часть результата. К завершению assessment должен быть определён хотя бы один **канонический** runtime-документ, supporting-документы и план устранения противоречий в docs.

> 🛑 **NO PRODUCT FALLBACKS:** PRD описывает один выбранный основной путь выполнения. При ошибке, неподдержанном типе или неполных данных он должен сохранить последний подтверждённый результат/checkpoint, вернуть видимую ошибку или partial/unsupported coverage и блокировать зависимую операцию. Запрещено автоматически подменять путь другим API/RPC, connector/источником, фильтром, кешем, локальной догадкой или полным перечитыванием истории. Discovery может проверять такие варианты как отдельные гипотезы; они не становятся runtime fallback. Добавить альтернативу можно только после явного решения пользователя сделать её новым основным путём.

### Режим по умолчанию: autonomous discovery + decision-gated flow

Для всего пайплайна действуют следующие правила:

1. Агент обязан **сам продвигать workflow вперёд**, пока не встретит реальную точку решения пользователя.
2. Формальные подтверждения вида «продолжить?», «идём дальше?», «есть ли новые вопросы?» запрещены, если:
   - нет `ESCALATE`;
   - нет изменения scope с ненулевыми последствиями;
   - нет выбора между содержательно разными вариантами;
   - нет вопроса, ответ на который нельзя подтвердить локально.
3. Если шаг завершился без пользовательского выбора, основной flow **автоматически** переходит к следующему шагу и показывает только краткий summary.
4. Для `Step 4`, `Step 5`, `Step 9`, `Step 11`, `Step 12`, `Step 14` действует **adversarial stance by default**:
   - искать не только основной path, но и adjacent / mirror / mixed-use пути;
   - проверять материал как скептичный архитектор, QA-перфекционист, новый член команды и вредный проджект без отдельного напоминания пользователя;
   - не завершать шаг, пока не предъявлен явный output-контракт: что найдено, что исключено, что требует решения пользователя.
   - для `Step 4` отдельно искать same-class sibling flows и shared runtime surfaces до поздних шагов.
5. Формулировки в стиле «если хотите, могу ещё проверить...» запрещены для обязательных follow-up внутри текущего шага. Если follow-up обязателен для замыкания анализа, агент делает его сам.

**Reference:** Подробные инструкции каждого шага находятся в `../instructions/`.

---

## Парсинг команды

```
/prd-assessment [описание или путь к файлу] [--team X] [--path Y] [--lang ru|en]
/prd-assessment --resume <feature-slug> [--from-step N] [--team X] [--lang ru|en]
```

- **Описание:** inline текст или ссылка на файл (@file.md). Если нет — промпт No Input (§10.2).
- **--team** — team key Linear; иначе list_teams для выбора (только при `--linear`).
- **--path** — путь проекта; по умолчанию — workspace path.
- **--lang** — ru | en; по умолчанию **en**.
- **--linear** — включить Step 13 (Get Team из Linear). По умолчанию: выключен.
- **--resume** — slug существующего фичи-ассессмента. Активирует **Resume Mode** (см. ниже).
- **--from-step** — номер шага (1-14), с которого начать или возобновить (только совместно с `--resume`).

**Авто-детект resume:** если передан `@file.md` и это уже сгенерированный `prd.md` (содержит секции `## Goals`, `## Functional Requirements`, `## Change Plan`) — предложить Resume Mode автоматически.

Из описания извлекай **feature-slug** латиницей и дефисами. Каталог assessment бери из `.agent/config.json` → `documentation.paths.prdAssessment` (если не задано, используй принятую в проекте папку документации или `docs/`). Папка фичи: `<assessment-dir>/<feature-slug>/` (например `docs/02-core-linear/`).

---

## Схема пайплайна

```
Step 1 (Business + Classification + MBQ + Corner Cases)
    ↓
Step 2 (Form PRD) ←──────── [если Step 4/5 изменил scope]
    ↓
Step 3 (Project Path)
    ↓
Step 4 (Codebase Analysis) ── Scope Checkpoint ──→ Step 2?
    ↓
Step 5 (Completeness Check) ── найти все интеграционные точки и legacy data patterns ──→ Step 2 (update FR)?
    ↓
Step 6 (Legacy Data Compatibility Check) ── есть ли existing bad data, которые healing не достанет? ──→ добавить FR?
    ↓
Step 7 (Format Consistency) ── сравнить JSON/формат между producer-ами ──→ update FR?
    ↓
Step 8 (Refactoring Regression Check) ── diff «старый код vs новый код» ──→ добавить FR/тесты?
    ↓
Step 9 (Technical Questions) ←── [мини-раунд ≤5 вопросов из Step 14]
    ↓
Step 10 (Final PRD: решения в FR + OQ Sync)
    ↓
Step 11 (PRD Self-Review: guard rails)
    ↓
Step 12 (Devil's Advocate Review: ролевой стресс-тест PRD)
    ↓
Step 13 (Linear Team) ── [только при --linear; иначе сразу Step 14]
    ↓
Step 14 (Decompose) ── Техвопросы? ──→ Step 9 mini
    ↓
  Завершение
```

**По умолчанию** (без флагов): Step 13 пропускается, переход идёт сразу из Step 12 в Step 14.

---

## Resume Mode

Активируется при `--resume <feature-slug>` или авто-детекте существующего `prd.md`.  
Позволяет заново пройти весь цикл или войти с конкретного шага — без потери накопленного контекста.

### Загрузка контекста

> ⚠️ **Duplicate Guard:** Перед загрузкой контекста проверить, существует ли папка `docs/<feature-slug>/` для этой фичи. Если да — использовать существующую папку. Если нет — создать новую с нормализованным feature-slug.

1. Прочитай `docs/<feature-slug>/prd.md` — это **базовый контекст** для всех шагов.
2. Если есть `codebase-analysis.md` в той же папке — прочитай его тоже.
3. Выведи сводку текущего PRD:

```
## Resume: <feature-slug>

**PRD:** docs/<feature-slug>/prd.md
**Goals:** <краткое резюме из секции Goals>
**FR count:** <число требований>
**Open Questions:** <число открытых OQ>
**Change Plan:** <статус: есть / нет>

---
```

### Выбор точки входа

Если `--from-step N` указан — начать с шага N.  
Иначе — предложить меню:

```
С какого шага продолжить?
  1 — Business Clarification (пересмотреть цели и вопросы)
  2 — Form PRD (переформировать Business PRD)
  3 — Project Path (сменить путь)
  4 — Codebase Analysis (перезапустить анализ кода)
  5 — Completeness Check (проверить интеграционные точки и legacy patterns)
  6 — Legacy Data Compatibility Check (перепроверить existing data и healing)
  7 — Format Consistency (сверить producer-ов и JSON-контракты)
  8 — Refactoring Regression Check (сравнить old/new path)
  9 — Technical Questions (задать заново / другой моделью)
  10 — Final PRD (перезаписать решения)
  11 — PRD Self-Review (прогнать проверку ещё раз)
  12 — Devil's Advocate Review (ролевой стресс-тест PRD)
  13 — Get Team из Linear (только при --linear)
  14 — Decompose (пересчитать декомпозицию)
  all — весь цикл с Step 1

Введите номер или «all»:
```

### Поведение при resume

| Шаг | Поведение при resume |
|-----|----------------------|
| Step 1 | Показать существующие ответы MBQ и бизнес-вопросы. Спросить: «Что изменилось?» Пересмотреть только изменённое. |
| Step 2 | Открыть существующий `prd.md`; применять StrReplace (Cursor), replace_file_content (Antigravity) или Edit (Claude Code) только к изменённым секциям. |
| Step 3 | Подтвердить или сменить путь проекта. |
| Step 4 | Перезапустить анализ кода; сравнить с предыдущим `codebase-analysis.md` (вывести diff-резюме). |
| Step 5 | Перепроверить интеграционные точки, legacy patterns и полноту codebase findings. |
| Step 6 | Пересчитать legacy data compatibility: existing data, healing path, фоновая обработка, нужные FR/Change Plan изменения. |
| Step 7 | Повторно сверить format consistency между producer-ами и consumer-ами; обновить FR при необходимости. |
| Step 8 | Повторно сравнить old/new path и зафиксировать возможные регрессии рефакторинга. |
| Step 9 | Показать прежние ответы на техвопросы. Спросить: «Переоткрыть все вопросы, только открытые OQ, или конкретные номера?» |
| Step 10 | Применить новые решения через StrReplace / замену контента файла; не удалять уже закрытые OQ. Обновить/создать секцию `## Diagrams` если изменился поток данных или состав компонентов. |
| Step 11 | Прогнать Self-Review по актуальному `prd.md`, включая чеклист §7e (Diagrams). |
| Step 12 | Прогнать Devil's Advocate Review заново по актуальному `prd.md`. Ранее ACCEPTED правки уже в PRD — не дублировать. Роли A6/A7 проверяют актуальность диаграмм. |
| Step 13 | Только при `--linear`. Перевыбрать команду или использовать прежнюю. |
| Step 14 | Пересчитать только затронутые строки Change Plan; сохранить неизменённые задачи. |

> ⚠️ Resume не создаёт новые файлы и не дублирует `prd.md`. Все изменения — через StrReplace / замену контента в существующем файле.  
> ⚠️ Если feature slug не найден (`docs/<feature-slug>/prd.md` не существует) — ошибка: «PRD не найден. Проверьте slug или запустите /prd-assessment для создания нового.»

---

## Шаги (последовательно)

🛑 Выполняй каждый шаг СТРОГО один за другим. После согласия пользователя — переходи к следующему. Для деталей — загружай (читай) инструкции `instructions/prd-*.md`:
- **Antigravity:** `view_file`
- **Claude Code / Codex:** `Read` для загрузки инструкций, `Task` / subagent для автономных шагов

### Правило оркестрации для Codex

Для **Claude Code / Codex**:

- Основной flow **не должен** самостоятельно выполнять тяжёлые автономные шаги `Step 4`, `Step 11`, `Step 12`, `Step 14`, если их можно делегировать субагенту без потери контекста пользователя.
- Для explicit `/prd-assessment` формулировку выше читать жёстко: эти шаги **обязаны** быть делегированы соответствующим project-scoped custom agents, а не просто "могут быть делегированы".
- Основной flow отвечает за: диалог с пользователем, checkpoint-вопросы, подтверждения переходов, показ пользователю только итогов шага и пунктов `ESCALATE`.
- Субагент отвечает за: чтение контекста, глубокий анализ, правки `prd.md` / `codebase-analysis.md`, подготовку summary для основного flow.
- После завершения субагента основной flow обязан:
  1. кратко сообщить результат шага;
  2. показать только нужные пользователю решения или вопросы;
  3. перейти **строго** к следующему шагу пайплайна.
- **Запрещено:** дублировать работу субагента в основном flow, кроме краткой проверки результата и интеграции checkpoint-решений пользователя.
- **Exit gate:** heavy step не может завершиться, пока основной flow не получил явный результат от обязательного subagent-а.

### Глобальное правило late-findings propagation

Если на любом шаге после Step 4 найден хотя бы один из сигналов:
- новый `main scenario`;
- новый `adjacent path`;
- новый `mixed-use task / queue / worker`;
- новая ambiguity в `source of truth / convergence`;
- новая критичная `docs topology / docs drift` проблема;
- новый execution-surface gap;

то основной flow **не имеет права** просто "дописать пару FR и идти дальше".

Он обязан:
1. определить минимальную точку возврата:
   - Step 5 — если изменилась поверхность системы;
   - Step 9 — если изменилась только техническая конкретизация;
   - Step 10 — если нужен только повторный freeze документа;
2. явно сообщить пользователю, почему происходит rewind;
3. после rewind заново пройти обязательные downstream-шаги.

### Step 1: Business Clarification
**Инструкция:** [prd-business-questions.md](../instructions/prd-business-questions.md)

> **Resume Mode:** Выполнить Step 0 из инструкции (Resume Check) **до всех остальных пунктов**. Если пользователь подтверждает без изменений — перейти к следующему шагу без вопросов.

1. Если нет ввода — промпт: «Опишите фичу или прикрепите файл с черновиком ТЗ.»
2. **Feature Classification** — первый вопрос до всего остального: тип фичи `UI-only / backend / integration / architecture`. Адаптирует глубину Step 4 и лимит техвопросов (см. TZ §3.1a).
3. Parse input; extract entities, goals, scope hints.
4. **Search first:** docs/, ai_docs/, codebase — ищи ответы на потенциальные вопросы.
5. **MBQ-1..4** (обязательно): статус сделки, ответственность за стороны, поведение при ошибке, аудит.
6. **Corner Cases (§6b):** проработать применимые категории.
7. Completeness block; затем остальные вопросы пачкой до 10 с вариантами (a, b, c), формат Nx.
8. **Переход:** если после Step 1 не осталось реальных бизнес-неизвестностей или решений пользователя — автоматически перейти к Step 2. Спрашивать пользователя только при подлинной неоднозначности требований.

### Step 2: Form Business PRD
**Инструкция:** [prd-form-prd.md](../instructions/prd-form-prd.md)

> ⚠️ **Duplicate Guard (выполнять первым):** Перед созданием файла проверить, не существует ли уже PRD для этой фичи в папке `docs/<feature-slug>/`. Если папка найдена — **не создавать новую**, использовать существующую. Если нет — создать новую папку с новым feature-slug.

1. Создай `docs/<feature-slug>/` если нет.
2. Create/Edit `prd.md` (StrReplace/Write/replace_file_content — Cursor/Antigravity; Edit/Write — Claude Code): Goals, Scope, AC, Non-Goals, User Stories.
2a. **PRD Friction Reducers (обязательно, но без догматизма):** при формировании PRD добавь и поддерживай следующие блоки как рабочие секции документа. Это не «истина по умолчанию», а гипотезы структуры, которые нужно наполнять только подтверждёнными данными из Step 1/4/5/6/7/8 или явно помеченным target-state:
   - `## Текущее состояние / Уже сделано / Вне scope` — что уже подтверждено в коде или runtime, что сделано частично, что точно не входит в эту фичу. Цель: не заставлять следующий шаг заново восстанавливать baseline и delta.
   - `## Таблица границ и зон ответственности` — таблица границ компонентов/сервисов/runtime-ролей: кто владеет логикой, кто вызывает, кто хранит state, что запрещено делать соседним компонентам.
   - `## Таблица контрактов` — канонические контракты для API/routes/events/return values/library constraints. Если библиотека или текущий код ограничивают желаемый контракт, это нужно зафиксировать явно.
   - `## Матрица конфигурации` — canonical env vars / settings / ports / healthchecks / timeouts / domain source of truth с dev/prod или local/compose примерами, если фича затрагивает runtime-конфиг.
   - `## Инварианты и негативные сценарии` — что нельзя сломать, какие legacy-сценарии обязаны пережить изменение, какие path'ы запрещены или остаются out of scope.
   - `## Матрица верификации` — какие тесты, smoke/regression checks и non-functional проверки обязаны пройти, чтобы change считался завершённым.
2b. **Принцип заполнения этих блоков:** не копируй туда общие рассуждения. Каждый пункт должен быть либо (a) подтверждён baseline-анализом, либо (b) сформулирован как целевое состояние реализации, либо (c) оформлен как OQ/риск, если данных не хватает.
3. **Язык PRD:** весь документ пишется **исключительно на русском языке**. Все разделы (Goals, FR, User Stories, Non-Goals, Open Questions, Diagrams descriptions) — на русском. Английский допускается только для: имён переменных/функций в коде, технических терминов без русского аналога, названий внешних API/сервисов. Проверка: если ≥10% текста на английском → переписать на русский.
4. Guard rails: Goals — поведение системы (не переговоры); FR — без «или» и «разумный» без диапазона; минимум один negative User Story.
5. Completeness.
6. Если после Step 2 не осталось реальных решений пользователя — автоматически перейти к Step 3. Формальный confirm на переход запрещён.

> ⚠️ Этот шаг может быть повторно вызван из Step 4 при изменении scope.

### Step 3: Project Path
1. Используй `--path` из команды или workspace root.
2. Если `--path` не задан, а workspace root выглядит валидным проектным корнем, принять его автоматически.
3. Спрашивать пользователя о пути только если найдено несколько правдоподобных корней, путь невалиден или есть риск анализировать не тот проект.
4. После выбора или автоподтверждения пути → Step 4.

### Step 4: Codebase Analysis
**Инструкция:** [prd-codebase-analysis.md](../instructions/prd-codebase-analysis.md)

> **Claude Code / Codex:** этот шаг выполнять через обязательный custom subagent `prd_explorer`. Основной flow передаёт субагенту подтверждённый `project path`, текущий `prd.md`, Feature Classification и путь для сохранения `codebase-analysis.md`. После завершения субагента основной flow показывает пользователю только summary анализа и проводит Scope Checkpoint.

Дополнительный ранний контракт для Step 4:
- subagent обязан поднять `Adjacent Scope Check` до поздних шагов;
- если найден same-class sibling flow с общими runtime/pipeline/status semantics, subagent обязан вернуть `decision_required` + `questions_for_user`, а основной flow обязан задать короткий scope-check;
- запрещено фиксировать PRD scope дальше Step 4, если найденный sibling flow ещё не получил явный статус `include` / `exclude` / `defer`.

1. Analyze: structure, package.json/requirements.txt/go.mod, README, entry points. Глубина — по Feature Classification из Step 1.
2. Map: integration points, constraints.
   - **Reuse search (обязательно):** для каждого типа utility-функции, упомянутого в PRD (сравнение строк/текстов, хэширование, парсинг, нормализация, форматирование) — grep по всему проекту на ключевые слова предметной области. Смежные модули (`classifier/`, `common/`, `utils/`, `shared/`) проверять явно, даже если основная работа ведётся в другом модуле. Найденные переиспользуемые утилиты фиксируются в секции `## Prior Art` codebase-analysis.md и должны быть указаны в FR как обязательное переиспользование.
 2a. **Baseline Reality Check (обязательно):** отделить подтверждённый baseline от target-state: что уже существует, что реализовано частично, чего нет. Если PRD называет основной продуктовый path «fallback» или описывает target-state как текущий baseline — вынести это в Scope Checkpoint и затем исправить формулировки.
 2b. **Runtime Docs Check (обязательно для критичных helper/API/status-contract):** проверить docstring, `Doc/`, `docs/`, `ai_docs/` и тесты для ключевых runtime-контрактов. Определить: canonical doc, supporting docs, historical/assessment docs, documentation drift.
 2c. **Boundary / Contract / Config extraction (обязательно по ситуации):** если фича затрагивает несколько компонентов, runtime-процессов, env vars, endpoints, очереди, healthchecks, домены, library adapters или migration существующей логики — собрать факты для будущих секций `Таблица границ и зон ответственности`, `Таблица контрактов`, `Матрица конфигурации`, `Инварианты и негативные сценарии`, `Матрица верификации`. Нельзя оставлять эти секции «на потом» без опоры на codebase-analysis.
 2d. **Adjacent Path Expansion Protocol (обязательно для shared runtime surface):** если найден task, queue, worker, callback path, polling/status path, helper, wrapper или shared entry point, субагент обязан:
   - найти все caller-ы и downstream hop-ы;
   - зафиксировать первый hop отдельно от downstream hop-ов;
   - проверить mirror/sibling flows и mixed-use call sites (`interactive`, `background`, `periodic`, `agent`, `preview`, `healing`);
   - перечислить не только найденные связанные пути, но и явно указать, какие соседние поверхности были проверены и исключены.
   Нельзя завершать Step 4 выводом вида «проблема локализована», если первый hop понятен, но соседние shared-use path'ы ещё не инвентаризированы.
3. **External System Analysis (§3b):** для `integration` — обязательно; для `backend`/`architecture` — если есть внешние зависимости. Изучить публичную документацию API (WebFetch/WebSearch), выяснить: аутентификацию, rate limits, формат данных, sandbox, SDK, SLA. Недоступная информация → OQ с владельцем и дедлайном.
4. Save `docs/<feature-slug>/codebase-analysis.md` (включая секцию `## External Systems`).
5. **Degraded:** invalid path → ошибка, спросить путь или skip (если user подтвердит).
5b. **Implicit Call Contract Audit (обязательно, если Feature добавляет новый код в существующий паттерн с обязательными side-effect вызовами):**
   **Инструкция:** [prd-implicit-call-contracts.md](../instructions/prd-implicit-call-contracts.md)
   Выявить паттерны, где обязательный вызов не защищён интерфейсом, составить таблицу веток, сверить FR на наличие side-effect assertion. Результат — секция `## Implicit Call Contracts` в `codebase-analysis.md`.

   **Сигналы, при которых шаг обязателен (конкретные триггеры):**
   - Feature создаёт/обновляет модель Django, у которой есть публичный метод (`update_relation`, `toggle_*`, `set_*` и т.п.) с side-effect внутри (Celery task, уведомление, кэш-инвалидация, сигнал) — вместо него нельзя использовать `.objects.*` напрямую
   - Feature добавляет новую ветку в pipeline (новый тип отклика, новый источник данных), где уже есть таблица `## Implicit Call Contracts`
   - Feature переносит логику из inline-кода в Celery-задачу или сервис — side-effects могут потеряться

   **Обязательная проверка:** для каждой модели, затрагиваемой Feature, — grep на `.objects.create/update/get_or_create/update_or_create` в новом коде. Если найдено — убедиться, что аналогичный вызов через публичный метод модели недоступен. Если публичный метод есть — использовать его, зафиксировать контракт в `## Implicit Call Contracts`.
6. **Scope Checkpoint:** «Анализ выявил [X]. Это меняет scope, baseline или формулировки Goals/FR? (да / нет)». При «да» → Step 2 (ревизия PRD), затем продолжить в Step 5.
7. **Incident vs Architecture Reframe Checkpoint (обязательно по ситуации):**
   - Если вход пользователя выглядел как локальный инцидент / bugfix / "почему не сработало", а codebase-analysis показал более широкий системный контур, основной flow обязан явно разделить:
     - подтверждённый incident baseline;
     - локальный root cause;
     - что именно теперь обобщается до architecture / platform scope.
   - Формат checkpoint:
     - a) Остаться в incident/fix scope
     - b) Расширить до architecture initiative
   - Если выбран `b`, перед переходом дальше нужно зафиксировать `Incident Baseline Freeze`: симптом, подтверждённая точка сбоя, ближайший локальный fix path и то, какие выводы переносятся в архитектурный scope.

**Контракт выхода для subagent (Step 4):**
- Сохранить полный `codebase-analysis.md`.
- Вернуть основному flow краткое резюме:
  - ключевые integration points;
  - найденные ограничения;
  - prior art / reuse;
  - confirmed baseline vs target-state;
  - main scenario vs edge/fallback;
  - adjacent / mirror / mixed-use surfaces и что было проверено отрицательным поиском;
  - adjacent scope check: `primary_surface`, `adjacent_similar_surfaces`, `shared_runtime_surfaces`, `scope_forks`;
  - canonical/supporting/historical docs и documentation drift;
  - architectural findings / implicit call contracts;
  - что именно нужно вынести в Scope Checkpoint.
- Не задавать пользователю вопросы напрямую. Все вопросы пользователю задаёт основной flow.
- Если найден sibling flow или boundary fork, который способен materially изменить PRD scope, subagent обязан вернуть `decision_required` и `questions_for_user`.
- Основной flow не имеет права закрыть Step 4 без ответа `prd_explorer`.

### Step 5: Completeness Check — Поиск всех интеграционных точек и execution surfaces
**Инструкция:** [prd-completeness-check.md](../instructions/prd-completeness-check.md)

1. Обязательно собрать:
   - `Execution Surface Matrix`
   - `Integration Points — Completeness`
   - `Analytical Debt Register`
2. Для `backend` / `integration` / `architecture` step считается незавершённым, пока не перечислены:
   - main paths;
   - adjacent paths;
   - mixed-use tasks / queues / workers / wrappers;
   - все ещё не закрытые аналитические хвосты.
3. Если найден новый `main scenario`, `adjacent path` или `mixed-use surface` → кандидат на возврат в Step 2.

### Step 6: Legacy Data Compatibility Check — Обработка существующих данных
⚠️ **ПОДКРИТИЧЕСКИЙ ШАГ:** Предотвращает циклические ловушки, когда исцеляющий механизм никогда не запустится для existing bad data.

**Проблема, решаемая этим шагом:**
- Миграция логики: старые данные имеют признаки system A, но новая логика ищет признаки system B → старые данные остаются необработанными
- Фолбэк-механизмы, которые срабатывают только "по требованию" → existing bad data никогда не будет обработана, потому что обращение отсекается на входе
- Пример: резюме с external_id (признак HH), но source='local' → пайплайн не запрашивает его из API → healing task никогда не запускается

**Процесс (применяется, если Feature изменяет логику обработки existing данных):**

1. **Выявить существующую популяцию данных**, которую затрагивает Feature:
   - Есть ли уже в БД записи, которые соответствуют старому формату/статусу, но будут обработаны по новой логике?
   - Пример: миграция `source='local'` → `source='hh'`; есть ли уже в БД резюме с признаками HH (external_id) но source='local'?
   - Query: укажи SQL/ORM-запрос для выявления этой популяции

2. **Проверить точку входа каждого механизма (в т.ч. healing/фонового механизма из Step 5):**
   - Каждый producer/consumer/healing task: где запускается? При каких условиях?
   - Будет ли эта точка входа достигнута для existing bad data?
   - Пример: heal task запускается только когда пайплайн запрашивает резюме (on-demand)
   - Пример: пайплайн отсекает source='local' ДО запроса → heal task никогда не запустится для existing bad data ❌

3. **Если механизм не достанет existing bad data:**
   - Добавить явный фоновый механизм (batch healing, background sync job) **в основной пайплайн** (не on-demand, а в существующий periodic job)
   - Пример: `update_user_resumes_from_hh_sync()` при авторизации/периодически должен исправлять source='local' → source='hh' для резюме с external_id
   - Добавить FR или уточнить существующую FR: «При синхронизации пользователя с [external system] должны исправляться существующие повреждённые записи (source/поля/статусы)»

4. **Результат в codebase-analysis.md:** секция `## Legacy Data Compatibility` с таблицей:
   ```
   | Популяция data | Существует? | Затрагивается feature? | Точка входа healing | Достанет legacy? | Добавить FR? |
   |---|---|---|---|---|---|
   | Resume(external_id=..., source='local', resume_text='') | ✅ | ✅ | get_full_resume_by_external_id_sync (on-demand) | ❌ | ✅ Добавить фоновый healing в sync job |
   ```

**Проверочный вопрос для пользователя (явный):**
«Анализ выявил, что существующая популяция данных [описание] не будет обработана механизмом [healing/sync] в новой логике. Нужно ли добавить фоновый механизм для исцеления existing data, или legacy data можно оставить как есть? (да, добавить FR / нет, оставить как есть)»

При «да» → добавить/уточнить FR в codebase-analysis.

**Проверочный вопрос для пользователя (общий):**
«Анализ выявил [N] producer-ов и [M] consumer-ов для [interface]. Это полный список? Есть ещё парсеры/интеграции/слушатели, которые пишут/читают из того же места? (да / нет / [список])»

При «да» или новый список → обновить codebase-analysis, пересмотреть FR, пересчитать оценки в Phase 2 (Telegram Ingestion, etc.).

**Переход:** после завершения Step 6 перейти в Step 7 с актуализированным списком producer-ов, consumer-ов и legacy data patterns.

### Step 7: Format Consistency — Проверка формата между producer-ами
**Инструкция:** [prd-format-consistency.md](../instructions/prd-format-consistency.md)

1. Для каждого interface с multiple producers сравнить payload contract и consumer expectations.
2. Результат — секция `## Format Contracts` в `codebase-analysis.md`.
3. Все обязательные поля должны быть отражены в FR и тестах до перехода дальше.

### Step 8: Refactoring Regression Check — Сверка «до / после»
**Инструкция:** [prd-regression-check.md](../instructions/prd-regression-check.md)

1. Сравнить old path vs new path.
2. Зафиксировать `## Refactoring Regression Diff`.
3. Все accidental потери должны стать FR / tests / OQ до перехода в Step 9.

### Step 9: Technical Questions
**Инструкция:** [prd-technical-questions.md](../instructions/prd-technical-questions.md)

1. Coverage check: для каждого AC — есть ли техвопрос? Нет → «AC-X: no decisions needed, covered by [file].»
2. **Verify-first rule (обязательно):** если ответ можно получить из кода, тестов, docstring или docs — не задавать вопрос пользователю. Зафиксировать это как `covered by [file/module/doc]` или как подтверждённый baseline.
3. **Source of Truth / Convergence / Symmetry check (обязательно по ситуации):**
   - для каждого изменяемого статуса или состояния, которое живёт в двух системах, зафиксировать source of truth, кто запускает side-effects и как достигается convergence;
   - для каждого UI/API контракта зафиксировать request envelope, success/error response, ID propagation и ответственность за валидацию;
   - для парных или зеркальных действий проверить симметрию по payload, auth, response, side-effects, tests и docs. Незаполненные места → Tier 1 вопросы.
4. **Вывести вопросы в чат** — только если после Verify-first действительно остались решения пользователя. Вопросы должны идти одним `Decision Packet`: Completeness + таблица + варианты (a, b, c). **НЕ писать** в prd.md.
5. Лимит вопросов — по Feature Classification (UI-only ≤3, backend ≤7, integration/architecture ≤10). Tier 1 first. Ответ Nx.
6. **Simplicity check (для architecture/integration):** для каждого архитектурного решения с повышенной сложностью (polling-loop, hot-reload конфига, event bus, distributed lock и т.п.) — явно предложить более простую альтернативу. Формат вопроса: «Вариант A: [сложный паттерн] — [зачем нужен]. Вариант B: [простая альтернатива] — работает при условии [Z], не требует [X]. Рекомендую B, если [условие]. Выберите: a/b.» Пользователь выбирает осознанно; выбор фиксируется в FR.
7. Если пользовательские решения нужны — дождаться ответов и затем автоматически перейти к Step 10. Если все вопросы были закрыты локально, не запрашивать формальное «готово» и переходить дальше автоматически.

> ⚠️ Этот шаг может быть вызван повторно из Step 14 (мини-раунд ≤5 вопросов, только Tier 1).

### Step 10: Final PRD
1. **StrReplace / write_to_file** в том же `prd.md`:
   - Каждое решение из Step 9 — вписать **в тот FR, который оно уточняет** (in-place). Только кросс-платформенные решения → Technical Considerations (опционально, max 3 строки).
   - **FR Grouping (обязательно):** если после Step 9 итоговых FR больше 7, перегруппировать `## Functional Requirements` в логические блоки с подзаголовками. Глобальная нумерация `FR-N` сохраняется, но блоки становятся обязательными.
   - **Acceptance Criteria (обязательно):** заполнить отдельный раздел `## Acceptance Criteria`. Для каждого логического FR-блока или ключевого сценария должен появиться минимум один проверяемый `AC-N`. Запрещено оставлять PRD без критериев приёмки.
   - **Architecture Consolidation (обязательно):** собрать архитектурную narrative-часть в единый раздел `## Architecture & Contracts`. В этом разделе должны жить boundaries, runtime flow, source of truth, execution surface, convergence, ownership, canonical docs и ключевые контракты. Архитектурное объяснение не должно быть размазано по PRD без одного central block.
   - **Current State / Delta Freeze (обязательно):** перед выходом из Step 10 заполнить `## Текущее состояние / Уже сделано / Вне scope`. Явно перечислить: подтверждённый baseline, уже реализованные куски, ожидаемый delta, out-of-scope. Если baseline известен не полностью — отметить это как неизвестность, а не домысливать.
   - **Incident Baseline Freeze (обязательно по ситуации):** если feature выросла из incident / bugfix / runtime anomaly, зафиксировать отдельно: исходный симптом, подтверждённый root cause, локальный incident scope и то, какая часть проблемы была обобщена в architecture / platform decision. Это нужно, чтобы incident-контекст не растворился внутри большого PRD.
   - **Таблица границ и зон ответственности (обязательно для backend/integration/architecture и всех multi-component flow):** зафиксировать по строкам компонент/сервис/runtime-процесс, зона ответственности, owned state, вход/выход, запрещённые зависимости или legacy-path'ы, которые нельзя возвращать. Эта таблица должна жить внутри `## Architecture & Contracts`, а не отдельным разрозненным блоком.
   - **Contract Freeze (обязательно):** перед выходом из Step 10 явно зафиксировать в PRD: request envelope, success/error response, source of truth, кто запускает side-effects, механизм convergence, canonical/supporting/historical docs. Если что-то из этого не подтверждено Step 4 — писать как target-state, а не как существующий baseline. Эти материалы должны быть собраны под `## Architecture & Contracts`.
   - **Execution Surface Freeze (обязательно):** если фича затрагивает async/multi-component surface, зафиксировать в PRD итоговую картину по основным и соседним путям: trigger, call site, task/wrapper, queue/topic, worker/process, urgency, side-effects. Если какой-то adjacent path не меняется, это должно быть сказано явно. Этот freeze должен быть частью `## Architecture & Contracts`, а не разбросан по разным разделам.
   - **Analytical Debt Sweep (обязательно):** перед выходом из Step 10 проверить `Analytical Debt Register`. Каждый хвост должен быть либо закрыт, либо переведён в явный OQ / Risk / Non-Goal. Неформальные хвосты запрещены.
   - **Таблица контрактов (обязательно):** помимо narrative-описания, добавить компактную таблицу по API/routes/events/library boundaries: producer/caller, consumer, request/input shape, success output, error/legacy semantics, library/runtime constraints, canonical source. Для migration существующей логики перечислить readers/writers и что именно меняется. Таблица должна быть частью `## Architecture & Contracts`.
   - **Матрица конфигурации (обязательно, если есть config/runtime surface):** перечислить canonical env vars / settings, deprecated aliases, single source of truth для домена/host/port, healthcheck, таймауты, dev/local/compose примеры и expected defaults. Если фича не затрагивает конфиг — напиши это явно.
   - **Database Changes (обязательно):** заполнить раздел `## Database Changes` на основе технических решений из Step 9 и анализа из Step 4. Перечислить: новые таблицы/модели, добавляемые/удаляемые поля, изменения индексов/constraints, необходимые миграции. Если фича не затрагивает БД — записать явно: «No database changes required.» Раздел **не может быть пустым или содержать placeholder**. **Backfill check:** для каждой новой таблицы-реестра/lookup (служит для поиска или индексации существующих данных) — обязательно добавить FR с `RunPython` бэкфиллом из существующих данных. Симптом пропуска: после деплоя таблица будет пустой, и все lookup-запросы вернут «не найдено».
   - **DB Schema Diagram (обязательно при изменениях БД):** если фича изменяет схему БД — добавить Mermaid ERD-диаграмму (`erDiagram`) в секцию `## Database Changes` (или `## Diagrams`). Диаграмма должна: (1) показывать все изменяемые таблицы с полным списком полей; (2) показывать соседние таблицы, необходимые для понимания связей (FK, триггеры); (3) помечать новые поля `<<NEW>>`, изменяемые ограничения `<<CHANGED>>`; (4) отражать FK-связи между таблицами. Если фича не затрагивает БД — диаграмма не нужна.
   - **Diagrams:** если секция `## Diagrams` содержит placeholder — заменить готовыми Mermaid-диаграммами (правила — в [prd-form-prd.md §3d](../instructions/prd-form-prd.md)). Если секции нет, а Feature Classification = `integration` или `architecture` — создать её.
   - **Инварианты и негативные сценарии (обязательно):** отдельной секцией перечислить инварианты уровня продукта и архитектуры: что запрещено ломать, какие данные нельзя потерять, какие админские/legacy/user paths должны остаться совместимыми, какие упрощения или rollback-path'ы запрещены.
   - **Testing & Documentation (обязательно):** заполнить раздел `## Testing & Documentation Strategy` в согласии со instruction [testing](../instructions/testing.md) и skill [docs](../references/docs.md). Перечислить:
      * **Unit Tests:** какие компоненты требуют unit-тестов, critical paths, edge cases (≥ 5 пунктов). Если тесты не требуются — обоснование.
      * **Integration Tests:** какие компоненты требуют integration-тестов, точки интеграции (если применимо).
      * **Documentation Updates:** какие существующие docs требуют актуализации, какие новые docs нужны (API docs, user guides, runbooks). Обязательно зафиксировать: canonical doc, supporting docs, historical/assessment docs и задачу на contradiction cleanup.
      * **Project test command (обязательно):** явно укажи как запускать тесты в этом проекте (из `.agent/config.json` → `testing.command`, и где лежат тесты: `testing.path`). Это должно попасть и в стратегию, и в Change Plan (отдельной задачей).
   - **Матрица верификации (обязательно):** кроме narrative-стратегии, добавить таблицу: change area / required tests / regression surfaces / non-functional checks / owner. Если есть инварианты, для каждого должен быть указан хотя бы один способ верификации или явная причина, почему автоматическая проверка невозможна.
   - **Section placement check (обязательно):** перед выходом из Step 10 пройтись по PRD и убедиться, что архитектурные объяснения не продублированы длинными narrative-абзацами в `Technical Considerations`, `Database Changes`, `Open Questions`, `Testing & Documentation Strategy`. Эти разделы должны содержать только свой специфичный материал и ссылаться на `## Architecture & Contracts`.
    - Change Plan skeleton: заголовки таблицы (см. [prd-change-plan-template.md](../instructions/prd-change-plan-template.md)).
2. Не создавать prd-v2.md или копии.
3. **OQ Sync:**
   - OQ с ответом → решение вписать прямо в соответствующий FR или TC «по месту», затем **удалить строку OQ** из таблицы
   - Если все OQ закрыты → удалить раздел `## Открытые вопросы` целиком (он не должен остаться в финальном PRD даже пустым)
   - Открытые OQ (подлинно неизвестное) → обязательно: **владелец** (конкретная роль) + **дедлайн** (дата или milestone)
   - OQ без владельца и дедлайна — не допускаются в финальный PRD
   - **«Опциональные» OQ** («OQ-X: порт health endpoint, default: 8080») — **удалить**; default-значение вписать прямо в соответствующий FR
   - **Placeholder sweep (обязательно):** после вписывания всех решений — grep по PRD на паттерны `Приложение:`, `Note:`, `будет определено`, `будет в Step`, `[TBD`, `[PLACEHOLDER`. Каждый найденный placeholder **должен быть заменён** финальным решением прямо в тексте FR/TC. Пустой placeholder без решения — оформить явным OQ с владельцем и дедлайном. Ноль placeholders — обязательное условие выхода из Step 10.

### Step 11: PRD Self-Review
**Инструкция:** [prd-self-review.md](../instructions/prd-self-review.md)

> **Claude Code / Codex:** этот шаг выполнять через обязательный custom subagent `prd_self_reviewer`. Основной flow передаёт субагенту актуальный `prd.md` и `codebase-analysis.md` (если он есть). Субагент сам прогоняет чеклист и вносит допустимые правки. Основной flow показывает пользователю только summary изменений и подключает пользователя лишь при необходимости реального решения.

1. Агент самостоятельно прогоняет PRD по чеклисту guard rails, включая terminology audit и confirmed-vs-target check.
1a. **Friction audit (обязательно):** отдельно проверить, что PRD не заставляет следующего исполнителя восстанавливать контекст из догадок:
   - есть ли явный `Текущее состояние / Уже сделано / Вне scope`;
   - есть ли `Таблица границ и зон ответственности` для всех спорных границ;
   - есть ли `Таблица контрактов` для API/routes/events/return semantics/library constraints;
   - есть ли `Матрица конфигурации`, если фича трогает runtime-конфиг;
   - есть ли `Инварианты и негативные сценарии`;
   - есть ли `Матрица верификации` с тестами и non-functional checks.
1b. Если какой-то из этих блоков не нужен, subagent должен явно подтвердить причину пропуска, а не молча оставить пробел.
2. Выводит найденные проблемы (или «PRD прошёл проверку»).
3. Исправляет найденные проблемы через StrReplace / замену контента файла, сообщает об изменениях.
4. Если subagent не вернул `rewind_required` и не осталось решений пользователя — автоматически перейти к Step 12.
5. Пользователь подключается только если self-review выявил спорное место, которое нельзя закрыть без решения владельца.

**Контракт выхода для subagent (Step 11):**
- Применить все механические и однозначные исправления из чеклиста.
- Вернуть основному flow:
  - список исправленных нарушений;
  - список оставшихся спорных мест, если они требуют решения пользователя;
  - итог: `pass with fixes` или `pass without changes`.
- Не вести диалог с пользователем и не решать за него checkpoint перехода.
- Если обнаружен новый adjacent path / mixed-use surface / main scenario / source-of-truth conflict → вернуть `rewind_required` вместо обычного pass.
- Если найдено спорное место, которое нельзя закрыть ни механической правкой, ни rewind'ом → вернуть `decision_required`.
- При `decision_required` вернуть `questions_for_user` для основного flow.
- Основной flow не имеет права закрыть Step 11 без ответа `prd_self_reviewer`.

### Step 12: Devil's Advocate Review
**Инструкция:** [prd-devils-advocate.md](../instructions/prd-devils-advocate.md)

> **Claude Code / Codex:** этот шаг выполнять через обязательный custom subagent `prd_devils_advocate`. Основной flow передаёт субагенту `prd.md`, `codebase-analysis.md` и Feature Classification. Субагент самостоятельно прогоняет атакующие роли и Защитника PRD, применяет допустимые ACCEPTED-правки и возвращает основному flow только итоговые `ESCALATE`-пункты и summary внесённых правок.

1. Агент последовательно играет до 7 атакующих ролей (Вредный проджект, Скептичный архитектор, Параноидальный безопасник, Капризный пользователь, QA-перфекционист, **Архитектор-визуал**, **Новый член команды**) — по 2–4 атаки каждая. Набор ролей адаптируется по Feature Classification.
   - Для релевантных фич обязательны атаки по осям: **symmetry**, **convergence**, **docs drift**, **main scenario misclassification**.
2. Защитник PRD (Lead) отвечает на каждую атаку со ссылкой на FR/Goal/Risk.
3. Вердикты: DEFENDED (нет действий), ACCEPTED (агент вносит правку в prd.md), ESCALATE (вопрос пользователю с a/b/c).
4. Автоправки: не более 5 за шаг.
5. **Переход:** если нет `ESCALATE` → автоматически перейти к Step 13 (или сразу к Step 14 без `--linear`). Если `ESCALATE` есть — дождаться только содержательных ответов пользователя, затем продолжить без дополнительного confirm.

**Контракт выхода для subagent (Step 12):**
- Вернуть основному flow:
  - список ACCEPTED-правок, уже внесённых в `prd.md`;
  - список DEFENDED-атак, которые не требуют действий;
  - только нерешённые `ESCALATE`-пункты в форме, пригодной для показа пользователю.
- Основной flow показывает пользователю только `ESCALATE` и краткое summary, а не весь внутренний спор ролей.
- Если ACCEPTED/ESCALATE изменили frozen surface, boundaries, docs topology или main scenario — subagent обязан вернуть `freeze_refresh_required` или `rewind_required`.
- Если обнаружена развилка scope / sibling-feature inclusion / boundary choice, которую нельзя честно закрыть локально — вернуть `decision_required` и `questions_for_user`.
- Основной flow не имеет права закрыть Step 12 без ответа `prd_devils_advocate`.

> ⚠️ Этот шаг — содержательный стресс-тест. Step 11 (Self-Review) — формальный чеклист. Оба обязательны для Antigravity и Cursor! Не пропускай его!

### Step 13: Get Team (Linear) [опционально — только при `--linear`]
**Инструкция:** [prd-linear.md](../instructions/prd-linear.md)

> ⚠️ **Этот шаг выполняется только при флаге `--linear`.** Без флага — пропустить, перейти к Step 14.

1. If `--team` — use it.
2. Else call `list_teams`:
   - **Cursor:** `plugin-linear-linear`
   - **Antigravity / Claude Code:** `mcp_linear-mcp-server_list_teams`

   Выведи: «Команды: [names]. Укажите team key или «пропустить».»
3. If team chosen → `list_users(team)`, `list_cycles(teamId, type="previous")` для velocity последнего завершённого цикла. Сохранить: номер цикла, completedScopeHistory[-1] (финальный velocity), длину спринта в днях. Эти данные используются в Step 14 для перевода pts → календарное время без коэффициентов накладных расходов.
4. **Degraded:** Linear fail → proceed without team/velocity.

### Step 14: Decompose
**Инструкция:** [prd-decompose.md](../instructions/prd-decompose.md)

> **Claude Code / Codex:** этот шаг выполнять через обязательный custom subagent `prd_decomposer`. Основной flow передаёт субагенту финальный `prd.md`, `codebase-analysis.md`, данные о team/velocity (если есть) и просит заполнить Change Plan в `prd.md`. После завершения субагента основной flow показывает пользователю summary декомпозиции и подключает его только если реально возникли новые решения пользователя.

1. Define phases (0: Prep, 1: Core, 2: Integration, 3: Tests …).
2. Break into tasks; CP-{phase}.{n}, Depends, Responsible.
   - В Phase 0 обязательно проверить, не нужны ли задачи на baseline capture (скриншоты, smoke, фиксация текущего поведения) до начала реализации.
   - Если фича меняет runtime-контракт, API/UI flow или критичный helper, Change Plan обязан включать docs audit, canonical doc update и contradiction cleanup между `Doc/`, `docs/` и `ai_docs/`.
   - **No-stub rule:** задача не может завершаться stub-реализацией («заглушка для следующей фазы», «TODO: wire up»). Если что-то нельзя реализовать сейчас — это отдельная задача с явной зависимостью, а не placeholder внутри другой.
   - **Cross-file consistency:** для задач, добавляющих новый тип/функцию в пакет — явно указать в описании, что нужно проверить отсутствие дублирующих определений в том же пакете/модуле (grep перед написанием).
3. **OQ Blocking Check:** задачи с зависимостью от открытых OQ → `[BLOCKED: OQ-X]`.
4. Fill Change Plan table: phase headers, task rows.
5. Записать таблицу в prd.md через Edit/StrReplace/replace_file_content.
6. **Mini-loop decision rule:** если декомпозиция выявила новые технические неизвестные, субагент сам классифицирует их:
   - `Step 9 mini` — если это реальные unresolved decisions пользователя;
   - `Step 5 rewind` — если найден новый adjacent path / mixed-use surface / main scenario;
   - `local close` — если неизвестных нет или они закрываются локально.
   Формальный вопрос пользователю «есть ли новые вопросы?» запрещён.
7. **Переход:** при отсутствии новых решений пользователя → Завершение.

**Контракт выхода для subagent (Step 14):**
- Заполнить Change Plan в `prd.md`.
- Вернуть основному flow:
  - краткую сводку фаз;
  - список критических зависимостей / блокеров;
  - список новых технических неизвестных, если они появились.
- Если найдены новые технические неизвестные, субагент **не задаёт** вопросы пользователю сам, а передаёт их основному flow уже классифицированными: `Step 9 mini`, `Step 5 rewind` или `local close`.
- Если найдена реальная scope / boundary развилка, которая требует решения пользователя, вернуть `decision_required` и `questions_for_user`.
- Основной flow не имеет права закрыть Step 14 без ответа `prd_decomposer`.


---

## Degraded Mode (§11)

| Ситуация | Действие |
|----------|----------|
| No input | Промпт; не продолжать |
| Invalid project path | Ошибка; спросить путь или skip |
| Linear MCP fails | Proceed без team; estimate = expert |
| Codebase analysis fails | Proceed с пустым Technical Appendix; flag в PRD |
| «стоп» / «отмена» | Stop; save state |

**Critical stop:** No input; path not found и user не подтвердил skip.

---

## Intellectual Opponent (§2)

Во всех шагах: не принимать выводы пользователя за истину. Анализировать аргументы, искать противоречия. Варианты a/b/c с обоснованием.

> **Devil's Advocate:** в Step 12 механизм Intellectual Opponent усилен ролевой симуляцией стейкхолдеров — агент явно играет атакующие роли и роль Защитника PRD.

---

## Результат и Остановка (Guardrail)

🛑 **КРИТИЧЕСКОЕ ПРАВИЛО:** После завершения последнего активного шага пайплайна `prd-assessment` (Step 14) агент **ОБЯЗАН ПОЛНОСТЬЮ ОСТАНОВИТЬСЯ**.
Тебе строго **ЗАПРЕЩЕНО**:
- Писать код или редактировать проектные файлы (например, `schema.prisma`).
- Создавать любые планы реализации (`implementation_plan.md`).
- Выполнять терминальные команды (миграции БД, деплой и т.д.).
- Самовольно начинать фазу реализации без явной команды пользователя.

Верни пользователю **ТОЛЬКО** следующий текст:
- Ссылку на финальный PRD: `docs/<feature-slug>/prd.md`
- Предложение: «Assessment завершён. Для перехода к реализации запустите:
  передайте путь к PRD в отдельной команде на реализацию»

Никаких других действий не предпринимать до явной новой команды.
