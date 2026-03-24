# API Documentation

## Core Modules

### 1. Data Ingestion (`src/connectors/data_ingestion.py`)

#### `DataIngestion.process_shift(date, shift_code)`

Полный цикл обработки смены.

**Parameters:**
- `date` (date): Дата для обработки
- `shift_code` (str): 'day' или 'night'

**Returns:** `snapshot_id` (int)

**Example:**
```python
from src.connectors.data_ingestion import data_ingestion
from datetime import date

snapshot_id = data_ingestion.process_shift(date.today(), 'day')
```

---

### 2. Metrics Calculator (`src/core/metrics.py`)

#### `MetricsCalculator.calculate_project_metrics(project_id, end_date, period_days)`

Расчёт метрик по проекту за период.

**Parameters:**
- `project_id` (int): ID проекта
- `end_date` (date): Конечная дата периода
- `period_days` (int): Длина периода в днях (default: 30)

**Returns:** `ProjectMetrics` или None

**Example:**
```python
from src.core.metrics import metrics_calculator
from datetime import date

metrics = metrics_calculator.calculate_project_metrics(
    project_id=123,
    end_date=date.today(),
    period_days=30
)

print(f"Risk Score: {metrics.risk_score}")
print(f"Miss Rate: {metrics.miss_rate}")
```

#### `MetricsCalculator.calculate_all_projects(target_date, period_days)`

Расчёт метрик для всех активных проектов.

**Returns:** List[ProjectMetrics]

---

### 3. Reason Analyzer (`src/analysis/reason_analyzer.py`)

#### `ReasonAnalyzer.classify_reason(text, shift_code, month)`

Классификация причины невыполнения с использованием AI.

**Parameters:**
- `text` (str): Текст причины
- `shift_code` (str): 'day' / 'night'
- `month` (int): Месяц (1-12)

**Returns:** Tuple[List[str], float] - (categories, confidence)

**Example:**
```python
from src.analysis.reason_analyzer import reason_analyzer

categories, confidence = reason_analyzer.classify_reason(
    text="Работник заболел, не смогли найти замену",
    shift_code='day',
    month=1
)

print(f"Categories: {categories}")
print(f"Confidence: {confidence}")
```

#### `ReasonAnalyzer.generate_hypotheses(project_id, shift_code, deficit, season)`

Генерация гипотез вероятных причин на основе истории.

**Returns:** List[Dict] - [{'reason', 'probability', 'basis'}]

---

### 4. Telegram Bot (`src/telegram/bot.py`)

#### `TelegramBotManager.send_incident_request(...)`

Отправка запроса менеджеру при невыполнении заявки.

**Parameters:**
- `incident_id` (int)
- `project_name` (str)
- `incident_date` (date)
- `shift_code` (str)
- `request` (int)
- `actual` (int)
- `deficit` (int)
- `manager_chat_id` (str)

---

### 5. CEO Reporter (`src/reporting/ceo_reports.py`)

#### `CEOReporter.generate_daily_digest(target_date)`

Генерация ежедневного дайджеста для CEO.

**Returns:** str (HTML formatted message)

**Example:**
```python
from src.reporting.ceo_reports import ceo_reporter
from datetime import date

digest = ceo_reporter.generate_daily_digest(date.today())
print(digest)
```

---

### 6. Orchestrator (`src/orchestrator.py`)

#### `CEOAgent.manual_process(target_date, shift_code)`

Ручной запуск обработки (для тестирования).

**Parameters:**
- `target_date` (date, optional): Дата для обработки
- `shift_code` (str, optional): 'day' / 'night' / None (обе смены)

**Example:**
```python
import asyncio
from src.orchestrator import agent
from datetime import date

async def manual_run():
    await agent.initialize()
    await agent.manual_process(date.today(), 'day')
    await agent.stop()

asyncio.run(manual_run())
```

---

## Data Models

### ProjectMetrics

```python
@dataclass
class ProjectMetrics:
    project_id: int
    project_name: str
    period_days: int
    
    # Counts
    shift_total: int = 0
    miss_count: int = 0
    deficit_sum: int = 0
    deficit_peak: int = 0
    
    # Streaks
    streak_max: int = 0
    
    # Rates
    miss_rate: float = 0.0
    day_miss_rate: float = 0.0
    night_miss_rate: float = 0.0
    
    # Risk
    risk_score: float = 0.0
    risk_class: str = "green"
    confidence: float = 0.0
```

### DataRecord

```python
class DataRecord:
    date: date
    shift_code: str
    project_id: int
    project_name: str
    request: Optional[int]
    actual: Optional[int]
    deficit: Optional[int]
    fulfilled: Optional[bool]
```

---

## Database Models

See `src/storage/models.py` for full SQLAlchemy models:

- **RefProject** - Справочник проектов
- **RefShift** - Справочник смен
- **Exit** - Фактические данные
- **Snapshot** - Срезы данных
- **SnapshotRecord** - Записи срезов
- **Incident** - Инциденты невыполнения
- **TelegramMessage** - История сообщений
- **Reason** - Причины невыполнения
- **RiskScore** - История Risk Score
- **AuditLog** - Аудит действий

---

## Configuration

See `config/settings.py` for all available settings.

Key settings:
- `DB_*` - Database connection
- `TELEGRAM_*` - Telegram integration
- `RISK_*` - Risk Score parameters
- `ALERT_*` - Alert thresholds

---

## Error Handling

All modules use `structlog` for structured logging.

**Example:**
```python
logger.info("action_completed", entity_id=123, duration=2.5)
logger.error("action_failed", entity_id=123, error=str(e))
```

All errors are also logged to `agent_audit_logs` table.

---

## Testing

See `test_agent.py` for interactive testing menu.

**Quick tests:**
```bash
python test_agent.py
```

**Pytest:**
```bash
pytest tests/
pytest --cov=src tests/
```

---

For more information, see:
- README.md - Overview
- docs/DEPLOYMENT.md - Deployment guide
