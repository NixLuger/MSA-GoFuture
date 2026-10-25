# ADR-002: Схема топиков Kafka

## Контекст

Определён каталог из 20 доменных событий (ADR-001), которые публикуются
в Kafka. Требуется детальный дизайн топиков: naming, партиционирование
по регионам, retention, compaction, replication, управление схемами.

Уже зафиксировано:
- 20 integration events (ADR-001).
- Все события содержат `region` в envelope (ADR-001).
- Kafka — единственная шина событий, RabbitMQ удаляется (ADR-003 Task1).
- 500 000 конкурентных поездок, 200 000+ активных водителей (ADR-001 Task1).

## Требования

1. Схема топиков с партиционированием по регионам.
2. Retention и compaction-политика по категориям событий.
3. Количество партиций под целевой throughput.
4. Replication и durability.
5. Управление схемами (Apicurio Registry).
6. ACL и quotas.
7. Не дублировать решения из ADR-001.

## Решение

### 1. Naming convention топиков

**Формат:** `<region>.<domain>.<category>`

| Часть | Значение | Пример |
|-------|----------|--------|
| `region` | `region-a`, `region-b`, `region-c` | `region-a` |
| `domain` | Домен-владелец (booking, driver, payment, payout, pricing, fraud, notification, platform) | `booking` |
| `category` | Тип события (events, commands, dlq) | `events` |

**Примеры:**
- `region-a.booking.events`
- `region-b.payment.events`
- `region-c.driver.events`

**DLQ:** `<region>.<domain>.dlq` — для сообщений, которые не удалось обработать (см. ADR-005 Reliable Delivery).

### 2. Региональное партиционирование (Вариант A)

**Решение:** региональные топики — каждый регион в своём наборе топиков.

**Обоснование:**
- **Data residency:** данные резидентов региона остаются в регионе (требование Task3, комплаенс).
- **Изоляция отказов:** проблема в `region-a` не влияет на `region-b`.
- **Проще failover:** можно восстановить один регион независимо (Task3).
- **Capacity planning:** у каждого региона свой throughput, проще масштабировать.
- **Соответствует бизнес-требованию:** «горизонтальное масштабирование в нескольких регионах» (ADR-001 Task1).

**Количество топиков:**
Топики — per-domain (один топик на домен на регион), не per-event.
9 типов топиков: booking, driver, driver.location, pricing, payment,
payout, notification, fraud, platform.

- 9 типов × 3 региона = **27 events-топиков**.
- + 9 × 3 = **27 DLQ-топиков**.
- **Итого: 54 топика** (не считая retry-топиков из ADR-005 — ещё 81).

**Управление:** топики создаются через GitOps (Terraform + Kafka operator), не вручную.

### 3. Retention и compaction по категориям

| Категория | Retention | Cleanup policy | Топики |
|-----------|-----------|----------------|--------|
| **Critical** | 30 дней | `delete` | `payment.events`, `payout.events` |
| **Important** | 14 дней | `delete` | `booking.events`, `driver.events`, `fraud.events`, `pricing.events`, `platform.events` |
| **Informational** | 72 часа | `delete` | `notification.events` |
| **Streaming (compaction)** | ∞ (последнее состояние) | `compact` | `driver.location.events` |
| **DLQ** | 30 дней | `delete` | `<region>.<domain>.dlq` |

**Обоснование:**

- **Critical (30 дней):** финансовые события — нужны для аудита, reconciliation, разбора инцидентов. 30 дней — стандарт для финансового аудита.
- **Important (14 дней):** replay на случай багов consumer'ов — 2 недели достаточно, чтобы починить и переиграть.
- **Informational (72 часа):** аналитика и уведомления — не критичны, 3 дня достаточно для разбора инцидентов.
- **Compaction для Driver.LocationUpdated:** это не история, а «последнее состояние водителя». Compaction по ключу `driver_id` гарантирует, что в топике останется одно (последнее) сообщение на водителя. Consumer'ы при старте читают все последние состояния.
- **DLQ (30 дней):** чтобы успеть разобрать «застрявшие» сообщения.

### 4. Количество партиций

**Формула:** `partitions = ceil(peak_throughput / target_partition_throughput)`

**Assumptions:**
- target_partition_throughput ~ 500 events/sec (для нашего профиля событий).
- peak_throughput = 2× average (пиковые нагрузки).

| Топик | Партиций | Обоснование |
|-------|----------|-------------|
| `driver.location.events` | **100** | 200k водителей × 1 update / 5 сек ≈ 40k events/sec |
| `booking.events` | **50** | 500k поездок × несколько событий на поездку |
| `payment.events` | **30** | Меньше, но критичные; с запасом под пики |
| `payout.events` | **20** | Батчевые выплаты (раз в сутки) |
| `pricing.events` | **20** | 1 расчёт на поездку + динамические обновления |
| `fraud.events` | **20** | Проверки на транзакции и поездки |
| `notification.events` | **10** | Не высоконагруженный |
| `platform.events` | **10** | Редкие события (tenants, users) |
| `driver.events` | **20** | StatusChanged + VerificationCompleted (низкий throughput) |

> **Важно:** это **рабочие оценки (assumption)**. Точные значения определяются
> нагрузочным тестированием на staging перед production. Партиции можно
> увеличивать (Kafka поддерживает), уменьшать — нет.

**Partition key:**
- Для большинства топиков — `<aggregate_id>` (например, `booking_id`), чтобы события одной сущности шли в одну партицию (порядок гарантирован).
- Для `driver.location.events` — `driver_id`.
- Для `platform.events` — `tenant_id`.

### 5. Replication и durability

| Параметр | Значение | Обоснование |
|----------|----------|-------------|
| `replication.factor` | **3** | Стандарт для production; выдерживает потерю 2 брокеров |
| `min.insync.replicas` | **2** | Producer получает ack только если запись на ≥ 2 репликах |
| `acks` (producer) | **all** | Максимальная durability, критично для Critical-событий |
| `enable.idempotence` (producer) | **true** | Защита от дублей на уровне producer |
| `unclean.leader.election` | **false** | Не теряем данные даже при отказе лидера |

**Кросс-региональная репликация:** **не используем** Kafka MirrorMaker между регионами. Каждый регион — изолированный кластер. Обмен между регионами — через API (если нужно), а не через Kafka. Обоснование: data residency + изоляция отказов.

### 6. Управление схемами (Apicurio Registry)

| Параметр | Значение |
|----------|----------|
| **Registry** | Apicurio Registry (self-hosted) |
| **Format** | Avro (основной), JSON Schema (для быстрых прототипов) |
| **Compatibility** | `BACKWARD` (по умолчанию), `FULL` для Critical |
| **Subject naming** | `<region>.<domain>.<event_type>-value` |
| **Schema evolution** | Только backward-compatible изменения (добавление optional полей) |
| **CI-проверка** | Schema lint + compatibility check в GitLab CI (ADR-003 Task1) |

**Почему Avro:** компактность, строгая схема, лучшая поддержка в Apicurio, зрелость.

**Почему `FULL` для Critical:** финансовые события не должны ломать ни producer, ни consumer при эволюции.

### 7. ACL и quotas

**ACL (Kafka ACLs):**

| Producer | Может писать в          | Обоснование |
|----------|-------------------------|-------------|
| `booking-service` | `*.booking.events`      | Владелец домена |
| `driver-status-service` | `*.driver.events`, `*.driver.location.events`     | Владелец домена |
| `payment-service` | `*.payment.events`      | Владелец домена |
| `payout-service` | `*.payout.events`       | Владелец домена |
| `pricing-service` | `*.pricing.events`      | Владелец домена |
| `fraud-service` | `*.fraud.events`        | Владелец домена |
| `notification-service` | `*.notification.events` | Владелец домена |
| `tenant-service`, `identity-service` | `*.platform.events`     | Владельцы домена |

**1 producer = 1 домен = 1 набор топиков** — жёсткая изоляция (ADR-001).

**Consumer groups:** ACL на чтение — по consumer groups (см. ADR-003 Event Processing).

**Quotas (per client):**
- Producer byte-rate: 10 МБ/сек (по умолчанию).
- Consumer byte-rate: 20 МБ/сек.
- Request rate: 1000 запросов/сек.
- **Приоритет для Critical:** отдельные квоты для `payment-service` / `payout-service` (×2 от обычных).

### 8. Сводная схема топиков

**Итого: 54 топика** (9 типов × 3 региона × 2 — events + DLQ), не считая retry-топиков (81, см. ADR-005).

**Пример для `region-a`:**

| Топик | Partitions | Retention | Cleanup |
|-------|------------|-----------|---------|
| `region-a.booking.events` | 50         | 14 дней | delete |
| `region-a.driver.events` | 20         | 14 дней | delete |
| `region-a.driver.location.events` | 100        | ∞ | compact |
| `region-a.pricing.events` | 20         | 14 дней | delete |
| `region-a.payment.events` | 30         | 30 дней | delete |
| `region-a.payout.events` | 20         | 30 дней | delete |
| `region-a.notification.events` | 10         | 72 часа | delete |
| `region-a.fraud.events` | 20         | 14 дней | delete |
| `region-a.platform.events` | 10         | 14 дней | delete |
| `region-a.booking.dlq` | 10         | 30 дней | delete |
| ... (аналогично для остальных DLQ) |            | | |

> **Замечание:** `driver.location.events` — отдельный топик, потому что
> у него особые retention/compaction и очень высокий throughput. Остальные
> driver-события (StatusChanged, VerificationCompleted) — в общем
> `driver.events`.

### 9. Что НЕ входит в это решение

- **Дизайн consumer-групп и их scaling** — ADR-003 (Event Processing).
- **Saga-топики (компенсации, оркестрация)** — ADR-004 (Saga).
- **Retry-топики и DLQ-механика** — ADR-005 (Reliable Delivery).
- **Kafka-кластеры (размеры, hardware)** — инфраструктурное решение.
- **MirrorMaker и межрегиональная репликация** — не используем (см. раздел 5).
- **Мониторинг Kafka** — ADR-006 (Monitoring).

## Альтернативы

1. **Глобальные топики + partition key по региону** — отклонено:
   нет изоляции по регионам, проблемы с data residency (Task3),
   failover сложнее.
2. **Один общий топик на все события** — отклонено: смешение категорий,
   невозможно применить разные retention/compaction.
3. **Kafka MirrorMaker между регионами** — отклонено: избыточная
   репликация, проблемы с data residency, усложняет failover.
4. **Confluent Schema Registry** — отклонено: vendor lock-in, платно
   для production.
5. **JSON Schema для всех событий** — отклонено: менее компактный, менее
   строгий, чем Avro. JSON Schema — только для быстрых прототипов.
6. **Compaction для всех driver-событий** — отклонено: `Driver.StatusChanged`
   и `Driver.VerificationCompleted` — история, а не состояние.

## Компромиссы

- **54 топика** — управляемое количество. Митигация: GitOps
  (Terraform + Kafka operator), автоматизация создания топиков.
- **Региональная изоляция** — нет кросс-региональных событий (например,
  пассажир в `region-a` не может заказать поездку в `region-b` через Kafka).
  Митигация: кросс-региональные операции — через API, не через Kafka.
  В будущем, если понадобится — отдельный глобальный топик для specific
  use cases.
- **Партиции — assumption** — могут быть неточными. Митигация: нагрузочное
  тестирование на staging, возможность увеличения (Kafka поддерживает).
- **Compaction `Driver.LocationUpdated`** — «бесконечное» хранение может
  привести к росту диска. Митигация: `min.cleanable.dirty.ratio`,
  `segment.ms` — тюнинг compaction; периодический мониторинг размера.
- **Avro-only** — менее удобно для ad-hoc debugging (бинарный формат).
  Митигация: `kafka-avro-console-consumer` + Apicurio UI для просмотра схем.
- **Отказ от MirrorMaker** — нет автоматического DR между регионами.
  Митигация: каждый регион — самодостаточный; восстановление — из
  собственных реплик (RF=3).