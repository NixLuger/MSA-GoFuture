# ADR-001: Каталог доменных событий

## Контекст

Переход на микросервисы (Task1) требует асинхронной коммуникации между
сервисами через Kafka. Событийная платформа должна обрабатывать 500 000
конкурентных поездок и обеспечивать динамическое ценообразование в реальном
времени.

Уже зафиксировано в Task1:
- Kafka — Event Bus (ADR-002/ADR-004 Task1).
- Монолит — только producer через outbox (ADR-004 Task1).
- 12 сервисов (10 бизнес + 2 платформенных).
- Feature flags и ACL (ADR-003/ADR-004 Task1).

Нужно определить **каталог integration events** — какие события публикуются
в Kafka, кто их producer, кто consumer, какие поля несут, как классифицируются.

## Требования

1. Определить список доменных событий (15–20 ключевых бизнес-значимых).
2. Зафиксировать naming convention, envelope, версионирование.
3. Для каждого события: producer, consumers, категория критичности.
4. Детальные схемы для критичных событий.
5. Не дублировать то, что уже зафиксировано в Task1.

## Решение

### 1. Naming convention

- **Past tense** — `BookingCreated`, не `CreateBooking`.
- **Namespace:** `<Domain>.<EventType>` — `Booking.Created`, `Payment.Succeeded`.
- **Версионирование:** через поле `event_version` (semver) в envelope,
  не через имя события (`BookingCreatedV2` — anti-pattern).
- **1 событие = 1 producer** — сервис-владелец домена. Остальные — только consumers.
- **Регион в envelope** — все события содержат `region`, независимо от
  схемы топиков (детали топиков — ADR-002).

### 2. Envelope события

Каждое событие в Kafka имеет единый envelope:

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "Booking.Created",
  "event_version": "1.0.0",
  "occurred_at": "2025-10-22T14:30:00.000Z",
  "producer": "booking-service",
  "tenant_id": "tenant-123",
  "region": "region-a",
  "correlation_id": "corr-456",
  "causation_id": "caus-789",
  "data": { }
}
```

| Поле | Назначение |
|------|------------|
| `event_id` | UUID, уникальный ID события (для дедупликации) |
| `event_type` | Namespace + тип события |
| `event_version` | Semver, для эволюции схемы |
| `occurred_at` | Когда произошло (ISO 8601, UTC) |
| `producer` | Кто опубликовал |
| `tenant_id` | Для multi-tenancy (Task5) |
| `region` | Для партиционирования по регионам |
| `correlation_id` | Сквозной ID бизнес-операции (для Saga) |
| `causation_id` | ID события-причины (для трассировки) |
| `data` | Payload события |

**Схемы хранятся в Apicurio Registry** (Avro / Protobuf / JSON Schema).
Backward-compatible эволюция — обязательна.

### 3. Классификация событий

| Категория | Потеря допустима? | Идемпотентность | Примеры |
|-----------|-------------------|-----------------|---------|
| **Critical** | ❌ Нет | Обязательна | `Payment.Succeeded`, `Payout.Completed` |
| **Important** | ❌ Нет (но не финансы) | Обязательна | `Booking.Created`, `Booking.Completed` |
| **Informational** | ✅ Да (аналитика, метрики) | Рекомендуется | `Driver.LocationUpdated` (высокочастотное) |

### 4. Каталог событий (20)

#### Booking (4 события)

| Событие | Producer | Consumers | Категория |
|---------|----------|-----------|-----------|
| `Booking.Created` | Booking | flink_pricing, flink_matching, flink_fraud, Notification, Analytics | Important |
| `Booking.Assigned` | flink_matching | Booking, Driver Status, Notification, Analytics | Important |
| `Booking.Cancelled` | Booking | Driver Status, Payments, Notification, Analytics | Important |
| `Booking.Completed` | Booking | Payments, Payouts, Notification, Analytics, flink_fraud | Important |

#### Driver (3 события)

| Событие | Producer | Consumers | Категория |
|---------|----------|-----------|-----------|
| `Driver.LocationUpdated` | Driver Status | flink_matching, Analytics | Informational |
| `Driver.StatusChanged` | Driver Status | flink_matching, Analytics | Important |
| `Driver.VerificationCompleted` | Driver Profile | Booking, Notification, Analytics | Important |

#### Pricing (1 событие)

| Событие | Producer | Consumers | Категория |
|---------|----------|-----------|-----------|
| `Pricing.PriceCalculated` | flink_pricing | Booking, Analytics | Important |

#### Payments (3 события)

| Событие | Producer | Consumers | Категория |
|---------|----------|-----------|-----------|
| `Payment.Succeeded` | Payments | Payouts, Booking, flink_fraud, Notification, Analytics | **Critical** |
| `Payment.Failed`    | Payments | Booking, flink_fraud, Notification, Analytics | **Critical** |
| `Payment.Refunded`  | Payments | Booking, flink_fraud, Notification, Analytics | **Critical** |

#### Payouts (3 события)

| Событие | Producer | Consumers | Категория |
|---------|----------|-----------|-----------|
| `Payout.Initiated` | Payouts | Notification, Analytics | **Critical** |
| `Payout.Completed` | Payouts | Notification, Analytics, svc_payments (Saga) | **Critical** |
| `Payout.Failed` | Payouts | Notification, Analytics, Payments (Saga orchestrator) | **Critical** |

#### Notification (2 события)

| Событие | Producer | Consumers | Категория |
|---------|----------|-----------|-----------|
| `Notification.Sent` | Notification | Analytics | Informational |
| `Notification.Failed` | Notification | Analytics, Fraud (аномалии) | Informational |

> **Примечание:** Notification — сервис, который **подписан на все доменные
> события** и решает сам, что отправить (routing rules внутри сервиса).
> Специальное событие `NotificationRequested` не нужно — это был бы лишний слой.

#### Fraud (2 события)

| Событие | Producer | Consumers | Категория |
|---------|----------|-----------|-----------|
| `Fraud.CheckCompleted` | flink_fraud | Booking, Payments | Important |
| `Fraud.AlertRaised` | flink_fraud | Notification, Analytics, Admin | Important |

#### Platform (2 события)

| Событие | Producer | Consumers | Категория |
|---------|----------|-----------|-----------|
| `Tenant.Onboarded` | Tenant Management | Identity, Notification, Analytics | Important |
| `User.Registered` | Identity | Tenant Management, Notification, Analytics | Important |

### 5. Детальные схемы ключевых событий

#### `Booking.Created`

```json
{
  "event_id": "uuid",
  "event_type": "Booking.Created",
  "event_version": "1.0.0",
  "occurred_at": "2025-10-22T14:30:00Z",
  "producer": "booking-service",
  "tenant_id": "uuid",
  "region": "region-a",
  "correlation_id": "uuid",
  "causation_id": "uuid",
  "data": {
    "booking_id": "uuid",
    "passenger_id": "uuid",
    "pickup": {
      "lat": 55.7558,
      "lon": 37.6173,
      "address": "Москва, Красная площадь"
    },
    "dropoff": {
      "lat": 55.7522,
      "lon": 37.6156,
      "address": "Москва, ГУМ"
    },
    "tariff_id": "economy",
    "estimated_price": {
      "amount": 350.00,
      "currency": "RUB"
    },
    "estimated_duration_min": 15,
    "requested_at": "2025-10-22T14:29:55Z"
  }
}
```

#### `Booking.Completed`

```json
{
  "event_id": "uuid",
  "event_type": "Booking.Completed",
  "event_version": "1.0.0",
  "occurred_at": "2025-10-22T14:45:00Z",
  "producer": "booking-service",
  "tenant_id": "uuid",
  "region": "region-a",
  "correlation_id": "uuid",
  "causation_id": "uuid",
  "data": {
    "booking_id": "uuid",
    "driver_id": "uuid",
    "passenger_id": "uuid",
    "actual_price": {
      "amount": 380.00,
      "currency": "RUB"
    },
    "distance_km": 4.2,
    "duration_min": 14,
    "completed_at": "2025-10-22T14:45:00Z",
    "payment_method": "card"
  }
}
```

#### `Payment.Succeeded`

```json
{
  "event_id": "uuid",
  "event_type": "Payment.Succeeded",
  "event_version": "1.0.0",
  "occurred_at": "2025-10-22T14:45:30Z",
  "producer": "payments-service",
  "tenant_id": "uuid",
  "region": "region-a",
  "correlation_id": "uuid",
  "causation_id": "uuid",
  "data": {
    "payment_id": "uuid",
    "booking_id": "uuid",
    "passenger_id": "uuid",
    "amount": {
      "amount": 380.00,
      "currency": "RUB"
    },
    "payment_method": "yandex_pay",
    "transaction_id": "external-transaction-id",
    "processed_at": "2025-10-22T14:45:30Z"
  }
}
```

#### `Payout.Completed`

```json
{
  "event_id": "uuid",
  "event_type": "Payout.Completed",
  "event_version": "1.0.0",
  "occurred_at": "2025-10-23T02:00:00Z",
  "producer": "payouts-service",
  "tenant_id": "uuid",
  "region": "region-a",
  "correlation_id": "uuid",
  "causation_id": "uuid",
  "data": {
    "payout_id": "uuid",
    "driver_id": "uuid",
    "amount": {
      "amount": 3500.00,
      "currency": "RUB"
    },
    "period": {
      "from": "2025-10-22T00:00:00Z",
      "to": "2025-10-23T00:00:00Z"
    },
    "bank_transaction_id": "external-bank-id",
    "completed_at": "2025-10-23T02:00:00Z"
  }
}
```

#### `Driver.LocationUpdated`

> **Особенность:** высокочастотное событие. В ADR-002 будет отдельная
> retention/compaction политика (compaction по `driver_id` — последнее
> состояние). Не подходит для «обычных» топиков с retention 7 дней.

```json
{
  "event_id": "uuid",
  "event_type": "Driver.LocationUpdated",
  "event_version": "1.0.0",
  "occurred_at": "2025-10-22T14:30:05.123Z",
  "producer": "driver-status-service",
  "tenant_id": "uuid",
  "region": "region-a",
  "correlation_id": null,
  "causation_id": null,
  "data": {
    "driver_id": "uuid",
    "location": {
      "lat": 55.7558,
      "lon": 37.6173,
      "accuracy_m": 10,
      "heading_deg": 90,
      "speed_kmh": 45
    },
    "status": "available",
    "timestamp": "2025-10-22T14:30:05.123Z"
  }
}
```

#### `Fraud.AlertRaised`

```json
{
  "event_id": "uuid",
  "event_type": "Fraud.AlertRaised",
  "event_version": "1.0.0",
  "occurred_at": "2025-10-22T14:46:00Z",
  "producer": "flink_fraud",
  "tenant_id": "uuid",
  "region": "region-a",
  "correlation_id": "uuid",
  "causation_id": "uuid",
  "data": {
    "alert_id": "uuid",
    "entity_type": "payment",
    "entity_id": "uuid",
    "risk_score": 0.92,
    "rule_id": "unusual_location",
    "severity": "high",
    "details": {
      "reason": "Payment from unexpected location",
      "user_id": "uuid",
      "attempted_amount": 50000.00,
      "currency": "RUB"
    }
  }
}
```

#### `Payout.Failed`

```json
{
  "event_id": "uuid",
  "event_type": "Payout.Failed",
  "event_version": "1.0.0",
  "occurred_at": "2025-10-23T02:05:00Z",
  "producer": "payouts-service",
  "tenant_id": "uuid",
  "region": "region-a",
  "correlation_id": "uuid",
  "causation_id": "uuid",
  "data": {
    "payout_id": "uuid",
    "driver_id": "uuid",
    "amount": {
      "amount": 3500.00,
      "currency": "RUB"
    },
    "failure_reason": "bank_rejected",
    "failure_details": "Invalid account number",
    "retry_count": 1,
    "failed_at": "2025-10-23T02:05:00Z"
  }
}
```

### 6. Что НЕ входит в каталог

- **Внутренние domain events** (внутри bounded context) — не публикуются в Kafka.
- **Технические события** (health checks, метрики инфраструктуры) — идут
  через OTel/Prometheus, не через Kafka.
- **Команды** (imperative) — заменены на события (Notification routing
  внутри сервиса).
- **Детальный дизайн топиков** (naming, partitions, retention) — ADR-002.
- **Saga-специфичные события** (компенсации, таймауты) — ADR-004.
- **Схемы для всех 20 событий** — детальные схемы только для 7 ключевых;
  остальные — по аналогии через Apicurio Registry.

## Альтернативы

1. **Оставить `NotificationRequested`** — отклонено: лишний слой, Notification
   подписан на все события и сам решает.
2. **`DriverAssigned` как отдельное событие** — отклонено: дублирует
   `Booking.Assigned` + `Driver.StatusChanged`.
3. **`DriverLocationUpdated` как stream, а не event** — отклонено: пользователь
   хочет масштабировать через event-модель (проще для кластеризации и
   горизонтального масштабирования).
4. **Command + Event** (CQRS с явными командами) — отклонено для Task2:
   усложняет модель, добавит второй каталог. В нашем случае достаточно
   только events.
5. **Версионирование в имени события** (`BookingCreatedV2`) — отклонено:
   anti-pattern, ломает контракты. Semver в envelope.

## Компромиссы

- **20 событий** — покрывают все 12 сервисов и все сценарии Saga.
  Если начнём добавлять `*Failed` для каждого сервиса — каталог вырастет
  вдвое. Митигация: добавляем события по мере необходимости, Apicurio
  Registry позволяет эволюционировать схемы.
- **1 producer на событие** — чёткое владение, но при смене сервиса-владельца
  нужно обновлять каталог. Митигация: владение зафиксировано в каталоге,
  изменение — через ADR.
- **`region` в envelope** — избыточно для глобальных топиков, но обязательно
  для регионального партиционирования (ADR-002). Митигация: `region`
  используется как partition key в региональных топиках.
- **`Notification.Sent` / `Notification.Failed`** — информационные события
  с низкой ценностью. Митигация: retention 7 дней, не для аналитики
  критичных сценариев.
- **Детальные схемы только для 7 событий** — остальные 13 в каталоге
  без полной схемы. Митигация: Apicurio Registry — схемы дозаполняются
  по мере реализации.
```