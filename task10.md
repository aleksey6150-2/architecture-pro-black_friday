### <a name="_b7urdng99y53"></a>**Название задачи:** Миграция на Cassandra для интернет-магазина "Мобильный мир"
### <a name="_hjk0fkfyohdk"></a>**Автор:** Львов А А
### <a name="_uanumrh8zrui"></a>**Дата:** 30 февраля 2026 года

## Архитектурный документ

## Задание 10.1: Анализ данных для миграции

### Обоснование выбора данных для Cassandra

**Cassandra отлично подходит для:**
1. **Time-series данных** (история заказов, события) — append-only модель
2. **Key-value нагрузок** (сессии пользователей) — быстрый доступ по ключу
3. **Высокой скорости записи** (логи, аудит) — до миллионов запис/сек
4. **Геораспределенных сценариев** — multi-DC репликация

**Cassandra НЕ подходит для:**
1. **Данных с транзакциями** (остатки, корзины) — нет ACID
2. **Частых обновлений** (активные корзины) — дорогие обновления
3. **Строгой консистентности** — eventual consistency по умолчанию

### Итоговый список сущностей для миграции

| Сущность                                     | Обоснование                                              |
|----------------------------------------------|----------------------------------------------------------|
| ** История заказов (order_history)**         | Time-series данные, append-only, высокая скорость записи |
| ** Пользовательские сессии (user_sessions)** | Key-value нагрузка, быстрое чтение/запись, TTL           |
| ** События аудита (audit_events)**           | Append-only, огромный объем, геораспределение            |
| ** Аналитика просмотров (product_views)**    | Time-series, агрегация, высокая скорость записи          |

---

## Задание 10.2: Концептуальная модель данных

### 10.2.1. Сущность: История заказов (order_history)

**Характеристики:**
- Запись: при каждом изменении статуса заказа
- Чтение: история по пользователю, по заказу
- Объем: ~100 млн записей/день

#### Модель данных:

```sql
CREATE TABLE order_history (
    user_id UUID,                    -- Partition key
    order_id UUID,                   -- Clustering key 1
    status_change_time TIMESTAMP,    -- Clustering key 2
    status TEXT,                     -- Статус заказа
    old_status TEXT,                  -- Предыдущий статус
    changed_by TEXT,                  -- Кто изменил
    changed_from_ip TEXT,              -- IP адрес
    PRIMARY KEY ((user_id), order_id, status_change_time)
) WITH CLUSTERING ORDER BY (order_id ASC, status_change_time DESC)
  AND default_time_to_live = 7776000; -- 90 дней
```

**Обоснование выбора ключей:**
- **Partition key (`user_id`)** — равномерное распределение по пользователям, нет горячих партиций
- **Clustering key (`order_id`, `status_change_time`)** — позволяет:
    - Быстро получить всю историю конкретного заказа (`WHERE user_id=? AND order_id=?`)
    - Получить последние изменения по пользователю (сортировка по времени DESC)
    - Эффективную time-series выборку

#### Альтернативный запрос: поиск по заказу

```sql
-- Создаем дополнительную таблицу для поиска по order_id
CREATE TABLE order_history_by_order (
    order_id UUID,                   -- Partition key
    status_change_time TIMESTAMP,     -- Clustering key
    user_id UUID,
    status TEXT,
    PRIMARY KEY ((order_id), status_change_time)
) WITH CLUSTERING ORDER BY (status_change_time DESC);
```

### 10.2.2. Сущность: Пользовательские сессии (user_sessions)

**Характеристики:**
- Запись: при каждом действии пользователя
- Чтение: проверка валидности сессии
- TTL: автоматическое удаление через 24 часа

#### Модель данных:

```sql
CREATE TABLE user_sessions (
    session_id UUID,                  -- Partition key
    user_id UUID,                      -- Данные сессии
    created_at TIMESTAMP,
    last_accessed TIMESTAMP,
    ip_address TEXT,
    user_agent TEXT,
    session_data TEXT,                  -- JSON с данными сессии
    PRIMARY KEY ((session_id))
) WITH default_time_to_live = 86400;    -- 24 часа
```

**Обоснование выбора ключей:**
- **Partition key (`session_id`)** — идеальный ключ для равномерного распределения
- Отсутствие clustering key — точечные чтения по сессии
- TTL автоматически удаляет старые сессии (no maintenance)

#### Дополнительная таблица для поиска сессий пользователя:

```sql
CREATE TABLE user_active_sessions (
    user_id UUID,                      -- Partition key
    session_id UUID,                    -- Clustering key
    last_accessed TIMESTAMP,
    PRIMARY KEY ((user_id), session_id)
) WITH default_time_to_live = 86400;
```

### 10.2.3. Сущность: События аудита (audit_events)

**Характеристики:**
- Запись: все значимые действия в системе
- Объем: до 1 млн событий/час
- Хранение: 30 дней

#### Модель данных:

```sql
CREATE TABLE audit_events (
    event_date DATE,                    -- Partition key (по дням)
    event_hour INT,                      -- Clustering key 1
    event_id TIMEUUID,                    -- Clustering key 2
    user_id UUID,
    event_type TEXT,                      -- login, order_create, etc.
    entity_type TEXT,                      -- order, product, cart
    entity_id UUID,
    old_value TEXT,
    new_value TEXT,
    ip_address TEXT,
    PRIMARY KEY ((event_date), event_hour, event_id)
) WITH CLUSTERING ORDER BY (event_hour DESC, event_id DESC)
  AND default_time_to_live = 2592000;     -- 30 дней
```

**Обоснование выбора ключей:**
- **Partition key (`event_date`)** — данные за день в одной партиции (предсказуемый размер)
- **Clustering key (`event_hour`, `event_id`)** — эффективная выборка за период
- TIMEUUID обеспечивает уникальность и сортировку по времени

**Предотвращение горячих партиций:**
- Размер партиции контролируется (1 день)
- Можно дополнительно шардировать по `event_hour` если нужно

### 10.2.4. Сущность: Аналитика просмотров (product_views)

**Характеристики:**
- Запись: каждый просмотр товара
- Чтение: топ товаров за период
- Объем: миллионы событий/день

#### Модель данных:

```sql
CREATE TABLE product_views (
    view_date DATE,                      -- Partition key
    product_id UUID,                      -- Clustering key 1
    view_hour INT,                        -- Clustering key 2
    view_count COUNTER,                    -- Счетчик просмотров
    PRIMARY KEY ((view_date), product_id, view_hour)
);

-- Материализованное представление для топ-товаров
CREATE MATERIALIZED VIEW top_products_daily AS
    SELECT view_date, product_id, SUM(view_count) as total_views
    FROM product_views
    WHERE view_date IS NOT NULL AND product_id IS NOT NULL
    PRIMARY KEY ((view_date), product_id)
    WITH CLUSTERING ORDER BY (product_id DESC);
```

**Обоснование:**
- **Counter** для атомарного инкремента
- Две гранулярности: по часам (детально) и по дням (агрегаты)
- Материализованное представление для топ-запросов

---

## Задание 10.3: Стратегии обеспечения целостности данных

### 10.3.1. Выбор стратегий по сущностям

| Сущность          | Hinted Handoff | Read Repair         | Anti-Entropy | Обоснование                                                            |
|-------------------|----------------|---------------------|--------------|------------------------------------------------------------------------|
| **order_history** | Always         | Probabilistic (10%) | Weekly       | История некритична к консистентности, важна скорость записи            |
| **user_sessions** | Always         | Probabilistic (50%) | No           | Сессии tolerate eventual consistency, read repair для активных         |
| **audit_events**  | Always         | No                  | No           | Аудит не требует строгой консистентности, важна пропускная способность |
| **product_views** | Always         | No                  | No           | Счетчики, eventual consistency допустима                               |

### 10.3.3. Настройка уровней согласованности

```sql
-- Для истории заказов: быстрая запись
INSERT INTO order_history ... 
WITH CONSISTENCY = QUORUM;  -- Гарантия записи на большинство

-- Для сессий: высокая доступность
SELECT * FROM user_sessions 
WHERE session_id = ? 
WITH CONSISTENCY = LOCAL_QUORUM;  -- Чтение из локального DC

-- Для аудита: максимальная скорость
INSERT INTO audit_events ... 
WITH CONSISTENCY = ANY;  -- Запись хотя бы на один узел
```

### 10.3.4. Пример настройки ремонтов

```bash
# Еженедельный full repair для критичных данных
nodetool repair order_history --full

# Инкрементальный repair для больших таблиц
nodetool repair audit_events --inc

# Проверка статуса ремонтов
nodetool repair_admin
```

### 10.3.5. Мониторинг целостности

```sql
-- Проверка расхождений в данных
SELECT * FROM system.repairs;

-- Мониторинг hinted handoff
SELECT * FROM system.hints;
```

---

## Заключение

### Преимущества предложенного решения:

1. **Масштабируемость**: линейное расширение без решардинга
2. **Отказоустойчивость**: leaderless архитектура
3. **Геораспределение**: multi-DC репликация
4. **Экономия**: автоматическое удаление данных через TTL

### Риски и митигации:

| Риск | Митигация |
|------|-----------|
| Горячие партиции | Равномерные partition keys, композитные ключи |
| Потеря данных при repair | Регулярные backup, commitlog |
| Сложность моделирования | Денормализация, CQRS паттерн |

### Итоговые рекомендации:

1. **Мигрировать в Cassandra**:
    - История заказов
    - Сессии пользователей
    - Аудит и события
    - Аналитика просмотров

2. **Оставить в MongoDB**:
    - Активные корзины
    - Остатки товаров
    - Каталог продуктов

3. **Гибридная архитектура**:
    - MongoDB для транзакционных данных
    - Cassandra для time-series и высоких нагрузок
    - Синхронизация через событийную шину