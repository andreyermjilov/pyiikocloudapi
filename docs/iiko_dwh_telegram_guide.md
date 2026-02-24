# Как построить сервер + БД для отчетов из iiko Cloud API с доступом через Telegram-бота

Ниже — практический план для production-решения: сбор данных из iiko Cloud API, хранение в своей БД, расчет метрик и выдача по командам в Telegram.

## 1) Целевая архитектура

- **Collector (ETL/ELT)**: периодически забирает данные из iiko Cloud API и складывает в staging-таблицы.
- **DWH слой (PostgreSQL)**: факты + измерения + агрегаты.
- **API-сервис (FastAPI)**: отдает готовые отчеты и выполняет SQL-запросы к витринам.
- **Telegram-бот**: принимает команды пользователя и запрашивает ваш API.
- **Планировщик**: APScheduler / Celery Beat / cron для регулярных загрузок.
- **Наблюдаемость**: логирование, retry, алерты, контроль лагов загрузки.

Минимальный production-стек:
- Python 3.11+
- PostgreSQL 14+
- Redis (для очередей/кеша, опционально)
- FastAPI + SQLAlchemy
- python-telegram-bot (или aiogram)
- Docker Compose (на старте) -> Kubernetes (по мере роста)

## 2) Какие данные и как тянуть из iiko

### Рекомендуемый подход

1. **Справочники** (организации, точки, сотрудники, меню, категории, скидки, типы оплат) — обновлять 1–4 раза в сутки.
2. **Операционные данные** (заказы, оплаты, доставки, статусы, курьеры, склады/движения) — инкрементально каждые 2–10 минут.
3. **Webhook-события** (если доступны под ваши сценарии) — принимать сразу и сохранять в event log.
4. **Ретро-перезагрузка** — отдельный джоб для backfill за произвольный период.

### Почему не считать метрики «на лету» из API iiko

- API ограничен лимитами и временем ответа.
- Исторические отчеты и сложные срезы будут медленные и дорогие.
- Вам нужна единая «правда» в своей БД для Telegram/BI/финмодели.

## 3) Модель данных (ядро)

Сделайте DWH по схеме **star schema**:

### Измерения (dimensions)
- `dim_date` (день, неделя, месяц, квартал, год, день недели, час)
- `dim_point` (точка/ресторан/терминал)
- `dim_employee` (сотрудник, роль)
- `dim_courier`
- `dim_product` (блюдо, категория)
- `dim_category`
- `dim_payment_type`
- `dim_discount_type`
- `dim_geo` (город/зона доставки/кластер)

### Факты (facts)
- `f_sales_checks` (чек/заказ: сумма, налоги, скидки, статус, канал)
- `f_sales_items` (строки чека: блюдо, qty, сумма, себестоимость)
- `f_delivery` (доставка: время назначения/вручения, SLA, опоздание)
- `f_cash_ops` (внесения/изъятия/кассовые операции)
- `f_stock_movements` (приход, списание, перемещение, корректировка)
- `f_procurements` (закупки)

### Агрегаты/витрины (marts)
- `m_revenue_hour_day_month`
- `m_avg_check`
- `m_discounts_taxes`
- `m_sales_by_product_category`
- `m_sales_by_employee_point`
- `m_delivery_sla_geo`
- `m_stock_cost_turnover`

## 4) Метрики из вашего списка: где считать

- **Выручка, чеки, средний чек, скидки, налоги** -> `f_sales_checks` + `m_*`.
- **Продажи по блюдам/категориям** -> `f_sales_items`.
- **Продажи по сотрудникам/точкам/часам/дням/месяцам** -> `f_sales_checks` + `dim_*`.
- **Приходы/списания/перемещения/корректировки** -> `f_stock_movements`.
- **Себестоимость в разрезе движений** -> `f_sales_items` + `f_stock_movements` (или отдельный cost-layer).
- **Закупки** -> `f_procurements`.
- **Доставки, среднее время, курьеры, SLA, опоздания, выручка доставки, география** -> `f_delivery` + `dim_courier` + `dim_geo`.

## 5) Инкрементальная загрузка (критично)

Для каждого источника храните:
- `last_successful_cursor` (timestamp/id)
- `last_attempt_at`
- `last_error`

Правила:
- Делайте загрузку с **overlap-окном** (например, последние 15 минут), чтобы поймать опоздавшие изменения.
- Используйте **idempotent upsert** (`ON CONFLICT ... DO UPDATE`).
- Добавьте дедупликацию по `source_id + updated_at`.
- Всегда храните raw payload (JSONB) в staging для аудита.

## 6) Пример минимального пайплайна на Python

1. Забираете данные из iiko (через `pyiikocloudapi`).
2. Пишете в `stg_*` таблицы (raw JSON + служебные поля).
3. Трансформируете в `f_*` и `dim_*`.
4. Обновляете `m_*` витрины (матвью или инкремент).
5. Telegram запрашивает только `m_*`/подготовленные SQL.

## 7) Контур Telegram-бота

Команды:
- `/revenue today point=all`
- `/avg_check 2026-01-01 2026-01-31`
- `/sales_by_product week`
- `/delivery_sla yesterday city=KZN`
- `/stock_movements month`

Практика:
- Бот **не ходит в iiko напрямую**.
- Бот ходит только в ваш API (JWT/API key + rate limit).
- Тяжелые отчеты -> асинхронно: «принял задачу» + отправка файла позже.

## 8) Пример SQL-метрик

```sql
-- Выручка, чеки, средний чек по дням
SELECT
  business_date,
  SUM(net_revenue) AS revenue,
  COUNT(DISTINCT check_id) AS checks,
  SUM(net_revenue) / NULLIF(COUNT(DISTINCT check_id), 0) AS avg_check
FROM f_sales_checks
WHERE business_date BETWEEN :date_from AND :date_to
GROUP BY business_date
ORDER BY business_date;
```

```sql
-- SLA доставки
SELECT
  business_date,
  COUNT(*) AS deliveries,
  AVG(delivery_minutes) AS avg_delivery_minutes,
  AVG(CASE WHEN is_late THEN 1 ELSE 0 END)::numeric(10,4) AS late_ratio
FROM f_delivery
WHERE business_date BETWEEN :date_from AND :date_to
GROUP BY business_date
ORDER BY business_date;
```

## 9) Безопасность и эксплуатация

- Секреты через env/vault, не в git.
- Ограничить IP и роли БД (read-only для бота).
- Retry с exponential backoff для iiko API.
- Dead-letter очередь для «битых» событий.
- Мониторинги: freshness, row count anomaly, ETL duration, API error rate.
- Бэкапы БД + тест восстановления.

## 10) Пошаговый план запуска (2–4 недели)

1. Поднять PostgreSQL + schema `stg/dwh/mart`.
2. Реализовать загрузку: организации, точки, меню, сотрудники.
3. Реализовать инкремент заказов/чеков/доставок.
4. Добавить складские движения и закупки.
5. Собрать первые витрины: revenue/checks/avg_check, delivery SLA.
6. Поднять FastAPI с 10–15 готовыми endpoint-отчетами.
7. Подключить Telegram-бота к API.
8. Добавить алерты и дашборд мониторинга.
9. Провести тест нагрузки и сверку цифр с iiko отчетами.

## 11) Минимальная структура проекта

```text
project/
  app/
    api/                 # FastAPI endpoints
    bot/                 # Telegram handlers
    etl/
      collectors/        # iiko clients + extract
      transforms/        # stg -> dwh
      marts/             # refresh marts
    db/
      migrations/
      models/
      sql/
    core/
      config.py
      logging.py
  docker-compose.yml
  .env
```

## 12) Важные нюансы iiko-проектов

- Некоторые показатели могут зависеть от бизнес-правил (возвраты, отмены, сторно, время закрытия смены).
- Заранее согласуйте «словарь метрик» с бухгалтерией/операционкой.
- Зафиксируйте методологию: что считать выручкой (gross/net), куда относить налоги/доставку/скидки.

---

Если хотите, следующим шагом можно сделать **готовый шаблон**:
- SQL DDL для таблиц `stg_*`, `f_*`, `m_*`;
- FastAPI endpoint’ы под ваши команды бота;
- каркас Telegram-бота с 10 командами и выгрузкой CSV/XLSX.
