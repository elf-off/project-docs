# TASK: AI-анализ рекламных объявлений (вакансий)

**Дата:** 2026-03-30
**Приоритет:** Высокий
**Проект:** CRM Worker (локальная копия, боевой — `/opt/openai/crm-worker/`)
**БД:** `2_kadry_4_crm_avito`

---

## Описание

Новая фича: AI-анализ текстов рекламных объявлений о вакансиях на Avito.

**Две функции:**
1. **Анализ** — Claude оценивает объявление по 11 категориям, выдаёт баллы (0-10) + рекомендации по каждой категории + общий балл + общие рекомендации
2. **Генерация A/B вариантов** — Claude создаёт 2 улучшенные версии текста объявления на основе результатов анализа

Вакансии уже есть в таблице `vacancies` (синхронизация каждые 30 минут). Поле `raw_description` содержит полный текст объявления. Поле `title` — заголовок.

---

## 1. SQL-миграция: таблица `ad_analysis`

Создать файл `migrations/004_ad_analysis.sql`:

```sql
CREATE TABLE IF NOT EXISTS ad_analysis (
    id                INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vacancy_id        INT UNSIGNED NOT NULL,
    account_id        INT UNSIGNED NOT NULL,
    
    -- Результат анализа (JSON)
    analysis_result   JSON         DEFAULT NULL COMMENT 'Полный результат анализа от Claude',
    overall_score     DECIMAL(3,1) DEFAULT NULL COMMENT 'Общий балл 0-10',
    
    -- Сгенерированные варианты (JSON)
    generated_variants JSON       DEFAULT NULL COMMENT 'A/B варианты текста объявления',
    
    -- Мета
    model_used        VARCHAR(50)  DEFAULT NULL COMMENT 'Модель Claude',
    analysis_tokens   INT UNSIGNED DEFAULT 0 COMMENT 'Токены на анализ (prompt+completion)',
    generation_tokens INT UNSIGNED DEFAULT 0 COMMENT 'Токены на генерацию вариантов',
    cost_usd          DECIMAL(10,6) DEFAULT 0,
    
    status            ENUM('pending','analyzing','generating','completed','failed') 
                      NOT NULL DEFAULT 'pending',
    error_message     TEXT         DEFAULT NULL,
    
    created_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    KEY idx_vacancy (vacancy_id),
    KEY idx_account (account_id),
    KEY idx_status (status),
    KEY idx_score (overall_score),
    CONSTRAINT fk_analysis_vacancy FOREIGN KEY (vacancy_id) REFERENCES vacancies(id),
    CONSTRAINT fk_analysis_account FOREIGN KEY (account_id) REFERENCES avito_accounts(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='AI-анализ рекламных объявлений';
```

---

## 2. SQLAlchemy-модель

Создать файл `models/ad_analysis.py`:

```python
from sqlalchemy import Column, Integer, String, Enum, Text, DateTime, DECIMAL, JSON, ForeignKey
from sqlalchemy.sql import func
from models.db import Base

class AdAnalysis(Base):
    __tablename__ = "ad_analysis"
    
    id = Column(Integer, primary_key=True, autoincrement=True)
    vacancy_id = Column(Integer, ForeignKey("vacancies.id"), nullable=False, index=True)
    account_id = Column(Integer, ForeignKey("avito_accounts.id"), nullable=False, index=True)
    
    analysis_result = Column(JSON, default=None)
    overall_score = Column(DECIMAL(3, 1), default=None, index=True)
    
    generated_variants = Column(JSON, default=None)
    
    model_used = Column(String(50), default=None)
    analysis_tokens = Column(Integer, default=0)
    generation_tokens = Column(Integer, default=0)
    cost_usd = Column(DECIMAL(10, 6), default=0)
    
    status = Column(Enum("pending", "analyzing", "generating", "completed", "failed"), 
                    nullable=False, default="pending")
    error_message = Column(Text, default=None)
    
    created_at = Column(DateTime, nullable=False, server_default=func.now())
    updated_at = Column(DateTime, nullable=False, server_default=func.now(), onupdate=func.now())
```

**Также:** добавить импорт в `models/__init__.py` (или `models/db.py` — посмотреть где остальные модели импортируются).

---

## 3. Сервис `services/ad_analysis_service.py`

Класс `AdAnalysisService` с 4 методами:

### 3.1 `analyze_vacancy(vacancy_id: int) -> AdAnalysis`

1. Загрузить вакансию из БД по `vacancy_id` (таблица `vacancies`)
2. Взять `raw_description` и `title`
3. Создать запись в `ad_analysis` со статусом `analyzing`
4. Вызвать Claude API с промптом анализа (см. раздел 5.1)
5. Распарсить ответ (JSON), сохранить в `analysis_result`, `overall_score`
6. Обновить статус → `completed` (или `failed` + `error_message`)
7. Сохранить token usage и cost

### 3.2 `generate_variants(analysis_id: int) -> AdAnalysis`

1. Загрузить `ad_analysis` по ID, проверить что `status == completed` и `analysis_result` не пуст
2. Загрузить вакансию (для `raw_description` и `title`)
3. Обновить статус → `generating`
4. Вызвать Claude API с промптом генерации (см. раздел 5.2), передать оригинальный текст + результат анализа
5. Распарсить ответ (JSON), сохранить в `generated_variants`
6. Обновить статус → `completed`
7. Сохранить token usage и cost

### 3.3 `get_analysis(analysis_id: int) -> AdAnalysis`

Простой SELECT по ID. Вернуть 404 если не найден.

### 3.4 `list_analyses(account_id: int = None, vacancy_id: int = None) -> list[AdAnalysis]`

Список анализов с опциональной фильтрацией по аккаунту и/или вакансии. ORDER BY created_at DESC.

### Важно:
- Для вызова Claude API — использовать **существующий** `services/ai_claude.py` (там уже настроен клиент, прокси, retry). Посмотреть как вызывается Claude в `ai_agent.py` и использовать аналогичный паттерн.
- Модель: использовать `settings.claude_model` из конфига
- Все операции — async (проект использует async SQLAlchemy)

---

## 4. API-эндпоинты

Создать файл `api/ad_analysis.py` (роутер FastAPI):

```
POST   /api/ads/analyze/{vacancy_id}     — запустить анализ вакансии
POST   /api/ads/generate/{analysis_id}   — сгенерировать A/B варианты
GET    /api/ads/analysis/{analysis_id}    — получить результат анализа
GET    /api/ads/analyses                  — список анализов (?account_id=&vacancy_id=)
GET    /api/ads/vacancies?account_id=N    — список вакансий аккаунта с инфо об анализе (для UI)
```

**Подключить роутер** в `main.py` (или где подключаются остальные роутеры — посмотреть паттерн по `api/admin.py` или `api/webhooks.py`).

Авторизация — аналогично admin-эндпоинтам (если есть проверка, использовать ту же).

---

## 5. Промпты для Claude API

### 5.1 Промпт анализа

System prompt:
```
Ты — эксперт по HR-маркетингу и рекламе вакансий на job-площадках (Avito, hh.ru, SuperJob).
Твоя задача — проанализировать рекламное объявление о вакансии и оценить его эффективность.

Отвечай ТОЛЬКО валидным JSON без markdown-обёрток и без пояснений.
```

User prompt:
```
Проанализируй рекламное объявление о вакансии.

Заголовок: {title}

Текст объявления:
{raw_description}

Оцени по следующим 11 категориям, каждую по шкале 0-10:

1. title — Заголовок: привлекательность, ключевые слова, длина
2. salary_motivation — Зарплата и мотивация: указана ли зарплата, конкурентность, бонусы
3. conditions — Условия работы: график, формат, соцпакет, питание, проезд
4. requirements — Требования: адекватность, ясность, не завышены ли
5. cta — Призыв к действию: есть ли, мотивирует ли откликнуться
6. structure_formatting — Структура и форматирование: читаемость, абзацы, списки, эмодзи
7. location_geo — Локация: указан ли адрес, метро, район, удобство
8. target_audience — Целевая аудитория: понятно ли кого ищут, tone of voice
9. search_visibility — Поисковая видимость: ключевые слова в заголовке/тексте для поиска
10. trust_signals — Сигналы доверия: название компании, отзывы, стаж, факты
11. competitive_edge — Конкурентное преимущество: чем лучше аналогичных вакансий

Формат ответа (строго JSON):
{
  "categories": {
    "title": {"score": 7, "comment": "Краткий комментарий что хорошо", "suggestion": "Что можно улучшить"},
    "salary_motivation": {"score": 5, "comment": "...", "suggestion": "..."},
    "conditions": {"score": 8, "comment": "...", "suggestion": "..."},
    "requirements": {"score": 6, "comment": "...", "suggestion": "..."},
    "cta": {"score": 3, "comment": "...", "suggestion": "..."},
    "structure_formatting": {"score": 7, "comment": "...", "suggestion": "..."},
    "location_geo": {"score": 9, "comment": "...", "suggestion": "..."},
    "target_audience": {"score": 6, "comment": "...", "suggestion": "..."},
    "search_visibility": {"score": 5, "comment": "...", "suggestion": "..."},
    "trust_signals": {"score": 4, "comment": "...", "suggestion": "..."},
    "competitive_edge": {"score": 5, "comment": "...", "suggestion": "..."}
  },
  "overall_score": 5.9,
  "general_recommendations": [
    "Рекомендация 1",
    "Рекомендация 2",
    "Рекомендация 3"
  ]
}
```

### 5.2 Промпт генерации A/B вариантов

System prompt:
```
Ты — копирайтер, специализирующийся на рекламных объявлениях о вакансиях для job-площадок (Avito, hh.ru).
Твоя задача — создать улучшенные варианты объявления на основе анализа.

Отвечай ТОЛЬКО валидным JSON без markdown-обёрток и без пояснений.
```

User prompt:
```
На основе анализа создай 2 улучшенных варианта рекламного объявления.

Оригинальный заголовок: {title}

Оригинальный текст:
{raw_description}

Результат анализа:
{analysis_result_json}

Создай 2 варианта:
- Вариант A: консервативное улучшение — исправить слабые места, сохранить общий стиль и структуру
- Вариант B: радикальное улучшение — переписать с нуля, максимизировать привлекательность

Формат ответа (строго JSON):
{
  "variant_a": {
    "title": "Улучшенный заголовок A",
    "description": "Полный текст улучшенного объявления A",
    "strategy": "Краткое описание стратегии: что изменено и почему"
  },
  "variant_b": {
    "title": "Улучшенный заголовок B",
    "description": "Полный текст улучшенного объявления B",
    "strategy": "Краткое описание стратегии: что изменено и почему"
  }
}
```

---

## 6. Обработка ответа Claude

Claude может вернуть JSON внутри markdown-блока. **Обязательно** очистить ответ перед парсингом:

```python
import json
import re

def parse_claude_json(text: str) -> dict:
    """Извлечь JSON из ответа Claude, даже если обёрнут в markdown."""
    cleaned = re.sub(r'^```(?:json)?\s*', '', text.strip())
    cleaned = re.sub(r'\s*```$', '', cleaned)
    return json.loads(cleaned)
```

---

## 7. Что НЕ делать

- **НЕ** менять существующие модели/сервисы (vacancy_sync, ai_agent, ai_claude и т.д.)
- **НЕ** запускать анализ автоматически при синке вакансий — только по запросу через API.

---

## 8. UI — вкладка "Анализ объявлений" в админке

Админка CRM Worker — `babito.kadry-24.ru/admin/`. Посмотреть как устроены существующие вкладки (аккаунты, логи, статистика, вакансии, тестирование) и сделать аналогично.

### 8.1 Новая вкладка: "Анализ объявлений"

Добавить в навигацию админки новую вкладку. Состоит из двух экранов (views):

---

### 8.2 Экран 1 — Список вакансий для анализа

**Сверху:**
- Dropdown выбора аккаунта (загружать из API `/api/admin/accounts` или аналогичный endpoint, где берутся аккаунты на вкладке "Аккаунты"). По умолчанию — первый аккаунт.

**Таблица вакансий (из эндпоинта `/api/ads/vacancies?account_id=N` — см. ниже новый эндпоинт 8.5):**

| Колонка | Источник |
|---------|----------|
| Заголовок | `vacancy.title` |
| Город | `vacancy.city` |
| Последний синк | `vacancy.last_synced_at` |
| Балл | `ad_analysis.overall_score` (если анализ уже делали), иначе "—" |
| Действие | Кнопка "Анализировать" (или "Повторить" если балл уже есть) |

- Цвет балла: красный (<4), жёлтый (4-7), зелёный (>7)
- Клик по строке с баллом → переход на Экран 2

**Поведение кнопки "Анализировать":**
1. Показать спиннер / disabled на кнопке
2. `POST /api/ads/analyze/{vacancy_id}`
3. Поллинг `GET /api/ads/analysis/{id}` каждые 3 сек, пока `status != completed/failed`
4. Когда ready → обновить балл в таблице + переход на Экран 2

---

### 8.3 Экран 2 — Результат анализа

**Шапка:**
- Кнопка "← Назад" (к списку вакансий)
- Заголовок вакансии
- Общий балл — крупно, цветом (красный <4, жёлтый 4-7, зелёный >7)

**Блок "Оценки по категориям" (11 строк):**

Каждая категория отображается как строка/карточка:
```
[████████░░] 8/10  Условия работы
✓ Указан график, проезд компенсируется
💡 Добавить информацию о питании и соцпакете
```
- Прогресс-бар (или числовой балл) + цвет
- `comment` — что хорошо (иконка ✓)
- `suggestion` — что можно улучшить (иконка 💡)

**Названия категорий на русском (маппинг ключ → отображение):**
```
title             → "Заголовок"
salary_motivation → "Зарплата и мотивация"
conditions        → "Условия работы"
requirements      → "Требования"
cta               → "Призыв к действию"
structure_formatting → "Структура и форматирование"
location_geo      → "Локация"
target_audience   → "Целевая аудитория"
search_visibility → "Поисковая видимость"
trust_signals     → "Сигналы доверия"
competitive_edge  → "Конкурентное преимущество"
```

**Блок "Общие рекомендации":**
- Нумерованный список из `general_recommendations`

**Блок "Варианты объявления":**
- Кнопка "Сгенерировать варианты" (если `generated_variants` пуст)
- Спиннер при генерации (аналогично поллинг каждые 3 сек)
- Когда готово — два блока рядом (или друг под другом на мобильных):

```
┌─────────────────────────────┐  ┌─────────────────────────────┐
│ Вариант A (консервативный)  │  │ Вариант B (радикальный)     │
│                             │  │                             │
│ Заголовок: ...              │  │ Заголовок: ...              │
│                             │  │                             │
│ Стратегия: Исправлены       │  │ Стратегия: Полностью        │
│ слабые места, сохранён      │  │ переписано с фокусом на...  │
│ общий стиль                 │  │                             │
│                             │  │                             │
│ ┌─────────────────────────┐ │  │ ┌─────────────────────────┐ │
│ │ Полный текст объявления │ │  │ │ Полный текст объявления │ │
│ │ (прокручиваемый блок)   │ │  │ │ (прокручиваемый блок)   │ │
│ └─────────────────────────┘ │  │ └─────────────────────────┘ │
│                             │  │                             │
│ [Копировать заголовок]      │  │ [Копировать заголовок]      │
│ [Копировать текст]          │  │ [Копировать текст]          │
└─────────────────────────────┘  └─────────────────────────────┘
```

- Кнопки "Копировать заголовок" и "Копировать текст" — `navigator.clipboard.writeText()`, с визуальным подтверждением (иконка галочки на 2 сек)

---

### 8.4 Стили

Использовать те же стили/CSS что в существующих вкладках админки. Не подключать новые CSS-фреймворки. Посмотреть как сделаны таблицы, кнопки, спиннеры на других вкладках и использовать аналогично.

---

### 8.5 Дополнительный API-эндпоинт для UI

Добавить в `api/ad_analysis.py`:

```
GET /api/ads/vacancies?account_id=N  — список вакансий аккаунта с инфо об анализе
```

Возвращает:
```json
[
  {
    "id": 42,
    "title": "Сборщик / Комплектовщик",
    "city": "Москва",
    "last_synced_at": "2026-03-30T10:00:00",
    "analysis": {
      "id": 7,
      "overall_score": 5.3,
      "status": "completed",
      "created_at": "2026-03-30T09:30:00"
    }
  },
  {
    "id": 43,
    "title": "Грузчик",
    "city": "Казань",
    "last_synced_at": "2026-03-30T10:00:00",
    "analysis": null
  }
]
```

SQL: LEFT JOIN `ad_analysis` на `vacancies`, взять последний анализ для каждой вакансии (MAX id или ORDER BY created_at DESC LIMIT 1 per vacancy).

---

## 9. Структура новых файлов

```
models/ad_analysis.py            # SQLAlchemy-модель
services/ad_analysis_service.py  # Бизнес-логика + вызовы Claude
api/ad_analysis.py               # FastAPI роутер (5 эндпоинтов)
migrations/004_ad_analysis.sql   # SQL-миграция
static/admin/  или templates/    # UI-файлы вкладки (HTML/JS/CSS) — куда именно класть, посмотреть по паттерну существующих вкладок
```

---

## 10. Проверка после выполнения

```bash
# Проверить что импорты работают
cd /путь/к/crm-worker
source venv/bin/activate
python -c "from models.ad_analysis import AdAnalysis; print('Model OK')"
python -c "from services.ad_analysis_service import AdAnalysisService; print('Service OK')"
python -c "from api.ad_analysis import router; print('Router OK')"
```

---

## 11. Деплой на сервер (выполняет человек вручную)

После того как Code завершит работу — перенести изменения на сервер:

1. Скопировать новые/изменённые файлы на сервер (MC / sshfs / scp)

2. Выполнить SQL-миграцию:
```bash
mysql -u root -p 2_kadry_4_crm_avito < /opt/openai/crm-worker/migrations/004_ad_analysis.sql
```

3. Перезапустить сервис:
```bash
sudo systemctl restart k24-crm-worker
```

4. Проверить:
```bash
# Health check
curl http://localhost:9800/health

# Список вакансий (взять любой vacancy_id)
mysql -u root -p 2_kadry_4_crm_avito -e "SELECT id, title FROM vacancies LIMIT 5;"

# Запустить анализ
curl -X POST http://localhost:9800/api/ads/analyze/1

# Посмотреть результат
curl http://localhost:9800/api/ads/analysis/1

# Сгенерировать варианты
curl -X POST http://localhost:9800/api/ads/generate/1

# Посмотреть с вариантами
curl http://localhost:9800/api/ads/analysis/1

# Проверить UI
# Открыть babito.kadry-24.ru/admin/ → вкладка "Анализ объявлений"
# Выбрать аккаунт → увидеть вакансии → нажать "Анализировать" → дождаться результата
```

---

## ⚠️ Важные правила

- Использовать **существующий** паттерн вызова Claude из `services/ai_claude.py`
- Использовать **существующий** `AsyncSessionFactory` из `models/db.py`
- Посмотреть как подключаются роутеры в `main.py` и сделать аналогично
- Посмотреть как устроены существующие вкладки админки (HTML/JS/CSS, шаблоны, маршруты) и **повторить тот же паттерн** для новой вкладки
- Все операции — async/await
- Документация и комментарии в коде — **на русском языке**
