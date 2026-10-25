# ADR-005: Надёжная доставка событий

## Контекст

Событийная платформа (ADR-001–004) обрабатывает 500 000 конкурентных поездок
и финансовые транзакции. Kafka гарантирует **at-least-once** delivery, но не
exactly-once end-to-end. Без дополнительных механизмов возможны:
- **Потеря событий** (producer упал между записью в БД и публикацией в Kafka).
- **Дубли событий** (consumer упал после обработки, но до коммита offset'а).
- **Зависание consumer'а** на «ядовитом» сообщении (poison message).

Для финансовых событий (`Payment.Succeeded`, `Payout.Completed`) потеря
или дубль недопустимы — это прямые финансовые потери.

Уже зафиксировано:
- Kafka с региональными топиками, RF=3, `acks=all`, `enable.idempotence=true` (ADR-002).
- Flink jobs с exactly-once checkpointing (ADR-003).
- Идемпотентность через `event_id` (ADR-004, раздел 5).
- Outbox pattern в монолите (ADR-004 Task1).

Нужно спроектировать **end-to-end надёжную доставку** для всех producer'ов
и consumer'ов (не только монолита).

## Требования

1. Гарантировать, что событие не потеряется (producer side).
2. Гарантировать, что дубли не приведут к двойной обработке (consumer side).
3. Обрабатывать ошибки consumer'а (retry, DLQ).
4. Обрабатывать poison messages.
5. Метрики надёжности (для ADR-006).
6. Не дублировать решения из ADR-001–004.

## Решение

### 1. Общая картина: 3 уровня надёжности

```
┌─────────────────────────────────────────────────────────────────┐
│  Уровень 1: PRODUCER SIDE (Outbox)                              │
│  Атомарная запись в БД + публикация в Kafka                     │
└─────────────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│  Уровень 2: KAFKA (Durability)                                  │
│  RF=3, acks=all, min.insync.replicas=2, idempotent producer     │
└─────────────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────────────┐
│  Уровень 3: CONSUMER SIDE (Idempotency + Retry + DLQ)           │
│  Дедупликация по event_id, retry-topic, DLQ                     │
└─────────────────────────────────────────────────────────────────┘
```

**Итоговая гарантия:** `at-least-once delivery + idempotent processing = effective exactly-once`.

### 2. Уровень 1: Outbox pattern (producer side)

**Для всех сервисов-продюсеров** (не только монолита).

#### 2.1 Как работает

1. Бизнес-операция в сервисе пишет изменения в БД **и** в таблицу `outbox` **в одной транзакции**.
2. Отдельный poller (или CDC через Debezium) читает `outbox` и публикует в Kafka.
3. После успешной публикации — помечает запись как `published`.
4. Периодическая очистка (TTL 7 дней).

**Ключевое:** бизнес-изменение и запись в outbox — атомарны. Если транзакция откатилась — событие не будет опубликовано (что правильно). Если закоммитилась — событие будет опубликовано рано или поздно.

#### 2.2 Схема таблицы `outbox`

```sql
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    event_id UUID NOT NULL UNIQUE,
    event_type VARCHAR(255) NOT NULL,
    aggregate_id VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    topic VARCHAR(255) NOT NULL,
    partition_key VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ,
    retry_count INT NOT NULL DEFAULT 0,
    last_error TEXT
);

CREATE INDEX idx_outbox_unpublished ON outbox (created_at)
    WHERE published_at IS NULL;
```

#### 2.3 Реализация poller'а

**Для обычных сервисов:** фоновый процесс в том же приложении (FastAPI + background task / Celery Beat) или отдельный sidecar.

**Для высоконагруженных:** Debezium (CDC) — читает `outbox` как изменение БД и публикует в Kafka через Kafka Connect. Меньше нагрузки на сервис, лучше для throughput.

| Сервис | Способ публикации |
|--------|-------------------|
| Монолит | Poller (ADR-004 Task1) |
| Booking, Payments, Payouts, Driver Status (высокочастотные) | **Debezium** (CDC) |
| Остальные (низкочастотные) | Poller в сервисе |

**Обоснование:** для высокочастотных CDC эффективнее (не блокирует сервис), для остальных poller проще.

#### 2.4 Idempotent producer

- `enable.idempotence=true` (ADR-002).
- `transactional.id` — для Flink sinks (ADR-003).
- Producer retry: `retries=Integer.MAX_VALUE`, `delivery.timeout.ms=120000`.

Даже при retry producer'а — Kafka не создаст дубль (защита на уровне broker'а через `producer_id + sequence_number`).

### 3. Уровень 3: Идемпотентность consumer'ов

#### 3.1 Принцип

Каждый consumer хранит `processed_events` — таблицу уже обработанных событий. Перед обработкой проверяет: если `event_id` уже есть — skip.

#### 3.2 Схема таблицы `processed_events`

```sql
CREATE TABLE processed_events (
    event_id UUID PRIMARY KEY,
    consumer_group VARCHAR(255) NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    correlation_id UUID,
    result JSONB
);

CREATE INDEX idx_processed_events_ttl ON processed_events (processed_at);
```

**TTL:** 30 дней (покрывает retention Critical-событий, ADR-002).

**Ключ:** `event_id` (UNIQUE) — гарантирует, что одно и то же событие не обработается дважды **в рамках одного consumer'а**.

**Разные consumer'ы обрабатывают одно событие независимо** — `consumer_group` в ключе не входит в PK, но используется для аналитики.

#### 3.3 Процесс обработки (для обычного consumer'а)

```
1. Читаем сообщение из Kafka.
2. BEGIN transaction.
3. INSERT INTO processed_events (event_id, ...) ON CONFLICT DO NOTHING.
4. IF row_count = 0 THEN
      -- уже обработано, skip
      COMMIT; continue;
   END IF.
5. Бизнес-обработка (запись в БД сервиса).
6. COMMIT (processed_events + бизнес-изменение — атомарно).
7. Commit Kafka offset.
```

**Ключевое:** `processed_events` + бизнес-изменение — в одной БД-транзакции. Если транзакция упала — ни событие не помечено обработанным, ни бизнес-изменение не применено. Kafka offset тоже не закоммичен → событие придёт снова → обработается корректно.

#### 3.4 Для Flink jobs

Flink уже обеспечивает exactly-once через checkpointing (ADR-003). Дополнительная идемпотентность нужна только на **sink**:

| Sink | Идемпотентность |
|------|-----------------|
| Kafka producer (из Flink) | `transactional.id` + `read_committed` |
| PostgreSQL sink | `INSERT ... ON CONFLICT (event_id) DO NOTHING` |
| ClickHouse sink | `ReplacingMergeTree` по `(event_id, window_start)` |

#### 3.5 Для Saga orchestrator'ов

Saga state хранится в БД orchestrator'а (`saga_state`). Перед обработкой шага — проверка: `event_id` уже обработан в этой Saga? Если да — skip.

```sql
CREATE TABLE saga_state (
    saga_id UUID PRIMARY KEY,
    saga_type VARCHAR(100) NOT NULL,  -- 'booking_lifecycle', 'payment_payout'
    correlation_id UUID NOT NULL,
    current_step VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL,  -- 'in_progress', 'completed', 'compensated', 'escalated'
    last_event_id UUID,
    started_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    timeout_at TIMESTAMPTZ,
    payload JSONB
);
```

### 4. Retry через отдельный топик

#### 4.1 Зачем отдельный топик

**Проблема in-memory retry:** consumer блокируется на «сон» (wait N сек), тратит ресурсы, не обрабатывает другие сообщения.

**Решение:** retry через отдельный топик с задержкой.

#### 4.2 Схема retry

**Топики:**
- `<region>.<domain>.events` — основной.
- `<region>.<domain>.retry.1` — retry 1 (задержка 1 мин).
- `<region>.<domain>.retry.2` — retry 2 (задержка 5 мин).
- `<region>.<domain>.retry.3` — retry 3 (задержка 15 мин).
- `<region>.<domain>.dlq` — DLQ (после 3 retry).

**Процесс:**
1. Consumer получил сообщение → ошибка обработки (transient).
2. Публикует в `retry.1` с заголовком `original-topic` + `retry-count=1`.
3. Consumer `retry.1` читает через 1 мин → повторяет.
4. Если снова ошибка → `retry.2` (5 мин).
5. Если снова ошибка → `retry.3` (15 мин).
6. Если снова ошибка → `dlq`.

**Реализация задержки:**
- **Вариант A:** Kafka-топики с `retention.ms` = задержке + consumer с `max.poll.interval` = задержке (плохо).
- **Вариант B:** Отдельный scheduler (Kafka Streams / Flink) читает retry-топик и публикует в основной через delay.
- **Вариант C:** Использовать **Kafka Delay Topic** через Flink или Kafka Connect.

**Мой выбор — B** (scheduler на Flink), потому что:
- Flink уже есть в стеке (ADR-003).
- Управляемая задержка через stateful-обработку.
- Единая инфраструктура.

#### 4.3 Классификация ошибок

| Тип ошибки | Действие |
|------------|----------|
| **Transient** (timeout, network, deadlock) | Retry (3 попытки) |
| **Permanent** (schema violation, business rule) | Сразу в DLQ (без retry) |
| **Poison** (не парсится, некорректный payload) | Сразу в DLQ |

**Как определить:** обработчик возвращает `RetryableError` или `PermanentError`. Middleware решает.

### 5. Dead Letter Queue (DLQ)

#### 5.1 Что попадает в DLQ

- Сообщения после 3 retry (transient-ошибки).
- Сообщения с permanent-ошибками.
- Poison messages (не парсятся).

#### 5.2 Схема DLQ-сообщения

```json
{
  "original_event": { /* полное исходное событие */ },
  "original_topic": "region-a.booking.events",
  "original_partition": 5,
  "original_offset": 12345,
  "error_type": "SchemaViolation",
  "error_message": "Field 'pickup' is missing",
  "retry_count": 3,
  "first_seen_at": "2025-10-22T14:30:00Z",
  "last_failed_at": "2025-10-22T14:50:00Z",
  "consumer_group": "booking-streaming-cg-v1"
}
```

#### 5.3 Обработка DLQ

- **Автоматически:** мониторинг + алерт (ADR-006), если DLQ растёт.
- **Вручную:** разбор инцидента, фикс, **replay** из DLQ в основной топик.
- **Replay-инструмент:** CLI + Admin UI (вне scope Task2, но зафиксируем требование).

**Retention DLQ:** 30 дней (достаточно для разбора).

### 6. Poison messages

**Определение:** сообщение, которое **никогда** не обработается (например, битый Avro, отсутствующее обязательное поле, бизнес-инвариант нарушен).

**Обработка:**
1. Consumer пытается распарсить → ошибка.
2. **Не retry** (это не transient) → сразу в DLQ.
3. Алерт (ADR-006).
4. Ручной разбор.

**Защита producer'а:** schema validation в Apicurio Registry (ADR-002) не даст опубликовать невалидное сообщение. Poison messages — редкость.

### 7. Метрики надёжности (для ADR-006)

Кратко (детали — в ADR-006):

| Метрика | Что показывает |
|---------|----------------|
| `outbox_unpublished_count` | Сколько событий в outbox не опубликовано (должно быть ~0) |
| `outbox_oldest_unpublished_age_seconds` | Возраст старейшего неопубликованного события (алерт при > 60 сек) |
| `processed_events_duplicates_total` | Сколько дублей отсечено (норма — небольшое число) |
| `retry_total{retry_level}` | Количество retry по уровням |
| `dlq_messages_total{topic}` | Количество сообщений в DLQ |
| `consumer_processing_latency_p99` | Задержка обработки |

### 8. Что НЕ входит в это решение

- **Детальный дизайн Kafka-топиков** — ADR-002.
- **Flink-обработка** — ADR-003.
- **Saga orchestration** — ADR-004.
- **Мониторинг (инструменты, дашборды)** — ADR-006.
- **Replay-инструмент для DLQ** — вне scope Task2 (требование зафиксировано).
- **Инфраструктура outbox poller'а (K8s, deployment)** — инфраструктурное решение.

## Альтернативы

1. **In-memory retry (без retry-топиков)** — отклонено: блокирует
   consumer, плохо масштабируется, теряет сообщения при рестарте.
2. **Только outbox без идемпотентности consumer'ов** — отклонено:
   at-least-once delivery даёт дубли, без дедупликации — двойные списания.
3. **Только идемпотентность без outbox** — отклонено: без outbox риск
   «записал в БД, но не опубликовал».
4. **2PC (two-phase commit) между БД и Kafka** — отклонено: не работает
   с Kafka (нет XA), не масштабируется.
5. **DLQ без retry** — отклонено: transient-ошибки (timeout, network)
   должны retry, иначе много шума в DLQ.
6. **Один retry-топик с разными задержками через headers** — отклонено:
   сложнее обрабатывать, нужен custom consumer. Отдельные топики проще.

## Компромиссы

- **Outbox для всех сервисов** — дополнительная таблица + poller в каждом.
  Митигация: shared библиотека `outbox-publisher` для Python-сервисов.
- **`processed_events` TTL 30 дней** — рост таблицы. Митигация: партиционирование
  по `processed_at`, автоочистка.
- **Retry через Flink scheduler** — сложнее, чем in-memory, но правильнее.
  Митигация: единая Flink-инфраструктура (ADR-003).
- **DLQ — операционная нагрузка** — ручной разбор. Митигация: алерты,
  runbook, replay-инструмент.
- **Debezium для высокочастотных** — дополнительная инфраструктура
  (Kafka Connect). Митигация: общий кластер Kafka Connect для всех
  высокочастотных producer'ов.
- **Effective exactly-once** — не абсолютный exactly-once (Kafka не даёт
  end-to-end), но на практике эквивалент. Митигация: мониторинг
  `duplicates_total` (должно быть ~0 на уровне бизнес-эффектов).