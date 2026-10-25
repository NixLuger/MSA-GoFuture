## О задании

Task4 — финальный спринт курса архитектуры. GoFuture переходит
с Django-монолита на микросервисы (Task1), внедряет событийную
архитектуру (Task2), разворачивается глобально (Task3). Task4 —
**мультитенантная платформа**: изоляция данных между партнёрами,
IAM с RBAC и SSO, автоматизированный onboarding, мониторинг
мультитенантного окружения.

**Требования задания:**
1. Модель изоляции данных между партнёрами (тенантами).
2. IAM-система с ролевым доступом и SSO.
3. Автоматизированный процесс подключения новых партнёров (onboarding).
4. Система мониторинга мультитенантного окружения.
5. Диаграммы C2 (или C3) с IAM, мониторингом и потоками данных
   онбординга.
6. Таблица ролей и уровней доступа к данным.

## Артефакты Task4

| Файл | Что внутри |
|---|---|
| `adr_001_tenant_isolation.md` | Модель изоляции: shared DB + RLS, `tenant_id` на всех уровнях (БД, Kafka, ClickHouse, S3, Redis), квоты, аудит |
| `adr_002_iam_rbac.md` | IAM: shared realm Keycloak, RBAC (3 категории ролей), JWT claims, SSO между приложениями GoFuture, defense in depth |
| `roles_matrix.md` | Таблица ролей × уровней доступа × данных × BFF × операций (12 ролей, 3 уровня) |
| `adr_003_onboarding.md` | Onboarding: два потока (self-service / enterprise), state machine + Saga, компенсации, идемпотентность |
| `adr_004_monitoring.md` | Мониторинг: tenant-aware observability, per-tenant SLO, usage metering, дашборды, алерты |
| `C2_MultiTenant.puml` | Основная C2: IAM + monitoring + onboarding + RLS, красным — новое |
| `C3_Onboarding.puml` | C3: компоненты onboarding-процесса (Workflow, KYC, Keycloak, Notification) |
| `Sequence_Onboarding.puml` | Sequence: поток self-service + enterprise (через `alt`) |
| `Sequence_IAM_Flow.puml` | Sequence: аутентификация → авторизация → RLS |
| `README.md` | Этот файл |

## Соответствие требованиям задания

| Требование | Артефакт |
|---|---|
| Модель изоляции данных | `adr_001_tenant_isolation.md` |
| IAM с RBAC и SSO | `adr_002_iam_rbac.md` + `roles_matrix.md` + `Sequence_IAM_Flow.puml` |
| Автоматизированный onboarding | `adr_003_onboarding.md` + `C3_Onboarding.puml` + `Sequence_Onboarding.puml` |
| Система мониторинга | `adr_004_monitoring.md` |
| C2 с IAM, monitoring, onboarding | `C2_MultiTenant.puml` |
| Потоки данных онбординга | `C3_Onboarding.puml` + `Sequence_Onboarding.puml` |
| Таблица ролей и уровней доступа | `roles_matrix.md` |

## Ключевые решения (сводка)

### 1. Модель изоляции (ADR-001)

- **Shared DB + `tenant_id` + RLS** для всех тенантов (включая
  enterprise, с усиленным аудитом). Enterprise с выделенной БД —
  отдельное решение, не сейчас.
- **Тенант — один домашний регион** (residency, Task3). RU-тенант
  не открывает такси в SEA.
- **Defense in depth:** 8 слоёв изоляции (app → БД → Kafka →
  ClickHouse → S3 → Redis → квоты → аудит).
- **RLS fail-closed:** если `app.tenant_id` не установлен, запрос
  падает.
- **Kafka — domain-level ACL + `tenant_id` в envelope + consumer
  filtering** (не tenant-per-topic — это взорвало бы количество
  топиков).

### 2. IAM / RBAC / SSO (ADR-002)

- **Keycloak — shared realm** `gofuture` (не realm-per-tenant).
  `tenant_id` — claim в JWT.
- **RBAC: 12 ролей** в 3 категориях:
    - Platform (5): platform_admin, platform_support, platform_auditor,
      platform_billing, super_admin (break-glass).
    - Tenant (4): tenant_admin, tenant_manager, tenant_analyst,
      tenant_support.
    - Business (3): dispatcher, driver, passenger.
- **JWT: `roles` + `scope`** — роль → набор scopes (mapping в Keycloak).
- **SSO** — между приложениями GoFuture (passenger / driver / corp /
  admin / web) внутри одного тенанта и региона. **Без** федерации
  с внешними IdP (Okta / Azure AD) — не было требований.
- **Defense in depth: 4 слоя** — Kong (JWT) → BFF (scope, RBAC
  Middleware — библиотека) → Service (business rules) → DB (RLS).

### 3. Onboarding (ADR-003)

- **Два потока:** self-service (SMB, авто) + enterprise (KYC,
  approval, договор).
- **Onboarding Workflow — модуль внутри `svc_tenant`** (state machine
    + Saga), не отдельный микросервис.
- **Общее ядро** (шаги 3–10) для обоих потоков. Разница — шаги 1–2
  (KYC + approval).
- **Компенсация:** при ошибке тенант остаётся `failed` (для аудита),
  Keycloak group / user откатываются.
- **Идемпотентность:** `request_id` + уникальный индекс
  `(email, home_region)`.
- **Новых событий не добавляем** — `Tenant.Onboarded` достаточно.
  Внутренние шаги — operational state, не бизнес-события.

### 4. Мониторинг (ADR-004)

- **Переиспользуем Task2 ADR-006** (OTel / Prometheus / Loki /
  Grafana / Alertmanager) — не переделываем.
- **Tenant-aware middleware** в BFF — добавляет `tenant_id` в
  метрики / логи / traces.
- **Per-tenant SLO по тарифу:** SMB 99,5% / Pro 99,9% /
  Enterprise 99,95% (assumption, вынесено в Tenant config).
- **Usage metering:** BFF middleware → Prometheus → flink_analytics
  → ClickHouse → billing (trips, API calls, storage).
- **Dashboards:** Tenant Health (Tenant Admin), Platform Tenants,
  Onboarding, Billing, Security, SLO.
- **Alerts:** per-tenant routes (email Tenant Admin, Slack Platform
  Support). Cross-tenant (RLS violation, unauthorized access) → P1
  в Security.
- **Noisy neighbor detection:** `tenant_resource_share > 30%` → P2.

## Соглашение о цветах на диаграммах

| Цвет | Значение |
|---|---|
| **Красный** (`<#FFEBEE>`) | Новое в Task4 (tenant-aware middleware, RBAC middleware, Onboarding Workflow, Usage Metering) |
| **Синий** (`<#E3F2FD>`) | Существующее из Task1–3 (Kong, BFF, сервисы, Keycloak, `svc_identity`, `svc_tenant`, Kafka, БД, Global Control Plane) |
| **Зелёный** (`<#E8F5E9>`) | Monitoring (Task2 observability + per-tenant extensions) |
| **Оранжевый** (`<#FFF3E0>`) | Global Control Plane (Task3) |
| **Фиолетовый** (`<#F3E5F5>`) | Внешнее (KYC Provider) |

Соответствует требованию задания «все изменения выделить красным».

## Разрешение связей с предыдущими Task

### С Task1 (C2 Final)
- **Переиспользуем:** `svc_tenant`, `svc_identity`, Keycloak, все BFF,
  все бизнес-сервисы, Kafka, БД per service.
- **Расширяем:** `svc_tenant` — Onboarding Workflow (модуль);
  BFF — tenant-aware middleware + RBAC middleware (библиотеки);
  Keycloak — groups `tenant:<uuid>`, protocol mappers, роли.
- **Не переделываем** — всё существующее остаётся.

### С Task2 (EDA)
- **Переиспользуем:** `tenant_id` в envelope, `Tenant.Onboarded`,
  `User.Registered`, outbox / DLQ / retry, observability
  (OTel / Prometheus / Loki / Jaeger / Grafana / Alertmanager).
- **Расширяем:** observability — `tenant_id` label, per-tenant SLO,
  usage metering; `Tenant.Onboarded` payload (backward-compatible).
- **Не добавляем** новых событий — `Tenant.Onboarded` достаточно.

### С Task3 (Global deployment)
- **Переиспользуем:** 4 региона (residency), Keycloak per-region,
  Global Control Plane (tenant configs), audit / комплаенс
  (Task3 ADR-005), security (Vault, mTLS — Task3 ADR-006).
- **Расширяем:** tenant configs — SLO, квоты, тариф (non-PII);
  audit — `tenant_id` в каждом audit-событии.
- **Не дублируем** — ссылаемся на ADR Task3.

## Роли (сводка)

### Platform roles (cross-tenant)
- **platform_admin** — управление платформой и тенантами.
- **platform_support** — поддержка (read-only на данные тенантов).
- **platform_auditor** — аудит (read-only, audit logs).
- **platform_billing** — биллинг и инвойсы.
- **super_admin** — break-glass (только admin-API, OTP, P1-алерт).

### Tenant roles (внутри тенанта)
- **tenant_admin** — full доступ к своему тенанту.
- **tenant_manager** — бизнес-операции (read + write).
- **tenant_analyst** — аналитика (read-only, агрегаты).
- **tenant_support** — поддержка (read + limited write).

### Business roles (бизнес-пользователи)
- **dispatcher** — распределение поездок.
- **driver** — только свои поездки и заработок.
- **passenger** — только свои поездки.

**Детально:** `roles_matrix.md`.

## Уровни доступа к данным

| Уровень | Что доступно | Кто |
|---|---|---|
| **own** | Свои данные | passenger, driver |
| **tenant** | Данные своего тенанта (RLS по `tenant_id`) | tenant_admin, tenant_manager, tenant_analyst, tenant_support, dispatcher |
| **platform** | Все тенанты (RLS bypass + audit + P1-алерт) | platform_*, super_admin |

## Assumptions (явно помечены)

| Assumption | Значение | Где зафиксировано |
|---|---|---|
| **Модель изоляции** | Shared DB + RLS для всех (enterprise — с усиленным аудитом) | ADR-001 Task4 |
| **Тенант — один регион** | Да, residency | ADR-001 Task4 |
| **Миграция тенанта** | Не поддерживается | ADR-001 Task4 |
| **Мультирегиональный тенант** | Не поддерживается (4 тенанта + consolidated billing) | ADR-001 Task4 |
| **Keycloak — shared realm** | Да (не realm-per-tenant) | ADR-002 Task4 |
| **Внешние IdP** | Не поддерживаются (только наш Keycloak) | ADR-002 Task4 |
| **Один пользователь — один тенант** | Да (сотрудник 2 тенантов — 2 аккаунта) | ADR-002 Task4 |
| **Cross-tenant SSO** | Нет | ADR-002 Task4 |
| **Cross-region SSO** | Нет (роуминг через API) | ADR-002 Task4 |
| **Self-service onboarding** | Авто, без approval | ADR-003 Task4 |
| **Enterprise onboarding** | KYC + approval, ≤ 5 раб. дней | ADR-003 Task4 |
| **SLO по тарифу** | SMB 99,5% / Pro 99,9% / Enterprise 99,95% | ADR-004 Task4 |
| **Usage metering** | Trips + API calls + storage, через ClickHouse | ADR-004 Task4 |
| **Noisy neighbor threshold** | 30% ресурсов | ADR-004 Task4 |

## Что НЕ входит в Task4 (осознанно)

- Модель изоляции на уровне инфраструктуры (K8s namespaces per tenant) —
  shared cluster, RLS достаточно.
- Выделенные БД для enterprise — отдельное решение (не сейчас).
- Федерация Keycloak с внешними IdP — не требуется (решение
  пользователя). Можно добавить позже.
- ABAC / ReBAC — базовый RBAC покрывает потребности.
- Миграция тенанта между регионами — вне scope.
- Billing Service детально (тарифы, invoice) — вне scope.
- Юридические тексты Privacy Policy / договоров — вне scope.
- Детальный дизайн KYC-провайдера (выбор, интеграция) — инфраструктурное
  решение.
- Runbook по onboarding failure — требование зафиксировано, реализация
  вне scope.
- Game day (тестирование onboarding / failover) — требование
  зафиксировано, реализация вне scope.

## Открытые вопросы / что можно улучшить

1. **Enterprise с выделенной БД** — если бизнес потребует, отдельное
   решение (schema-per-tenant или DB-per-tenant для enterprise).
2. **Федерация Keycloak с внешними IdP** — если корп-клиенты попросят
   SSO через свой Okta / Azure AD, отдельное решение.
3. **Мультирегиональный тенант** — если enterprise с офисами в 4
   регионах, consolidated billing + cross-region aggregation (вне
   scope).
4. **ABAC** — если появятся сложные политики (доступ по времени,
   по атрибутам ресурса), отдельное решение.
5. **Миграция тенанта** — если бизнес потребует смену региона,
   export / import + residency-согласование (вне scope).
6. **Kafka per-tenant lag** — ограниченная точность (domain-level
   топики). Возможно, отдельный stream для per-tenant lag (вне scope).

## Структура директории Task4
```
Task4/
├── README.md
├── adr_001_tenant_isolation.md
├── adr_002_iam_rbac.md
├── roles_matrix.md
├── adr_003_onboarding.md
├── adr_004_monitoring.md
├── C2_MultiTenant.puml
├── C3_Onboarding.puml
├── Sequence_Onboarding.puml
└── Sequence_IAM_Flow.puml
```

## Ссылки на предыдущие Task

- **Task1** — домены, сервисы, C2 Final, миграция (Strangler Fig).
- **Task2** — EDA, Kafka-топики, outbox, DLQ, retry, observability.
- **Task3** — 4 региона, репликация, геомаршрутизация, failover,
  комплаенс, security.
- **Task4** — эта директория.