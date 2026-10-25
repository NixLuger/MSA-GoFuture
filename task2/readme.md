# Task2 — Event-Driven архитектура

## О чём этот раздел

Задание 2 итогового спринта: спроектировать событийную платформу для
обработки 500 000 конкурентных поездок с динамическим ценообразованием
в реальном времени.

Задание выполнено в виде **6 архитектурных решений (ADR)** и **1 диаграммы
C4 уровня C2** — архитектура событийной платформы (To-Be).

## Артефакты

### Architecture Decision Records (ADR)

| # | Документ | Что фиксирует |
|---|----------|---------------|
| 1 | [ADR-001: Каталог доменных событий](adr_001_event_catalog.md) | 20 integration events: envelope, naming convention, классификация, producer/consumers, детальные схемы 7 ключевых |
| 2 | [ADR-002: Схема топиков Kafka](adr_002_kafka_topics.md) | 54 топика (per-domain × регион), retention/compaction, партиции, ACL, Apicurio Registry |
| 3 | [ADR-003: Обработка событий (Flink jobs)](adr_003_event_processing.md) | 4 PyFlink job'а (Pricing, Matching, Fraud, Analytics), exactly-once, checkpointing, watermarks |
| 4 | [ADR-004: Saga (распределённые транзакции)](adr_004_saga.md) | Гибрид: 2 orchestration-Saga (Booking Lifecycle, Payment+Payout) + 3 choreography-паттерна + 1 choreography-сценарий |
| 5 | [ADR-005: Надёжная доставка событий](adr_005_reliable_delivery.md) | Outbox, идемпотентные consumer'ы, retry-топики, DLQ, poison messages |
| 6 | [ADR-006: Мониторинг событийной платформы](adr_006_monitoring.md) | Подход (3 столпа + бизнес-метрики), 7 категорий метрик, SLO, инструменты, alerting |

### Диаграмма C4 (уровень C2)

| Диаграмма | Файл | Что показывает |
|-----------|------|----------------|
| **C2 Event-Driven Platform** | [c2_eda.puml](c2_eda.puml) | Архитектура событийной платформы: Kafka (events, retry, DLQ), 4 Flink jobs, outbox, Apicurio Registry, ML Platform, Monitoring. Цветовая маркировка: 🔵 Producer / 🟣 Consumer / 🟪 Both / 🟠 Flink / 🟡 Kafka / 🟢 Monitoring / 🔷 ML |

> Диаграмма в формате PlantUML. Для рендера: [PlantUML Online](https://www.plantuml.com/plantuml/uml/), плагин PlantUML для VS Code / IntelliJ, или GitHub-расширение PlantUML.

## Ключевые решения (сводка)

- **Каталог событий (ADR-001):** 20 integration events по 8 доменам (Booking, Driver, Pricing, Payments, Payouts, Notification, Fraud, Platform). Единый envelope с `event_id`, `event_type`, `event_version`, `region`, `correlation_id`, `causation_id` (для трассировки Saga). Схемы хранятся в Apicurio Registry, backward-compatible эволюция.

- **Топики Kafka (ADR-002):** региональное партиционирование (3 региона × 9 типов = 27 events + 27 DLQ = 54 топика, не считая retry). Retention по категориям: Critical 30 дней, Important 14 дней, Informational 72 часа; `driver.location` — compaction по `driver_id`. RF=3, `acks=all`, `enable.idempotence=true`.

- **Flink-обработка (ADR-003):** 4 независимых PyFlink job'а — `pricing-streaming`, `matching-streaming`, `fraud-streaming`, `analytics-streaming`. Exactly-once через checkpointing + idempotent sinks. Event time + watermark 5 сек. State backend — RocksDB.

- **Saga (ADR-004):** гибридная модель. Orchestration — Booking Lifecycle (orchestrator в `svc_booking`) и Payment+Payout (orchestrator в `svc_payments`), с явными шагами, таймаутами и компенсациями. Choreography — Fraud-check, Notification, Analytics, Tenant Onboarding. `correlation_id` для сквозной трассировки.

- **Надёжная доставка (ADR-005):** 3 уровня — outbox (атомарная запись в БД + Kafka), durability в Kafka (RF=3, acks=all), идемпотентные consumer'ы (дедупликация по `event_id`). Retry через отдельные топики (1/5/15 мин), DLQ, poison messages.

- **Мониторинг (ADR-006):** 3 столпа observability (metrics + logs + traces) + бизнес-метрики. 7 категорий метрик: Kafka infrastructure, Producer/outbox, Consumer, Flink jobs, Saga, Business, Data quality. SLO: pricing p99 < 100 мс, matching p99 < 500 мс, fraud p99 < 200 мс. Инструменты: Prometheus + Grafana + Loki + OTel Collector + Jaeger + Alertmanager.

## Что сдаётся (по заданию)

- [x] Диаграмма C2 с архитектурой событийной платформы — `c2_eda.puml.txt`
- [x] Текстовое описание подхода к мониторингу с обоснованием и списком метрик — `adr_006_monitoring.md` (разделы 1–3, 6–7)
- [x] Список выбранных инструментов мониторинга и их отражение на C2 — `adr_006_monitoring.md` (раздел 4) + `c2_eda.puml.txt`

## Связь с другими заданиями

- **Task1** — фундамент: 12 сервисов, Kafka как Event Bus (ADR-002/ADR-004), outbox в монолите. Task2 детализирует event-driven слой поверх этой архитектуры.
- **Task3** — глобальное развёртывание: региональное партиционирование (ADR-002) станет основой для data residency и failover.
- **Task4** — Data Platform / ML: `flink_analytics` → ClickHouse (ADR-003) + ML Platform для моделей Pricing/Fraud (ADR-003 раздел 3.3).
- **Task5** — Multi-tenancy / IAM: `tenant_id` в envelope (ADR-001) — задел для мультитенантной изоляции.

## Компромиссы архитектуры (сводка)

- **20 событий** — покрывают все 12 сервисов и все сценарии Saga. Расширение возможно через Apicurio Registry.
- **54 топика** — управляемое количество, но требует GitOps-автоматизации.
- **4 Flink job'а** — больше инфраструктуры, чем один job, но независимое масштабирование и деплой.
- **Гибридная Saga** — orchestration для критичных финансовых сценариев, choreography для простых.
- **Effective exactly-once** — at-least-once + идемпотентность = практический эквивалент exactly-once.
- **Много инструментов мониторинга** — операционная нагрузка, но полное покрытие observability.