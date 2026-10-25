# ADR-006: Мониторинг событийной платформы

## Контекст

Событийная платформа (ADR-001–005) включает:
- 20 доменных событий (ADR-001).
- 54 топика Kafka (27 events + 27 DLQ), не считая retry-топиков (ADR-002).
- 4 Flink job'а для потоковой обработки (ADR-003).
- 2 orchestration-Saga + 3 choreography-паттерна + 1 choreography-сценарий (ADR-004).
- Outbox, идемпотентные consumer'ы, retry-топики, DLQ (ADR-005).

Распределённая система такого масштаба **невозможна без observability**:
без метрик и алертов мы не узнаем о проблемах (consumer lag, DLQ growth,
Saga stuck, poison messages) до того, как они станут инцидентом.

Уже зафиксировано (Task1 ADR-001):
- Observability-требования: 100% покрытие критических сервисов метриками и алертами, structured logging, distributed tracing, SLO/SLI.
- Стек: Prometheus + Grafana + Loki + OTel Collector + Alertmanager.

Нужно спроектировать **подход к мониторингу событийной платформы**,
список метрик и SLO, выбрать инструменты и отразить их на C2.

## Требования

1. Описать подход к мониторингу с обоснованием.
2. Составить список метрик по категориям.
3. Определить SLO для событийной платформы.
4. Выбрать инструменты мониторинга.
5. Отразить инструменты на диаграмме C2.
6. Не дублировать решения из ADR-001–005.

## Решение

### 1. Подход к мониторингу: 3 столпа observability + бизнес-метрики

**Три столпа (стандарт):**

| Столп | Что | Зачем |
|-------|-----|-------|
| **Metrics** | Числовые временные ряды | Тренды, алерты, SLO |
| **Logs** | Структурированные события | Разбор инцидентов, аудит |
| **Traces** | Сквозные цепочки запросов | Трассировка Saga, дебаг |

**Плюс четвёртый для событийной платформы:**

| **Business Metrics** | Бизнес-события в секунду | Понимание «здоровья» бизнеса |

**Обоснование:**
- **Metrics** — основной инструмент алертинга. Позволяют обнаружить проблему до того, как её заметит пользователь.
- **Logs** — нужны для post-mortem разбора. Structured JSON + correlation_id для поиска.
- **Traces** — критичны для Saga (сквозной correlation_id) и дебага распределённых сценариев.
- **Business Metrics** — без них мониторинг инфраструктуры не показывает, что бизнес «упал» (например, `bookings_created/sec = 0` — критично, даже если Kafka жива).

### 2. Категории метрик

#### 2.1 Kafka infrastructure

| Метрика | Описание | Алерт |
|---------|----------|-------|
| `kafka_brokers_up` | Количество живых брокеров | < RF (3) |
| `kafka_under_replicated_partitions` | Партиции с < RF реплик | > 0 |
| `kafka_offline_partitions` | Партиции без лидера | > 0 |
| `kafka_isr_shrinks_total` | Сжатия ISR (проблемы с репликацией) | rate > baseline |
| `kafka_disk_usage_percent` | Использование диска брокера | > 80% |
| `kafka_controller_active` | Активен ли controller | = 0 |

**Источник:** Kafka Exporter.

#### 2.2 Producer (outbox)

| Метрика | Описание | Алерт |
|---------|----------|-------|
| `outbox_unpublished_count` | Событий в outbox, не опубликованных | > 100 |
| `outbox_oldest_unpublished_age_seconds` | Возраст старейшего неопубликованного | > 60 сек |
| `outbox_publish_errors_total` | Ошибки публикации | rate > 0 |
| `schema_violations_total` | Нарушения схемы (Apicurio) | > 0 |
| `producer_record_send_total` | Отправленные записи | для throughput |
| `producer_record_error_total` | Ошибки отправки | rate > baseline |

**Источник:** custom metrics из сервисов + Kafka producer metrics.

#### 2.3 Consumer

| Метрика | Описание | Алерт |
|---------|----------|-------|
| `kafka_consumer_lag` | Lag по consumer group / партиции | p99 > 10 сек |
| `consumer_processing_latency_p99` | Время обработки | > SLO |
| `consumer_processing_errors_total` | Ошибки обработки | rate > baseline |
| `consumer_retry_total{level}` | Retry по уровням (1/2/3) | rate растёт |
| `dlq_messages_total{topic}` | Сообщений в DLQ | rate > 0.01% |
| `processed_events_duplicates_total` | Дубли отсечены | informational |
| `consumer_rebalance_total` | Ребалансировки | rate > baseline |

**Источник:** custom middleware `idempotent_consumer` + Kafka consumer metrics.

#### 2.4 Flink jobs

| Метрика | Описание | Алерт |
|---------|----------|-------|
| `flink_job_uptime_seconds` | Uptime job'а | < 5 мин |
| `flink_num_restarts_total` | Рестарты | rate > 0 |
| `flink_checkpoint_duration_seconds` | Длительность checkpoint | > 30 сек |
| `flink_checkpoint_failures_total` | Провалы checkpoint | rate > 0 |
| `flink_last_checkpoint_age_seconds` | Возраст последнего checkpoint | > interval × 3 |
| `flink_backpressure_ratio` | Backpressure | > 0.5 |
| `flink_watermark_delay_seconds` | Задержка watermark | > 5 сек |
| `flink_state_size_bytes` | Размер state | > 80% лимита |
| `flink_records_in_per_second` | Throughput in | baseline |
| `flink_records_out_per_second` | Throughput out | baseline |

**Источник:** Flink Metrics Reporter (Prometheus).

#### 2.5 Saga

| Метрика | Описание | Алерт |
|---------|----------|-------|
| `sagas_in_progress{saga_type}` | Саг в работе | > baseline × 2 |
| `saga_duration_p99{saga_type}` | Длительность Saga | > SLO |
| `saga_compensations_total{saga_type, reason}` | Компенсации | rate > baseline |
| `saga_timeouts_total{saga_type}` | Таймауты | rate > baseline |
| `saga_escalations_total{saga_type}` | Эскалации | > 0 |

**Источник:** custom metrics из `BookingSagaOrchestrator` и `PaymentPayoutSagaOrchestrator`.

#### 2.6 Business metrics

| Метрика | Описание | Алерт |
|---------|----------|-------|
| `bookings_created_total{region}` | Созданных поездок | drop > 50% от baseline |
| `bookings_completed_total{region}` | Завершённых | drop > 50% |
| `payments_succeeded_total{region}` | Успешных платежей | drop > 50% |
| `payments_failed_total{region}` | Провалов платежей | spike > 5% |
| `payouts_completed_total{region}` | Выплат | drop > 50% |
| `fraud_alerts_total{severity}` | Fraud-алертов | spike |
| `driver_locations_received_total{region}` | Location updates | drop > 50% |
| `notifications_sent_total{channel}` | Уведомлений | drop > 50% |

**Источник:** custom metrics из сервисов (счётчики в Prometheus).

#### 2.7 Data quality

| Метрика | Описание | Алерт |
|---------|----------|-------|
| `events_out_of_order_total{topic}` | События не по порядку | rate > baseline |
| `events_duplicate_total{topic}` | Дубли (в Kafka) | informational |
| `events_missing_total{topic}` | Пропуски (по gap detection) | > 0 |
| `schema_compatibility_violations_total` | Нарушения backward-compatible | > 0 |

**Источник:** consumer middleware + Apicurio Registry.

### 3. SLO событийной платформы

| SLO | Значение | Обоснование |
|-----|----------|-------------|
| **Event delivery latency** (producer → consumer) | p99 < 1 сек | Real-time обработка |
| **Pricing latency** (Booking.Created → PriceCalculated) | p99 < 100 мс | UX (цена в приложении) |
| **Matching latency** (Booking.Created → Booking.Assigned) | p99 < 500 мс | UX (назначение водителя) |
| **Fraud latency** (Payment.Succeeded → CheckCompleted) | p99 < 200 мс | Безопасность транзакций |
| **Booking Saga completion** | p99 < 1 час | Бизнес-ожидание (короткая поездка) |
| **Payment+Payout Saga completion** | p99 < 24 часа | Батчевые выплаты |
| **DLQ rate** | < 0.01% от общего объёма событий | Качество обработки |
| **Consumer lag** | p99 < 10 сек | Real-time |
| **Kafka availability** | 99,95% | Наследуется из ADR-001 Task1 |
| **Flink job availability** | 99,95% | Критичные job'ы |
| **Duplicate processing rate** | < 0.001% (effective) | Financial correctness |
| **Outbox age** | p99 < 60 сек | Свежесть событий |

**Error budget:** 99,95% → 4,38 часа недоступности в год → 21,9 мин в месяц.

**Burn rate alerts:**
- 2% budget за 1 час → PagerDuty (critical).
- 5% budget за 6 часов → Slack (warning).
- 10% budget за 3 дня → Slack (info).

### 4. Инструменты мониторинга

#### 4.1 Общий стек (наследуется из Task1)

| Инструмент | Роль | Специфика Task2 |
|------------|------|-----------------|
| **Prometheus** | Метрики (TSDB) | + Kafka Exporter, + Flink Metrics Reporter |
| **Grafana** | Дашборды | + дашборды Kafka / Flink / Saga / Business |
| **Loki** | Логи | + structured logs из consumer'ов |
| **OTel Collector** | Трейсы | + сквозные traces для Saga |
| **Alertmanager** | Алерты | + SLO burn rate alerts |

#### 4.2 Специфичные для событийной платформы

| Инструмент | Роль | Что мониторит |
|------------|------|---------------|
| **Kafka Exporter** (danielqsj) | Prometheus exporter | Kafka lag, ISR, under-replicated, controller health |
| **Flink Metrics Reporter** | Prometheus reporter | Flink jobs (checkpoint, backpressure, watermark) |
| **Jaeger** (или OTel Traces backend) | Distributed tracing | Сквозные traces по `correlation_id` для Saga |
| **Apicurio Registry Metrics** | Custom exporter | Schema violations, compatibility |
| **Custom middleware** `observability` | Python library | Business metrics, retry_total, duplicates_total |

**Обоснование выбора:**
- **Kafka Exporter** — де-факто стандарт, минимальная конфигурация.
- **Flink Metrics Reporter** — встроенный в Flink, нативный Prometheus.
- **Jaeger** — open-source, хорошо интегрируется с OTel Collector.
- **Apicurio** — уже выбран в ADR-001, метрики — через встроенный endpoint.
- **Custom middleware** — нужен для бизнес-метрик, которых нет в стандартных exporter'ах.

#### 4.3 Дашборды Grafana (ключевые)

| Дашборд | Что показывает |
|---------|----------------|
| **Kafka Cluster** | Brokers up, ISR, under-replicated, disk, lag per consumer group |
| **Flink Jobs** | Uptime, checkpoint, backpressure, throughput per job |
| **Saga Health** | Sagas in progress, duration p99, compensations, timeouts |
| **Business Metrics** | Bookings/sec, payments/sec, fraud alerts, notifications/sec |
| **SLO Dashboard** | Error budget burn rate, SLO compliance |
| **DLQ & Retry** | DLQ messages per topic, retry rate per level |

### 5. Alerting strategy

**Уровни алертов:**

| Severity | Условие | Куда |
|----------|---------|------|
| **Critical (P1)** | SLO burn rate > 2% / 1 час; Kafka offline partitions > 0; `outbox_age > 5 мин` | PagerDuty (on-call) |
| **Warning (P2)** | SLO burn rate > 5% / 6 часов; DLQ rate > 0.05%; consumer lag > 30 сек | Slack #alerts |
| **Info (P3)** | DLQ rate > 0.01%; retry rate растёт; saga escalations | Slack #warnings |

**Runbook для каждого алерта:** ссылка в аннотации Alertmanager на runbook (в Git-репозитории).

**Пример алерта (Prometheus rule):**

```yaml
- alert: HighConsumerLag
  expr: kafka_consumer_lag > 10000
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High consumer lag on {{ $labels.consumer_group }}"
    runbook: "https://github.com/gofuture/runbooks/consumer-lag"
```

### 6. Distributed tracing (Saga)

**Сквозной `correlation_id`** — единый на всю Saga (ADR-004).

**OTel instrumentation:**
- Producer'ы: `trace_id` = `correlation_id`, `span` на каждую публикацию.
- Consumer'ы: продолжают `trace_id` из заголовков сообщения.
- Flink: `openTelemetry` reporter (Flink 1.16+).
- Saga orchestrator: root span на Saga, child spans на каждый шаг.

**Инструмент:** Jaeger UI для поиска trace по `correlation_id`.

**Пример trace (Booking Saga):**
```
[Booking.Created] ─► [Fraud.CheckCompleted] ─► [Booking.Assigned] ─► [Booking.Completed] ─► [Payment.Succeeded] ─► [Payout.Completed]
      5ms                   3ms                      450ms               14min                300ms                2h
```

### 7. Structured logging

**Формат:** JSON (structured), обязательные поля:
- `timestamp`, `level`, `service`, `event_type`
- `correlation_id`, `causation_id` (для поиска по Saga)
- `tenant_id`, `region` (для фильтрации)
- `message`, `error` (если есть)

**Loki integration:** log shipping через Promtail / Fluent Bit.

**Пример лога:**
```json
{
  "timestamp": "2025-10-22T14:30:00.123Z",
  "level": "INFO",
  "service": "pricing-streaming-job",
  "event_type": "Pricing.PriceCalculated",
  "correlation_id": "corr-456",
  "region": "region-a",
  "message": "Price calculated",
  "data": {"booking_id": "...", "price": 350.00}
}
```

### 8. Отражение на C2

Инструменты мониторинга отражены на **C2 Event-Driven Platform**
(см. диаграмму ниже):

- **Kafka Exporter** — подключён к Kafka, отдаёт метрики в Prometheus.
- **Flink Metrics Reporter** — в каждом Flink job, отдаёт в Prometheus.
- **OTel Collector** — собирает traces от producer'ов, consumer'ов, Flink.
- **Prometheus** — TSDB, к нему подключены все exporter'ы.
- **Grafana** — читает из Prometheus + Loki + Jaeger.
- **Loki** — агрегирует logs.
- **Jaeger** — хранит traces.
- **Alertmanager** — получает алерты из Prometheus.

### 9. Что НЕ входит в это решение

- **Инфраструктура мониторинга (K8s, storage)** — инфраструктурное решение.
- **Runbook'и** — операционная документация, вне scope Task2.
- **On-call rotation** — организационное решение.
- **Business Intelligence (BI)** — Task4 (Data Platform).
- **ML-модели мониторинга** (anomaly detection) — Task4.
- **Cost monitoring** (FinOps) — вне scope Task2.

## Альтернативы

1. **Только Metrics (без Logs/Traces)** — отклонено: невозможно разобрать
   инцидент без логов и трейсов.
2. **Datadog / New Relic (SaaS)** — отклонено: vendor lock-in, дорого
   при нашем объёме. Open-source стек проще масштабировать.
3. **ELK вместо Loki** — отклонено: ELK тяжелее, дороже, Loki лучше
   интегрируется с Prometheus.
4. **Zipkin вместо Jaeger** — отклонено: Jaeger более зрелый, лучше
   с OTel.
5. **Только infrastructure metrics (без business)** — отклонено:
   инфраструктура может быть «здорова», а бизнес — «лежать».
6. **Alerting только в Grafana** — отклонено: Alertmanager — стандарт,
   лучше работает с SLO burn rate.

## Компромиссы

- **Много инструментов** — операционная нагрузка. Митигация: единый
  стек (Prometheus + Grafana + Loki + Jaeger), Helm charts для быстрого
  развёртывания.
- **Prometheus с высокой кардинальностью** (метки tenant_id, region,
  event_type) — риск «взрыва» TSDB. Митигация: кардинальность под контролем,
  recording rules для агрегатов.
- **Loki дорогой при высоком объёме логов** — structured logging помогает
  фильтровать. Митигация: только INFO+ в production, sample для DEBUG.
- **OTel collector — single point** — если упал, traces теряются. Митигация:
  HA deployment (2+ реплики), буферизация в Kafka.
- **Jaeger storage** — растёт быстро. Митигация: sampling (10% traces),
  retention 7 дней.
- **SLO 99,95% для событийной платформы** — жёстко. Митигация: error
  budget, canary deployments, chaos engineering.