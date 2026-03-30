# TASK: Фаза 2.1 — Схема БД для откликов (регионы, очереди, назначения)

**Дата:** 2026-03-30
**Приоритет:** Высокий
**Проект:** `/var/www/new.kadry-24.ru/`
**Зависимости:** Фаза 1 завершена, БД `2_kadry_4_maps` существует

---

## КОНТЕКСТ

Фаза 2 — интеграция откликов из CRM Worker в портал операторов.

**Архитектура:**
- CRM Worker (`2_kadry_4_crm_avito`, порт 9800) продолжает работать автономно — Елена обрабатывает отклики 24/7 и пишет в `handover_cards`
- Main-api (`2_kadry_4_maps`, порт 8100) читает `handover_cards` cross-database (read-only) и ведёт логику распределения у себя
- Распределение: "умный pull" — карточки лежат в очереди региона, оператор получает порцию при открытии вкладки "Отклики"

**Цепочка:** код вакансии → регион → очередь → порция оператору

---

## 1. НОВЫЕ ТАБЛИЦЫ В `2_kadry_4_maps`

### 1.1 Регионы

```sql
CREATE TABLE IF NOT EXISTS regions (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL COMMENT 'Название региона (Ростов, Москва, СПб...)',
    timezone    VARCHAR(50)  NOT NULL DEFAULT 'Europe/Moscow' COMMENT 'Часовой пояс региона',
    is_active   TINYINT(1)   NOT NULL DEFAULT 1,
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_region_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='Регионы для распределения откликов';
```

### 1.2 Маппинг код вакансии → регион

```sql
CREATE TABLE IF NOT EXISTS vacancy_code_regions (
    id              INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    vacancy_code    VARCHAR(50)  NOT NULL COMMENT 'Код вакансии из CRM Worker (напр. РОС-001)',
    region_id       INT UNSIGNED NOT NULL,
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_vacancy_code (vacancy_code),
    CONSTRAINT fk_vcr_region FOREIGN KEY (region_id) REFERENCES regions(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='Маппинг кодов вакансий на регионы (источник: Google Docs)';
```

### 1.3 Привязка операторов к регионам

```sql
CREATE TABLE IF NOT EXISTS operator_regions (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     INT UNSIGNED NOT NULL COMMENT 'ID оператора из 2_kadry_4.users',
    region_id   INT UNSIGNED NOT NULL,
    created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_region (user_id, region_id),
    CONSTRAINT fk_or_region FOREIGN KEY (region_id) REFERENCES regions(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='Привязка операторов к регионам (один оператор может быть в нескольких)';
```

### 1.4 Расписание рабочего времени региона

```sql
CREATE TABLE IF NOT EXISTS region_work_schedule (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    region_id   INT UNSIGNED NOT NULL,
    day_of_week TINYINT      NOT NULL COMMENT '0=пн, 1=вт, ..., 6=вс',
    start_time  TIME         NOT NULL DEFAULT '09:00:00',
    end_time    TIME         NOT NULL DEFAULT '18:00:00',
    is_working  TINYINT(1)   NOT NULL DEFAULT 1 COMMENT '0 = выходной',
    UNIQUE KEY uk_region_day (region_id, day_of_week),
    CONSTRAINT fk_rws_region FOREIGN KEY (region_id) REFERENCES regions(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='Расписание рабочего времени по регионам (для корректировки Елены)';
```

### 1.5 Назначения карточек на операторов

```sql
CREATE TABLE IF NOT EXISTS response_assignments (
    id                  INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    handover_card_id    INT UNSIGNED NOT NULL COMMENT 'ID из 2_kadry_4_crm_avito.handover_cards',
    region_id           INT UNSIGNED NOT NULL,
    operator_user_id    INT UNSIGNED DEFAULT NULL COMMENT 'ID оператора (NULL = в очереди)',
    
    -- Статус обработки
    status              ENUM('queued','assigned','not_suitable','refused','thinking','recorded','expired')
                        NOT NULL DEFAULT 'queued'
                        COMMENT 'queued=в очереди, assigned=выдана оператору, not_suitable=не подходит, refused=отказался, thinking=думает, recorded=записан, expired=просрочена',
    
    -- Результат обработки
    call_comment        TEXT         DEFAULT NULL COMMENT 'Комментарий оператора после звонка',
    application_id      INT UNSIGNED DEFAULT NULL COMMENT 'ID заявки если статус=recorded (связь с applications)',
    
    -- Временные метки
    assigned_at         DATETIME     DEFAULT NULL COMMENT 'Когда выдана оператору',
    processed_at        DATETIME     DEFAULT NULL COMMENT 'Когда обработана (финальный статус)',
    created_at          DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_handover_card (handover_card_id),
    KEY idx_status (status),
    KEY idx_region_status (region_id, status),
    KEY idx_operator (operator_user_id),
    CONSTRAINT fk_ra_region FOREIGN KEY (region_id) REFERENCES regions(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='Назначения карточек откликов на операторов (умный pull)';
```

**Примечание:** FK на `handover_cards` не ставим — это cross-database, MariaDB не поддерживает. Целостность через код.

### 1.6 Настройки системы

```sql
CREATE TABLE IF NOT EXISTS system_settings (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    setting_key VARCHAR(100) NOT NULL,
    setting_value VARCHAR(500) NOT NULL,
    description VARCHAR(255) DEFAULT NULL,
    updated_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_key (setting_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='Системные настройки портала';

-- Настройка размера порции карточек
INSERT INTO system_settings (setting_key, setting_value, description) VALUES
('responses.batch_size', '5', 'Количество карточек откликов, выдаваемых оператору за раз');
```

---

## 2. НОВЫЕ PERMISSIONS

```sql
-- Подключиться к 2_kadry_4_users
USE 2_kadry_4_users;

-- Добавить permissions
INSERT INTO permissions (name, description) VALUES
('operator.responses.view', 'Просмотр откликов из CRM'),
('operator.responses.process', 'Обработка откликов (взять, статус, записать)'),
('admin.regions.view', 'Просмотр регионов'),
('admin.regions.manage', 'Управление регионами и расписанием'),
('admin.settings.manage', 'Управление системными настройками')
ON DUPLICATE KEY UPDATE description = VALUES(description);

-- Привязать к роли operator
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id 
FROM roles r, permissions p
WHERE r.name = 'operator' 
  AND p.name IN ('operator.responses.view', 'operator.responses.process')
ON DUPLICATE KEY UPDATE role_id = role_id;

-- Привязать к роли admin
INSERT INTO role_permissions (role_id, permission_id)
SELECT r.id, p.id 
FROM roles r, permissions p
WHERE r.name = 'admin' 
  AND p.name IN ('admin.regions.view', 'admin.regions.manage', 'admin.settings.manage',
                  'operator.responses.view', 'operator.responses.process')
ON DUPLICATE KEY UPDATE role_id = role_id;
```

---

## 3. CROSS-DATABASE ДОСТУП

### 3.1 Права для main-api (portal-user)

```sql
-- portal-user уже имеет доступ к 2_kadry_4 и 2_kadry_4_maps
-- Добавить read-only доступ к CRM Worker БД
GRANT SELECT ON 2_kadry_4_crm_avito.handover_cards TO 'portal-user'@'localhost';
GRANT SELECT ON 2_kadry_4_crm_avito.applications TO 'portal-user'@'localhost';
GRANT SELECT ON 2_kadry_4_crm_avito.messages TO 'portal-user'@'localhost';
GRANT SELECT ON 2_kadry_4_crm_avito.chats TO 'portal-user'@'localhost';
GRANT SELECT ON 2_kadry_4_crm_avito.ai_sessions TO 'portal-user'@'localhost';
GRANT SELECT ON 2_kadry_4_crm_avito.vacancies TO 'portal-user'@'localhost';
FLUSH PRIVILEGES;
```

### 3.2 Права для CRM Worker на чтение расписания

```sql
-- CRM Worker читает расписание из 2_kadry_4_maps
-- Узнать пользователя CRM Worker:
-- grep DB_USER /opt/openai/crm-worker/.env
GRANT SELECT ON 2_kadry_4_maps.regions TO 'crm_user'@'localhost';
GRANT SELECT ON 2_kadry_4_maps.region_work_schedule TO 'crm_user'@'localhost';
GRANT SELECT ON 2_kadry_4_maps.vacancy_code_regions TO 'crm_user'@'localhost';
FLUSH PRIVILEGES;
```

**⚠️ ВАЖНО:** Заменить `'crm_user'@'localhost'` на реального пользователя из `/opt/openai/crm-worker/.env` (`DB_USER`).

---

## 4. МИГРАЦИЯ — СОЗДАНИЕ ФАЙЛА

Создать файл `backend/migrations/005_phase2_responses.sql` со всеми SQL из пунктов 1-3.

---

## 5. ПРОВЕРКА

```bash
# Проверить таблицы созданы
mysql -u root -p 2_kadry_4_maps -e "SHOW TABLES LIKE 'region%'; SHOW TABLES LIKE 'response%'; SHOW TABLES LIKE 'vacancy_code%'; SHOW TABLES LIKE 'operator_regions'; SHOW TABLES LIKE 'system_settings';"

# Проверить cross-db доступ portal-user
mysql -u portal-user -p 2_kadry_4_maps -e "SELECT COUNT(*) FROM 2_kadry_4_crm_avito.handover_cards;"

# Проверить permissions
mysql -u root -p 2_kadry_4_users -e "SELECT p.name FROM permissions p JOIN role_permissions rp ON rp.permission_id = p.id JOIN roles r ON r.id = rp.role_id WHERE r.name = 'operator' AND p.name LIKE 'operator.responses%';"
```

### Чеклист:
- [ ] 6 таблиц созданы в `2_kadry_4_maps`
- [ ] Permissions добавлены в `2_kadry_4_users`
- [ ] Cross-db SELECT для portal-user на `2_kadry_4_crm_avito` работает
- [ ] Cross-db SELECT для crm_user на `2_kadry_4_maps` работает
- [ ] Файл миграции `005_phase2_responses.sql` создан

---

## ⚠️ НЕ ДЕЛАТЬ

- Не трогать таблицы в `2_kadry_4_crm_avito` — CRM Worker работает автономно
- Не создавать FK между базами — MariaDB не поддерживает cross-db FK
- Не менять существующие таблицы `handover_cards`, `applications` и т.д.
