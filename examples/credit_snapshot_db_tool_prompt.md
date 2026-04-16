# Prompt-ТЗ для генерации кастомной LangFlow tool

Скопируй весь блок ниже и отдай внешней LLM (DeepSeek или аналог) как задание на генерацию кода компонента.

---

Ты senior Python engineer, пишешь production-ready custom component для LangFlow 1.5.x.

## Цель

Нужен компонент-инструмент (tool) для агента, который:
1. Принимает структурированный JSON от LLM.
2. Валидирует JSON по обязательным полям.
3. Если JSON невалиден — возвращает понятную machine-readable ошибку.
4. Если JSON валиден — сохраняет записи в SQLite.
5. Возвращает результат выполнения в JSON-формате со статусом.

## Технические ограничения

- Только Python.
- Без внешних зависимостей кроме стандартной библиотеки и импортов LangFlow.
- Совместимость с LangFlow 1.5.x.
- Компонент должен работать как tool для агента (`to_toolkit` / `component_as_tool`).
- Должен быть безопасный и предсказуемый обработчик ошибок.

## Имя компонента

- `display_name = "Credit Snapshot DB Tool"`
- `name = "CreditSnapshotDBTool"`
- `icon = "database"`

## Входы компонента

1. `record_json` (обязательный, `MessageTextInput`, `tool_mode=True`)  
   JSON-строка со структурированным результатом от LLM.
2. `snapshot_id` (необязательный, `MessageTextInput`, `tool_mode=True`)  
   Если пустой — компонент сам генерирует UUID.
3. `database_url` (обязательный, `MessageTextInput`, `advanced=True`)  
   Значение по умолчанию: `sqlite:////tmp/credit_products.db`

## Выход компонента

- Один action-метод: `save_credit_snapshot`
- Возвращаемый тип: `Data`
- В `Data.data` обязательно положить:
  - `status`: `ok | invalid_json | error`
  - `snapshot_id`: `string | null`
  - `inserted_rows`: `int`
  - `errors`: `string[]`
  - `text`: JSON-строка с тем же payload (для удобного показа в логах/чате)

## Ожидаемая структура входного JSON (`record_json`)

```json
{
  "run_summary": {
    "topic": "string|null",
    "generated_at": "string|null"
  },
  "banks": [
    {
      "bank": "string",
      "product": "string|null",
      "rate": "any",
      "apr_psk": "any",
      "max_amount": "any",
      "term": "any",
      "special_conditions": [],
      "source_url": "string",
      "confidence": "string",
      "missing_fields": []
    }
  ],
  "comparison": {
    "best_rate_hint": "string|null",
    "notes": []
  }
}
```

## Валидация

Минимальные правила:
- JSON должен парситься.
- Top-level объект.
- Обязательные ключи верхнего уровня: `run_summary`, `banks`, `comparison`.
- `banks` — непустой массив.
- Для каждого элемента в `banks` обязательны:
  - `bank` (непустая строка)
  - `source_url` (непустая строка)
  - `confidence` (непустая строка)

При нарушении:
- `status=invalid_json`
- `inserted_rows=0`
- `errors` содержит детальные сообщения по каждому нарушению.

## Логика записи в БД

- Поддержать только SQLite.
- Нормализовать путь из `database_url` формата `sqlite:///...`.
- Создать директорию, если не существует.
- Подключиться через `sqlite3`.
- Создать таблицу при необходимости:

```sql
CREATE TABLE IF NOT EXISTS credit_product_snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  snapshot_id TEXT NOT NULL,
  topic TEXT,
  generated_at TEXT,
  bank TEXT,
  product TEXT,
  rate TEXT,
  apr_psk TEXT,
  max_amount TEXT,
  term TEXT,
  special_conditions TEXT,
  source_url TEXT,
  confidence TEXT,
  missing_fields TEXT,
  raw_record TEXT,
  created_at TEXT NOT NULL
);
```

- Для каждого элемента `banks[]` вставить строку.
- Поля со сложными типами сериализовать через `json.dumps(..., ensure_ascii=False)`.
- `created_at` ставить в UTC ISO-формате.

## Поведение при ошибках

- Любая ошибка парсинга JSON -> `invalid_json`.
- Любая ошибка БД/IO -> `error`.
- Исключения наружу не пробрасывать; всегда возвращать структурированный ответ.

## Критерии приемки

1. Пустой `record_json` -> `invalid_json`.
2. Битый JSON -> `invalid_json`.
3. JSON без `banks` -> `invalid_json`.
4. Валидный JSON с 4 банками -> `ok`, `inserted_rows=4`.
5. После успешной записи в SQLite таблица существует, строки добавлены.

## Что нужно выдать в ответ

1. Полный Python-код компонента (готовый к вставке в LangFlow Custom Component).
2. Короткую инструкцию, как подключить компонент в `agent.tools`.
3. 3 тестовых payload:
   - валидный,
   - невалидный (битый JSON),
   - невалидный (нет обязательных полей).

---

## Короткий реалистичный промт (быстрый подход)

Нужен Python custom component для LangFlow, который будет работать как tool агента: принимать `record_json`, валидировать структуру (`run_summary`, `banks`, `comparison`, плюс обязательные поля `bank/source_url/confidence` в каждом банке) и писать валидные записи в SQLite `sqlite:////tmp/credit_products.db`.  
Если JSON невалидный — верни `status=invalid_json` и список ошибок, если все ок — `status=ok`, `snapshot_id`, `inserted_rows`.  
Сделай полный готовый код компонента и короткую инструкцию по подключению в `agent.tools`.

---

## Примеры JSON для тестирования

### Валидный JSON (пример 1, 2 банка)

```json
{
  "run_summary": {
    "topic": "Сравнение потребкредитов",
    "generated_at": "2026-04-13T09:30:00Z"
  },
  "banks": [
    {
      "bank": "Сбер",
      "product": "Кредит наличными",
      "rate": "от 11.9%",
      "apr_psk": "13.2%",
      "max_amount": "5000000",
      "term": "60 мес",
      "special_conditions": ["зарплатным клиентам скидка"],
      "source_url": "https://example.org/sber-credit",
      "confidence": "medium",
      "missing_fields": []
    },
    {
      "bank": "ВТБ",
      "product": "Кредит наличными",
      "rate": "от 12.5%",
      "apr_psk": "14.1%",
      "max_amount": "7000000",
      "term": "84 мес",
      "special_conditions": [],
      "source_url": "https://example.org/vtb-credit",
      "confidence": "medium",
      "missing_fields": []
    }
  ],
  "comparison": {
    "best_rate_hint": "Сбер",
    "notes": ["Ставки нужно уточнить по ПСК"]
  }
}
```

### Валидный JSON (пример 2, с пустыми полями)

```json
{
  "run_summary": {
    "topic": "Сравнение кредитов 4 банков",
    "generated_at": null
  },
  "banks": [
    {
      "bank": "Т-Банк",
      "product": null,
      "rate": null,
      "apr_psk": null,
      "max_amount": null,
      "term": null,
      "special_conditions": [],
      "source_url": "https://example.org/tbank-credit",
      "confidence": "low",
      "missing_fields": ["rate", "apr_psk", "max_amount", "term"]
    }
  ],
  "comparison": {
    "best_rate_hint": null,
    "notes": []
  }
}
```

### Невалидный JSON (битый синтаксис)

```json
{
  "run_summary": {
    "topic": "demo"
  },
  "banks": [
    {
      "bank": "Сбер",
      "source_url": "https://example.org/sber",
      "confidence": "medium"
    }
  ],
  "comparison": {
    "best_rate_hint": "Сбер",
    "notes": []
  }
```

### Невалидный JSON (нет обязательных ключей)

```json
{
  "run_summary": {
    "topic": "demo",
    "generated_at": "2026-04-13T10:00:00Z"
  },
  "comparison": {
    "best_rate_hint": "Сбер",
    "notes": []
  }
}
```

### Невалидный JSON (в banks нет обязательных полей)

```json
{
  "run_summary": {
    "topic": "demo",
    "generated_at": "2026-04-13T10:00:00Z"
  },
  "banks": [
    {
      "bank": "",
      "product": "Кредит наличными",
      "source_url": "",
      "confidence": ""
    }
  ],
  "comparison": {
    "best_rate_hint": null,
    "notes": []
  }
}
```
