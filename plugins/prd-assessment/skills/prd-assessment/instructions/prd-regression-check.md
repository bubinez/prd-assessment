---
name: prd-regression-check
description: Step 8 of /prd-assessment. Refactoring Regression Check — compare old and new paths to preserve required side-effects and contracts.
---

# PRD Assessment — Step 8: Refactoring Regression Check

При выполнении Step 8 пайплайна `/prd-assessment` — следуй этой инструкции.

## Цель

Не допустить, чтобы новый путь реализовал только "главную идею", но потерял поля, side-effects, сигналы, вызовы, кэш-инвалидацию или старые обязательные побочные действия.

---

## Input

- `prd.md`
- `codebase-analysis.md`
- `Execution Surface Matrix`

---

## Когда шаг обязателен

- Feature Classification = `architecture`
- PRD описывает замену старого механизма новым
- Логика переносится между модулями / процессами / задачами
- Inline-path заменяется task/service/wrapper path
- Shared helper / shared task получает новый routing / urgency split

---

## Процесс

1. Найти old path.
2. Найти proposed/new path.
3. Построить diff:
   - поля;
   - side-effects;
   - callbacks;
   - queue/worker routing;
   - logging / audit;
   - docs / tests expectations.
4. Для каждой потери классифицировать:
   - intentional;
   - accidental;
   - unknown.
5. Все accidental / unknown потери должны стать:
   - FR;
   - regression test;
   - или OQ с владельцем и дедлайном.

---

## Output

Сохранить в `codebase-analysis.md` секцию:

```markdown
## Refactoring Regression Diff

| Surface | Старый путь | Новый путь | Потеря? | Намеренно? | Что нужно добавить |
|---------|-------------|------------|---------|------------|--------------------|
```

---

## Переход

После завершения Step 8 перейти в Step 9.
