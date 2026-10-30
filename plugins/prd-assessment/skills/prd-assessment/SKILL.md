---
name: prd-assessment
description: >
  Формирует PRD и проводит декомпозицию фичи с уточнением требований,
  анализом кодовой базы и подготовкой change plan. Use when user asks to
  create a PRD, decompose a feature, turn a draft feature spec into a PRD,
  break a feature into implementation tasks, invokes /prd-assessment or
  /create-prd, says "декомпозируй фичу", "сформируй PRD", "создай PRD",
  "разбей фичу на задачи", or attaches a draft spec for decomposition. Do
  NOT use for simple
  brainstorming, isolated code review, one-off implementation questions, or
  generic architecture advice without a PRD or decomposition request.
---

# PRD Assessment

Когда пользователь запрашивает **формирование PRD**, **декомпозицию фичи** или **уточнение ТЗ**, запусти пайплайн `/prd-assessment` и веди пользователя по нему последовательно.

Этот skill — только тонкий маршрутизатор.
Полное поведение, порядок шагов, resume-логика, checkpoint-ы и результат находятся в [prd-assessment.md](workflows/prd-assessment.md).
Если между skill и workflow есть расхождение — всегда следуй workflow.

## Триггеры

- `/prd-assessment [описание]`
- `/create-prd [описание]` — **тот же пайплайн**, не прямой черновик
- «Декомпозируй фичу …»
- «Сформируй PRD для …», «Создай PRD для …»
- «Разбей фичу на задачи …»
- Прикреплён файл с черновиком ТЗ и запрос на декомпозицию

## Не использовать

- Для свободного брейншторминга без запроса на PRD или декомпозицию
- Для точечного code review
- Для разового вопроса по реализации или архитектуре без подготовки PRD
- Для общих продуктовых обсуждений без требования оформить итоговый артефакт

## Как работать

- Следуй [prd-assessment.md](workflows/prd-assessment.md) в каталоге этого скилла.
- Если пользователь вызывает `/create-prd`, используй тот же пайплайн, что и для `/prd-assessment`.
- Если пользователь приложил черновик ТЗ, используй его как вход в пайплайн, а не как готовый финальный PRD.
- **Без fallback-поведения в продуктовых требованиях:** если выбранный основной путь недоступен, ошибочен или не покрывает данные, PRD должен описывать видимую ошибку/partial/unsupported и блокировку зависимой операции с сохранением последнего подтверждённого состояния. Не добавляй автоматическую подмену другим API, источником, connector, фильтром или полным перечитыванием. Альтернативу можно исследовать отдельно, но включать её как автоматический fallback нельзя.
- Если пользователь **явно** вызвал `/prd-assessment` или `/create-prd`, запускай subagent-backed вариант пайплайна:
  - `Step 4` через `prd_explorer`
  - `Step 11` через `prd_self_reviewer`
  - `Step 12` через `prd_devils_advocate`
  - `Step 14` через `prd_decomposer`
- Не считай эти шаги завершёнными без результата соответствующего subagent-а.

## Результат

Финальный результат определяется workflow-файлом, а не этим skill.
