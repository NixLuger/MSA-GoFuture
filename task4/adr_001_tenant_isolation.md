# ADR-001: Модель изоляции данных между тенантами

## Контекст

Task4 требует «модель изоляции данных между партнёрами» для мультитенантной
платформы GoFuture. Партнёры (тенанты) — корпоративные клиенты, региональные
операторы, франшизы — должны работать на одной платформе с **полной изоляцией
данных** друг от друга.

Уже зафиксировано:
- **Task1 C2 Final:** `svc_tenant` (Tenant Management) — управление
  партнёрами/тенантами, квоты, изоляция данных. Своя БД `Tenant DB`.
  `tenant_id` — поле в модели данных.
- **Task2 ADR-001:** `tenant_id` в envelope всех событий (multi-tenancy).
- **Task2 ADR-002:** `tenant_id` — partition key для `platform.events`.
- **Task2 ADR-001:** `Tenant.Onboarded` — событие создания тенанта.
- **Task3 ADR-002:** tenant configs — в Global Control Plane (non-PII),
  синхронизируются во все регионы.
- **Task3 ADR-001:** 4 региона, residency. Тенант живёт в **одном**
  регионе (решение пользователя: «RU не может открыть такси в SEA»).
- **Task3 ADR-005:** cross-jurisdiction PII (поездка в регионе оказания
  услуги).

**Задача:** зафиксировать модель изоляции на всех уровнях хранения
и обработки данных: БД, Kafka, ClickHouse, object storage, cache,
квоты, аудит.

## Требования

1. Полная изоляция данных между тенантами (тенант A не видит данные
   тенанта B).
2. Тенант живёт в **одном домашнем регионе** (residency, Task3).
3. Модель — shared DB + `tenant_id` + RLS (решение пользователя).
4. Изоляция на всех уровнях: БД, Kafka, ClickHouse, object storage,
   cache.
5. Квоты per tenant (защита от «шумного соседа»).
6. Аудит доступа к данным тенанта (кто, когда, к чему).
7. Не дублировать Task1–3; переиспользовать `svc_tenant`, `tenant_id`
   в событиях, tenant configs в Global Control Plane.

## Решение

### 1. Принципы изоляции (defense in depth)

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Application — tenant_id из JWT, фильтрация в коде │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: Database — RLS (Row-Level Security) в PostgreSQL  │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: Event Bus — tenant_id в envelope, ACL per domain  │
├─────────────────────────────────────────────────────────────┤
│ Layer 4: Analytics — tenant_id как колонка в ClickHouse    │
├─────────────────────────────────────────────────────────────┤
│ Layer 5: Object Storage — prefix per tenant                │
├─────────────────────────────────────────────────────────────┤
│ Layer 6: Cache — ключи с tenant_id                         │
├─────────────────────────────────────────────────────────────┤
│ Layer 7: Quotas — per-tenant лимиты                        │
├─────────────────────────────────────────────────────────────┤
│ Layer 8: Audit — лог доступа к данным тенанта              │
└─────────────────────────────────────────────────────────────┘
```

**Принцип:** каждый слой независим. Компрометация одного (например, баг
в коде без `WHERE tenant_id = ...`) не приводит к утечке — RLS на уровне
БД отсечёт.

### 2. Домашний регион тенанта

**Решение:** каждый тенант привязан к **одному домашнему региону**
(`home_region` в Tenant config).

| Аспект | Как работает |
|---|---|
| **Создание тенанта** | При onboarding выбирается `home_region` (или определяется по юрисдикции партнёра) |
| **Данные тенанта** | Хранятся только в `home_region` (БД, Kafka, ClickHouse, object storage) |
| **Пользователи тенанта** | Аутентификация — в Keycloak `home_region`. JWT содержит `tenant_id` + `region` |
| **API-вызовы** | Если пользователь тенанта физически в другом регионе (роуминг) — API-вызов к `home_region` (как пассажиры, ADR-003 Task3) |
| **Миграция тенанта** | **Не поддерживается** в базовой модели. Export/import — отдельное решение (вне scope) |
| **Мультирегиональный тенант** | **Не поддерживается** (решение пользователя). Enterprise с офисами в 4 регионах — 4 отдельных тенанта, объединённых на уровне billing (вне scope) |

**Residency:** RU-тенант → данные в region-ru. SEA-тенант → данные
в region-sea. Cross-region данных тенанта нет.

### 3. Модель изоляции в БД (shared DB + RLS)

#### 3.1 Схема данных

**Все бизнес-таблицы содержат `tenant_id`:**

```sql
-- Пример: bookings
CREATE TABLE bookings (
    booking_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,           -- обязательно
    passenger_id UUID NOT NULL,
    driver_id UUID,
    pickup_lat NUMERIC,
    pickup_lon NUMERIC,
    status VARCHAR(50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- ... другие поля
);

-- Индекс для RLS и запросов
CREATE INDEX idx_bookings_tenant ON bookings (tenant_id);
```

**RLS policy (PostgreSQL):**

```sql
-- Включаем RLS
ALTER TABLE bookings ENABLE ROW LEVEL SECURITY;

-- Политика: пользователь видит только свой tenant_id
CREATE POLICY tenant_isolation ON bookings
    USING (tenant_id = current_setting('app.tenant_id')::UUID);

-- Принудительно для всех ролей (включая владельца таблицы)
ALTER TABLE bookings FORCE ROW LEVEL SECURITY;
```

**Как `app.tenant_id` попадает в сессию:**

1. BFF / сервис получает JWT с `tenant_id` в claims.
2. При открытии соединения с БД — `SET app.tenant_id = '<tenant_id>'`.
3. RLS policy автоматически фильтрует все `SELECT`, `UPDATE`, `DELETE`.
4. `INSERT` проверяется через `WITH CHECK (tenant_id = ...)`.

**Защита от «забыли SET»:** если `app.tenant_id` не установлен —
`current_setting('app.tenant_id')` вернёт ошибку → запрос упадёт.
Fail-closed, не fail-open.

#### 3.2 Уровни доступа (роли БД)

| Роль БД | Доступ | Кто использует |
|---|---|---|
| `app_tenant_user` | RLS включён, видит только свой `tenant_id` | Сервисы (Booking, Payments, Driver…) |
| `app_platform_admin` | RLS bypass (для платформенных операций) | `svc_tenant`, `svc_identity` (только для администрирования) |
| `app_readonly` | RLS включён, только `SELECT` | Analytics read-path (per tenant) |

**Platform Admin bypass — только для сервисов, не для пользователей.**
Пользователь-платформенный-админ работает через `app_tenant_user`
с явным `tenant_id` в контексте (или через отдельный admin-сервис).

#### 3.3 Общие (shared) таблицы

Не все таблицы — per tenant. Некоторые — глобальные:

| Таблица | Модель | Обоснование |
|---|---|---|
| **tenants** (svc_tenant) | Shared, без RLS | Список всех тенантов, платформенный ассет |
| **users** (svc_identity) | Shared + `tenant_id` + RLS | Пользователь принадлежит тенанту |
| **bookings, payments, payouts** | Shared + `tenant_id` + RLS | Бизнес-данные |
| **drivers, passengers** | Shared + `tenant_id` + RLS | Принадлежат тенанту |
| **kafka_topics** (метаданные) | Shared, без RLS | Платформенный ассет |
| **apicurio_schemas** | Global Control Plane | Non-PII, глобальный ассет |
| **feature_flags** | Global Control Plane | Non-PII, глобальный ассет |

**Правило:** PII и бизнес-данные — всегда с `tenant_id` + RLS.
Платформенные метаданные (тенанты, схемы, флаги) — без RLS,
но с ограниченным доступом (только platform admin).

### 4. Изоляция в Kafka

**Что уже есть (Task2 ADR-002):**
- Топики per-domain × per-region (не per-tenant).
- `tenant_id` в envelope всех событий.
- `tenant_id` — partition key для `platform.events`.

**Что добавляем:**

#### 4.1 Consumer-side filtering

Каждый consumer **обязан** фильтровать по `tenant_id` из контекста:

```python
# Псевдокод
for event in consumer:
    if event.tenant_id != context.tenant_id:
        continue  # skip
    process(event)
```

**Защита:** middleware в consumer library проверяет `tenant_id` до
бизнес-обработки. Если `tenant_id` отсутствует в контексте — ошибка
(fail-closed).

#### 4.2 Producer-side validation

Producer **обязан** проставлять `tenant_id` в envelope. Валидация
в outbox-publisher: если `tenant_id` пустой — событие не публикуется
(ошибка).

#### 4.3 ACL на уровне Kafka

**Не tenant-level** (топики не per-tenant, это было бы 100+ тенантов ×
9 доменов × 4 региона = тысячи топиков). **Domain-level** (Task2
ADR-002): `booking-service` пишет в `*.booking.events`, читает —
`booking-consumer-group`. Внутри — фильтрация по `tenant_id`.

**Обоснование:** tenant-level ACL в Kafka = взрыв топиков. Domain-level +
`tenant_id` в envelope + consumer filtering — стандартный паттерн для
multi-tenant SaaS.

#### 4.4 Изоляция в DLQ

DLQ — per domain (Task2 ADR-005). При replay — фильтрация по
`tenant_id`, чтобы не переиграть события чужого тенанта.

### 5. Изоляция в ClickHouse (аналитика)

**Что уже есть (Task2 ADR-003):** `flink_analytics` → ClickHouse,
агрегаты.

**Что добавляем:** `tenant_id` как **обязательная колонка** в каждой
таблице ClickHouse:

```sql
CREATE TABLE trip_aggregates (
    tenant_id UUID,
    window_start DateTime,
    window_end DateTime,
    zone_id String,
    trips_count UInt64,
    avg_price Decimal(10,2),
    -- ... другие поля
) ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(window_start)
ORDER BY (tenant_id, window_start, zone_id);
```

**Row-level policies в ClickHouse:** ClickHouse поддерживает row policies
(аналог RLS) — `CREATE ROW POLICY tenant_isolation ON trip_aggregates
USING tenant_id = currentUser()`. Но проще — фильтрация в запросах
(BI-инструменты и API-слой добавляют `WHERE tenant_id = ...`).

**Изоляция в Global Analytics (Task3 ADR-002):** только агрегаты
(hashed, non-PII), но с `tenant_id`. Глобальные отчёты — по запросу,
с фильтрацией.

### 6. Изоляция в object storage

**Что:** KYC-документы, медиа, ML-модели, бэкапы.

**Модель:** prefix per tenant в S3-compatible хранилище:

```
s3://gofuture-media/
├── tenant-<uuid-1>/
│   ├── kyc/
│   ├── driver-photos/
│   └── reports/
├── tenant-<uuid-2>/
│   └── ...
└── shared/
    └── ml-models/      # глобальные модели, non-PII
```

**IAM-политика на bucket:** сервис тенанта имеет доступ только к своему
prefix (`s3:prefix = tenant-<uuid>/*`). Cross-tenant доступ — запрещён
на уровне IAM.

**Шифрование:** SSE-S3 (per-region key из Vault, Task3 ADR-006).

### 7. Изоляция в cache (Redis)

**Модель:** ключи с `tenant_id` в префиксе:

```
tenant:<uuid>:booking:<booking_id>
tenant:<uuid>:driver:<driver_id>:location
tenant:<uuid>:session:<session_id>
```

**Защита:** библиотека-обёртка над Redis автоматически добавляет
`tenant_id` в ключ. Прямой доступ к Redis без префикса — запрещён
(ревью, линтер).

**Изоляция:** Redis — per-region (Task3), tenant'ы одного региона
делят инстанс, но ключи изолированы.

### 8. Квоты per tenant (защита от «шумного соседа»)

| Ресурс | Квота | Как реализуется |
|---|---|---|
| **API rate limit** | per tenant (например, 1000 req/sec) | Kong rate limiting (per JWT `tenant_id`) |
| **DB connections** | per tenant (например, 50) | Connection pool + `tenant_id` в контексте |
| **Kafka throughput** | per tenant (например, 10 MB/sec) | Quotas в Kafka (Task2 ADR-002) |
| **ClickHouse storage** | per tenant (например, 100 GB) | Partition per tenant? Нет — `tenant_id` + мониторинг размера |
| **Object storage** | per tenant (например, 500 GB) | S3 bucket policies + мониторинг |
| **Trips per month** | per tenant (тариф) | `svc_tenant` (usage metering, ADR-004) |

**При превышении:** мягкий лимит (алерт тенанту) → жёсткий лимит
(429 / throttle). Настройки — в Tenant config (Global Control Plane).

### 9. Аудит доступа к данным тенанта

**Что логируем (уже есть в Task3 ADR-005, расширяем):**
- `tenant_id` в каждом audit-событии.
- Cross-tenant запросы (platform admin) — отдельный лог, P1 алерт.
- RLS-violations (попытки обойти) — P1 алерт.

**Куда:** Loki (per-region, Task3 ADR-006), retention 5 лет для
финансовых, 1 год для остальных.

**Кто смотрит:** Tenant Admin (свой tenant), Platform Auditor (все
тенанты, read-only), DPO (PII-доступ).

### 10. Диаграмма

Модель изоляции отражается на `C2_MultiTenant.puml`:
- `svc_tenant` — управление тенантами.
- RLS-политики — аннотация на БД (или отдельный C3).
- Kafka — `tenant_id` в envelope (аннотация).
- ClickHouse — `tenant_id` колонка.
- Object storage — prefix per tenant.
- Квоты — Kong + Kafka quotas.

**Отдельная диаграмма** для RLS-flow (как `tenant_id` проходит от JWT
до RLS) — `Sequence_IAM_Flow.puml` (ADR-002).

### 11. Что НЕ входит в это решение

- RBAC-модель (роли, права) — **ADR-002**.
- Onboarding-процесс — **ADR-003**.
- Мониторинг per tenant — **ADR-004**.
- Миграция тенанта между регионами — вне scope.
- Enterprise с выделенной БД — отдельное решение (не сейчас).
- Федерация Keycloak с внешними IdP — не требуется (решение
  пользователя).
- Мультирегиональный тенант — не поддерживается.

## Альтернативы

1. **DB-per-tenant для всех.** Отклонено: дорого (100+ тенантов ×
   4 региона × N сервисов = тысячи БД), ops-нагрузка, не масштабируется
   для SMB. Enterprise с выделенной БД — отдельное решение (не сейчас).
2. **Schema-per-tenant для всех.** Отклонено: PostgreSQL плохо
   масштабируется на тысячи схем, миграции сложные, connection pool
   per schema — дорого. RLS проще.
3. **Только application-level фильтрация (без RLS).** Отклонено:
   баг в коде = утечка. RLS — defense in depth.
4. **Только RLS без application-фильтрации.** Отклонено: RLS — последний
   рубеж, но приложение тоже должно фильтровать (производительность,
   явность). Defense in depth — оба.
5. **Tenant-per-Kafka-topic.** Отклонено: взрыв топиков (100 тенантов ×
   9 доменов × 4 региона = 3600 топиков), не масштабируется.
   Domain-level + `tenant_id` в envelope — стандарт.
6. **Мультирегиональный тенант (данные в 4 регионах).** Отклонено:
   нарушает residency (Task3), усложняет граф данных. Тенант — один
   регион.
7. **Миграция тенанта между регионами.** Отклонено: вне scope, сложно
   (export/import, downtime, residency). Если потребуется — отдельное
   решение.
8. **Внешние IdP (Okta, Azure AD) для SSO.** Отклонено: не было
   требований (решение пользователя). Keycloak можно интегрировать
   позже — отдельное решение.
9. **Отдельная БД для enterprise.** Отклонено: не сейчас (решение
   пользователя — shared DB + RLS для всех). Может быть расширением.

## Компромиссы

- **Shared DB + RLS** — «шумный сосед» возможен (один тенант грузит
  БД). Митигация: квоты (раздел 8), connection pool per tenant,
  мониторинг (ADR-004).
- **RLS — performance overhead** (каждый запрос проверяет policy).
  Митигация: индексы по `tenant_id`, бенчмарки, RLS только на
  tenant-scoped таблицах (не на платформенных).
- **`app.tenant_id` через `SET`** — если забыли SET, запрос упадёт
  (fail-closed). Митигация: библиотека-обёртка, линтер, тесты.
- **Kafka consumer filtering** — каждый consumer обязан фильтровать.
  Митигация: middleware в consumer library, code review, тесты.
- **Tenant-level ACL в Kafka не используется** — доменный ACL + фильтрация.
  Митигация: middleware, аудит.
- **Мультирегиональный тенант не поддерживается** — enterprise с офисами
  в 4 регионах = 4 тенанта. Митигация: consolidated billing (вне scope).
- **Миграция тенанта не поддерживается** — если тенант хочет сменить
  регион, это export/import (вне scope). Митигация: явно в документации.
- **Object storage prefix** — если IAM-политика настроена неверно, утечка.
  Митигация: IaC (Terraform), аудит, тесты.
- **Redis без префикса** — если библиотека не добавила `tenant_id`,
  утечка. Митигация: линтер, code review.
- **Квоты — per tenant, но не per user** — один пользователь тенанта
  может исчерпать квоту. Митигация: дополнительные per-user лимиты
  (вне scope, если потребуется).

## Ссылки на предыдущие решения

- `svc_tenant`, `tenant_id`, `svc_identity` — **Task1 C2 Final**.
- `tenant_id` в envelope, `Tenant.Onboarded`, `User.Registered` —
  **Task2 ADR-001**.
- Kafka-топики per-domain, `tenant_id` как partition key — **Task2
  ADR-002**.
- Outbox / DLQ / retry — **Task2 ADR-005**.
- Observability — **Task2 ADR-006**.
- 4 региона, residency — **ADR-001 Task3**.
- Tenant configs в Global Control Plane — **ADR-002 Task3**.
- Роуминг через API — **ADR-003 Task3**.
- Комплаенс, аудит — **ADR-005 Task3**.
- Security (Vault, mTLS) — **ADR-006 Task3**.
- RBAC, роли — **ADR-002 Task4**.
- Onboarding — **ADR-003 Task4**.
- Мониторинг — **ADR-004 Task4**.