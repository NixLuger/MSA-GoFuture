# ADR-004: Saga (распределённые транзакции)

## Контекст

Событийная платформа (ADR-001, ADR-002) и Flink-обработка (ADR-003) готовы.
Требуется управлять распределёнными транзакциями между сервисами:
бизнес-процессы, которые затрагивают несколько bounded context и требуют
согласованности (не через БД-транзакции, а через компенсирующие события).

В Task1 (ADR-002) уже зафиксировано, что каждая БД — своя (Database per
Service), распределённых ACID-транзакций нет. Значит, для многошаговых
бизнес-операций нужна Saga.

Уже зафиксировано:
- 20 событий (ADR-001, после добавления `Payout.Failed`).
- Гибридная Saga: choreography + orchestration (ADR-001 Task2).
- `correlation_id` и `causation_id` в envelope — для трассировки Saga.

## Требования

1. Классифицировать события: какие требуют Saga, какие — нет.
2. Спроектировать orchestration-Saga для критичных сценариев.
3. Спроектировать choreography-паттерны для простых сценариев.
4. Определить компенсации (компенсирующие события) для каждого шага.
5. Таймауты, retry, escalation.
6. Не дублировать решения из ADR-001/002/003.

## Решение

### 1. Классификация событий: где нужна Saga

**Критерий Saga:** многошаговый бизнес-процесс в разных сервисах + есть
компенсации (откат через компенсирующие события).

| Событие | Требует Saga? | Почему |
|---------|---------------|--------|
| `Booking.Created` | ✅ **Orchestration** (Booking Lifecycle) | Начало многошагового процесса |
| `Booking.Assigned` | ✅ Шаг Booking Saga | |
| `Booking.Cancelled` | ✅ **Компенсация** Booking Saga | |
| `Booking.Completed` | ✅ Триггер Payment Saga | Конец Booking Saga, начало Payment Saga |
| `Driver.LocationUpdated` | ❌ Нет | Streaming, не транзакция |
| `Driver.StatusChanged` | ❌ Нет | Простое изменение статуса |
| `Driver.VerificationCompleted` | ❌ Нет | Одношаговое |
| `Pricing.PriceCalculated` | ❌ Нет | Одношаговый расчёт |
| `Payment.Succeeded` | ✅ **Orchestration** (Payment+Payout Saga) | |
| `Payment.Failed` | ✅ **Компенсация** Payment Saga | |
| `Payment.Refunded` | ✅ **Компенсация** (Booking + Payment) | |
| `Payout.Initiated` | ✅ Шаг Payment+Payout Saga | |
| `Payout.Completed` | ✅ Шаг Payment+Payout Saga (успех) | |
| `Payout.Failed` | ✅ **Компенсация** Payment+Payout Saga | |
| `Notification.Sent` | ❌ Нет | Fire-and-forget |
| `Notification.Failed` | ❌ Нет | Информационное |
| `Fraud.CheckCompleted` | ⚠️ Choreography (внутри Booking Saga) | Реагирует на `Booking.Created`, публикует результат |
| `Fraud.AlertRaised` | ❌ Нет | Уведомление/алерт |
| `Tenant.Onboarded` | ⚠️ Choreography | Простой сценарий (2–3 шага), без сложных компенсаций |
| `User.Registered` | ❌ Нет | Одношаговое |

**Итого:**
- **2 orchestration-Saga:** Booking Lifecycle, Payment+Payout.
- **3 choreography-паттерна:** Fraud-check, Notification, Analytics.
- **1 choreography-сценарий:** Tenant Onboarding.
- **Остальные события** — не участвуют в Saga.

### 2. Saga 1: Booking Lifecycle (Orchestration)

**Orchestrator:** внутри `svc_booking` (компонент `BookingSagaOrchestrator`).

**Почему orchestration:** многошаговый процесс, разные компенсации на
разных шагах, нужен контроль времени и явный «финиш».

**Участники:** Booking (orchestrator), Fraud, Driver Status (через
matching-streaming-job), Notification.

**Шаги и компенсации:**

| Шаг | Событие / действие | Таймаут | Компенсация при сбое |
|-----|--------------------|---------|----------------------|
| 1 | `Booking.Created` (старт) | — | — |
| 2 | Fraud check (choreography: Fraud слушает → `Fraud.CheckCompleted`) | 5 сек | `Booking.Cancelled` (fraud detected) |
| 3 | Matching (Flink matching-job) → `Booking.Assigned` | 5 мин | `Booking.Cancelled` (no driver found) |
| 4 | Trip execution (пассивно, статусы внутри Booking) | до 24 ч | — |
| 5 | `Booking.Completed` (успех) | — | — |

**Диаграмма состояний Booking Saga:**
```
[START]
│
▼
Booking.Created ──► [WAIT_FRAUD] ──Fraud.CheckCompleted(ok)──► [WAIT_MATCH]
│                    │                                           │
│                    │ Fraud detected                            │ Booking.Assigned
│                    ▼                                           ▼
│              Booking.Cancelled                              [IN_TRIP]
│                    │                                           │
│                    ▼                                           │ Booking.Completed
│                 [END: CANCELLED]                              ▼
│                                                          [END: COMPLETED]
└── Booking.Cancelled (user cancel) → [END: CANCELLED]
```

**Компенсации:**
- **Fraud detected** → `Booking.Cancelled` + `Notification.Sent` (push user «Заказ отклонён»).
- **No driver found (timeout 5 min)** → `Booking.Cancelled` + refund (если предоплата).
- **User cancel** (до назначения) → `Booking.Cancelled`.
- **User cancel** (после назначения) → `Booking.Cancelled` + штраф (если применимо).

**Кто вызывает компенсации:** `BookingSagaOrchestrator` публикует
`Booking.Cancelled` с `causation_id = event_id` исходного `Booking.Created`
(для трассировки).

**Timeout реализации:** Flink job или планировщик (Celery / Quartz) внутри
Booking-сервиса — таймеры на 5 мин и 24 ч.

### 3. Saga 2: Payment + Payout (Orchestration)

**Orchestrator:** внутри `svc_payments` (компонент `PaymentPayoutSagaOrchestrator`).

**Почему orchestration:** финансовый сценарий, критичные компенсации,
нужна эскалация при сбоях.

**Участники:** Payments (orchestrator), Fraud (вторичная проверка),
Payouts, Notification.

**Триггер:** `Booking.Completed` (после завершения поездки).

**Шаги и компенсации:**

| Шаг | Событие / действие | Таймаут | Компенсация |
|-----|--------------------|---------|-------------|
| 1 | Списание с пассажира → `Payment.Succeeded` / `Payment.Failed` | 30 сек | Retry × 3 (1 мин, 5 мин, 15 мин) → `Payment.Failed` (terminal) + `Notification.Sent` |
| 2 | Fraud re-check (для крупных сумм) | 10 сек | `Payment.Refunded` + `Fraud.AlertRaised` |
| 3 | `Payout.Initiated` → банковский API | 10 сек | Retry × 3 |
| 4 | `Payout.Completed` (успех) | до 24 ч | — |
| 5 | `Payout.Failed` | — | Retry × 3 → эскалация (`Fraud.AlertRaised` + ручной разбор) |

**Диаграмма состояний Payment+Payout Saga:**

```
[START: Booking.Completed]
│
▼
Charge passenger ──► [WAIT_PAYMENT]
│                     │ Payment.Succeeded       Payment.Failed
│                     ▼                             │
│               Fraud re-check (если > threshold)  ▼
│                     │ Fraud ok              [RETRY × 3]
│                     ▼                             │
│               [WAIT_PAYOUT]                  [END: FAILED]
│                     │
│                     │ Payout.Initiated
│                     ▼
│               Payout.Completed ──► [END: COMPLETED]
│                     │
│                     │ Payout.Failed
│                     ▼
│               Retry × 3 ──► эскалация
│                     │
│                     ▼
│               [END: ESCALATED]
└── Booking.Cancelled → Payment.Refunded → [END: REFUNDED]
```

**Компенсации:**
- **Payment.Failed (terminal)** → `Notification.Sent` (push user «Не удалось списать»), `Booking` в статус `PAYMENT_FAILED`.
- **Fraud re-check failed** → `Payment.Refunded` + `Fraud.AlertRaised`.
- **Payout.Failed (после retry)** → `Fraud.AlertRaised` (severity: high) + Notification в Admin.
- **Booking.Cancelled после оплаты** → `Payment.Refunded`.

**Особенность:** выплата водителю — батчевая (раз в сутки) в реальной жизни,
но в нашей архитектуре `Payout.Initiated` публикуется после каждого
`Payment.Succeeded` (выплата агрегируется в Payouts-сервисе и через
банковский API отправляется батчем). Saga ждёт `Payout.Completed` с
таймаутом 24 ч.

**Retry:**
- Exponential backoff: 1 мин → 5 мин → 15 мин.
- Max 3 retry.
- После — эскалация.

### 4. Choreography-паттерны (без orchestrator)

#### 4.1 Fraud Check (choreography)

**Участники:** Fraud (consumer `Booking.Created`, `Payment.Succeeded`),
Booking/Payments (consumer `Fraud.CheckCompleted`).

**Поток:**
1. Booking публикует `Booking.Created`.
2. Fraud подписан → выполняет проверку (streaming-модель из ADR-003).
3. Fraud публикует `Fraud.CheckCompleted` (с результатом в payload).
4. Booking (Saga orchestrator) читает `Fraud.CheckCompleted` с
   `correlation_id = Booking.Created` → решает, продолжать или `Booking.Cancelled`.

**Почему choreography:** простой паттерн «запрос → ответ», не требует
orchestrator. Fraud сам решает, как проверять.

#### 4.2 Notification (choreography)

**Участники:** Notification (consumer всех событий).

**Поток:**
1. Любой сервис публикует событие.
2. Notification подписан → по routing rules решает, что отправить.
3. Публикует `Notification.Sent` / `Notification.Failed`.

**Почему choreography:** fire-and-forget, не участвует в транзакциях.

#### 4.3 Analytics (choreography)

**Участники:** Analytics (consumer всех событий, ADR-003 analytics-streaming-job).

**Поток:**
1. Любой сервис публикует событие.
2. Analytics агрегирует и пишет в Data Platform.

**Почему choreography:** read-only, не участвует в транзакциях.

#### 4.4 Tenant Onboarding (choreography)

**Участники:** Tenant Management (producer `Tenant.Onboarded`), Identity
(consumer → создание admin user), Notification (welcome email), Analytics.

**Поток:**
1. Tenant Management публикует `Tenant.Onboarded`.
2. Identity создаёт admin user (по данным из события).
3. Identity публикует `User.Registered`.
4. Notification отправляет welcome email.

**Почему choreography:** 2–3 шага, компенсации тривиальные (если упало —
повторная попытка / ручной фикс). Не оправдывает orchestrator.

### 5. Идемпотентность и дедупликация

**Ключевой риск Saga:** дубли событий (at-least-once, ADR-005) могут
привести к двойным списаниям / выплатам.

**Решение:**
- Каждый consumer хранит `processed_events` (event_id + correlation_id + timestamp).
- Перед обработкой проверяет: если `event_id` уже обработан — skip (idempotent consumer).
- Для финансовых операций — уникальный ключ в БД: `(payment_id, idempotency_key)`.
- Saga orchestrator хранит state Saga в БД: `saga_state` (saga_id, step, status, event_ids).

**Хранение:** PostgreSQL, TTL 30 дней (покрывает retention Critical-событий).

### 6. Timeouts и эскалация

| Сценарий | Timeout | Действие |
|----------|---------|----------|
| Booking: Fraud check | 5 сек | Retry × 2 → если не пришло — `Booking.Cancelled` (fail-secure) |
| Booking: Matching | 5 мин | `Booking.Cancelled` (no driver found) |
| Booking: Trip execution | 24 ч | Alert в Admin (аномально долгая поездка) |
| Payment: Charge | 30 сек | Retry × 3 (1/5/15 мин) → `Payment.Failed` |
| Payment: Payout | 24 ч | Alert в Admin, если `Payout.Completed` не пришёл |
| Payment: Payout retry | 3 × (1/5/15 мин) | Эскалация (`Fraud.AlertRaised` severity high) |

**Эскалация = алерт в Admin** + `Fraud.AlertRaised` (для финансовых) +
`Notification.Sent` (пуш оператору).

### 7. Трассировка Saga

**`correlation_id`** — единый на всю Saga (Booking Lifecycle или Payment+Payout).
**`causation_id`** — ID события-причины для каждого шага.

**Пример Booking Saga:**
- `Booking.Created` (event_id=A, correlation_id=C)
- `Fraud.CheckCompleted` (event_id=B, causation_id=A, correlation_id=C)
- `Booking.Assigned` (event_id=D, causation_id=A, correlation_id=C)
- `Booking.Completed` (event_id=E, causation_id=D, correlation_id=C)

**Запрос «покажи всю Saga по booking_id»:**
```sql
SELECT * FROM events
WHERE correlation_id = (SELECT correlation_id FROM events WHERE event_id = ?)
ORDER BY occurred_at
```

**Инструменты:** OpenTelemetry + Jaeger (Task1 ADR-004) — трейсы сквозные,
Saga видна как единый trace.

### 8. Что НЕ входит в это решение

- **Детальный дизайн Kafka-топиков** — ADR-002.
- **Flink-обработка matching/fraud** — ADR-003.
- **Reliable delivery (retry, DLQ, outbox)** — ADR-005.
- **Мониторинг Saga** — ADR-006.
- **Business-level UI для разбора эскалаций** — вне scope Task2.

## Альтернативы

1. **Только choreography (без orchestrator)** — отклонено: для Booking
   и Payment+Payout слишком сложно отслеживать состояние, дебажить и
   добавлять компенсации.
2. **Только orchestration** — отклонено: для Notification/Analytics/Fraud
   избыточно, добавит центральные точки отказа.
3. **Один глобальный orchestrator на всё** — отклонено: единая точка
   отказа, «мини-монолит», не соответствует bounded context.
4. **2PC (two-phase commit) вместо Saga** — отклонено: распределённых
   ACID-транзакций в Kafka+PostgreSQL нет, 2PC не масштабируется.
5. **Saga без явных компенсаций (только retry)** — отклонено: финансовые
   сценарии требуют refund при сбоях.

## Компромиссы

- **2 orchestrator'а** — дублирование инфраструктуры (state, tracing,
  retry). Митигация: общая библиотека `saga-framework` (Python).
- **State Saga в PostgreSQL** — дополнительная нагрузка на БД Booking
  и Payments. Митигация: TTL 30 дней, периодическая очистка.
- **Timeout 5 сек на Fraud check** — жёстко; если Fraud тормозит — Booking
  отменяется (fail-secure). Митигация: мониторинг latency Fraud (ADR-006),
  алерт при p99 > 3 сек.
- **Payout timeout 24 ч** — Saga «висит» сутки. Митигация: метрика
  `sagas_in_progress` + алерт при росте, ручной разбор через Admin.
- **Choreography для Tenant Onboarding** — если шаги станут сложнее (4+),
  придётся переделать на orchestration. Митигация: пока 3 шага, ок;
  при росте — отдельный ADR.
- **Идемпотентность через `event_id`** — требует хранения `processed_events`
  во всех consumer'ах. Митигация: shared middleware `idempotent_consumer`.
