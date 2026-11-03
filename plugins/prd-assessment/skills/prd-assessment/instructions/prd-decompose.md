---
name: prd-decompose
description: Step 14 of /prd-assessment. Decompose into phases/tasks, fill Change Plan table, StrReplace in prd.md. Use when executing Step 14 of prd-assessment workflow.
---

# PRD Assessment — Step 14: Decompose

При выполнении Step 14 пайплайна /prd-assessment — следуй этой инструкции.

**Reference:** [prd-change-plan-template.md](prd-change-plan-template.md)

---

## Input
- Final PRD (prd.md)
- Team (list_users) — для Responsible
- Codebase analysis

---

## Действия

### 1. Define Phases
- **0:** Preparation (baseline, screenshots, smoke tests)
- **1:** Core (main feature implementation)
- **2:** Integration (API, UI wiring)
- **3:** Tests (unit, e2e)
- Дополнительные по необходимости.

Обязательные проверки при разбиении на фазы:
- Если фича меняет UI, API или важный runtime-контракт — в **Phase 0** должна появиться задача на baseline capture (скриншоты, smoke, фиксация текущего поведения).
- Если фича меняет runtime docs или source-of-truth контракт — в **Phase 3** должны появиться задачи на docs audit, canonical doc update и contradiction cleanup.

### 1c. Integration Support Phase — обязательно при Feature Classification = `integration` с external party

Если фича типа `integration` и хотя бы одна сторона реализации выполняется **внешней командой** (клиент, партнёр, другой отдел):

1. Добавить отдельную **Phase S: Integration Support** после основных фаз.
2. Обязательный минимум задач:

| ID | Задача | Когда включать |
|----|--------|----------------|
| CP-S.1 | Integration support: ответы на вопросы внешней команды, review их реализации | Всегда при external party |
| CP-S.2 | Joint E2E testing — совместный тест с внешней командой (обе стороны) | Всегда при joint acceptance |
| CP-S.3 | Bug-fix buffer: исправление дефектов, выявленных внешней командой в нашем API | Всегда при external party |

3. **Когда выполняется:** Phase S начинается, когда внешняя команда приступает к своей части (зависит от CP-1.x или аналогичных задач Core-фазы нашей стороны).
4. **Ответственный:** тот же разработчик, что реализовал Core-часть.
5. **Depends:** все задачи Phase S зависят от последней задачи Core-фазы нашей стороны.

### 1b. Design Tasks — обязательно при наличии UI-работы

Если Feature Classification = `UI-only` **или** фича включает frontend-компоненты (любой тип с UI):

1. Добавить дизайн-задачи в **Phase 0** (Preparation) до всех задач разработки.
2. Обязательный минимум дизайн-задач:

| ID | Задача | Когда включать |
|----|--------|----------------|
| CP-0.x | Wireframes / UX flow | Всегда при UI |
| CP-0.x | UI mockups / high-fidelity design | Если нет готового макета |
| CP-0.x | Design review & approval | Всегда при UI |

3. **Responsible:** Designer (или «TBD» если нет в команде).
4. Все задачи разработки UI (Phase 1+) **должны зависеть** от завершения дизайн-задач — указать в `Depends: CP-0.x`.
5. Если макеты уже готовы (упомянуты в PRD / Design Considerations) — включить только задачу «Design review & approval».

### 2. Break into Tasks
- Assign IDs: `CP-{phase}.{n}` (например CP-0.1, CP-1.2).
- Depends: comma-separated CP-IDs; `—` если нет.
- **Правило декомпозиции:** если задача оказывается слишком обширной — обязательно декомпозировать на подзадачи. 
- **Правило backfill-задач:** если задача включает backfill или data migration существующих строк — уточнить в описании задачи: (a) ожидаемый объём строк (из codebase-analysis.md или PRD); (b) метод: `INSERT ... SELECT` / `UPDATE ... FROM` на стороне БД vs Python ORM итерация. При объёме >50k строк метод ORM-итерации недопустим без обоснования. При объёме >500k строк — добавить отдельную подзадачу на проверку порядка операций миграции (триггеры, индексы, CHECK constraints создаются ПОСЛЕ массовых операций).
- **Правило порядка операций в миграции:** если задача включает одновременно (a) создание триггера/constraint и (b) массовый UPDATE/INSERT на той же таблице — явно зафиксировать в описании задачи порядок: массовые операции идут ДО создания триггеров и ограничений.
- **Правило docs end-state:** если PRD меняет API/UI flow, source-of-truth или важный helper-контракт, план обязан содержать не абстрактное `update docs`, а как минимум три отдельные задачи: (1) docs audit; (2) update canonical doc; (3) contradiction cleanup для supporting/historical docs.
- **Правило symmetry:** если фича содержит парные действия, в задачах должно быть видно, что обе стороны контракта обновляются согласованно (backend, frontend, tests, docs), а не только «одна кнопка/один endpoint».
- **Правило traceability:** каждая строка `Execution Surface Matrix`, которая требует изменения, должна иметь хотя бы одну задачу в Change Plan. Если строка матрицы не покрыта задачей, шаг не завершён.

### 3. Change Plan Table Structure

**Phase-строки:** Phase N, ID `—`, Task = цель фазы (bold).

**Task-строки:** Phase, ID `CP-{phase}.{n}`, Task, Responsible, Depends.

### 3b. OQ Blocking Check

Проверить все Open Questions в prd.md:

1. Для каждой задачи — есть ли OQ, ответ на который необходим для её реализации?
2. Если да:
   - Добавить в колонку `Depends`: `[BLOCKED: OQ-X]`

### 4. Responsible
- Из `list_users(team)` — displayName.
- Если нет team — «TBD».

### 5. Edit-in-Place
- **StrReplace** таблицу Change Plan в том же `prd.md`.
- **Запрещено:** prd-v2.md, копии.

### 6. Language
- PRD, Change Plan и итоговое сообщение всегда пишутся **исключительно на русском языке**. Английский допускается только для имён файлов, кода, переменных, функций, внешних API и технических терминов без корректного русского аналога. Флаг `--lang` не изменяет это правило.

### 7. Mini-loop Decision Rule (→ Step 9 / при необходимости → Step 5)

После заполнения Change Plan субагент **сам** принимает решение о mini-loop, а не задаёт пользователю формальный вопрос.

Он обязан выдать один из четырёх статусов:
- `step9_mini_required`
- `step5_rewind_required`
- `local_close`
- `decision_required`

**При `step9_mini_required`:**
1. Выполнить Step 9 в режиме мини-раунда: **только Tier 1**, **≤5 вопросов**.
2. Получить ответы.
3. Вписать решения в соответствующие FR in-place (StrReplace в prd.md).
4. Вернуться сюда: пересмотреть задачи.
5. Если после мини-раунда выявлены ещё неизвестные — повторить не более **одного** дополнительного раза (максимум 2 мини-раунда суммарно).

**При `step5_rewind_required`:**
1. Вернуться в Step 5.
2. Обновить `Execution Surface Matrix` и `Analytical Debt Register`.
3. Затем заново пройти Step 9 → Step 10 → Step 11, если изменилась поверхность системы или contract freeze.
4. После этого вернуться к декомпозиции.

**При `local_close`:** перейти к финальному сообщению.

**При `decision_required`:**
1. Не задавать вопрос пользователю напрямую из subagent.
2. Вернуть в основной flow `questions_for_user`:
   - `observed`
   - `why_it_matters`
   - `question`
   - `options`
3. Использовать этот статус только для настоящих scope/boundary fork, а не для фактов, которые можно проверить локально.

Формальный вопрос пользователю «есть ли новые вопросы?» запрещён. Пользователь подключается только если реально нужны ответы на мини-пакет решений Step 9.

### 8. Final Message
«Assessment завершён. PRD: docs/<feature-slug>/prd.md. Change Plan заполнен. Для реализации дайте отдельную команду.»

---

## Notes
1. Phase headers first.
2. Чекбокс `[ ]` только для task-строк.
3. Метаданные: Feature, Team — обнови в prd.md.

## Соразмерная декомпозиция
Группируй небольшие связанные действия в проверяемые задачи; не вводи отдельную фазу или блокирующий эксперимент только ради шаблона. Для каждого блокера укажи конкретную неизвестность и зависимость; подтверждённые факты не перепроверяй без причины. Сохраняй обязательные проектные проверки, тесты и документацию, необходимые выбранному scope. План не должен добавлять новые состояния, абстракции или продуктовые требования.
