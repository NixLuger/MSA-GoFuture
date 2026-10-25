# Task1 — Проектирование доменов и декомпозиция монолита GoFuture

## О чём этот раздел

Задание 1 итогового спринта: разработать поэтапный план декомпозиции
монолита GoFuture на доменные сервисы с минимизацией времени простоя.

Задание выполнено в виде **5 архитектурных решений (ADR)** и **2 диаграмм C4
уровня C2** — целевого и переходного состояний.

## Артефакты

### Architecture Decision Records (ADR)

| # | Документ | Что фиксирует |
|---|----------|---------------|
| 1 | [ADR-001: Нефункциональные требования](adr_001_adr_nft.md) | НФТ из бизнес-требований: 200k поездок, 99,95% доступности, time-to-market ≤ 2 недель, мультирегион, комплаенс |
| 2 | [ADR-002: Карта сервисов и доменные границы](adr_002_service_map.md) | 12 сервисов (10 бизнес + 2 платформенных), 4 BFF, инфраструктурные компоненты |
| 3 | [ADR-003: Приоритетная очерёдность](adr_003_priority.md) | 7 волн миграции за 12 месяцев по критериям Foundation / Простота / Риск / Бизнес-ценность |
| 4 | [ADR-004: Механизмы обратной совместимости](adr_004_acl.md) | ACL в монолите, Kong-роутинг v1/v2, транзакционный outbox, feature flags |
| 5 | [ADR-005: План миграции данных](adr_005_migration.md) | Pre-copy + CDC + dual-write + reconciliation, per-service планы |

### Диаграммы C4 (уровень C2)

| Диаграмма | Файл | Что показывает |
|-----------|------|----------------|
| **C2 Final State** | [C2_FinalState.puml.txt](C2_FinalState.puml) | Целевое состояние после миграции (месяц 12+): 12 сервисов, 5 BFF, монолит отсутствует |
| **C2 Transition State** | [C2_TransitionState.puml.txt](C2_TransitionState.puml) | Состояние во время миграции (месяцы 1–11): монолит + ACL + новые сервисы. Цветовая маркировка: 🔴 legacy / 🟢 new / 🟠 ACL |

> Диаграммы в формате PlantUML. Для рендера: [PlantUML Online](https://www.plantuml.com/plantuml/uml/), плагин PlantUML для VS Code / IntelliJ, или GitHub-расширение PlantUML.

## Ключевые решения (сводка)

- **Карта сервисов (ADR-002):** 10 бизнес-сервисов (Booking, Driver Profile, Driver Status, Pricing, Payments, Payouts, Notification, Geography, Analytics, Fraud) + 2 платформенных (Identity/Auth, Tenant Management). Database per Service, bounded context по DDD, sync только где нужен немедленный ответ.

- **Очерёдность (ADR-003):** волна 0 — инфраструктура и ACL-каркас; волна 1 — Identity/Auth и Notification; волны 2–4 — foundation-сервисы (Tenant, Geography, Driver Profile, Pricing, Driver Status); волна 5 — денежные (Payments, Payouts); волна 6 — ядро (Booking). Analytics и Fraud — параллельно.

- **Обратная совместимость (ADR-004):** ACL внутри монолита (routing, v2 Controllers, Model Translators, HTTP Clients, Kafka Producer через outbox). Kong-роутинг: `/v1/*` → монолит, `/v2/*` → ACL, `/bff/*` → BFF. Feature flags для управления v1/v2 и kill switch. Монолит — только producer.

- **Миграция данных (ADR-005):** общий паттерн pre-copy → CDC (Debezium) → dual-write → verification → cutover → cleanup. Zero-downtime для критичных сервисов, 5–15 мин для не-критичных. Analytics — гибрид (последние 6–12 мес в новую платформу, старое в архив). Driver Status — без миграции истории (volatile).

## Что сдаётся (по заданию)

- [x] Список нефункциональных требований — `adr_001_adr_nft.md`
- [x] Диаграмма C2 To-Be с механизмом обратной совместимости — `C2_FinalState.puml.txt`, `C2_TransitionState.puml.txt`, `adr_004_acl.md`
- [x] Описание очерёдности выделения сервисов с обоснованием — `adr_003_priority.md`
- [x] План миграции — `adr_005_migration.md`

## Связь со следующими заданиями

- **Task2 (Event-Driven):** детализирует Kafka-топики, доменные события, Saga поверх архитектуры из ADR-002/ADR-004.
- **Task3 (Global deployment):** детализирует мультирегиональность из ADR-001.
- **Task4 (Data Platform / ML):** раскрывает Data Platform и ML Platform из ADR-002.
- **Task5 (Multi-tenancy / IAM):** раскрывает Tenant Management и Identity из ADR-002.