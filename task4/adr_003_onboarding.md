# ADR-003: Автоматизированный onboarding новых партнёров (тенантов)

## Контекст

Task4 требует «автоматизированный процесс подключения новых партнёров
(onboarding)». Партнёры (тенанты) — корпоративные клиенты, региональные
операторы, франшизы. Onboarding должен быть **автоматизированным**, но
с двумя потоками: SMB/self-service (без approval) и enterprise
(с approval, KYC, кастомные квоты).

Уже зафиксировано:
- **Task1 C2 Final:** `svc_tenant` (Tenant Management) — управление
  тенантами, квоты, изоляция. Своя `Tenant DB`.
- **Task2 ADR-001:** `Tenant.Onboarded` (producer — Tenant Management,
  consumers — Identity, Notification, Analytics), `User.Registered`
  (producer — Identity, consumer — Tenant Management).
- **Task2:** choreography-сценарий Tenant Onboarding.
- **Task3 ADR-002:** tenant configs — в Global Control Plane (non-PII),
  синхронизируются во все регионы.
- **Task3 ADR-001:** 4 региона, residency. Тенант — один домашний регион.
- **ADR-001 Task4:** модель изоляции (shared DB + RLS, `tenant_id`).
- **ADR-002 Task4:** RBAC, Keycloak shared realm, группы `tenant:<uuid>`,
  роли, `tenant_admin` назначается при onboarding.
- **ADR-002 Task4:** SSO, JWT с `tenant_id` и `region`.

**Задача:** спроектировать onboarding-процесс: шаги, участники,
автоматизация, события, обработка ошибок, идемпотентность, аудит.

## Требования

1. Два потока: self-service (SMB) и enterprise (approval).
2. Полная автоматизация после триггера (self-service — сразу;
   enterprise — после approval).
3. Создание тенанта в `svc_tenant` с `home_region`.
4. Создание Keycloak group `tenant:<uuid>` + роли (ADR-002).
5. Создание первого `tenant_admin` (Keycloak user + роль).
6. Применение RLS-политик (shared DB — новые строки с `tenant_id`).
7. Настройка квот (per tenant, ADR-001 Task4).
8. Настройка billing (тариф).
9. Публикация `Tenant.Onboarded` (Task2).
10. Уведомление партнёра (email / webhook).
11. Tenant configs — в Global Control Plane (Task3 ADR-002).
12. Идемпотентность (повторный onboarding не создаёт дублей).
13. Аудит всех шагов (Task3 ADR-005).
14. Не дублировать Task1–3; переиспользовать `svc_tenant`,
    `svc_identity`, Keycloak, `Tenant.Onboarded`.

## Решение

### 1. Два потока onboarding

| Аспект | Self-service (SMB) | Enterprise |
|---|---|---|
| **Триггер** | Самостоятельная заявка (веб-форма) | Контакт с sales → договор |
| **Approval** | Нет (авто) | Да (внутренний approval) |
| **KYC партнёра** | Валидация email / домена | KYC (юрлицо, документы, банк) |
| **Договор** | Terms of Service (click-through) | Подписанный договор |
| **Квоты** | Лимиты по умолчанию (тариф SMB) | Кастомные (по договору) |
| **Роли** | `tenant_admin` (self-service) | `tenant_admin` (назначается sales) |
| **SLA** | Стандартный (best-effort) | По договору |
| **Время onboarding** | ≤ 5 мин (авто) | ≤ 5 рабочих дней (KYC + договор) |
| **Billing** | Авто (карта, подписка) | Invoice / договор |

**Общее ядро:** после триггера (self-service — сразу; enterprise —
после approval) выполняется **один и тот же автоматизированный
workflow** (шаги 3–10 ниже). Разница — в шагах 1–2.

### 2. Шаги onboarding (workflow)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Заявка (self-service форма / enterprise контакт)         │
├─────────────────────────────────────────────────────────────┤
│ 2. [Enterprise only] KYC + approval + договор               │
├─────────────────────────────────────────────────────────────┤
│ 3. Валидация (email, домен, дубликат тенанта)               │
├─────────────────────────────────────────────────────────────┤
│ 4. Создание тенанта в svc_tenant (home_region, тариф)       │
├─────────────────────────────────────────────────────────────┤
│ 5. Создание Keycloak group tenant:<uuid> + роли             │
├─────────────────────────────────────────────────────────────┤
│ 6. Создание первого tenant_admin (Keycloak user + роль)     │
├─────────────────────────────────────────────────────────────┤
│ 7. Применение конфигурации (квоты, billing, RLS)            │
├─────────────────────────────────────────────────────────────┤
│ 8. Публикация Tenant.Onboarded (Task2)                      │
├─────────────────────────────────────────────────────────────┤
│ 9. Уведомление партнёра (email + welcome)                   │
├─────────────────────────────────────────────────────────────┤
│ 10. Tenant configs → Global Control Plane (Task3)           │
├─────────────────────────────────────────────────────────────┤
│ 11. Аудит всех шагов (Task3 ADR-005)                        │
└─────────────────────────────────────────────────────────────┘
```

**Шаги 3–10 — автоматизированы** (Orchestrator). Шаг 2 — enterprise
(ручной / semi-авто: KYC-провайдер + sales approval).

### 3. Onboarding Orchestrator

Сервис/компонент, который управляет workflow.
**Не новый микросервис** — модуль внутри `svc_tenant` (или отдельный
lightweight-сервис, если `svc_tenant` разрастётся). На диаграмме —
`Onboarding Workflow` (компонент внутри `svc_tenant`).

**Функции:**
- State machine: `initiated → validated → created → provisioned →
  completed` (или `failed` на любом шаге).
- Идемпотентность: повторный вызов с тем же `request_id` не создаёт
  дублей.
- Retry: transient-ошибки (Keycloak недоступен, БД timeout) —
  retry с backoff.
- Компенсация: если шаг 6 упал (не удалось создать admin) — откат
  шага 5 (удаление group) и шага 4 (тенант в статусе `pending`,
  не активен).
- Audit: каждый шаг → Loki (Task3 ADR-005).

**State в БД `svc_tenant`:**

```sql
CREATE TABLE onboarding_requests (
    request_id UUID PRIMARY KEY,
    tenant_id UUID,                    -- NULL до шага 4
    flow_type VARCHAR(20) NOT NULL,    -- 'self_service' | 'enterprise'
    status VARCHAR(30) NOT NULL,       -- initiated | validated | created | provisioned | completed | failed
    home_region VARCHAR(20) NOT NULL,  -- region-ru | region-sea | region-latam | region-mena
    request_payload JSONB NOT NULL,    -- исходная заявка
    error_details JSONB,               -- при failed
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMPTZ
);
```

### 4. Self-service flow (SMB)

**Триггер:** партнёр заполняет форму на сайте (или в API).

**Шаги:**

1. **Заявка** — партнёр заполняет: название, email, страна, тариф
   (SMB), желаемый `home_region` (или определяется по стране).
2. **Валидация:**
    - Email валиден? (домен не одноразовый)
    - Дубликат? (тенант с таким email/доменом уже есть → ошибка)
    - Страна поддерживается? (есть регион для этой юрисдикции)
    - `home_region` определён (по стране или явно).
3. **Создание тенанта** в `svc_tenant`:
    - `tenant_id` (UUID), `name`, `home_region`, `tariff = smb`,
      `status = pending`.
4. **Keycloak group** `tenant:<uuid>` — создаётся.
5. **Keycloak roles** — назначаются на group (tenant_admin,
   tenant_manager, ...).
6. **Первый tenant_admin:**
    - Keycloak user (email партнёра) — создаётся.
    - Пароль — **временный**, отправляется на email с ссылкой
      «установить пароль» (или magic link).
    - Роль `tenant_admin` — назначается.
7. **Конфигурация:**
    - Квоты по умолчанию (SMB): API rate limit, trips/month, storage.
    - Billing: подписка SMB (карта).
    - RLS: применяется автоматически (shared DB — все новые строки
      с `tenant_id`).
8. **`Tenant.Onboarded`** — публикуется (Task2). Consumers: Identity
   (создать пользователя в `svc_identity`), Notification (welcome),
   Analytics (tenant created).
9. **Уведомление:** welcome email с инструкциями, ссылка на
   corp-portal / admin-panel.
10. **Tenant configs → Global Control Plane** (Task3 ADR-002) —
    квоты, тариф, `home_region` синхронизируются.
11. **Тенант `status = active`.**
12. **Мониторинг:** с первого дня (ADR-004 Task4).

**Время:** ≤ 5 мин.

**Ошибки:**
- Email дубликат → 409.
- Страна не поддерживается → 400 (список поддерживаемых).
- Keycloak недоступен → retry, статус `failed`, партнёр получает
  письмо «повторим позже».
- Тенант `pending` дольше 1 ч → алерт, ручной разбор.

### 5. Enterprise flow

**Триггер:** sales фиксирует контакт, создаёт `onboarding_request`
с `flow_type = enterprise`.

**Шаги:**

1. **Заявка** — sales заполняет: юрлицо, ИНН/налоговый номер, страна,
   контакт, желаемый `home_region`, тариф (enterprise), кастомные
   квоты.
2. **KYC партнёра** (ручной / semi-авто):
    - Проверка юрлица (реестры, санкционные списки).
    - Документы (устав, лицензии — если применимо).
    - Банковские реквизиты (для invoice).
    - KYC-провайдер (внешний, per-region: например, для RU — локальный провайдер, для SEA — Refinitiv / Dow Jones).
3. **Approval** (внутренний):
    - Sales manager + Legal + Finance.
    - Если approval — `onboarding_request.status = approved`.
    - Если rejected — письмо партнёру, `status = rejected`.
4. **Создание тенанта** — как self-service, но:
    - `tariff = enterprise`.
    - Кастомные квоты (из договора).
    - Billing: invoice (не карта).
5. **Keycloak group + роли** — как self-service.
6. **Первый tenant_admin** — назначается sales (или партнёр сам
   устанавливает пароль через email).
7. **Конфигурация** — кастомные квоты, billing invoice.
8. **`Tenant.Onboarded`** — публикуется.
9. **Уведомление:** welcome + контакт account manager.
10. **Tenant configs → Global Control Plane.**
11. **Тенант `status = active`.**
12. **Мониторинг** — с первого дня + SLA-дашборд.

**Время:** ≤ 5 рабочих дней (KYC + approval).

**Ошибки:**
- KYC failed → `status = rejected`, письмо.
- Approval rejected → `status = rejected`.
- Договор не подписан → `status = pending_contract`.

### 6. События (расширение Task2)

**Что уже есть (Task2 ADR-001):**
- `Tenant.Onboarded` — публикуется после завершения onboarding.
- `User.Registered` — при создании первого `tenant_admin`.

**Что добавляем (новые события? нет — используем существующие +
state machine):**
- **Не добавляем новые события** — `Tenant.Onboarded` достаточно.
  Внутренние шаги orchestrator'а — не события Kafka (это
  operational state, не бизнес-событие).
- **`Tenant.Onboarded`** — расширяем payload: `tenant_id`, `name`,
  `home_region`, `tariff`, `flow_type`, `created_by`, `onboarded_at`.

**Почему не добавляем `Tenant.Created` / `Tenant.Provisioned`:**
- Это внутренние шаги orchestrator'а, не бизнес-события для других
  сервисов.
- Consumers (Identity, Notification, Analytics) реагируют на
  `Tenant.Onboarded` — факт «тенант готов к работе».
- Если понадобится — добавим позже (backward-compatible).

### 7. Идемпотентность

**Проблема:** повторный вызов onboarding (retry, дубль формы) не
должен создавать дубль тенанта.

**Решение:**
- `request_id` (UUID) — генерируется при создании заявки.
- `svc_tenant` проверяет: `request_id` уже есть? → возвращает
  существующий результат (idempotent).
- Дополнительно: уникальный индекс по `(email, home_region)` для
  активных тенантов — дубль email в одном регионе запрещён.
- Keycloak group `tenant:<uuid>` — создаётся с проверкой
  существования (idempotent).
- `Tenant.Onboarded` — публикуется один раз (outbox + `event_id`
  дедупликация, Task2 ADR-005).

### 8. Компенсация при ошибках (Saga)

**Onboarding — это Saga** (orchestration, ADR-004 Task2 — паттерн
«Booking Lifecycle»). Orchestrator — в `svc_tenant`.

**Компенсации:**

| Шаг | Действие | Компенсация при ошибке |
|---|---|---|
| 4. Создание тенанта | INSERT в `svc_tenant` | `status = failed`, не удаляем (для аудита) |
| 5. Keycloak group | Create group | Delete group |
| 6. Keycloak user | Create user | Delete user (или disable) |
| 7. Конфигурация | Set quotas, billing | Rollback (или `status = failed`) |
| 8. `Tenant.Onboarded` | Publish | Не публикуем (тенант не готов) |

**Правило:** если шаг 4–7 упал — тенант остаётся в `status = failed`,
не активен. Партнёр получает письмо «проблема, мы разбираемся».
Ручной разбор → либо retry, либо удаление.

**Идемпотентная компенсация:** повторный retry не создаёт дублей
(`request_id`).

### 9. Tenant configs → Global Control Plane

**После `Tenant.Onboarded`:**
- `svc_tenant` публикует tenant config (non-PII): `tenant_id`, `name`,
  `home_region`, `tariff`, `quotas`, `billing_info` (без PII
  платёжных данных — только тип billing) → в Global Control Plane
  (Task3 ADR-002).
- Все регионы получают read-копию (pull-on-start + webhook).
- Другие регионы знают о тенанте (для роуминга — API к `home_region`,
  ADR-003 Task3).

### 10. Уведомления

**Self-service:**
- Email партнёру: welcome + ссылка «установить пароль» + ссылка
  на docs.
- Email `tenant_admin`: инструкции по настройке.

**Enterprise:**
- Email партнёру: welcome + контакт account manager.
- Email sales: «тенант создан, можно начинать работу».
- Email finance: «invoice настроен».

**Канал:** `svc_notification` (Task1) — email / push / webhook.

### 11. Аудит

**Что логируем (Task3 ADR-005):**
- Каждый шаг orchestrator'а: `request_id`, `tenant_id`, `step`,
  `status`, `duration`, `error` (если есть).
- Кто инициировал (self-service — email; enterprise — sales user).
- KYC-результат (enterprise).
- Approval (enterprise) — кто, когда, решение.
- Keycloak операции (group / user / role) — Keycloak audit.

**Куда:** Loki (per-region, Task3 ADR-006), retention 1 год
(для onboarding) / 5 лет (для KYC / договоров).

### 12. Диаграммы

**`C3_Onboarding.puml`** — компоненты вокруг onboarding:
- `svc_tenant` (существующий) + `Onboarding Workflow` (новый
  компонент внутри).
- `svc_identity` (существующий).
- Keycloak (существующий).
- `svc_notification` (существующий).
- KYC Provider (внешний, enterprise) — новый.
- Billing Service (существующий или новый — уточнить).
- Global Control Plane (Task3) — существующий.
- Kafka (Task2) — существующий.
- Admin panel (BFF Admin, Task1) — существующий.

**`Sequence_Onboarding.puml`** — поток:
- Self-service: form → validation → tenant → Keycloak → admin →
  Tenant.Onboarded → welcome.
- Enterprise: sales → KYC → approval → (то же ядро).
- Показать оба потока через `alt`.

**Красным** — новое (Onboarding Workflow, KYC Provider, state machine,
компенсации). **Синим** — существующее (svc_tenant, Keycloak,
svc_identity, Tenant.Onboarded, Global Control Plane).

### 13. Что НЕ входит в это решение

- RBAC-модель — **ADR-002 Task4**.
- Мониторинг per tenant — **ADR-004 Task4**.
- Модель изоляции — **ADR-001 Task4**.
- Детальный дизайн KYC-провайдера (выбор, интеграция) — инфраструктурное
  решение.
- Billing Service детально (тарифы, invoice) — вне scope Task4.
- Юридические тексты договоров / ToS — вне scope.
- Миграция существующих тенантов (из Django-монолита) — Task1 ADR-005
  (Strangler Fig), не дублируем.
- Федерация Keycloak с внешними IdP — не требуется (ADR-002 Task4).

## Альтернативы

1. **Один поток для всех (без enterprise approval).** Отклонено:
   enterprise требует KYC + договор + кастомные квоты. Без approval —
   риск fraud / неоплаты.
2. **Полностью ручной onboarding для всех.** Отклонено: не масштабируется
   (100+ тенантов), TTM ≤ 2 нед (Task1 ADR-001).
3. **Полностью авто для enterprise (без KYC).** Отклонено: регуляторные
   требования (KYC юрлица), риск fraud.
4. **Отдельный микросервис `onboarding-service`.** Отклонено: overkill,
   дублирует `svc_tenant`. Orchestrator — модуль внутри `svc_tenant`.
5. **Новые события `Tenant.Created` / `Tenant.Provisioned`.** Отклонено:
   внутренние шаги orchestrator'а, не бизнес-события. `Tenant.Onboarded`
   достаточно.
6. **Realm-per-tenant в Keycloak (при onboarding — новый realm).**
   Отклонено: ADR-002 Task4 (не масштабируется). Group `tenant:<uuid>`
   в shared realm.
7. **Federation с внешними IdP при onboarding (enterprise приносит
   свой Okta).** Отклонено: не требуется (решение пользователя).
   Если потребуется — отдельное решение.
8. **Автоматический KYC без ручного approval (enterprise).** Отклонено:
   KYC-провайдер даёт сигнал, но финальное решение — человек (sales /
   legal / finance).
9. **Без компенсации (Saga).** Отклонено: если шаг 6 упал — тенант
   в неконсистентном состоянии (group есть, admin нет). Компенсация
   обязательна.
10. **Без идемпотентности.** Отклонено: retry / дубль формы → дубль
    тенанта. `request_id` + уникальный индекс.

## Компромиссы

- **Два потока** — сложнее, чем один. Митигация: общее ядро (шаги 3–10),
  разница только в шагах 1–2.
- **Onboarding Orchestrator внутри `svc_tenant`** — если `svc_tenant`
  разрастётся, выделим в отдельный сервис. Пока — модуль.
- **KYC-провайдер внешний** — дополнительная интеграция, стоимость.
  Митигация: только для enterprise (SMB — валидация email), per-region
  провайдер (локальные требования).
- **Approval ручной (enterprise)** — задержка до 5 дней. Митигация:
  параллельные approval (sales + legal + finance), SLA на approval.
- **Компенсация не удаляет тенант** (только `status = failed`) —
  данные остаются для аудита. Митигация: периодическая очистка failed
  тенантов (ручная, после разбора).
- **`Tenant.Onboarded` payload расширяется** — backward-compatible
  (добавляем поля, не удаляем). Task2 ADR-001 — schema evolution
  через Apicurio.
- **Нет новых событий** — только `Tenant.Onboarded`. Если понадобится
  granular (для интеграций) — добавим позже (backward-compatible).
- **Billing — вне scope** — предполагается, что `svc_billing`
  существует (Task1? или добавляем). Уточним на этапе реализации.
- **KYC для RU / SEA / LatAm / MENA — разные провайдеры** —
  per-region регуляторика. Митигация: единый интерфейс KYC Provider,
  разные реализации.
- **Идемпотентность через `request_id`** — если партнёр дважды
  заполнил форму (разные `request_id`) → два тенанта. Митигация:
  уникальный индекс по `(email, home_region)` для активных тенантов.

## Ссылки на предыдущие решения

- `svc_tenant`, `svc_identity`, `svc_notification`, Keycloak,
  BFF Admin — **Task1 C2 Final**.
- `Tenant.Onboarded`, `User.Registered`, outbox / DLQ — **Task2
  ADR-001, ADR-005**.
- Saga (orchestration) — **Task2 ADR-004**.
- Tenant configs в Global Control Plane — **ADR-002 Task3**.
- 4 региона, residency — **ADR-001 Task3**.
- Аудит, breach — **ADR-005 Task3**.
- Модель изоляции (RLS, `tenant_id`) — **ADR-001 Task4**.
- RBAC, Keycloak group, роли — **ADR-002 Task4**.
- Матрица ролей — **`roles_matrix.md`**.
- Мониторинг — **ADR-004 Task4**.


