# ADR-004: Мониторинг мультитенантного окружения

## Контекст

Task4 требует «систему мониторинга мультитенантного окружения». ADR-001
зафиксировал модель изоляции (RLS, `tenant_id`), ADR-002 — RBAC и JWT,
ADR-003 — onboarding. Теперь нужно спроектировать мониторинг, который
видит не только «платформу в целом», но и **каждого тенанта отдельно**:
SLO per tenant, usage metering (для биллинга), tenant health dashboard,
per-tenant алерты.

Уже зафиксировано:
- **Task2 ADR-006:** observability — 3 столпа (metrics / logs / traces),
  7 категорий метрик, SLO (pricing p99 < 100 мс, matching < 500 мс,
  fraud < 200 мс), Alertmanager (P1 → PagerDuty, P2/P3 → Slack),
  Grafana dashboards.
- **Task2 ADR-006:** OTel Collector, Prometheus, Loki, Jaeger, Grafana,
  Alertmanager, Custom Middleware (business metrics).
- **Task3 ADR-006:** observability переиспользуется, audit logs → Loki.
- **ADR-001 Task4:** квоты per tenant (API rate, trips, storage).
- **ADR-002 Task4:** JWT с `tenant_id`, RBAC.
- **ADR-003 Task4:** onboarding, `Tenant.Onboarded`.

**Задача:** расширить observability (Task2 ADR-006) на мультитенантность:
tenant-aware метрики, per-tenant SLO, usage metering, дашборды, алерты.
Не дублировать Task2 — переиспользовать, добавить tenant dimensions.

## Требования

1. `tenant_id` во всех метриках / логах / traces (tenant-aware observability).
2. SLO per tenant (доступность, latency, error rate).
3. Usage metering (trips, API calls, storage, Kafka throughput).
4. Tenant health dashboard (для Tenant Admin — свой тенант).
5. Platform dashboard (для Platform Admin — все тенанты).
6. Per-tenant алерты (SLO burn rate, quota).
7. Алерты на cross-tenant нарушения (RLS violation, unauthorized access) — P1.
8. Noisy neighbor detection (один тенант грузит платформу).
9. Не дублировать Task2 ADR-006; переиспользовать OTel / Prometheus /
   Loki / Grafana / Alertmanager.
10. Usage metering → billing (ADR-003 Task4, billing service).

## Решение

### 1. Что уже есть (Task2 ADR-006) — переиспользуем

| Компонент | Что делает | Изменения |
|---|---|---|
| **OTel Collector** | Сбор traces / metrics / logs | + `tenant_id` в resource attributes |
| **Prometheus** | TSDB метрик | + `tenant_id` label в метриках |
| **Loki** | Логи | + `tenant_id` в structured logs |
| **Jaeger** | Traces | + `tenant_id` в span attributes |
| **Grafana** | Дашборды | + per-tenant фильтры, новые дашборды |
| **Alertmanager** | Алерты | + per-tenant routes |
| **Custom Middleware** | Бизнес-метрики | + `tenant_id` label |

**Не переделываем** — расширяем.

### 2. Что добавляем (tenant-aware observability)

| Компонент | Новое? | Что делает |
|---|---|---|
| **Tenant-aware middleware** (в BFF) | 🔴 Новое | Извлекает `tenant_id` из JWT → добавляет в метрики / логи / traces |
| **Per-tenant SLO** | 🔴 Новое | SLO для каждого тенанта (по тарифу) |
| **Usage Metering** | 🔴 Новое | Счётчики usage → ClickHouse → billing |
| **Tenant Health Dashboard** | 🔴 Новое | Grafana dashboard для Tenant Admin |
| **Platform Tenants Dashboard** | 🔴 Новое | Grafana dashboard для Platform Admin |
| **Per-tenant Alerts** | 🔴 Новое | Alertmanager routes по `tenant_id` |
| **Noisy Neighbor Detection** | 🔴 Новое | Алерт, если тенант > X% ресурсов |
| **Cross-tenant Alerts** | 🔴 Новое | RLS violation, unauthorized cross-tenant access → P1 |

### 3. Tenant-aware middleware (в BFF)

**Что:** библиотека в каждом BFF (аналогично RBAC middleware, ADR-002).

**Функции:**
1. Извлекает `tenant_id` из JWT (claim).
2. Добавляет `tenant_id` в:
   - Prometheus labels (`http_requests_total{tenant_id="..."}`)
   - Structured logs (`{"tenant_id": "...", ...}`)
   - OTel span attributes (`span.set_attribute("tenant.id", ...)`)
3. Счётчики usage:
   - `api_calls_total{tenant_id, endpoint, status}`
   - `http_request_duration_seconds{tenant_id, endpoint}`
4. Пробрасывает `tenant_id` в downstream (сервисы, Kafka).

**Почему в BFF, а не в сервисах:** BFF — точка входа для клиентских
запросов. Сервисы получают `tenant_id` из контекста (уже с меткой).

**Для platform-ролей:** `tenant_id = "platform"` (специальное значение).
Метрики platform-запросов отделены от tenant-запросов.

### 4. Per-tenant SLO (по тарифу)

**Тарифы** (assumption, уточняется с бизнесом):

| Тариф | Доступность | Latency p95 | Error rate | Кто |
|---|---|---|---|---|
| **SMB** | 99,5% | ≤ 200 мс | ≤ 1% | Self-service тенанты |
| **Pro** | 99,9% | ≤ 150 мс | ≤ 0,5% | Средние тенанты |
| **Enterprise** | 99,95% | ≤ 100 мс | ≤ 0,1% | Крупные тенанты (по договору) |

**Реализация:**
- SLO определяются в Tenant config (Global Control Plane, Task3 ADR-002).
- Prometheus rules вычисляют SLO per tenant (error budget, burn rate).
- Grafana показывает SLO per tenant (dashboard).
- Alertmanager алертит при burn rate > 2% (multi-window: 1h / 6h / 3d).

**Пример Prometheus rule:**

```yaml
groups:
  - name: tenant_slo
    rules:
      - record: tenant:slo_availability:ratio
        expr: |
          sum by (tenant_id) (rate(http_requests_total{status!~"5.."}[5m]))
          /
          sum by (tenant_id) (rate(http_requests_total[5m]))
      - alert: TenantSLOBreach
        expr: tenant:slo_availability:ratio < 0.995
        for: 5m
        labels:
          severity: P2
          tenant_id: "{{ $labels.tenant_id }}"
        annotations:
          summary: "SLO breach for tenant {{ $labels.tenant_id }}"
```

**Multi-window burn rate** (по Google SRE Workbook):
- 1h window: burn rate > 14.4 → P1 (быстрый burn)
- 6h window: burn rate > 6 → P2
- 3d window: burn rate > 1 → P3 (медленный burn)

### 5. Usage metering (для биллинга)

**Что считаем:**

| Метрика | Как | Куда |
|---|---|---|
| **Trips per month** | `trips_total{tenant_id}` (из Booking) | Prometheus → ClickHouse → billing |
| **API calls** | `api_calls_total{tenant_id, endpoint}` (из BFF middleware) | Prometheus → ClickHouse → billing |
| **Storage** | Периодический job (S3 inventory, DB size) | ClickHouse → billing |
| **Kafka throughput** | `kafka_producer_bytes_total{tenant_id}` | Prometheus → ClickHouse |
| **Compute (если нужно)** | K8s resource usage per tenant (если shared) | Prometheus → ClickHouse |

**Pipeline:**

```
BFF middleware → Prometheus (метрики)
                     ↓
              flink_analytics (Task2) — агрегация per tenant
                     ↓
              ClickHouse (usage_metering table)
                     ↓
              Billing Service (ADR-003 Task4) — invoice
```

**Почему ClickHouse, а не Prometheus:** Prometheus — для real-time
метрик (retention 15–30 дней). Usage metering — для billing (нужны
месячные агрегаты, retention 5 лет). ClickHouse — оптимально
(Task2 ADR-003).

**Таблица в ClickHouse:**

```sql
CREATE TABLE usage_metering (
    tenant_id UUID,
    metric_name String,           -- 'trips', 'api_calls', 'storage_bytes'
    window_start DateTime,
    window_end DateTime,
    value UInt64,
    unit String                   -- 'count', 'bytes'
) ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(window_start)
ORDER BY (tenant_id, metric_name, window_start);
```

**Идемпотентность:** `ReplacingMergeTree` по `(tenant_id, metric_name,
window_start)` — повторная агрегация не дублирует.

### 6. Tenant Health Dashboard (Grafana)

**Для Tenant Admin** (tenant_admin, tenant_manager):

| Панель | Что показывает |
|---|---|
| **SLO** | Доступность, latency p95, error rate за 24h / 7d / 30d |
| **Usage** | Trips, API calls, storage — текущий месяц vs квота |
| **Quotas** | Использование квот (80% → warning, 100% → critical) |
| **Errors** | Топ ошибок (4xx, 5xx) за последние 24h |
| **Latency** | p50 / p95 / p99 по операциям |
| **Incidents** | История инцидентов тенанта |
| **Drivers / Passengers** | Активные водители, пассажиры (агрегаты) |

**Доступ:** только пользователи этого тенанта (RBAC, ADR-002). URL:
`grafana.gofuture.io/d/tenant-health?var-tenant_id=<uuid>` —
фильтр по `tenant_id` из JWT.

**Для Platform Admin** (platform_admin, platform_support, platform_auditor):

| Панель | Что показывает |
|---|---|
| **All Tenants** | Список тенантов, SLO, usage, статус |
| **Top by Load** | Топ тенантов по API calls, trips, storage |
| **SLO Breaches** | Тенанты с пробитым SLO |
| **Noisy Neighbors** | Тенанты > X% ресурсов |
| **Cross-tenant Alerts** | RLS violations, unauthorized access |
| **Onboarding Funnel** | Self-service / enterprise, success rate, time to activate |

**Доступ:** platform-роли (RBAC, ADR-002). URL:
`grafana.gofuture.io/d/platform-tenants`.

### 7. Метрики (расширение Task2 ADR-006)

**Было (Task2 ADR-006):** 7 категорий метрик (Kafka, Producer,
Consumer, Flink, Saga, Business, Data quality).

**Добавляем `tenant_id` label + новые метрики:**

| Категория | Метрика | Labels | Назначение |
|---|---|---|---|
| **HTTP** | `http_requests_total` | `tenant_id, endpoint, status` | SLO, usage |
| **HTTP** | `http_request_duration_seconds` | `tenant_id, endpoint` | SLO latency |
| **Business** | `trips_total` | `tenant_id, status` | Usage, SLO |
| **Business** | `payments_total` | `tenant_id, status` | Usage, SLO |
| **API** | `api_calls_total` | `tenant_id, endpoint` | Usage metering |
| **Storage** | `storage_bytes` | `tenant_id, type` | Usage metering, quotas |
| **Quota** | `quota_usage` | `tenant_id, quota_type` | Quota alerts |
| **Quota** | `quota_limit` | `tenant_id, quota_type` | Quota alerts |
| **SLO** | `slo_burn_rate` | `tenant_id, slo_name, window` | Per-tenant SLO |
| **Kafka** | `kafka_consumer_lag` | `tenant_id, topic, consumer_group` | Per-tenant lag (где возможно) |
| **Security** | `rls_violation_total` | `tenant_id` | P1 alert |
| **Security** | `cross_tenant_access_total` | `source_tenant, target_tenant` | P1 alert |
| **Noisy neighbor** | `tenant_resource_share` | `tenant_id, resource` | Noisy neighbor detection |

**Ограничение по Kafka:** топики — domain-level (Task2 ADR-002), не
per-tenant. `tenant_id` в partition key только для `platform.events`.
Поэтому `kafka_consumer_lag` per tenant — только на consumer side
(сколько событий tenant'а в lag). Реализация — custom middleware
в consumer.

### 8. Алерты (расширение Task2 ADR-006)

**Существующие (Task2 ADR-006):** P1 → PagerDuty, P2/P3 → Slack.

**Добавляем per-tenant routes:**

| Алерт | Severity | Кому | Условие |
|---|---|---|---|
| **TenantSLOBreach** | P2 | Tenant Admin (email) + Platform Support (Slack) | SLO burn rate > 2% |
| **TenantSLOBreachCritical** | P1 | Platform Support (PagerDuty) | SLO burn rate > 14.4 (1h) |
| **QuotaWarning** | P3 | Tenant Admin (email) | 80% квоты |
| **QuotaExceeded** | P2 | Tenant Admin (email) + Platform Support (Slack) | 100% квоты |
| **NoisyNeighbor** | P2 | Platform Support (Slack) | Тенант > 30% ресурсов |
| **RLSViolation** | P1 | Security Team (PagerDuty) | Любое нарушение |
| **CrossTenantAccess** | P1 | Security Team (PagerDuty) | Любая попытка |
| **OnboardingFailed** | P3 | Platform Support (Slack) | Onboarding failed > 1h |

**Alertmanager routes:**

```yaml
route:
  routes:
    - match:
        alertname: RLSViolation
      receiver: security-pagerduty
      group_by: [source_tenant, target_tenant]
    - match:
        alertname: TenantSLOBreach
      receiver: tenant-admin-email
      group_by: [tenant_id]
    - match:
        severity: P1
      receiver: platform-pagerduty
    - match:
        severity: P2
      receiver: platform-slack
```

**Per-tenant email routing:** Tenant Admin получает алерты только
по своему тенанту (email из Tenant config, Global Control Plane).

### 9. Noisy Neighbor Detection

**Проблема:** один тенант грузит платформу (много API calls, trips,
storage) → деградация для остальных.

**Как детектим:**
- `tenant_resource_share{tenant_id, resource}` — доля тенанта в общем
  потреблении (API calls, DB connections, Kafka throughput, storage).
- Алерт `NoisyNeighbor`, если `tenant_resource_share > 30%` на
  протяжении 15 мин.

**Действия:**
1. P2 алерт в Slack (Platform Support).
2. Проверка: легитимная нагрузка (enterprise) или abuse?
3. Если abuse — throttle (Kong rate limit, ADR-001 Task4, квоты).
4. Если легитимная — upgrade тарифа (кастомные квоты).

### 10. Cross-tenant Security Alerts

**RLS violation** (ADR-001 Task4, раздел 9):
- Любая попытка обойти RLS → `rls_violation_total{tenant_id}` → P1.
- Alertmanager → Security Team (PagerDuty).
- Runbook: проверить логи, изолировать, post-mortem.

**Cross-tenant access** (platform admin доступ к данным тенанта):
- Каждый доступ → `cross_tenant_access_total{source, target}` → P1.
- Alertmanager → Security Team + Platform Admin (Slack).
- Audit log (Task3 ADR-005) — детали.

**Unauthorized cross-tenant** (попытка пользователя тенанта A
получить данные тенанта B):
- → P1, немедленная эскалация.
- Признак: RLS отказ + `tenant_id` в JWT ≠ `tenant_id` в запросе.

### 11. Дашборды (Grafana)

| Dashboard | Для кого | Что | URL |
|---|---|---|---|
| **Tenant Health** | Tenant Admin, Manager | SLO, usage, quotas, errors, incidents | `/d/tenant-health?var-tenant_id=<uuid>` |
| **Platform Tenants** | Platform Admin, Support, Auditor | Все тенанты, SLO, top by load, noisy neighbors | `/d/platform-tenants` |
| **Onboarding** | Platform Admin, Sales | Funnel self-service / enterprise, time to activate, failures | `/d/onboarding` |
| **Billing** | Platform Billing, Finance | Usage per tenant, invoice status | `/d/billing` |
| **Security** | Security Team | RLS violations, cross-tenant access, auth failures | `/d/security` |
| **SLO Overview** | Platform Admin | Global SLO + per-tenant SLO | `/d/slo-overview` |

**Доступ:** RBAC (ADR-002). Tenant Admin видит только свой `tenant_id`
(фильтр в Grafana из JWT). Platform — все.

### 12. Диаграмма

`C2_MultiTenant.puml` — основная C2 для Task4. Показывает:
- **IAM:** Keycloak + svc_identity (синее), tenant-aware middleware
  в BFF (красное).
- **Onboarding:** svc_tenant + Onboarding Workflow (красное), KYC
  Provider (внешнее).
- **Monitoring:** OTel / Prometheus / Loki / Grafana / Alertmanager
  (синее, Task2 ADR-006) + Usage Metering (красное), per-tenant
  dashboards (красное), per-tenant alerts (красное).
- **Изоляция:** RLS на БД (аннотация), `tenant_id` в JWT.
- **Все остальные компоненты** (Kong, BFF, сервисы, Kafka, ClickHouse,
  Global Control Plane, externals) — синие (Task1–3).

Красным — новое в Task4:
- Tenant-aware middleware
- Onboarding Workflow
- Usage Metering
- Per-tenant dashboards
- Per-tenant alerts
- KYC Provider (enterprise)
- RLS policies (аннотация)

### 13. Что НЕ входит в это решение

- Базовый observability (3 столпа, метрики, логи, traces) —
  **Task2 ADR-006**.
- Модель изоляции (RLS, `tenant_id`) — **ADR-001 Task4**.
- RBAC, JWT — **ADR-002 Task4**.
- Onboarding — **ADR-003 Task4**.
- Billing Service детально (тарифы, invoice) — вне scope Task4.
- Детальный дизайн Usage Metering (поля, retention) — уточняется
  на этапе реализации.
- Логирование PII — **Task3 ADR-005** (audit).
- SLO для platform-сервисов (не tenant) — Task2 ADR-006.

## Альтернативы

1. **Отдельная observability-инфраструктура per tenant.** Отклонено:
   дорого, ops-кошмар. Shared Prometheus / Loki / Grafana + `tenant_id`
   label — стандарт.
2. **Prometheus per tenant.** Отклонено: 100+ инстансов Prometheus,
   не масштабируется.
3. **Usage metering через Prometheus (без ClickHouse).** Отклонено:
   Prometheus retention 15–30 дней, billing нужны месячные агрегаты
   с retention 5 лет. ClickHouse — оптимально.
4. **SLO одинаковые для всех тенантов.** Отклонено: тарифы разные
   (SMB / Pro / Enterprise), SLO должны соответствовать.
5. **Дашборды per tenant (отдельный Grafana instance).** Отклонено:
   shared Grafana + `tenant_id` фильтр — проще.
6. **Алерты только глобальные (без per-tenant).** Отклонено: Tenant
   Admin должен видеть проблемы своего тенанта.
7. **Cross-tenant alerts только в audit log (без P1).** Отклонено:
   RLS violation — критическая безопасность, P1 обязателен.
8. **Noisy neighbor detection вручную.** Отклонено: не масштабируется,
   автоматизация через Prometheus rules.
9. **Отдельный Usage Metering Service.** Отклонено: overkill,
   middleware + ClickHouse достаточно. Если разрастётся — выделим.
10. **Kafka lag per tenant через топики per tenant.** Отклонено:
    Task2 ADR-002 (domain-level топики). Per-tenant lag — только
    на consumer side.

## Компромиссы

- **Shared observability** — «шумный сосед» в метриках (кардинальность
  Prometheus). Митигация: лимит на кол-во `tenant_id` labels
  (не более N активных), cardinality limits, recording rules.
- **`tenant_id` label во всех метриках** — рост кардинальности.
  Митигация: `tenant_id` только на tenant-scoped метриках; platform
  метрики без `tenant_id`.
- **SLO по тарифу (assumption)** — тарифы не финализированы.
  Митигация: вынести в Tenant config (Global Control Plane), менять
  без релиза.
- **Usage metering через ClickHouse** — задержка агрегации (не
  real-time). Митигация: для billing — приемлемо (месячные агрегаты);
  для real-time — Prometheus.
- **Per-tenant email alerts** — нужен email Tenant Admin в Tenant
  config. Митигация: onboarding (ADR-003) заполняет email.
- **Noisy neighbor detection** — порог 30% (assumption). Митигация:
  настраивается per тариф (enterprise может > 30%).
- **Cross-tenant alerts P1** — много шума, если легитимные platform
  операции. Митигация: whitelist platform-ролей (только super_admin
  и platform_admin), но всё равно P1 + audit.
- **Grafana per tenant фильтр** — если URL утёк, можно увидеть чужой
  tenant_id? Нет — фильтр применяется на backend (JWT), не только
  в URL. Митигация: Grafana datasource с RLS-like фильтром.
- **Kafka lag per tenant** — ограниченная точность (топики
  domain-level). Митигация: custom middleware в consumer, где возможно.
- **Usage metering retention 5 лет** — рост ClickHouse. Митигация:
  партиционирование по месяцам, TTL, архивация в S3.

## Ссылки на предыдущие решения

- Observability (3 столпа, метрики, SLO, Alertmanager) — **Task2
  ADR-006**.
- Kafka-топики (domain-level) — **Task2 ADR-002**.
- ClickHouse, flink_analytics — **Task2 ADR-003**.
- Audit logs → Loki — **Task3 ADR-005**.
- Tenant configs в Global Control Plane — **ADR-002 Task3**.
- Модель изоляции, RLS, квоты — **ADR-001 Task4**.
- RBAC, JWT, tenant-aware middleware — **ADR-002 Task4**.
- Onboarding — **ADR-003 Task4**.
- Матрица ролей — **`roles_matrix.md`**.
