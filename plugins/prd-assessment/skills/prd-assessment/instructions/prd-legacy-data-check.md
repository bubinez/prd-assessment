# Step 6: Legacy Data Compatibility Check

**Цель:** Выявить, есть ли в БД существующие данные, которые соответствуют старому формату/статусу, но новая логика не сможет их обработать или оставит в неконсистентном состоянии.

**Применяется когда:**
- Feature меняет логику обработки данных (миграция формата, введение нового статуса, изменение фильтрации)
- Feature вводит фоновый healing/sync механизм (но он может быть on-demand и никогда не запуститься для existing data)
- Feature меняет точку входа для обработки (например, раньше обрабатывались все резюме, теперь только `source='hh'`, но есть старые `source='local'` с признаками HH)

---

## Процесс

### 1. Выявить существующую популяцию данных

**Вопрос:** Есть ли уже в БД записи, которые соответствуют старому формату/статусу, но будут затронуты новой логикой?

**Метод:**
- Прочитать миграции и коммиты, связанные с затронутой таблицей (см. git log, git blame)
- Спросить у пользователя/код-ревьюера: когда эти данные попали в БД? Есть ли массовое обновление?
- Написать SQL/ORM-запрос для выявления: сколько записей соответствует паттерну старого формата?
- Пример:
  ```sql
  SELECT COUNT(*) FROM Resume 
  WHERE external_id IS NOT NULL AND external_id != '' 
    AND source = 'local';  -- Есть признаки HH, но source='local'
  ```

**Результат:** конкретное число или подтверждение "популяция отсутствует".

---

### 2. Проверить, будет ли новая логика обработана для existing data

**Вопрос:** Каждый механизм (producer, consumer, healing task, background sync) — где запускается? Будет ли он достигнут для existing bad data?

**Метод:**

Для каждого механизма из Step 5 (list of producers/consumers/healing tasks):

| Механизм | Где запускается? | Триггер | Будет ли вызван для existing data? |
|----------|-----------------|---------|-----|
| `heal_hh_resume_source_task` | `get_full_resume_by_external_id_sync` | Пайплайн запрашивает резюме (on-demand) | ❌ Нет — пайплайн отсекает `source='local'` на входе |
| `update_user_resumes_from_hh_sync` | `hh_callback` + periodic job | Авторизация + schedule | ✅ Да — запустится при авторизации |
| Фоновый batch job (если его нет) | `—` | `—` | ❌ Нет |

**Критерий:** Если механизм на-demand (like `heal_hh_resume_source_task.delay()`), проверь, вызывается ли он когда пользователь обращается к пайплайну, ИЛИ добавь явный фоновый вызов (в periodic job).

**Результат:** для каждого механизма — ✅ или ❌.

---

### 3. Если механизм не достанет existing bad data

**Действие:** Добавить явный фоновый механизм или уточнить existing sync job.

**Варианты:**

1. **Расширить existing periodic job** (рекомендуется):
   - Найти background sync job, который уже работает периодически (например, `update_user_resumes_from_hh_sync`)
   - Добавить в него логику исцеления existing data
   - Пример FR: «При синхронизации пользователя с HH (авторизация или periodic job) должны исправляться существующие резюме с external_id но source='local' → переставить source='hh'»

2. **Добавить новую periodic/batch задачу** (если existing job не подходит):
   - Отдельная Celery задача, запускающаяся 1 раз в день / неделю
   - Обрабатывает все existing bad data в батчах
   - Пример: `heal_legacy_resumes_batch_task.delay()` → обновляет все Resume(external_id!=null, source='local')

3. **Добавить миграцию Django** (только если данные стабильны и не меняются):
   - `RunPython` миграция, которая одноразово исправляет все existing data
   - Пример: `UPDATE Resume SET source='hh' WHERE external_id IS NOT NULL AND external_id != '' AND source='local'`
   - Применяется только если: existing data стабильны и не будут добавляться в старом формате

**Выбор:**
- Если data будут добавляться и после миграции (например, парсеры периодически добавляют резюме) → выбрать вариант 1 или 2 (фоновый механизм)
- Если это была одноразовая миграция (например, миграция в S3 произошла в определённый момент, и после этого новые данные уже в правильном формате) → вариант 3

---

### 4. Сформировать результат в codebase-analysis.md

**Раздел:** `## Legacy Data Compatibility`

**Формат:**

```markdown
## Legacy Data Compatibility

### Identified Populations

| Популяция | Query | Count | Затронута feature? | Healing механизм | Достанет legacy? | Action |
|-----------|-------|-------|---|---|---|---|
| Resume(external_id!=null, source='local') | `SELECT COUNT(*) FROM Resume WHERE external_id IS NOT NULL AND source='local'` | 47 | ✅ Да | on-demand: `get_full_resume_by_external_id_sync` | ❌ Нет | Расширить `update_user_resumes_from_hh_sync` добавить логику исправления source |

### Mitigation Strategy

1. Расширить `update_user_resumes_from_hh_sync(user_id)` чтобы при синхронизации она переставляла source='hh' для резюме с external_id.
2. Добавить FR: «При синхронизации пользователя с HH должны исправляться existing Resume с external_id но source='local'».
3. При развёртывании: один раз запустить sync для всех пользователей (management command).
```

---

### 5. Явный проверочный вопрос для пользователя

**Вопрос:**
```
Анализ выявил, что существующая популяция данных:
- [описание: e.g. "Resume с external_id но source='local'"]
- Количество: [число или "примерно X"]

не будет обработана механизмом [healing/sync] новой логики.

Нужно ли добавить фоновый механизм (background sync job / batch task / миграция) для исцеления existing data?

a) Да, расширить existing sync job — добавить логику исправления source
b) Да, добавить новую periodic batch task для healing
c) Да, добавить миграцию Django (RunPython) — одноразовое исправление
d) Нет, legacy data можно оставить как есть (указать причину)
```

**При выборе a/b/c:**
- Добавить/уточнить FR
- Обновить Change Plan (добавить фазу "Phase 0: Legacy Data Healing")
- Обновить codebase-analysis.md раздел Legacy Data Compatibility

**При выборе d:**
- Задокументировать причину в Non-Goals или Assumptions
- Указать, как long-term будут обработаны existing data (оставить на будущее?)

---

## Примеры

### Пример 1: Resume fallback (025_hh-resume-fallback)

**Популяция:** Resume с external_id (признак HH) но source='local' (из миграции в S3)

**Healing механизм:** `heal_hh_resume_source_task.delay()` — запускается in `get_full_resume_by_external_id_sync` при запросе резюме

**Проблема:** Пайплайн генерации писем отсекает source='local' на входе → healing никогда не запустится

**Действие:** Расширить `update_user_resumes_from_hh_sync()` чтобы переставлять source='hh' для резюме с external_id

**FR добавить:**
> «При синхронизации пользователя с HH (авторизация или periodic job) система должна исправлять существующие резюме, у которых проставлен external_id (признак HH-резюме) но source='local' — переставлять source='hh' и загружать resume_text если пусто.»

### Пример 2: Миграция парсера (гипотетическая)

**Популяция:** RawContent от старого парсера с parsing_status='pending' и пустой text

**Healing механизм:** Новый унифицированный парсер, запускается при ingestion

**Проблема:** Старые pending записи не переиндексируются автоматически

**Действие:** Добавить batch task `reparse_legacy_raw_content.delay()` чтобы переобработать старые pending записи

---

## Checklist

- [ ] Выявлена существующая популяция data (SQL запрос выполнен)
- [ ] Проверены все механизмы (producer/consumer/healing) — будут ли они вызваны для existing data
- [ ] Если механизм на-demand — проверено, что его точка входа не блокирует existing data
- [ ] Выбран способ healing (расширить sync job / новый batch task / миграция)
- [ ] Добавлена или уточнена FR с явным требованием на исправление
- [ ] Обновлена codebase-analysis.md раздел Legacy Data Compatibility
- [ ] Пользователь подтвердил, что way forward правильный
