---
name: prd-format-consistency
description: Step 7 of /prd-assessment. Format Consistency — compare producer payloads and consumer expectations across shared interfaces.
---

# PRD Assessment — Step 7: Format Consistency

При выполнении Step 7 пайплайна `/prd-assessment` — следуй этой инструкции.

## Цель

Поймать format drift между несколькими producer-ами одного интерфейса и закрепить единый контракт до финализации PRD.

---

## Input

- `codebase-analysis.md` после Step 5/6
- `Execution Surface Matrix`
- `Integration Points — Completeness`

---

## Процесс

Для каждого interface с multiple producers или shared consumer:

1. Прочитать serialization-код каждого producer-а.
2. Выписать поля, типы, required/optional.
3. Сравнить с consumer expectation.
4. Проверить:
   - отсутствующие mandatory fields;
   - producer-specific optional fields;
   - silent drops;
   - несовместимые naming/typing conventions.
5. Зафиксировать единый mandatory contract и parser-specific extensions.

---

## Output

Сохранить в `codebase-analysis.md` секцию:

```markdown
## Format Contracts

| Interface | Producer | Mandatory fields | Optional fields | Consumer expectation | Gap? |
|-----------|----------|------------------|-----------------|---------------------|------|
```

Если есть расхождения:
- обновить FR;
- добавить тестовые требования на формат каждого producer-а.

---

## Переход

После завершения Step 7 перейти в Step 8.
