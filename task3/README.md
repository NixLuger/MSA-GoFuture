## О задании

Task3 — финальный спринт курса архитектуры. GoFuture переходит
с Django-монолита на микросервисы (Task1) и внедряет событийную
архитектуру (Task2). Task3 — глобальное развёртывание: выбор регионов,
репликация, геомаршрутизация, failover, комплаенс, защита.

**Требования задания:**
1. Текстовое обоснование выбора регионов.
2. Схема репликации.
3. Схема реализации геомаршрутизации.
4. Диаграмма аварийного переключения (C4) с точками отказа
   и таблица сценариев отказа.
5. Доработать C2 с security-компонентами + текстовое обоснование
   механизмов защиты.
6. (Доп.) Стратегия комплаенса.

**Все изменения — красным цветом** на диаграммах.

## Артефакты Task3

| Файл | Что внутри |
|---|---|
| `adr_001_regions.md` | Выбор 4 регионов, регуляторные зоны, 2 зоны на регион, арифметика топиков 3→4 |
| `adr_002_replication.md` | 5 корзин репликации, DR-модель, placement Global Control Plane (EU/Франкфурт), RTO/RPO |
| `adr_003_georouting.md` | Слои ingress, GeoDNS + GLB, роуминг пассажиров, PII в региональных данных |
| `adr_004_failover.md` | 4 уровня отказа, Failover Orchestrator, reconciliation, таблица 14 сценариев |
| `adr_005_compliance.md` | 4 регуляторных зоны, cross-jurisdiction, GDPR, breach notification, audit |
| `adr_006_security.md` | Обоснование механизмов защиты (WAF, DDoS, Firewall, mTLS, Vault) |
| `C2_Replication.puml` | 4 boundary регионов × 2 зоны, Global Control Plane, Global Analytics, репликация |
| `C2_Georouting.puml` | Edge-слой (GeoDNS → WAF → DDoS → Firewall → GLB → Kong), роуминг |
| `C2_Failover.puml` | Точки отказа, Failover Orchestrator, шаги переключения, reconciliation |
| `README.md` | Этот файл |

## Выбранные регионы (кратко)

| Регион | Зона 1 | Зона 2 | Регуляторика |
|---|---|---|---|
| **region-ru** | Москва | Санкт-Петербург | 152-ФЗ |
| **region-sea** | Сингапур | Батам (частичный DR, non-PII) | PDPA SG + PDPA ID |
| **region-latam** | Сан-Паулу | Рио-де-Жанейро | LGPD BR |
| **region-mena** | Дубай | Абу-Даби | UAE DP Law |

Подробное обоснование — `adr_001_regions.md`.

## Как закрыты требования задания

| Требование | Артефакт |
|---|---|
| Текстовое обоснование регионов | `adr_001_regions.md` |
| Схема репликации | `adr_002_replication.md` + `C2_Replication.puml` |
| Схема геомаршрутизации | `adr_003_georouting.md` + `C2_Georouting.puml` |
| Диаграмма failover (C4) + точки отказа | `adr_004_failover.md` + `C2_Failover.puml` |
| Таблица сценариев отказа | `adr_004_failover.md`, раздел «Таблица сценариев отказа» (14 сценариев) |
| Доработка C2 с security + обоснование | `C2_Georouting.puml` (полная цепочка защиты) + `C2_Replication.puml` (`Regional Edge & Security` в каждом регионе) + `adr_006_security.md` |
| (Доп.) Комплаенс | `adr_005_compliance.md` |

**Про security-overlay:** отдельный `C2_Security_Overlay.puml`
**не создавался** — визуальное требование «доработать C2» закрыто
двумя существующими диаграммами:
- `C2_Georouting.puml` — полная цепочка `GeoDNS → WAF → DDoS →
  Network Firewall → GLB → Regional Ingress (Kong)`, per-region,
  с health checks.
- `C2_Replication.puml` — `Regional Edge & Security` в каждом регионе.

Дублировать эти компоненты в отдельном overlay было бы избыточно.
Текстовое обоснование механизмов — `adr_006_security.md`.

## Соглашение о цветах на диаграммах

| Цвет | Значение |
|---|---|
| **Красный** (`<#FFEBEE>` / `<#FFCDD2>`) | Новое в Task3 (репликация, DR, failover, edge, externals новых регионов, точки отказа) |
| **Синий** (`<#E3F2FD>`) | Существующее из Task1/Task2 (C2 Final, сервисы, Kong, RU-платёжные системы) |
| **Оранжевый** (`<#FFF3E0>`) | Global Control Plane / Global Analytics (новое, но глобальное) |
| **Фиолетовый** (`<#F3E5F5>`) | DR-реплики во второй зоне |

Соответствует требованию задания «все изменения выделить красным».

## Разрешение конфликтов с предыдущими Task

### 99,95% → 99,99%
- Task1 ADR-001 зафиксировал 99,95% как milestone.
- Task2 уже зафиксировал 99,99% (в схемах и ADR).
- Task3 подтверждает 99,99% как целевое. Достижимость: TIER IV + ESXi +
  K8s с резервированием + автоматический failover за ≤ 45 мин.

### 3 → 4 региона
- Task2 ADR-002 использовал 3 placeholder-региона (`region-a/b/c`).
- Task3 материализует 4 реальных региона (SEA и LatAm — обязательны
  по заданию; RU — домашний рынок; MENA — часовые пояса + растущий
  рынок).
- **Арифметика топиков пересчитана:** 135 → 180 (см. `adr_001_regions.md`,
  раздел 5).

### 200k → 500k конкурентных поездок
- Task1 ADR-001: 200k как milestone 2 мес.
- Task2: 500k как целевая нагрузка.
- Task3: 500k как текущая для capacity planning.

### Kafka cross-region
- Task2 ADR-002: MirrorMaker отвергнут (residency + изоляция).
- Task3: DR через БД-репликацию + reconciliation (не через Kafka).
  См. `adr_002_replication.md`, `adr_004_failover.md`.

### PII в региональных данных
- Уточнено в `adr_003_georouting.md`, раздел 5:
  - Booking DB — **с PII** (данные поездок региона).
  - ClickHouse — **без PII** (только агрегаты).
  - Global Identity Directory — **без PII** (`user_id → home_region`).
  - Edge Cache — **без PII** (только токены и хэши).

## Assumptions (явно помечены)

| Assumption | Значение | Где зафиксировано |
|---|---|---|
| **RTO** | ≤ 45 мин критичные, ≤ 4 ч не-критичные | ADR-002, ADR-004 |
| **RPO** | ≤ 15 мин финансы, ≤ 1 ч остальные | ADR-002 |
| **SEA-PII RPO** | ∞ (принятый риск) | ADR-001, ADR-004 |
| **p95 latency ingress** | ≤ 100 мс | ADR-003 |
| **Health check interval** | 10 сек (GLB), 5 сек (PG) | ADR-003, ADR-004 |
| **Reconciliation окно** | ≤ 30 мин после failover | ADR-004 |
| **Data subject rights срок** | 10 рабочих дней (по 152-ФЗ) | ADR-005 |
| **Breach notification** | 24 ч (по 152-ФЗ) | ADR-005 |

## Что НЕ входит в Task3 (осознанно)

- Детальный дизайн Kafka-топиков — Task2 ADR-002.
- Outbox / DLQ / retry — Task2 ADR-005.
- Observability (метрики, дашборды) — Task2 ADR-006.
- Инфраструктура (TIER IV, ESXi, K8s sizing) — вне scope ADR.
- Runbook по failover — требование зафиксировано, реализация вне scope.
- Юридические тексты Privacy Policy — вне scope.
- Game day (тестирование failover) — требование зафиксировано,
  реализация вне scope.
- CDN для статики — предполагается Cloudflare CDN, детали не фиксируем.

## Открытые вопросы / что можно улучшить

1. **SEA-PII RPO = ∞** — принятый риск. Если бизнес готов на cross-border
   DR (Батам как full-DR) — потребуется согласование с PDPC SG и Komdigi
   ID. Вне scope Task3.
2. **Global Control Plane в EU** — политически нейтрально, но формально
   «пятая зона». Альтернатива — Швейцария (дороже). Решение зафиксировано.
3. **Fallback PSP** — список фиксирован (ADR-005), но интеграция каждого
   PSP — отдельная задача (вне scope).
4. **Driver roaming** — не поддерживается (assumption). Если бизнес
   потребует — граф данных усложнится (driver profile, KYC, payouts
   cross-border), потребуется отдельный ADR.
5. **Audit log schema** — стандартизирована, но детальный дизайн
   (поля, retention, доступ) — вне scope Task3.

## Структура Task3

```
Task3/
├── README.md
├── adr_001_regions.md
├── adr_002_replication.md
├── adr_003_georouting.md
├── adr_004_failover.md
├── adr_005_compliance.md
├── adr_006_security.md
├── C2_Replication.puml
├── C2_Georouting.puml
└── C2_Failover.puml
```