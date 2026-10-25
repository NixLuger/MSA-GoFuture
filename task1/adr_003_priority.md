# ADR-003: Приоритетная очерёдность выделения сервисов

## Контекст

Определены 12 сервисов (10 бизнес-доменных + 2 платформенных), зафиксирована
карта доменных границ (ADR-002). Определён механизм обратной совместимости
(ADR-004): ACL внутри монолита, Kong-роутинг v1/v2, транзакционный outbox,
feature flags.

Требуется определить очерёдность выделения сервисов с учётом:
- бизнес-требования «time-to-market ≤ 2 недель» к концу года;
- отсутствия жёсткого дедлайна на сам рефакторинг;
- стратегии Strangler Fig с ACL (ADR-004);
- минимизации простоя (простой недопустим).

НФТ — ADR-001. Карта сервисов — ADR-002. Механизм обратной совместимости —
ADR-004.

## Требования

1. Определить очерёдность выделения сервисов.
2. Обосновать логику очерёдности.
3. Для каждой волны указать конкретный механизм ACL (sync HTTP или async Kafka).
4. Разложить процесс на волны, синхронизированные с целью «через год».

## Решение

### 1. Критерии приоритизации

| Приоритет | Критерий | Вес |
|-----------|----------|-----|
| 1 | Foundation-фактор (от сервиса зависят другие) | 40% |
| 2 | Простота выноса (слабая связность, малый объём) | 30% |
| 3 | Снижение риска (узкие места монолита) | 20% |
| 4 | Бизнес-ценность (влияние на time-to-market) | 10% |

### 2. Оценка сервисов по критериям

**Шкала:** 5 — максимальное значение критерия, 1 — минимальное.

**Веса (из раздела 1):** Foundation — 0.4; Простота — 0.3; Риск-снижение — 0.2; Бизнес-ценность — 0.1.

| # | Сервис | Foundation (0.4) | Простота (0.3) | Риск (0.2) | Бизнес (0.1) | Итоговый вес | Волна |
|---|--------|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | Identity/Auth | 5 | 4 | 3 | 4 | **4.2** | 1 |
| 2 | Notification | 4 | 5 | 2 | 2 | **3.7** | 1 |
| 3 | Tenant Management | 4 | 4 | 3 | 3 | **3.7** | 2 |
| 4 | Geography | 5 | 3 | 3 | 4 | **3.9** | 2 |
| 5 | Driver Profile | 4 | 3 | 3 | 3 | **3.4** | 2 |
| 6 | Pricing | 4 | 3 | 4 | 5 | **3.8** | 3 |
| 7 | Driver Status | 3 | 2 | 4 | 5 | **3.1** | 4 |
| 8 | Payments | 4 | 2 | 4 | 5 | **3.5** | 5 |
| 9 | Payouts | 3 | 3 | 3 | 4 | **3.1** | 5 |
| 10 | Fraud | 2 | 3 | 2 | 3 | **2.4** | Парал. |
| 11 | Analytics | 1 | 5 | 1 | 2 | **2.3** | Парал. |
| 12 | Booking | 2 | 1 | 5 | 5 | **2.6** | 6 |

**Итоговый вес = Foundation × 0.4 + Простота × 0.3 + Риск × 0.2 + Бизнес × 0.1.**

**Важное уточнение.** Итоговый вес — это **sanity-check**, а не жёсткий порядок волн. Финальная очерёдность **корректируется зависимостями между сервисами**:

- **Geography (3.9) идёт в Волне 2, а не в Волне 1**, потому что требует готовности ACL-каркаса и опыта на простых сервисах. Но именно она — самая «ценная» по весу после Identity.
- **Pricing (3.8) идёт в Волне 3, а не в Волне 2**, потому что зависит от Geography (маршруты/зоны). Без неё Pricing не может работать в новой архитектуре.
- **Payments (3.5) идёт в Волне 5, а не в Волне 3**, потому что требует готовности Fraud (shadow mode) и Payouts (для связи payout-логики). Хотя по весу Payments выше Driver Status (3.1).
- **Driver Status (3.1) и Payouts (3.1) идут в Волне 4–5**, потому что зависят от Geography и Payments соответственно.
- **Booking (2.6) идёт в Волне 6 (последняя)**, потому что ядро и зависит от всех foundation-сервисов.
- **Fraud (2.4) и Analytics (2.3) — параллельные волны** (месяцы 3–8), потому что не блокируют основные волны и могут выноситься по мере готовности.

Иначе говоря: **порядок волн = топология зависимостей + приоритет foundation + опыт Strangler Fig**, а вес — для валидации, что никто не выносится «за зря раньше времени».
### 3. Волны выделения

#### Волна 0 (месяцы 1–2): Платформенная подготовка и ACL-каркас

**Инфраструктура:**
- Managed Kubernetes, CI/CD (GitLab CI), Docker Registry.
- Kong как API Gateway (ADR-004) с маршрутизацией:
    - `/api/v1/*` → Monolith Legacy API.
    - `/api/v2/*` → Monolith ACL.
    - `/api/bff/*` → BFF-сервисы (по мере появления).
- Apache Kafka + Schema Registry.
- Observability-стек: Prometheus, Grafana, Loki, OTel Collector, Alertmanager.
- **Feature Flags сервис** (Unleash / LaunchDarkly) — для управления
  переключением v1/v2 и shadow/dual режимов.

**ACL-каркас в монолите (ADR-004):**
- Django app `acl/`: routing, v2 Controllers, Model Translators, HTTP Clients.
- OIDC middleware (валидация JWT от Keycloak для v2-запросов).
- Kafka Producer + таблица `outbox` в `db_main` (транзакционный outbox).
- Outbox poller (в `app_worker`) — публикация событий в Kafka.
- Feature flag-обвязка для маршрутизации.

**Keycloak:**
- Развёрнут **изолированно**. Монолит **не использует** его для v1.
- Настроен realm, клиенты OIDC для будущего v2-фронтенда.
- Готов к подключению в Wave 1+ (для новых сервисов и новых клиентов).

**Бизнес-логика не меняется.** v2-контроллеры пока проксируют в legacy-логику.

#### Волна 1 (месяцы 2–4): Простые foundation-сервисы

**Identity/Auth, Notification.**

**Identity/Auth — уточнение:**
- Keycloak уже развёрнут (Wave 0).
- **Монолит не меняет auth** для v1 (session / DRF Token).
- Новые сервисы и v2-клиенты — через OIDC (Keycloak).
- **Механизм ACL:** ACL валидирует OIDC JWT для v2-запросов. Монолит
  v1 продолжает со своей auth. Cross-service вызовы монолит → сервис —
  через service account (API key / mTLS).
- Миграция существующих клиентов с v1 на v2 (и, соответственно, на OIDC) —
  постепенная, вне жёсткого годового плана (по мере обновления приложений).

**Notification:**
- `svc_notification` создан: FastAPI + Kafka consumers + своя PostgreSQL.
- **Механизм ACL:**
    - Legacy-вызовы `notification_domain.send()` в монолите перехватываются
      ACL.
    - ACL пишет `NotificationRequested` в `outbox` (в одной транзакции с
      бизнес-изменением).
    - Outbox poller публикует в Kafka.
    - `svc_notification` подписан, отправляет.
    - `notification_tasks` (Celery) отключается feature flag
      `notification.via_kafka=true`.
    - `notification_domain` (Django app) → read-only → удаление.

**Rollback:** feature flag `notification.via_kafka=false` → возврат на Celery.

#### Волна 2 (месяцы 4–6): Foundation среднего размера

**Tenant Management, Geography, Driver Profile.**

**Механизм ACL (для каждого):**
- **Tenant Management:** sync HTTP из ACL в `svc_tenant`. Dual-read
  на время перехода (ACL читает из сервиса, fallback — монолит).
- **Geography:** sync HTTP из ACL в `svc_geography`. Модели транслируются
  через `Model Translators`. Shadow mode для маршрутов (сравнение
  результатов).
- **Driver Profile:** sync HTTP из ACL в `svc_driver_profile`. Dual-write
  на время перехода, reconciliation job сверяет данные.

**Data migration:** CDC (Debezium) для начального backfill + dual-write
для инкрементальных изменений.

#### Волна 3 (месяцы 6–7): Foundation + бизнес-критичность

**Pricing.**

- **Механизм ACL:** sync HTTP из ACL в `svc_pricing`. **Shadow mode**
  обязателен: ACL параллельно вызывает старый и новый расчёт, сравнивает,
  логирует расхождения. Переключение только после 7 дней без расхождений.
- **Feature flag:** `pricing.shadow_mode=true` → `pricing.use_new=true`.

#### Волна 4 (месяцы 7–9): Highload foundation

**Driver Status.**

- **Механизм ACL:**
    - Location updates (высокочастотные) — **async через outbox** →
      Kafka `DriverLocationUpdated`. `svc_driver_status` подписан,
      обновляет Redis/PostGIS.
    - Запросы статуса (низкочастотные) — **sync HTTP** из ACL в
      `svc_driver_status`.
- **Feature flag:** `driver_status.location_via_kafka=true`.

#### Волна 5 (месяцы 9–10): Критичные денежные сервисы

**Payments, Payouts.**

- **Механизм ACL:**
    - Sync HTTP из ACL в `svc_payments` / `svc_payouts`.
    - **Dual-write** транзакций в монолит и сервис (≤ 4 недели).
    - Reconciliation job: сверка балансов, алерты на расхождения.
    - Feature flag постепенного rollout: `payments.routing_percent` =
      1 → 10 → 50 → 100.

#### Волна 6 (месяцы 10–11): Ядро

**Booking.**

- **Механизм ACL:**
    - Новые поездки — sync HTTP в `svc_booking` (по feature flag).
    - Старые поездки — legacy-логика монолита.
    - **Kong-роутинг:** по заголовку / feature flag.
    - Постепенная миграция: 1% → 10% → 50% → 100% новых поездок.
- **Feature flag:** `booking.routing_percent`.

#### Параллельные волны (месяцы 3–8): Независимые

**Analytics, Fraud.**

- **Analytics:** async через Kafka (outbox). `svc_analytics` подписан на
  все доменные события. Dual-read отчётов на время перехода.
- **Fraud:** sync HTTP из ACL в `svc_fraud` (критичные проверки) +
  async через Kafka (batch-проверки). Shadow mode для сравнения со старой
  fraud-логикой.

### 4. Сводная таблица волн

| Волна | Месяцы | Сервисы | Механизм ACL |
|-------|--------|---------|--------------|
| 0 | 1–2 | Инфраструктура + ACL-каркас + Keycloak (изолированно) | — |
| 1 | 2–4 | Identity/Auth (v2 только), Notification | Kafka outbox |
| 2 | 4–6 | Tenant, Geography, Driver Profile | Sync HTTP + dual-read/write |
| 3 | 6–7 | Pricing | Sync HTTP + shadow mode |
| 4 | 7–9 | Driver Status | Kafka (location) + sync HTTP (запросы) |
| 5 | 9–10 | Payments, Payouts | Sync HTTP + dual-write + reconciliation |
| 6 | 10–11 | Booking | Sync HTTP + постепенный rollout |
| Парал. | 3–8 | Analytics, Fraud | Kafka (Analytics) + HTTP+shadow (Fraud) |

**Месяц 12** — стабилизация, вывод legacy-кода и ACL.

### 5. Судьба Celery и RabbitMQ

Celery — **распределённый монолит** (anti-pattern): отдельные процессы,
но общий код, общая БД, совместный деплой. Архитектурно — часть монолита.

**Поэтапное сжатие `app_worker`:**

| Волна | Что уходит из Celery | Куда уходит |
|-------|----------------------|-------------|
| 1 | `notification_tasks` | `svc_notification` (Kafka consumer + async send) |
| Парал. | `analytics_tasks` | `svc_analytics` (Kafka consumer) |
| 5 | `payout_tasks` | `svc_payouts` (собственный механизм async) |
| Финал | `app_worker` удаляется | Каждый сервис выбирает свой async-механизм |

**RabbitMQ:** после выноса всех Celery-задач **удаляется** из стека.
Kafka — единственная шина.

### 6. Feature Flags (сводка)

| Флаг | Волна | Что переключает |
|------|-------|-----------------|
| `notification.via_kafka` | 1 | Celery → Kafka (Notification) |
| `tenant.use_service` | 2 | Монолит → svc_tenant |
| `geography.use_service` | 2 | Монолит → svc_geography |
| `driver_profile.use_service` | 2 | Монолит → svc_driver_profile |
| `pricing.shadow_mode` | 3 | Параллельный расчёт |
| `pricing.use_new` | 3 | Монолит → svc_pricing |
| `driver_status.location_via_kafka` | 4 | DB write → Kafka |
| `payments.dual_write` | 5 | Dual-write платежей |
| `payments.routing_percent` | 5 | % трафика на сервис |
| `payouts.dual_write` | 5 | Dual-write выплат |
| `booking.routing_percent` | 6 | % новых поездок на сервис |
| `analytics.use_service` | Парал. | Монолит → svc_analytics |
| `fraud.shadow_mode` | Парал. | Параллельная проверка |

### 7. Критерии завершения волны (go/no-go)

1. Новый сервис обрабатывает 100% своего домена (для волн 1–5) или
   заданный % (для волны 6).
2. SLO соответствуют целевым.
3. Dual-write/shadow без расхождений ≥ 7 дней.
4. Rollback протестирован (≤ 30 мин).
5. Observability настроена.
6. Legacy-код в монолите переведён в read-only.

### 8. Что НЕ входит в это решение

- Детальный дизайн Kafka-топиков и consumer groups — отдельное решение.
- План миграции данных по каждому сервису — отдельное решение.
- Мультирегиональное развёртывание — отдельное решение.
- Multi-tenancy изоляция — отдельное решение.

## Альтернативы

1. Начать с Booking. Отклонено: максимальный риск.
2. Начать с Payments. Отклонено: высокий риск без опыта Strangler Fig.
3. Начать с Analytics. Отклонено: не foundation, не снижает риск.
4. Big Bang. Отклонено: длительный простой.
5. Параллельный вынос всеми командами. Отклонено: не хватит инфраструктуры.

## Компромиссы

- **Долгий путь до Booking (месяц 11).** Ядро ~10 месяцев в монолите.
  Митигация: монолит обрабатывает новые поездки без блокировки фич.
- **Зависимость волн.** Митигация: параллельные волны (Analytics, Fraud).
- **Dual-write.** Митигация: ≤ 4 недели на сервис, reconciliation.
- **ACL — долгоживущий компонент.** Митигация: строгая граница (никакой
  бизнес-логики).
- **Feature flags везде.** Митигация: централизованный сервис, аудит.
- **Риск «вечного перехода».** Митигация: жёсткие go/no-go критерии.