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


## 13) Какие методы и эндпоинты iiko Cloud API использовать

Ниже — практическая карта «метрика -> источник». Важно: **не все управленческие/складские отчеты доступны напрямую в iiko Cloud API (Transport)**. Для части показателей нужен iikoBiz/OLAP (или выгрузка из back-office), либо расчет в вашей БД из доступных первичных данных.

### 13.1 База для авторизации и справочников

- Авторизация: `POST /api/1/access_token`
- Организации: `POST /api/1/organizations`
- Группы терминалов/точки: `POST /api/1/terminal_groups`
- Типы оплат: `POST /api/1/payment_types`
- Скидки/надбавки: `POST /api/1/discounts`
- Номенклатура (блюда, категории): `POST /api/1/nomenclature`
- Сотрудники/курьеры: `POST /api/1/employees/info`, `POST /api/1/employees/couriers`

### 13.2 Продажи и чеки

- Получение заказов доставки по периоду/статусу:
  - `POST /api/1/deliveries/by_delivery_date_and_status`
  - `POST /api/1/deliveries/by_delivery_date_and_source_key_and_filter`
- Детали конкретного заказа:
  - `POST /api/1/order/by_id`

Что считать из этих данных в DWH:
- выручка;
- количество чеков/заказов;
- средний чек;
- скидки и налоги;
- продажи по блюдам/категориям/точкам/часам/дням/месяцам;
- продажи по сотрудникам (если в заказе есть нужные идентификаторы).

### 13.3 Доставка и SLA

- Основной источник: те же delivery endpoint’ы:
  - `POST /api/1/deliveries/by_delivery_date_and_status`
  - `POST /api/1/deliveries/by_delivery_date_and_source_key_and_filter`
- Дополнительно по персоналу:
  - `POST /api/1/employees/couriers`
  - `POST /api/1/employees/shift/by_courier`

Что считать:
- количество доставок;
- среднее время доставки;
- SLA и доля опозданий;
- выручка доставки;
- география (если сохраняете адрес/город/зону в своей витрине).

### 13.4 Что в Cloud API обычно закрывается частично или не закрывается

Для метрик ниже в большинстве проектов нужен **дополнительный источник** (iikoBiz/OLAP/внутренние отчеты back-office), а не только Transport API:

- приходы;
- списания;
- перемещения;
- складские движения;
- закупки;
- корректировки;
- себестоимость «в разрезе движений»;
- часть кассовых операций (в зависимости от вашей схемы учета и доступности данных).

Практический вариант:
1. Delivery/Order-аналитику строить через Cloud API endpoint’ы выше.
2. Склад/себестоимость/закупки тянуть из iikoBiz/OLAP-выгрузок в те же `stg_*` и сводить в единый DWH.

### 13.5 Соответствие методам в `pyiikocloudapi`

В этой библиотеке уже есть обертки над ключевыми endpoint’ами:

- `organizations()` -> `/api/1/organizations`
- `terminal_groups()` -> `/api/1/terminal_groups`
- `payment_types()` -> `/api/1/payment_types`
- `discounts()` -> `/api/1/discounts`
- `nomenclature()` -> `/api/1/nomenclature`
- `employees_info()` -> `/api/1/employees/info`
- `couriers()` -> `/api/1/employees/couriers`
- `order_by_id()` -> `/api/1/order/by_id`
- `by_delivery_date_and_status()` -> `/api/1/deliveries/by_delivery_date_and_status`
- `by_delivery_date_and_source_key_and_filter()` -> `/api/1/deliveries/by_delivery_date_and_source_key_and_filter`

Это удобный минимальный набор, чтобы стартовать с продаж и доставки.

## 14) Как получать данные из iikoBiz/OLAP

Ниже — рабочая схема для случаев, когда Transport (Cloud API) не покрывает склад/себестоимость/закупки в нужной глубине.

### 14.1 Что обычно берут из iikoBiz/OLAP

- складские движения (приход, списание, перемещение, корректировки);
- закупки/поставки;
- себестоимость и валовая маржа;
- расширенные кассовые и управленческие срезы;
- регламентные отчеты, которых нет в Transport API.

### 14.2 Базовые способы интеграции

Практически в проектах используют один из 3 подходов (или комбинацию):

1. **Регламентная выгрузка отчетов (CSV/XLSX) из iikoBiz/OLAP**
   - Настраивается набор отчетов и расписание выгрузки.
   - Файлы складываются в S3/MinIO/FTP/сетевую папку.
   - Ваш ETL подбирает новые файлы, парсит и грузит в `stg_biz_*`.

2. **API-выгрузка отчетов iikoBiz (если доступна в вашей инсталляции/тарифе)**
   - Вы вызываете endpoint формирования отчета.
   - Получаете `job_id`/идентификатор задачи.
   - Опрашиваете статус, затем скачиваете файл результата.
   - Дальше стандартно: `stg -> dwh -> mart`.

3. **Гибрид**
   - Delivery/Order в near-real-time из Cloud API.
   - Склад/себестоимость/закупки пакетно из iikoBiz/OLAP (каждые 1–24 часа).

> Важно: конкретные URL/контракты iikoBiz API могут отличаться по версии/окружению. Поэтому в проде лучше фиксировать «интеграционный контракт» (какой отчет, какие поля, какой формат, какая периодичность) и версионировать его в вашем репозитории.

### 14.3 Рекомендуемый pipeline для iikoBiz-выгрузок

1. **Export job**: инициировать отчет (или дождаться файла по расписанию).
2. **Landing**: сохранить исходник без изменений (`/landing/iikobiz/{report}/{dt}/...`).
3. **Staging**: распарсить в `stg_biz_*` + сохранить `raw_row` (JSONB).
4. **Normalize**: привести ключи (organization_id, point_id, product_id, employee_id, doc_id).
5. **Deduplicate**: ключ вида `source_report + source_doc_id + row_num + updated_at`.
6. **Merge to facts**: загрузить в `f_stock_movements`, `f_procurements`, `f_cost`.
7. **Rebuild marts**: пересчитать `m_stock_cost_turnover`, маржу и себестоимость.
8. **Quality checks**: сверка сумм/количества строк с итогами исходного отчета.

### 14.4 Минимальный формат служебной таблицы загрузок

```sql
CREATE TABLE IF NOT EXISTS etl_report_loads (
    id bigserial PRIMARY KEY,
    source_system text NOT NULL,             -- iiko_cloud / iikobiz
    report_name text NOT NULL,
    report_period_from timestamp,
    report_period_to timestamp,
    source_file_name text,
    source_file_checksum text,
    row_count integer,
    loaded_at timestamp NOT NULL DEFAULT now(),
    status text NOT NULL,                    -- success / failed / partial
    error_text text
);
```

### 14.5 Практика сверки (обязательно)

Для каждого отчета храните 3 контрольных значения:
- `rows_source` — строк в исходнике;
- `rows_loaded` — строк после staging;
- `amount_source` vs `amount_dwh` — контрольная сумма (выручка/себестоимость/приход).

Если расхождение выше порога (например, 0.1% или фиксированный лимит),
- помечайте загрузку `partial`/`failed`,
- отправляйте алерт в Telegram/Slack,
- не публикуйте витрину как «готовую».

### 14.6 Как связать с Telegram-ботом

- В Telegram отдавайте данные только из `mart`-таблиц.
- Для складских команд (`/stock_movements`, `/cogs`, `/procurements`) добавьте в ответ:
  - `data_freshness` (время последней успешной загрузки из iikoBiz),
  - `source` = `iikobiz_olap`.
- Если свежей загрузки нет — бот должен явно писать: «данные неактуальны, последняя успешная загрузка: ...».
