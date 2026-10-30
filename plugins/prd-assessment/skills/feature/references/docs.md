---
name: docs
description: Генерация документации. Use when creating README, docstrings, API docs, or onboarding materials.
---

# Docs

Стандарты и шаблоны для документации.

## Docstrings / JSDoc

```
/**
 * Краткое описание.
 * @param {Type} name - описание
 * @returns {Type} описание
 * @example
 * fn(arg) // => result
 */
```

## README-секции

- **Установка** — как установить и запустить
- **Использование** — основные сценарии
- **API** — ссылка на детальную документацию или краткий обзор

## Где документировать по workflow

| Workflow | Scope documenter | Пути |
|----------|------------------|------|
| **scaffold, implement** | docstrings, README-секция, API-описание | Локально в коде и проекте. Не создавать `ai_docs/`. |
| **feature** | План, итоговый отчёт | План → `ai_docs/develop/plans`, отчёт → `ai_docs/develop/reports` (из config.json). |

## Итоговый отчёт (workflow-feature)

- Что реализовано (список компонентов/модулей)
- Структура (где что лежит)
- Как использовать (примеры)
- Что тестировать (ключевые сценарии)
