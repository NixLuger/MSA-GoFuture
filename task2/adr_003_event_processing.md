# ADR-003: Обработка событий (Flink jobs)

## Контекст

Определены 20 доменных событий (ADR-001) и схема топиков Kafka с региональным
партиционированием (ADR-002). Требуется спроектировать слой потоковой
обработки для реализации динамического ценообразования, матчинга водителей,
streaming-fraud-детекции и аналитики в реальном времени.

Уже зафиксировано:
- Kafka — Event Bus, 27 events-топиков + 27 DLQ = 54 (ADR-002).
- PyFlink — выбранный движок потоковой обработки (обоснование — раздел 1.1).
- Use cases: Dynamic Pricing, Driver Matching, Fraud streaming, Analytics.
- 500 000 конкурентных поездок, 200 000+ активных водителей (ADR-001 Task1).

## Требования

1. Спроектировать Flink jobs для каждого use case.
2. Определить consumer groups и их scaling.
3. Обеспечить exactly-once processing.
4. Описать stateful-обработку (окна, джойны, state).
5. Определить checkpointing / savepoints.
6. Не дублировать решения из ADR-001/ADR-002.

## Решение

### 1. Архитектура: 4 независимых Flink job'а

**Решение:** отдельный Flink job на каждый use case.

| Job | Назначение | Приоритет |
|-----|------------|-----------|
| `pricing-streaming-job` | Динамическое ценообразование в реальном времени | **Critical** |
| `matching-streaming-job` | Матчинг водителей с заказами | **Critical** |
| `fraud-streaming-job` | Streaming-скоринг транзакций | **Critical** |
| `analytics-streaming-job` | Агрегации для Data Platform | Important |

**Обоснование:**
- **Независимое масштабирование:** pricing и matching требуют больше ресурсов, чем fraud и analytics.
- **Независимый деплой:** обновление fraud не роняет pricing.
- **Изоляция отказов:** падение одного job'а не влияет на остальные.
- **Соответствует микросервисному подходу** (ADR-002 Task1).
- **Разные SLA:** pricing — p99 < 100 мс; analytics — p99 < 5 сек.

#### 1.1 Выбор движка: PyFlink

**Сравнение кандидатов:**

| Критерий | PyFlink | Kafka Streams | Spark Streaming |
|----------|---------|---------------|-----------------|
| Стек | **Python** (соответствует ADR-002 Task1) | Java / Kotlin | Scala / Java / PySpark |
| Модель обработки | True streaming (мс) | True streaming (мс) | Micro-batch (сек) |
| Stateful-обработка | RocksDB, окна, joins | Только Kafka state store | Микро-батчи с state |
| Event time + watermarks | ✅ Сильнее | ✅ Есть | ✅ Есть |
| Гибкость источников | Kafka + JDBC + Kinesis + ... | **Только Kafka** | Kafka + files + JDBC |
| Exactly-once | ✅ Checkpointing + 2PC | ✅ Транзакции | ✅ Идемпотентные sinks |
| Сложность эксплуатации | Средняя (K8s operator) | Низкая (library) | Высокая (кластер) |

**Почему PyFlink:**

1. **Python-стек.** В Task1 ADR-002 зафиксирован Python для всех сервисов. PyFlink — единственный зрелый streaming-движок с Python API. Kafka Streams — Java-only (нет PyFlink-аналога), Spark Streaming — PySpark требует отдельного кластера и JVM-стека.
2. **Stateful-обработка.** Matching требует хранения состояния всех активных водителей (RocksDB); pricing — оконные агрегации по зонам; fraud — история транзакций за 30 дней. PyFlink даёт всё это «из коробки».
3. **Event time + watermarks.** Критично для корректных окон pricing и matching при задержках сети между регионами (Task3). Watermark strategy 5 сек — см. раздел 7.
4. **Латентность.** Pricing (p99 < 100 мс) и matching (p99 < 500 мс) несовместимы с micro-batch моделью Spark Streaming (секундная латентность).
5. **Гибкость источников.** Fraud-job обогащается справочниками из PostgreSQL — Flink поддерживает JDBC-коннекторы. Kafka Streams — только Kafka.
6. **Exactly-once.** Checkpointing + two-phase commit sinks (Kafka transactional producer, ClickHouse idempotent sink) — см. раздел 5.

**Почему не Spark Streaming:** micro-batch даёт секундную латентность — не подходит для real-time pricing/matching. Плюс PySpark требует отдельного кластера, JVM-стека и не даёт той же гибкости stateful-обработки.

**Почему не Kafka Streams:** Java/Kotlin (не соответствует Python-стеку Task1 ADR-002); привязка к Kafka как единственному источнику (нужно обогащение из PostgreSQL); отсутствие Python-аналога.

**Почему не custom Python-приложение:** пришлось бы писать с нуля checkpointing, watermarks, stateful-обработку, exactly-once sinks — это месяцы работы и огромный операционный риск. Flink — зрелое решение с 10+ годами production-эксплуатации.

### 2. Naming convention

**Flink job ID:** `<domain>-streaming-job`

**Consumer group:** `<domain>-streaming-cg-v<version>`

Пример: `pricing-streaming-cg-v1`, `matching-streaming-cg-v1`.

**Version в consumer group** — для безопасного обновления: новый job стартует
с новой группой, читает с `earliest` или `latest`, затем старая группа
выводится из эксплуатации.

### 3. Детальное описание Flink jobs

#### 3.1 `pricing-streaming-job` (Critical)

| Параметр | Значение |
|----------|----------|
| **Input** | `*.booking.events` (Booking.Created), `*.driver.location.events` (для предложения) |
| **Output** | `*.pricing.events` (Pricing.PriceCalculated) |
| **State** | Скользящее окно 5 мин по спросу/предложению на зону |
| **Processing** | Агрегация спроса (кол-во booking requests) и предложения (кол-во available drivers) по зонам; расчёт коэффициента; умножение на базовый тариф |
| **Parallelism** | 16 (по зонам через `keyBy(zone_id)`) |
| **Checkpoint interval** | 10 сек |
| **Exactly-once** | ✅ (checkpoint + idempotent sink) |
| **SLA** | p99 < 100 мс от `Booking.Created` до `Pricing.PriceCalculated` |

**Окна:**
- **Tumbling window 1 мин** — для быстрых изменений (пиковые нагрузки).
- **Sliding window 5 мин / slide 1 мин** — для сглаживания.

**Ключ партиционирования:** `zone_id` (зона — часть региона).

#### 3.2 `matching-streaming-job` (Critical)

| Параметр | Значение |
|----------|----------|
| **Input** | `*.booking.events` (Booking.Created), `*.driver.location.events` (Driver.LocationUpdated), `*.driver.events` (Driver.StatusChanged) |
| **Output** | `*.booking.events` (Booking.Assigned) |
| **State** | Текущее состояние всех активных водителей (позиция, статус, доступность) |
| **Processing** | Джойн Booking.Created с актуальным state водителей; выбор кандидатов в радиусе; учёт «умного» перераспределения (избегание горячих зон) |
| **Parallelism** | 32 (по регионам + зонам) |
| **Checkpoint interval** | 5 сек |
| **Exactly-once** | ✅ |
| **SLA** | p99 < 500 мс от `Booking.Created` до `Booking.Assigned` |

**Ключевой алгоритм:** «умное перераспределение» (бизнес-требование из кейса):
- Не просто ближайший водитель, а **ожидание подходящего** водителя.
- Учёт «горячих зон» (где концентрация водителей слишком высокая).
- Приоритет — равномерное распределение по зонам.

**State backend:** RocksDB (state может быть большим — все активные водители).

#### 3.3 `fraud-streaming-job` (Critical)

| Параметр | Значение |
|----------|----------|
| **Input** | `*.payment.events` (Payment.*), `*.booking.events` (Booking.Created, Booking.Completed) |
| **Output** | `*.fraud.events` (Fraud.CheckCompleted, Fraud.AlertRaised) |
| **State** | История транзакций пользователя (последние 30 дней, ключ — `user_id`), velocity-счётчики |
| **Processing** | Применение ML-модели (загружается из ML Platform) + правила (velocity, geography anomaly, amount anomaly) |
| **Parallelism** | 16 |
| **Checkpoint interval** | 10 сек |
| **Exactly-once** | ✅ |
| **SLA** | p99 < 200 мс |

**ML-модель:**
- Обучается в Data Platform (ADR-002 Task1).
- Inference — внутри job'а (модель загружается при старте + обновляется по событию `ModelUpdated`).
- Feature store: онлайн-фичи из state (velocity, история).

**Правила:**
- Velocity: > 5 транзакций / 5 мин.
- Geography: транзакция из неожиданной локации (> 100 км от последней).
- Amount: > 3σ от среднего по пользователю.

#### 3.4 `analytics-streaming-job` (Important)

| Параметр | Значение |
|----------|----------|
| **Input** | Все `*.events` (подписка на все домены) |
| **Output** | `*.analytics.events` (агрегаты) + запись в Data Platform (ClickHouse) |
| **State** | Агрегаты по окнам (1 мин, 1 час, 1 день) |
| **Processing** | Агрегация: bookings/min, revenue/min, drivers_online, cancellations и т.д. |
| **Parallelism** | 8 |
| **Checkpoint interval** | 30 сек |
| **Exactly-once** | ✅ (ClickHouse sink с идемпотентностью) |
| **SLA** | p99 < 5 сек (не critical) |

**Запись в ClickHouse:** через Kafka sink connector или custom sink с
идемпотентностью по `(event_id, window_start)`.

### 4. Consumer groups и scaling

| Job | Consumer group | Партиции input | Parallelism | Task managers |
|-----|----------------|----------------|-------------|---------------|
| Pricing | `pricing-streaming-cg-v1` | 50 (booking) + 100 (location) | 16 | 4 × 4 slots |
| Matching | `matching-streaming-cg-v1` | 50 (booking) + 100 (location) + 50 (driver) | 32 | 8 × 4 slots |
| Fraud | `fraud-streaming-cg-v1` | 30 (payment) + 50 (booking) | 16 | 4 × 4 slots |
| Analytics | `analytics-streaming-cg-v1` | все топики | 8 | 2 × 4 slots |

**Правило:** parallelism ≤ количество партиций input-топика (иначе часть
task'ов простаивает).

**Scaling:**
- **Вертикально:** увеличение slots per task manager.
- **Горизонтально:** добавление task managers (Kubernetes operator Flink).
- **Auto-scaling:** на основе `kafka_consumer_lag` (метрика из ADR-006).

### 5. Exactly-once processing

**Flink checkpointing:**
- **Interval:** 5–30 сек (зависит от SLA job'а).
- **Mode:** exactly-once.
- **Storage:** S3 / MinIO (persistent).
- **Timeout:** 10 мин (если checkpoint не завершён — job падает).

**Sink exactly-once:**

| Sink | Как обеспечиваем |
|------|------------------|
| Kafka producer | `transactional.id` + `isolation.level=read_committed` |
| ClickHouse | Идемпотентность по `(event_id, window_start)` + `ReplacingMergeTree` |
| PostgreSQL (если нужно) | Идемпотентные upsert по `event_id` |

**Two-phase commit:**
- Flink checkpoint barrier → sink готовит транзакцию.
- Checkpoint завершён → sink коммитит.
- Checkpoint failed → sink откатывает.

### 6. Savepoints и обновления

**Savepoint** — снимок state job'а, используется для:
- **Обновления job'а** (новый код + restore из savepoint).
- **Миграции** (переезд на другой кластер).
- **Rollback** (если новая версия сломана).

**Стратегия обновлений:**
1. Остановить job с savepoint.
2. Задеплоить новую версию кода.
3. Запустить job с restore из savepoint.
4. Проверить метрики (5–10 мин).
5. Если плохо — restore из старого savepoint.

**Savepoint перед каждым production-деплоем** — обязательно.

### 7. Watermarks и event time

**Event time** — используем для всех job'ов (не processing time).

**Watermark strategy:**
- **Bounded out-of-orderness:** 5 сек (события приходят с задержкой из-за сети/региона).
- **Idle source timeout:** 30 сек (если партиция «молчит», не блокируем окна).

**Зачем:** корректная агрегация в окнах даже при задержках. Для `Driver.LocationUpdated` (свежие данные) — допускаем out-of-orderness 2 сек.

### 8. Resource management

| Job | CPU | Memory | Storage (state) |
|-----|-----|--------|-----------------|
| Pricing | 16 cores | 32 GB | 10 GB (RocksDB) |
| Matching | 32 cores | 64 GB | 100 GB (RocksDB — все водители) |
| Fraud | 16 cores | 32 GB | 50 GB (RocksDB) |
| Analytics | 8 cores | 16 GB | 20 GB |

**Итого:** ~72 cores, 144 GB RAM, 180 GB state storage.

**State backend:** RocksDB (для больших state), HashMap (для маленьких).

### 9. Что НЕ входит в это решение

- **Детальный дизайн Kafka-топиков** — ADR-002.
- **Saga (orchestration/compensation)** — ADR-004.
- **Reliable delivery (retry, DLQ, outbox)** — ADR-005.
- **Мониторинг Flink** — ADR-006.
- **ML-модели (обучение, feature store)** — Task4.
- **Flink-кластер (hardware, K8s operator)** — инфраструктурное решение.

### 10. Эволюция относительно Task1

В Task1 C2_FinalState `pricing`, `fraud`, `matching`, `analytics` были
обозначены как FastAPI-сервисы. В Task2 детализируется: real-time обработка
вынесена в отдельные Flink jobs, FastAPI-сервисы остаются как API-фасады
и для batch-операций.

**Изменения относительно Task1 C2_FinalState:**

- **Добавлено:** 4 Flink job'а (`pricing-streaming`, `matching-streaming`,
  `fraud-streaming`, `analytics-streaming`), Apicurio Registry как отдельный
  компонент, retry-топики, DLQ.
- **Уточнено:**
    - `svc_pricing` — API + БД тарифов. Расчёт цены → `flink_pricing`.
    - `svc_fraud` — API управления правилами. Streaming-скоринг → `flink_fraud`.
    - `svc_booking` — API + Saga orchestrator. Матчинг → `flink_matching`.
    - `svc_analytics` — read-path API для BI. Агрегации → `flink_analytics`.
- **Producer для отдельных событий уточнён:**
    - `Pricing.PriceCalculated` → producer = `flink_pricing`.
    - `Fraud.CheckCompleted`, `Fraud.AlertRaised` → producer = `flink_fraud`.
    - `Booking.Assigned` → producer = `flink_matching`.
    - `Booking.Created`, `Booking.Cancelled`, `Booking.Completed` → producer = `svc_booking`.
- **ML Platform:** модели обучаются в ML Platform и деплоятся
  в `flink_pricing` / `flink_fraud` через Model Registry.

**Не участвуют в EDA (осознанно):**

- **`svc_geography`** — работает через sync REST (геопоиск, маршруты),
  не публикует и не подписывается ни на одно событие каталога ADR-001.
- **`spark`** — batch-обработка (исторические агрегации, ML training),
  предмет Task4 (Data Platform).
- **`svc_analytics`** — read-path API для BI, читает из ClickHouse,
  не участвует в event-driven контуре.
- **`bi`, все client apps, BFF, `keycloak`, `ff`** — sync-слой,
  не участвуют в EDA.
- **Монолит** — на момент Task2 монолит уже мигрирован (Task1 ADR-003/004),
  его роль как producer заменена новыми сервисами. На Transition State
  монолит был producer через outbox (ADR-004 Task1), но к целевому
  состоянию он исчезает.

## Альтернативы

1. **Один большой Flink job на всё** — отклонено: невозможно независимо
   масштабировать, деплой одного use case роняет остальные.
2. **Kafka Streams вместо Flink** — отклонено: Java-only (стек Python),
   нет PyFlink-аналога. Подробнее — раздел 1.1.
3. **Batch-обработка вместо streaming** — отклонено: не соответствует
   требованию «реального времени» (динамическое ценообразование, fraud).
4. **Processing time вместо event time** — отклонено: некорректные
   агрегации при задержках, что критично для pricing и fraud.
5. **At-least-once вместо exactly-once** — отклонено: для Critical-событий
   (payment, fraud) дубликаты недопустимы.

## Компромиссы

- **4 отдельных job'а** — больше инфраструктуры, чем один job. Митигация:
  единый Flink operator (Kubernetes), GitOps для деплоя.
- **RocksDB state** — быстрый, но требует дискового I/O. Митигация: SSD,
  тюнинг RocksDB (block cache, write buffer).
- **Checkpoint interval 5 сек для matching** — частая нагрузка на storage.
  Митигация: incremental checkpoints (Flink 1.15+).
- **Exactly-once с Kafka transactional producer** — латентность выше,
  чем at-least-once (~10–20 мс). Митигация: только для Critical job'ов;
  analytics — at-least-once + идемпотентность в sink.
- **Watermark 5 сек** — окна закрываются с задержкой. Митигация: для
  pricing используем «ранние» эмиссии (early firing) — промежуточные
  результаты, затем финальные.
- **ML-модель в fraud-job** — при обновлении модели нужен restart.
  Митигация: периодическая перезагрузка по `ModelUpdated`-событию,
  без полного restart job'а.