# ADR-002: IAM — RBAC и Keycloak (без внешних IdP)

## Контекст

Task4 требует «IAM-систему с ролевым доступом и SSO». ADR-001 зафиксировал
модель изоляции (shared DB + RLS, `tenant_id` в JWT и данных). Теперь нужно
спроектировать аутентификацию и авторизацию: какие роли, какие права, как
Keycloak расширяется под мультитенантность.

Уже зафиксировано:
- **Task1 C2 Final:** `Keycloak` (OIDC-провайдер), `svc_identity`
  (управление пользователями, ролями, сессиями), 5 BFF.
- **Task1:** `svc_identity` работает через Keycloak Admin API.
- **Task2 ADR-001:** `User.Registered` (producer — Identity).
- **Task3 ADR-001:** Keycloak per-region, replication между зонами.
- **Task3 ADR-002:** Keycloak realm + backup в той же регуляторной зоне.
- **ADR-001 Task4:** `tenant_id` в JWT, RLS по `tenant_id`.

**Задача:** зафиксировать RBAC-модель, структуру Keycloak, JWT claims,
authorization flow, уровни доступа к данным. Детальная матрица ролей —
в `roles_matrix.md`.

## Требования

1. RBAC: роли и права, привязанные к `tenant_id`.
2. SSO между приложениями GoFuture (один пользователь — один токен).
3. Keycloak — единственный IdP, без внешней федерации.
4. JWT содержит `tenant_id`, роли, scope, `region` (Task3).
5. Authorization — defense in depth: coarse-grained (Kong) +
   fine-grained (app / RLS).
6. Уровни доступа к данным: own / tenant / platform.
7. Аудит доступа (Task3 ADR-005, ADR-001 Task4).
8. Не дублировать Task1–3; расширять Keycloak, не переделывать.

## Решение

### 1. Keycloak — shared realm (не realm-per-tenant)

**Решение:** **один shared realm** `gofuture` на регион, все тенанты
внутри. `tenant_id` — атрибут пользователя и claim в JWT.

**Почему не realm-per-tenant:**
- Keycloak плохо масштабируется на тысячи realm'ов (память, CPU,
  Admin API latency).
- 100+ тенантов × 4 региона = 400+ realm'ов → ops-кошмар.
- Shared realm + `tenant_id` в claims + RBAC — стандартный паттерн
  для multi-tenant SaaS.

**Что внутри shared realm:**

| Сущность Keycloak | Использование |
|---|---|
| **Users** | Все пользователи всех тенантов + платформенные |
| **Groups** | `tenant:<uuid>` — группа тенанта; пользователь в одной группе |
| **Roles** | Platform roles + Tenant roles (см. ADR-002, раздел 3) |
| **Clients** | `passenger-app`, `driver-app`, `corp-portal`, `admin-panel`, `web-frontend` — по одному на приложение |
| **User attributes** | `tenant_id`, `home_region` |
| **Protocol mappers** | Добавляют `tenant_id`, `region`, `roles` в JWT |
| **Authentication flows** | Стандартный (username/password + OTP для админов) |
| **SSO sessions** | Одна сессия на пользователя, SSO между клиентами |

**Realm per region:** Keycloak развёрнут в каждом регионе (Task3).
Realm `gofuture` в каждом регионе — **свой**, с пользователями этого
региона (residency). Cross-region федерации realm'ов нет (роуминг —
через API, ADR-003 Task3).

**SSO между приложениями:** пользователь логинится в passenger-app →
получает SSO session cookie → при открытии corp-portal (того же тенанта,
того же региона) Keycloak видит активную сессию → выдаёт токен без
повторного логина.

### 2. JWT claims

**Структура JWT (access token):**

```json
{
  "iss": "https://keycloak.region-ru.gofuture.io/realms/gofuture",
  "sub": "<user_id>",
  "aud": ["passenger-app", "corp-portal"],
  "exp": 1735689600,
  "iat": 1735686000,
  "tenant_id": "<tenant_uuid>",
  "region": "region-ru",
  "roles": ["tenant_admin", "dispatcher"],
  "scope": "booking:read booking:write payout:read",
  "email": "user@example.com",
  "email_verified": true,
  "preferred_username": "user@example.com"
}
```

| Claim | Назначение | Кто использует |
|---|---|---|
| `tenant_id` | Тенант пользователя | RLS, Kafka filtering, API |
| `region` | Домашний регион | Роуминг, residency |
| `roles` | Роли (RBAC) | Kong (coarse), app (fine) |
| `scope` | Права (permissions) | App (fine-grained) |
| `aud` | Аудитория (приложение) | Keycloak, Kong |
| `sub` | user_id | Все сервисы |

**Про `roles` vs `scope`:**
- `roles` — **кто ты** (tenant_admin, dispatcher, driver).
- `scope` — **что можно** (booking:read, payout:write).
- Роль → набор scopes (mapping в Keycloak).
- App проверяет **scope**, не роль (гибче: добавили scope роли —
  не меняем код).

**Token lifetime:** access 15 мин, refresh 8 ч. Rotation — refresh
token rotation включён.

### 3. RBAC-модель (базовый набор ролей)

RBAC Middleware — библиотека внутри BFF, не отдельный сервис

**Три категории ролей:**

#### 3.1 Platform roles (платформенные, cross-tenant)

| Роль | Что делает | Уровень доступа |
|---|---|---|
| **platform_admin** | Управление платформой: создание/удаление тенантов, глобальные настройки, feature flags | Все тенанты (read + admin) |
| **platform_support** | Поддержка: просмотр данных тенантов для решения инцидентов | Все тенанты (read-only) |
| **platform_auditor** | Аудит: просмотр audit logs, compliance-отчёты | Все тенанты (read-only, audit logs) |
| **platform_billing** | Биллинг: тарифы, квоты, инвойсы | Все тенанты (billing-данные) |

#### 3.2 Tenant roles (внутри тенанта)

| Роль | Что делает | Уровень доступа |
|---|---|---|
| **tenant_admin** | Администратор тенанта: пользователи, настройки, квоты, billing | Свой тенант (full) |
| **tenant_manager** | Менеджер: поездки, водители, отчёты | Свой тенант (read + write бизнес) |
| **tenant_analyst** | Аналитик: отчёты, дашборды | Свой тенант (read-only аналитика) |
| **tenant_support** | Поддержка тенанта: инциденты, поездки | Свой тенант (read-only + limited write) |

#### 3.3 Business roles (бизнес-пользователи, с привязкой к тенанту)

| Роль | Что делает | Уровень доступа |
|---|---|---|
| **dispatcher** | Диспетчер: распределение поездок | Свой тенант (booking write) |
| **driver** | Водитель: поездки, заработок | Свои данные + свои поездки |
| **passenger** | Пассажир: заказ поездок | Свои данные + свои поездки |

#### 3.4 Cross-tenant role (ограниченная)

| Роль | Что делает | Уровень доступа |
|---|---|---|
| **super_admin** | Только для платформенных операций: миграция данных, reconciliation, incident response | Все тенанты (full, но только через admin-API, с аудитом и P1-алертом) |

**`super_admin` — не для повседневной работы.** Каждое действие —
отдельный аудит, P1-алерт в Slack. Используется только в инцидентах.

**Детальная матрица ролей × scopes — в `roles_matrix.md`.**

### 4. Authorization flow (defense in depth)

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Kong (coarse-grained)                             │
│   — Валидация JWT (signature, exp, iss, aud)               │
│   — Проверка, что user имеет доступ к клиенту (aud)        │
│   — Rate limiting per tenant_id                             │
│   — НЕ проверяет scopes (fine-grained)                     │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: BFF (fine-grained, scope check)                   │
│   — Проверка scope: есть ли нужный scope в JWT?            │
│   — Если нет → 403                                          │
│   — Если да → передаёт в сервис с контекстом (user_id,      │
│     tenant_id, scopes)                                      │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: Service (business rules)                          │
│   — Проверка бизнес-инвариантов (например, водитель       │
│     не может завершить чужую поездку)                       │
├─────────────────────────────────────────────────────────────┤
│ Layer 4: DB (RLS)                                          │
│   — SET app.tenant_id = <tenant_id из контекста>           │
│   — RLS фильтрует строки (ADR-001 Task4)                    │
└─────────────────────────────────────────────────────────────┘
```

**Kong (Layer 1):**
- JWT validation через JWKS (публичные ключи Keycloak, кэш).
- `aud` check — токен выдан для этого клиента.
- Rate limiting per `tenant_id` (квоты, ADR-001 Task4).
- **Не проверяет scopes** — это fine-grained, Kong не знает бизнес-правил.

**BFF (Layer 2):**
- Каждый endpoint декларирует требуемый scope (например,
  `@require_scope("booking:read")`).
- Middleware проверяет: `scope ∈ JWT.scope`? Если нет — 403.
- Передаёт в сервис контекст: `user_id`, `tenant_id`, `scopes`.

**Service (Layer 3):**
- Бизнес-инварианты: «водитель может завершить только свою поездку»,
  «dispatcher может назначить только водителя своего тенанта».

**DB (Layer 4):**
- RLS (ADR-001 Task4) — последний рубеж.

**Почему defense in depth:** если баг в BFF (забыли scope check) —
RLS отсечёт. Если баг в сервисе (забыли business rule) — audit log
поймает. Каждый слой независим.

### 5. SSO между приложениями

**Сценарий:** пользователь-менеджер тенанта логинится в corp-portal
→ получает SSO session → открывает admin-panel (того же тенанта,
того же региона) → Keycloak видит активную сессию → выдаёт токен
без повторного логина.

**Что нужно:**
- **Клиенты Keycloak** — по одному на приложение (`passenger-app`,
  `driver-app`, `corp-portal`, `admin-panel`, `web-frontend`).
- **Session cookie** — общая для всех клиентов одного realm
  (домен Keycloak).
- **`aud` в JWT** — токен выдан для конкретного клиента, но session
  общая.

**Cross-tenant SSO:** пользователь принадлежит **одному** тенанту
(`tenant_id` в JWT). Один пользователь — один тенант. Если сотрудник
работает в двух тенантах — два аккаунта (разные `sub`), один email,
разные `tenant_id`. Это **осознанное ограничение** (упрощает RBAC,
RLS, audit).

**Cross-region SSO:** нет. Пользователь логинится в Keycloak своего
`home_region`. Роуминг (пассажир в другом регионе) — через API к
`home_region` (ADR-003 Task3). SSO между realm'ами разных регионов
не настроен.

### 6. Уровни доступа к данным (own / tenant / platform)

| Уровень | Что доступно | Кто имеет |
|---|---|---|
| **own** | Свои данные (свой профиль, свои поездки, свой заработок) | passenger, driver (свои поездки) |
| **tenant** | Данные своего тенанта (все поездки тенанта, все водители, отчёты) | tenant_admin, tenant_manager, tenant_analyst, tenant_support, dispatcher |
| **platform** | Все тенанты (с аудитом) | platform_admin, platform_support, platform_auditor, platform_billing, super_admin |

**Реализация «own»:** RLS по `tenant_id` + дополнительное условие
`user_id = current_user` для passenger/driver (проверка в сервисе
или отдельная RLS policy).

**Реализация «tenant»:** RLS по `tenant_id` (ADR-001 Task4).

**Реализация «platform»:** RLS bypass (platform_admin DB role,
ADR-001 Task4, раздел 3.2) — только для platform-сервисов,
с аудитом и P1-алертом.

**Детально: какая роль к какому уровню — в `roles_matrix.md`.**

### 7. Keycloak расширение (что добавляем к Task1)

| Что | Как | Зачем |
|---|---|---|
| **Groups `tenant:<uuid>`** | Создаются при onboarding (ADR-003) | Группировка пользователей тенанта |
| **Roles (platform / tenant / business)** | Создаются в realm | RBAC |
| **Protocol mappers** | `tenant_id`, `region`, `roles`, `scope` в JWT | JWT claims для downstream |
| **Client scopes** | Per client (passenger-app → booking:read, booking:write...) | Fine-grained permissions |
| **Authentication flows** | Standard + OTP для platform_admin, super_admin | Усиленная защита привилегированных ролей |
| **Audit events** | Keycloak audit → Loki (Task3 ADR-006) | Аудит логинов, смены ролей |

**Что НЕ добавляем:**
- **Keycloak Authorization Services (UMA)** — overkill для базового
  RBAC. Роли в JWT + scope check в BFF достаточно.
- **Федерация с внешними IdP** — не требуется (решение пользователя).
- **Realm per tenant** — не масштабируется (см. раздел 1).
- **ABAC / ReBAC** — базовый RBAC покрывает потребности; ABAC — если
  появятся сложные политики (вне scope).

### 8. Sequence-диаграмма IAM flow

`Sequence_IAM_Flow.puml` — отдельный файл Task4. Показывает:
1. Пользователь → Keycloak (login, SSO session).
2. Keycloak → Пользователь (JWT с claims).
3. Пользователь → Kong (request + JWT).
4. Kong → Keycloak (JWKS validation).
5. Kong → BFF (после валидации).
6. BFF → BFF RBAC middleware (scope check).
7. BFF → Service (контекст: user_id, tenant_id, scopes).
8. Service → DB (SET app.tenant_id, RLS).
9. DB → Service (только свой tenant).
10. Service → BFF → Пользователь (response).

**Красным** — новое в Task4 (RBAC middleware, protocol mappers,
audit). **Синим** — существующее (Keycloak, Kong, BFF из Task1).

### 9. Аудит доступа

**Что логируем (расширение Task3 ADR-005):**
- Логины (успешные / неуспешные) — Keycloak audit.
- Смена ролей / групп — Keycloak audit.
- Проверка scope — BFF (успех / 403) — app log.
- Platform admin доступ к данным тенанта — отдельный audit log,
  P1-алерт.
- RLS violations — P1-алерт (ADR-001 Task4).

**Куда:** Loki (per-region, Task3 ADR-006), retention 5 лет для
финансовых, 1 год для остальных.

### 10. Что НЕ входит в это решение

- Детальная матрица ролей × scopes — **`roles_matrix.md`**.
- Onboarding (создание тенанта, Keycloak groups, первого админа) —
  **ADR-003**.
- Мониторинг per tenant — **ADR-004**.
- Федерация с внешними IdP — не требуется (решение пользователя).
- MFA для всех — только OTP для platform_admin / super_admin
  (базовый набор).
- ABAC / ReBAC — вне scope.
- Password policies — стандартные Keycloak, детали не фиксируем.

## Альтернативы

1. **ABAC (Attribute-Based Access Control).** Отклонено: базовый
   RBAC покрывает потребности; ABAC сложнее в администрировании.
   Если появятся сложные политики (например, «доступ в зависимости
   от времени суток») — ABAC отдельным решением.
2. **ReBAC (Relationship-Based, как Google Zanzibar).** Отклонено:
   overkill, сложная инфраструктура. RBAC + RLS достаточно.
3. **Keycloak realm-per-tenant.** Отклонено: не масштабируется
   (100+ тенантов × 4 региона), ops-кошмар.
4. **Keycloak Authorization Services (UMA).** Отклонено: overkill
   для базового RBAC, роли в JWT + scope check проще.
5. **Внешние IdP (Okta, Azure AD) как primary.** Отклонено: не было
   требований (решение пользователя). Можно добавить позже.
6. **Отдельный IAM-сервис (не Keycloak).** Отклонено: Keycloak уже
   есть (Task1), проверен, покрывает OIDC + SSO + RBAC.
7. **Проверка ролей, а не scopes.** Отклонено: роли → scopes mapping
   в Keycloak даёт гибкость (добавили scope роли — не меняем код).
8. **JWT без `tenant_id` (tenant в заголовке).** Отклонено: RLS,
   Kafka filtering, audit — всем нужен `tenant_id`. JWT — единый
   источник.
9. **Cross-tenant SSO (один пользователь — несколько тенантов).**
   Отклонено: усложняет RBAC, RLS, audit. Один пользователь —
   один тенант. Если нужно — два аккаунта.
10. **Cross-region SSO.** Отклонено: residency (Task3), Keycloak
    per-region. Роуминг — через API.

## Компромиссы

- **Shared realm** — все тенанты в одном realm, изоляция только
  через `tenant_id` + RBAC. Митигация: RLS (ADR-001 Task4), audit,
  тесты. Если enterprise потребует realm-per-tenant — отдельное
  решение (не сейчас).
- **Один пользователь — один тенант** — сотрудник двух тенантов
  имеет два аккаунта. Митигация: явно в документации; SSO внутри
  тенанта работает.
- **Нет федерации с внешними IdP** — корп-клиенты не могут
  использовать свой Okta/Azure AD. Митигация: если потребуется —
  отдельное решение (Keycloak federation). Сейчас — только наш
  Keycloak.
- **JWT 15 мин** — если роли изменились, пользователь может работать
  со старыми до истечения токена. Митигация: refresh token rotation,
  возможность revoke (Keycloak admin API).
- **Scope в JWT** — токен становится большим (если много scopes).
  Митигация: scope grouping (роли → набор scopes), JWT size ≤ 4 KB.
- **Kong не проверяет scopes** — только JWT validity. Митигация:
  BFF проверяет scopes (Layer 2); defense in depth.
- **`super_admin` — full access** — высокий риск. Митигация: OTP,
  audit, P1-алерт, только admin-API.
- **Нет ABAC** — сложные политики (например, «доступ только в рабочие
  часы») не поддерживаются. Митигация: если потребуется — ABAC
  отдельным решением.
- **RLS fail-closed** — если `app.tenant_id` не установлен, запрос
  падает. Митигация: библиотека-обёртка, тесты (ADR-001 Task4).
- **Keycloak audit → Loki** — дополнительная интеграция. Митигация:
  стандартный keycloak-event-listener, OTel.

## Ссылки на предыдущие решения

- Keycloak, `svc_identity`, BFF, Kong — **Task1 C2 Final**.
- `User.Registered` — **Task2 ADR-001**.
- Observability (Loki, OTel) — **Task2 ADR-006**.
- Keycloak per-region, replication — **ADR-001, ADR-002 Task3**.
- Роуминг через API — **ADR-003 Task3**.
- Аудит, комплаенс — **ADR-005 Task3**.
- Vault, mTLS — **ADR-006 Task3**.
- Модель изоляции, RLS, `tenant_id` в JWT — **ADR-001 Task4**.
- Матрица ролей — **`roles_matrix.md`**.
- Onboarding — **ADR-003 Task4**.
- Мониторинг — **ADR-004 Task4**.
