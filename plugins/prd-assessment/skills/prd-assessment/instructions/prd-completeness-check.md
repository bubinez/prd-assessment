---
name: prd-completeness-check
description: Step 5 of /prd-assessment. Completeness Check — inventory all integration points, adjacent flows, execution surfaces, and unresolved analytical gaps before moving to technical questions.
---

# PRD Assessment — Step 5: Completeness Check

При выполнении Step 5 пайплайна `/prd-assessment` — следуй этой инструкции.

## Цель

Не допустить ситуацию, когда PRD уже начал "замерзать", а соседние пути, mixed-use задачи, очереди, воркеры, callback-сценарии или фоновые вызовы всплывают по одному слишком поздно.

Step 5 — это не "ещё один обзор". Это обязательный инвентаризационный шлюз перед Legacy / Format / Regression / Technical Questions.

---

## Input

- Актуальный `prd.md`
- Актуальный `codebase-analysis.md`
- Feature Classification

---

## Обязательные артефакты шага

### 1. Execution Surface Matrix

Для каждого main path и каждого соседнего user-facing или runtime-significant path собрать таблицу:

| Trigger | Сценарий | Call site / entry point | Task / wrapper | Queue / topic / route | Worker / process | Urgency | Side-effects | Docs / tests owner |
|---------|----------|-------------------------|----------------|-----------------------|------------------|---------|--------------|--------------------|

Правила:
- Покрыть не только "основной happy path", но и adjacent flows, которые используют тот же runtime surface.
- Если одна и та же задача может вызываться и срочно, и фоново — это должны быть **две отдельные строки**.
- Если queue/worker/process не подтверждены локально — пометить как gap, а не домысливать.
- Для `integration` / `architecture` отсутствие этой матрицы = шаг не завершён.

### 2. Analytical Debt Register

Собрать все ещё не закрытые аналитические хвосты в отдельную таблицу:

| ID | Хвост / неизвестность | Почему важно | Блокирует шаг? | Должен быть закрыт к шагу | Как закрывается |
|----|-----------------------|--------------|----------------|---------------------------|-----------------|

Правила:
- Запрещены неформальные хвосты в стиле "если хотите, потом посмотрю".
- Каждый найденный хвост должен стать либо:
  - блокером текущего шага,
  - явным входом в следующий шаг,
  - либо осознанным `OQ` / `Risk`.

### 3. Integration Points — Completeness

Для каждого внешнего интерфейса / очереди / API / callback surface:
- найти всех producer-ов;
- найти всех consumer-ов;
- найти все трансформации данных на пути;
- найти все adjacent flows, использующие тот же task / queue / helper / wrapper.

Результат — секция `## Integration Points — Completeness` в `codebase-analysis.md`.

### 4. Adjacent Flow Audit

Обязательный для `backend`, `integration`, `architecture`, если feature трогает:
- очереди,
- Celery / background jobs,
- callbacks / bot flows,
- mixed-use helpers,
- shared tasks / wrappers,
- source-of-truth state в нескольких местах.

Нужно ответить:
- Какие соседние сценарии используют тот же runtime surface?
- Какие из них пользователь воспринимает как UX-critical?
- Какие из них идут по "тому же механизму, но другой срочности"?
- Что именно сейчас **ошибочно выглядит вне scope**, хотя фактически зависит от тех же задач / очередей / воркеров?

### 5. Negative Search Ledger

Шаг не считается завершённым, пока агент не покажет не только то, что найдено, но и что **специально искалось и не нашлось**.

Минимальный формат:

| Что проверяли | Как проверяли | Что ожидали найти | Результат |
|---------------|---------------|-------------------|-----------|

Примеры:
- mirror-flow для того же task;
- sibling callback/update path;
- agent/onboarding/preview call site;
- отдельный first hop перед dedicated worker;
- альтернативный writer/consumer того же state.

---

## Процесс

### 1. Inventory-first

Для каждого интерфейса / queue / callback / endpoint:
- grep по writer-ам;
- grep по reader-ам;
- grep по wrapper/task names;
- grep по queue / route / topic / endpoint strings;
- зафиксировать все найденные call sites.

### 1a. First-Hop Rule

Если user-facing или UX-critical цепочка имеет несколько hop-ов:
- агент обязан явно отделить **first hop** от downstream hop-ов;
- наличие dedicated queue/worker только на downstream hop **не считается достаточной изоляцией**;
- шаг нельзя закрыть, пока не подтверждено, где именно может происходить starvation до входа в dedicated contour.

### 1b. Mixed-Use Search

Для каждого найденного task / helper / wrapper / endpoint агент обязан проверить, не используется ли он в разных product intents:
- `interactive`
- `background`
- `periodic`
- `agent`
- `preview`
- `healing / sync`

Если да — это должны быть отдельные строки в `Execution Surface Matrix`, а не одна общая формулировка.

### 1c. Mirror / Sibling Audit

Если найден конкретный path, агент обязан проверить:
- mirror action (`prepare/send`, `like/dislike`, `create/delete`, `preview/auto`);
- sibling path, использующий тот же queue / worker / helper;
- polling / callback / status-update path вокруг того же сценария.

Отсутствие находок допустимо, но должно быть записано в `Negative Search Ledger`.

### 2. Surface grouping

Сгруппировать строки не только по "типу задачи", а по общей runtime-поверхности:
- shared queue
- shared worker
- shared helper
- shared callback/update path
- shared config source of truth

### 3. Urgency split

Если один и тот же механизм обслуживает разные product intents:
- interactive / UX-critical
- background / periodic
- migration / healing

то это должно быть зафиксировано явно.

### 4. Completeness gate

Перед завершением шага агент обязан явно ответить:
- все ли main paths перечислены?
- все ли adjacent paths перечислены?
- есть ли mixed-use task / helper / queue?
- есть ли ещё не закрытые аналитические хвосты?
- перечислены ли first hop и downstream hop для каждой multi-hop цепочки?
- предъявлен ли `Negative Search Ledger` по соседним и mirror-путям?

Если хотя бы на один вопрос ответ "не уверен" — шаг нельзя считать завершённым без записи в `Analytical Debt Register`.

---

## Output

Сохранить в `codebase-analysis.md`:

```markdown
## Execution Surface Matrix
[таблица]

## Integration Points — Completeness
[producer/consumer graph, adjacent flows, mixed-use surfaces]

## Negative Search Ledger
[что проверили и исключили]

## Analytical Debt Register
[таблица]
```

---

## Scope / Rewind Checkpoint

После выполнения шага вывести checkpoint **только если** действительно требуется решение о расширении scope. Если картина полна и новых scope-level решений нет — автоматически переходить в Step 6.

```markdown
## Completeness Checkpoint (Step 5)

Проверка полноты выявила:
- [новые adjacent flows / mixed-use surfaces / analytical debts]

Это требует:
- a) Вернуться в Step 2 и расширить / переписать Goals / FR
- b) Продолжить в Step 6, потому что картина уже достаточно полна
```

Правило:
- новый `main scenario`;
- новый `adjacent path`;
- новый `mixed-use task`;
- новый runtime source-of-truth conflict

= автоматический кандидат на возврат в Step 2.

---

## Guard Rails

- Не считать шаг завершённым только потому, что "основной сценарий уже понятен".
- Не прятать соседние сценарии как "вне scope", если они идут через тот же runtime surface.
- Не задавать пользователю вопросы о том, что можно проверить grep/read локально.
- Если feature затрагивает shared async surface, но `Execution Surface Matrix` не заполнена — переход к Step 6 запрещён.
- Если найден dedicated worker только на downstream hop, а first hop не проверен отдельно — шаг не завершён.
- Если отсутствует `Negative Search Ledger`, шаг не завершён.

## Полнота без расширения scope
Инвентаризация смежных путей нужна для выявления влияния и регрессий, а не для автоматического добавления новых функций. Каждую предложенную обработку оцени по конкретной вероятности и последствию; при отсутствии существенного влияния зафиксируй неприменимость или ограничение, а не создавай новый FR, блокер или подсистему. Следуй явному пользовательскому scope и проектным правилам.
