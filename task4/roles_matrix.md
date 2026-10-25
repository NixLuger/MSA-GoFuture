# Матрица ролей и уровней доступа к данным

## Назначение документа

Требование Task4: «Таблица с выбором ролей и необходимыми уровнями доступа к данным».

Документ детализирует ADR-002 (`adr_002_iam_rbac.md`): перечень ролей, их scopes, уровни доступа к данным и привязка к BFF / клиентам. Источник данных — RBAC-модель из ADR-002 (shared realm, `roles` + `scope` в JWT, RLS по `tenant_id`).

## Уровни доступа к данным

| Уровень | Что доступно | Реализация |
|---|---|---|
| **own** | Только свои данные (свой профиль, свои поездки, свой заработок) | RLS по `tenant_id` + фильтр `user_id = current_user` (в сервисе или отдельной RLS policy) |
| **tenant** | Все данные своего тенанта (все поездки, все водители, отчёты, billing) | RLS по `tenant_id` |
| **platform** | Все тенанты (с аудитом, P1-алертом) | RLS bypass (platform_admin DB role, ADR-001 Task4, раздел 3.2) — только для platform-сервисов |

## Категории ролей

| Категория | Привязка | Кол-во ролей |
|---|---|---|
| **Platform** | Cross-tenant (платформа) | 5 (вкл. super_admin) |
| **Tenant** | Внутри одного тенанта | 4 |
| **Business** | Внутри одного тенанта, бизнес-пользователь | 3 |

## Сводная матрица ролей

### Platform roles

| Роль | Scope | Уровень доступа | BFF / клиент | Кто назначает |
|---|---|---|---|---|
| **platform_admin** | `tenant:create`, `tenant:read`, `tenant:update`, `tenant:delete`, `tenant:config:write`, `feature_flag:write`, `payout:read:all`, `booking:read:all` | **platform** | `BFF Admin` | Другой platform_admin (двухключевой принцип) |
| **platform_support** | `tenant:read`, `booking:read:all`, `payment:read:all`, `driver:read:all`, `user:read:all`, `support:write` (комментарии к инцидентам) | **platform** (read-only) | `BFF Admin` | platform_admin |
| **platform_auditor** | `audit:read:all`, `compliance:read:all`, `tenant:read` (без бизнес-данных) | **platform** (read-only, audit logs) | `BFF Admin` | platform_admin |
| **platform_billing** | `tenant:read`, `billing:read:all`, `billing:write:all`, `invoice:read:all` | **platform** (billing-данные) | `BFF Admin` | platform_admin |
| **super_admin** | `*:*` (full, но только через admin-API, с OTP + аудитом + P1-алертом) | **platform** (full) | `BFF Admin` (только admin-API) | Только вручную (break-glass, 2 человека) |

**Особенности platform-ролей:**
- **Не привязаны к `tenant_id`** — `tenant_id` в JWT отсутствует или = `null`. RLS bypass (ADR-001 Task4, раздел 3.2).
- **Каждый доступ к данным тенанта → audit log + P1-алерт** в Slack (ADR-002, раздел 9).
- **super_admin** — не для повседневной работы. Break-glass процедура: 2 человека, OTP, аудит, P1-алерт, post-mortem.

### Tenant roles

| Роль | Scope | Уровень доступа | BFF / клиент | Кто назначает |
|---|---|---|---|---|
| **tenant_admin** | `tenant:read`, `tenant:update` (свой), `user:manage` (свой тенант), `role:assign` (в своём тенанте), `booking:read`, `booking:write`, `driver:read`, `driver:write`, `payout:read`, `payout:write`, `report:read`, `billing:read`, `billing:write`, `config:write` (квоты своего тенанта) | **tenant** (full) | `BFF Corporate`, `BFF Admin` | platform_admin (при onboarding) или другой tenant_admin |
| **tenant_manager** | `booking:read`, `booking:write`, `driver:read`, `driver:write`, `report:read`, `payout:read`, `user:read` | **tenant** (read + write бизнес) | `BFF Corporate` | tenant_admin |
| **tenant_analyst** | `report:read`, `booking:read:aggregated`, `driver:read:aggregated`, `payout:read:aggregated`, `analytics:read` | **tenant** (read-only аналитика) | `BFF Corporate` | tenant_admin |
| **tenant_support** | `booking:read`, `booking:cancel` (ограниченное), `driver:read`, `user:read`, `support:write` (комментарии) | **tenant** (read + limited write) | `BFF Corporate` | tenant_admin |

**Особенности tenant-ролей:**
- **Всегда привязаны к `tenant_id`** — JWT содержит `tenant_id`, RLS фильтрует.
- **`role:assign` у tenant_admin** — ограничен: не может назначить platform-роль, не может назначить себя super_admin, не может назначить роль вне своего тенанта.
- **`booking:cancel` у tenant_support** — ограниченное: только отмена поездки в определённых статусах (не в `in_progress`, не в `completed`).

### Business roles

| Роль | Scope | Уровень доступа | BFF / клиент | Кто назначает |
|---|---|---|---|---|
| **dispatcher** | `booking:read`, `booking:assign` (назначить водителя), `driver:read:location`, `driver:read:status` | **tenant** (booking + driver в своём тенанте) | `BFF Corporate` | tenant_admin, tenant_manager |
| **driver** | `booking:read:own`, `booking:accept`, `booking:complete`, `driver:profile:read:own`, `driver:profile:update:own`, `payout:read:own`, `location:write:own` | **own** (только свои поездки и заработок) | `BFF Driver` | tenant_admin, tenant_manager |
| **passenger** | `booking:create`, `booking:read:own`, `booking:cancel:own`, `payment:create:own`, `profile:read:own`, `profile:update:own` | **own** (только свои поездки) | `BFF Passenger`, `BFF Web` | self-registration (после onboarding тенанта) |

**Особенности business-ролей:**
- **`driver` и `passenger` — always `own`** — RLS по `tenant_id` + доп. фильтр `user_id = current_user` (ADR-001 Task4, раздел 6).
- **`dispatcher` — `tenant`** — видит всех водителей и все поездки своего тенанта, но не PII пассажиров в открытом виде (только `hashed` или маскированные данные, если не требуется для операции).
- **`passenger` — self-registration** — регистрируется сам через passenger-app в рамках существующего тенанта (тенант определяется по региону + бренду).

## Матрица «роль × уровень доступа × данные»

| Данные → | Профиль (own) | Профиль (tenant) | Поездки (own) | Поездки (tenant) | Поездки (platform) | Финансы (own) | Финансы (tenant) | Финансы (platform) | Audit | Config / billing |
|---|---|---|---|---|---|---|---|---|---|---|
| **platform_admin** | — | read (all) | — | read (all) | read (all) | — | read (all) | read (all) | read | write |
| **platform_support** | — | read (all) | — | read (all) | read (all) | — | read (all) | read (all) | — | — |
| **platform_auditor** | — | read (all, без PII) | — | — | — | — | — | — | read (all) | — |
| **platform_billing** | — | read (all) | — | — | — | — | read (all) | read (all) | — | read (billing) |
| **super_admin** | full | full | full | full | full | full | full | full | full | full |
| **tenant_admin** | — | read/write (свой) | — | read/write (свой) | — | — | read/write (свой) | — | — | read/write (свой) |
| **tenant_manager** | — | read (свой) | — | read/write (свой) | — | — | read (свой) | — | — | read (свой) |
| **tenant_analyst** | — | read (aggregated, свой) | — | read (aggregated, свой) | — | — | read (aggregated, свой) | — | — | — |
| **tenant_support** | — | read (свой) | — | read + cancel (свой) | — | — | read (свой) | — | — | — |
| **dispatcher** | — | read (свой, маскированный) | — | read/assign (свой) | — | — | — | — | — | — |
| **driver** | read/write (own) | — | read/accept/complete (own) | — | — | read (own) | — | — | — | — |
| **passenger** | read/write (own) | — | create/read/cancel (own) | — | — | create (own) | — | — | — | — |

**Легенда:**
- `read` / `write` — чтение / изменение.
- `own` — только свои данные (RLS + `user_id` фильтр).
- `tenant` — все данные своего тенанта (RLS по `tenant_id`).
- `platform` — все тенанты (RLS bypass + audit).
- `aggregated` — только агрегаты (без сырых PII).
- `маскированный` — PII в маскированном виде (например, `+7***1234`).
- `—` — нет доступа.

## Привязка ролей к BFF / клиентам

| BFF / клиент | Клиенты | Роли, которые используются |
|---|---|---|
| **BFF Passenger** | Passenger app, Web frontend | `passenger` |
| **BFF Driver** | Driver app | `driver` |
| **BFF Corporate** | Corp portal | `tenant_admin`, `tenant_manager`, `tenant_analyst`, `tenant_support`, `dispatcher` |
| **BFF Admin** | Admin panel | `platform_admin`, `platform_support`, `platform_auditor`, `platform_billing`, `super_admin`, `tenant_admin` (для управления своим тенантом) |
| **BFF Web** | Web frontend | `passenger` |

**Один пользователь может иметь несколько ролей** — например, tenant_admin + dispatcher. В JWT `roles: ["tenant_admin", "dispatcher"]`, scopes объединяются.

## Матрица «роль × BFF»

| Роль → BFF | Passenger | Driver | Corporate | Admin | Web |
|---|---|---|---|---|---|
| **platform_admin** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **platform_support** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **platform_auditor** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **platform_billing** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **super_admin** | ❌ | ❌ | ❌ | ✅ (admin-API) | ❌ |
| **tenant_admin** | ❌ | ❌ | ✅ | ✅ (свой тенант) | ❌ |
| **tenant_manager** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **tenant_analyst** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **tenant_support** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **dispatcher** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **driver** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **passenger** | ✅ | ❌ | ❌ | ❌ | ✅ |

**Ограничение в Keycloak:** `aud` в JWT содержит только те клиенты, к которым у пользователя есть доступ. Например, `driver` имеет `aud: ["driver-app"]` — не может получить токен для `admin-panel`.

## Матрица «роль × операция (пример: Booking)»

| Операция → Роль | Create booking | Read own booking | Read all bookings (tenant) | Assign driver | Cancel booking | Complete booking |
|---|---|---|---|---|---|---|
| **platform_admin** | — | — | ✅ (all) | — | ✅ (all, с аудитом) | — |
| **super_admin** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **tenant_admin** | — | — | ✅ (свой) | ✅ (свой) | ✅ (свой) | — |
| **tenant_manager** | — | — | ✅ (свой) | ✅ (свой) | ✅ (свой) | — |
| **tenant_support** | — | — | ✅ (свой) | — | ✅ (ограниченно) | — |
| **dispatcher** | — | — | ✅ (свой) | ✅ (свой) | — | — |
| **driver** | — | ✅ (own) | — | — | — | ✅ (own) |
| **passenger** | ✅ (own) | ✅ (own) | — | — | ✅ (own) | — |

**Ограничения:**
- `tenant_support` `cancel` — только в статусах `created`, `assigned`; не в `in_progress`, `completed`.
- `platform_admin` `cancel` — только через admin-API, с аудитом и P1-алертом.

## Что важно зафиксировать

1. **Один пользователь — один тенант** (ADR-002, раздел 5). Сотрудник двух тенантов — два аккаунта.
2. **Platform-роли не привязаны к `tenant_id`** — RLS bypass, каждый доступ к данным тенанта → audit + P1-алерт.
3. **super_admin — break-glass** — 2 человека, OTP, только admin-API, post-mortem.
4. **`driver` и `passenger` — always `own`** — RLS + `user_id` фильтр.
5. **`dispatcher` — `tenant` с маскированием PII** — видит поездки, но PII пассажира маскирован, если не требуется для операции.
6. **`tenant_analyst` — только агрегаты** — без сырых PII.
7. **Scopes группируются по ролям** — роль → набор scopes в Keycloak.
8. **Проверка scopes — в BFF** (RBAC Middleware — библиотека, ADR-002).
9. **`aud` в JWT** — ограничивает клиентов, к которым есть доступ.
10. **Cross-tenant доступа нет** — кроме platform-ролей и super_admin (с аудитом).

## Ссылки

- Модель изоляции (RLS, `tenant_id`) — **ADR-001 Task4**.
- RBAC-модель, Keycloak, JWT, SSO — **ADR-002 Task4**.
- Onboarding (создание ролей, первого админа) — **ADR-003 Task4**.
- Мониторинг (per-tenant audit, SLO) — **ADR-004 Task4**.