# ADR-004: План аварийного переключения (failover) и таблица сценариев отказа

## Контекст

ADR-001 материализовал 4 региона с двумя зонами, ADR-002 зафиксировал
схему репликации и RTO/RPO, ADR-003 — georouting и health checks.
Задание Task3 требует «диаграмму аварийного переключения в нотации C4
с основными точками отказа и механизмами переключения, таблицу сценариев
отказа».

Уже зафиксировано:
- 4 региона, 2 зоны на регион (SEA — частичный DR).
- RTO ≤ 45 мин для критичных, ≤ 4 ч для не-критичных (ADR-002).
- RPO ≤ 15 мин финансы, ≤ 1 ч остальные (ADR-002).
- DR-репликация: Postgres streaming, ClickHouse backup, Keycloak replica,
  Object storage — внутри регуляторной зоны.
- Kafka cross-region **не реплицируется** (Task2 ADR-002).
- Global Control Plane и Global Analytics — в нейтральной зоне EU/Франкфурт
  (ADR-002), с read-копиями во всех регионах.
- GLB health checks каждые 10 сек (ADR-003).
- SEA-PII RPO = ∞ (принятый риск, ADR-001).

**Задача:** спроектировать уровни отказа, механизмы переключения,
оркестратор, процедуру восстановления и таблицу сценариев — так, чтобы
RTO/RPO были достижимы.

## Требования

1. Автоматический failover для критичных сценариев (ручной = 2–4 ч,
   не укладывается в RTO ≤ 45 мин).
2. Покрыть 4 уровня отказа: AZ → город → регион → global.
3. Явно зафиксировать точки отказа (failure points) и механизмы
   переключения для каждой.
4. Reconciliation после failover (Kafka in-flight, внешние PSP).
5. Отразить на C4-диаграмме (красным — новое, синим — существующее).
6. Не дублировать ADR-002 (репликация) и ADR-003 (georouting).

## Решение

### 1. Четыре уровня отказа

| Уровень | Что ломается | Модель | Механизм | RTO |
|---|---|---|---|---|
| **L1: AZ** | Одна зона доступности внутри города | HA (не DR) | K8s reschedule + Postgres sync replica в другой AZ + Kafka RF=3 | < 5 мин |
| **L2: Город (зона 1 региона)** | Целый город (Москва / Сингапур / Сан-Паулу / Дубай) | DR (warm standby) | Failover Orchestrator: promote replica → DNS switch | ≤ 45 мин |
| **L3: Регион (обе зоны)** | Обе зоны региона | DR cross-region | GLB выводит регион → трафик в другие; degraded mode для роуминга | ≤ 45 мин (для остальных регионов) |
| **L4: Global** | Global Control Plane / Global Analytics (EU) | Read-копии в регионах | Регионы работают на локальных копиях; master восстанавливается | ≤ 30 мин |

**Ключевое разделение:** L1 — это HA (уже решено в Task2: RF=3, per-service
реплики, multi-AZ). Task3 фокусируется на **L2–L4** — это DR.

### 2. Failover Orchestrator (per-region, в зоне 2)

**Что это:** автоматический процесс (Python + Temporal / K8s operator),
живущий в **зоне 2** каждого региона. Триггерится при потере зоны 1.

**Функции:**
1. **Health check-агрегация** — слушает алерты от Prometheus / GLB,
   определяет «зона 1 недоступна» (3 фейла подряд).
2. **Promote replica** — переводит Postgres replica (зона 2) в primary.
3. **Keycloak replica activate** — переключает Keycloak на replica.
4. **DNS switch** — обновляет GeoDNS / GLB, чтобы трафик шёл в зону 2
   (через региональный GLB pool).
5. **K8s scale-up** — увеличивает реплики сервисов в зоне 2
   (warm standby → active).
6. **Reconciliation** — запускает сверку outbox ↔ processed_events
   и сверку с внешними PSP (см. раздел 4).
7. **Alerting** — уведомляет on-call, переводит в incident mode.

**Где живёт:** зона 2. Если потеряна зона 1 — orchestrator жив. Если
потерян регион целиком — orchestrator мёртв, failover делается вручную
из соседнего региона (L3).

**Идемпотентность:** orchestrator хранит state в БД зоны 2. Повторный
запуск не ломает систему (promote уже-promoted replica — no-op).

### 3. Health checks (детекция)

| Компонент | Кто проверяет | Интервал | Таймаут | Порог |
|---|---|---|---|---|
| **Regional Ingress (Kong)** | GLB (ADR-003) | 10 сек | 3 сек | 3 фейла |
| **Postgres primary** | Failover Orchestrator | 5 сек | 2 сек | 3 фейла |
| **Keycloak** | Failover Orchestrator | 10 сек | 3 сек | 3 фейла |
| **Kafka broker** | Kafka controller + Prometheus | 10 сек | 5 сек | 1 фейл (RF=3) |
| **Global Control Plane (EU)** | Read-копии в регионах | 30 сек | 10 сек | 3 фейла |
| **External PSP** | Payment Service (circuit breaker) | per-request | 5 сек | 5 фейлов за 60 сек |

**Детекция L2 (город):** если все health checks зоны 1 фейлят
одновременно (Kong + Postgres + Keycloak), orchestrator считает
«зона 1 недоступна» → запускает failover.

**False positive protection:** 3 фейла подряд + подтверждение от второго
источника (Prometheus + GLB) → иначе не триггерим. Иначе «моргание»
сети вызовет лишний failover.

### 4. Reconciliation после failover

**Проблема:** после failover зона 2 становится primary, но:
- Kafka in-flight события потеряны (cross-region не реплицируется);
- outbox может содержать неопубликованные события;
- внешние PSP могли обработать платёж, но наш consumer не успел
  записать результат.

**Процедура:**
1. **Outbox reconciliation:** Failover Orchestrator читает `outbox`
   (где `published_at IS NULL`) → публикует в Kafka повторно
   (идемпотентно, `event_id` дедуплицируется consumer'ами).
2. **Processed events reconciliation:** сверка `outbox` ↔
   `processed_events` — какие события опубликованы, но не обработаны
   → повторная публикация.
3. **External PSP reconciliation:** Payment Service делает
   `GET /payments?from=<T-1h>&to=now` к внешнему PSP → сверяет
   с локальной БД → досоздаёт пропущенные `Payment.Succeeded` /
   `Payment.Failed`.
4. **Payout reconciliation:** Payouts Service сверяет батчи с Bank API.
5. **Alert:** если reconciliation нашёл расхождения — P1 инцидент,
   ручной разбор.

**Окно reconciliation:** ≤ 30 мин после failover. RTO 45 мин включает
reconciliation.

### 5. SEA — особый сценарий (частичный DR)

При потере Сингапура целиком:
- **Non-PII** (outbox, агрегаты, конфиги, схемы, ML-модели, образы) —
  восстанавливаются из Батама за ≤ 4 ч.
- **PII** (Booking, Payments, Driver Profile, KYC) — **потеряно**
  (принятый риск, ADR-001). Восстановление только из бэкапов
  в том же регионе (если уцелели).
- **Degraded mode:** SEA-операции недоступны, трафик → MENA / LatAm,
  роуминг пассажиров SEA → API к домашнему региону.
- **RTO для non-PII:** ≤ 4 ч (не входит в 99,99% SLA).
- **RPO для SEA-PII:** ∞ (потеря).

**Явно фиксируется в алертах и runbook:** «SEA city loss = PII loss,
принятый риск. Эскалация к CTO».

### 6. Global Control Plane — failover

При потере EU/Франкфурт:
- **Регионы продолжают работать** на локальных read-копиях
  (Apicurio, Unleash, fraud rules, tenant configs, identity directory,
  Vault — pull-on-start + кэш).
- **Синк приостанавливается** — новые флаги / правила / схемы не
  доставляются. Сервисы работают по последней известной версии.
- **Восстановление master в EU:** ≤ 30 мин (из бэкапа / standby).
- **RPO:** ≤ 5 мин (потеря последних изменений конфигов).
- **Не критично для 99,99%:** регионы не падают.

### 7. Диаграмма

`C2_Failover.puml` — отдельный файл Task3. Фокус — точки отказа
и механизмы переключения. Не детализирует BFF / сервисы (они в Task1 C2).
Красным — механизмы failover (orchestrator, health checks, promote, DNS switch),
синим — существующие компоненты (GoFuture Platform, DR-реплики из ADR-002).

### 8. Таблица сценариев отказа

См. раздел «Таблица сценариев отказа» ниже.

### 9. Что НЕ входит в это решение

- Схема репликации (что куда) — **ADR-002**.
- Геомаршрутизация — **ADR-003**.
- Регуляторные детали — **ADR-005**.
- Security-overlay — **ADR-006**.
- HA внутри зоны (K8s reschedule, Postgres sync replica в AZ) —
  Task2 / инфраструктурное решение.
- Runbook (пошаговые инструкции) — вне scope ADR, требование
  зафиксировано.
- Game day (тестирование failover) — вне scope, требование
  зафиксировано (раз в квартал).

## Таблица сценариев отказа

| # | Сценарий | Уровень | Что ломается | Детекция | Механизм переключения | RTO | RPO | Остаточные потери | Авто? |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Отказ AZ (одна зона в городе) | L1 | Ноды в AZ | K8s health | K8s reschedule + Postgres sync replica + Kafka RF=3 | < 5 мин | 0 | Нет (HA) | ✅ |
| 2 | Отказ города (зона 1 региона RU / LatAm / MENA) | L2 | Москва / Сан-Паулу / Дубай | Health checks (Kong + PG + Keycloak) | Failover Orchestrator: promote PG replica → Keycloak activate → DNS switch → K8s scale-up → reconciliation | ≤ 45 мин | ≤ 15 мин (финансы), ≤ 1 ч (остальные) | Kafka in-flight (reconciliation), дубли событий (идемпотентность) | ✅ |
| 3 | Отказ Сингапура (SEA, частичный DR) | L2 | Сингапур целиком | Health checks | Failover Orchestrator: restore non-PII из Батама → DNS switch → degraded mode для PII | ≤ 4 ч (non-PII), PII = ∞ | Non-PII ≤ 1 ч, PII = ∞ | **SEA-PII потерян** (принятый риск) | ⚠️ Частично |
| 4 | Отказ региона целиком (обе зоны) | L3 | RU / LatAm / MENA целиком | GLB health check | GLB выводит регион → трафик в соседние; роуминг через API; PII региона недоступна | ≤ 45 мин (для остальных) | PII = ∞ (в регионе), non-PII восстанавливается | PII региона потеряна до восстановления из бэкапа | ✅ |
| 5 | Отказ Global Control Plane (EU) | L4 | Apicurio / Unleash / Vault / fraud rules | Health check от read-копий | Регионы на локальных копиях; master восстанавливается из backup / standby | ≤ 30 мин | ≤ 5 мин | Новые флаги / правила не доставляются (работа на старой версии) | ✅ |
| 6 | Отказ Global Analytics (EU) | L4 | Global ClickHouse / ML training / BI | Health check | Не критично; регионы работают; восстановление из backup | ≤ 4 ч | ≤ 1 ч | Отчёты устаревают | ✅ |
| 7 | Отказ Kafka broker (1–2 из 3) | L1 | Брокеры | Kafka controller + Prometheus | Leader election, ISR rebalance | < 1 мин | 0 | Нет (RF=3) | ✅ |
| 8 | Отказ Postgres primary (зона 1) | L1/L2 | Primary | Failover Orchestrator | Promote replica (зона 2 или AZ-2) | ≤ 5 мин (AZ) / ≤ 45 мин (город) | ≤ 15 мин / ≤ 1 ч | Транзакции в in-flight | ✅ |
| 9 | Отказ Keycloak (primary) | L1/L2 | Keycloak | Health check | Keycloak replica activate | ≤ 5 мин | ≤ 5 мин | Сессии | ✅ |
| 10 | Отказ внешнего PSP (Yandex Pay / GrabPay / Mercado Pago / Network Intl) | — | Внешний | Circuit breaker (Payment Service) | Retry (3 уровня, ADR-005) + fallback на резервный PSP (если есть) + degraded: только наличные | per-request | ≤ 15 мин | Платежи в очереди | ⚠️ Частично (fallback PSP) |
| 11 | DDoS-атака (L3-L7) | Edge | Ingress | WAF / DDoS detection | Cloudflare / AntiDDoS mitigation (авто) | < 1 мин | 0 | Нет | ✅ |
| 12 | Network partition зона 1 ↔ зона 2 | L2 | Связь | Health checks | Orchestrator: считает зону 1 недоступной → failover | ≤ 45 мин | ≤ 15 мин / ≤ 1 ч | Split-brain (защита: quorum) | ✅ |
| 13 | Отказ Failover Orchestrator (зона 2) | L2 | Orchestrator | Health check (внешний) | Ручной failover из соседнего региона (runbook) | 2–4 ч | ≤ 15 мин / ≤ 1 ч | Ручное вмешательство | ❌ Ручной |
| 14 | Отказ DNS / GLB (edge) | Edge | GeoDNS / GLB | Внешний мониторинг | Cloudflare failover (multi-region edge) | < 5 мин | 0 | Нет | ✅ |

**Легенда:** ✅ авто, ⚠️ частично авто, ❌ ручной (runbook).

**Ключевые выводы из таблицы:**
- **13 из 14 сценариев — автоматические** (или частично).
- **Единственный ручной** — отказ самого Failover Orchestrator
  (сценарий 13), но это редкий кейс (orchestrator живёт в зоне 2,
  защищён от потери зоны 1).
- **SEA-PII и потеря региона целиком** — принятые риски с потерей PII.
  Явно в алертах.
- **Сценарии 10 (PSP)** — частично, зависит от наличия fallback PSP.
  Для RU (Yandex Pay) fallback — Тинькофф / СБП. Для SEA — GrabPay /
  Xendit. Для LatAm — Mercado Pago / Pix. Для MENA — Network Intl /
  Checkout.com. Фиксируется в ADR-005 (комплаенс).

## Альтернативы

1. **Ручной failover везде (без автоматики).** Отклонено: RTO 2–4 ч
   не укладывается в 45 мин, 99,99% недостижим.
2. **Active-active cross-region для всех сервисов.** Отклонено:
   cross-region PII запрещён residency (ADR-002). Для non-PII можно,
   но выигрыш не оправдывает сложность.
3. **Kafka MirrorMaker для DR.** Отклонено: Task2 ADR-002 уже
   зафиксировал отказ.
4. **Sync-репликация БД cross-region (RPO=0).** Отклонено:
   неприемлемая latency, физический предел скорости света.
5. **Global Control Plane в одном из регионов.** Отклонено в ADR-002
   (зависимость от региона).
6. **Без reconciliation после failover.** Отклонено: двойные платежи,
   потерянные события — финансовые потери.
7. **Split-brain без quorum.** Отклонено: два primary → конфликт
   данных. Защита: Postgres quorum (2 из 3), K8s etcd quorum.

## Компромиссы

- **Автоматический failover** — риск false positive (моргание сети
  → лишний failover). Митигация: 3 фейла подряд + подтверждение
  от двух источников.
- **Reconciliation ≤ 30 мин** — окно, в которое возможны дубли
  событий. Митигация: идемпотентность consumer'ов (Task2 ADR-005).
- **SEA-PII = ∞** — принятый риск. Митигация: алерт, эскалация к CTO,
  бэкапы в том же регионе.
- **Orchestrator в зоне 2** — если потеряна зона 2 (не зона 1) —
  orchestrator мёртв, но зона 1 работает (degraded: нет автоматики,
  но сервис жив). Митигация: orchestrator можно поднять вручную
  из зоны 1 или соседнего региона.
- **Global Control Plane RTO ≤ 30 мин** — синк флагов / правил
  приостановлен. Митигация: регионы работают на последней версии,
  не падают.
- **RTO 45 мин для L2** — включает reconciliation. Если reconciliation
  найдёт много расхождений — RTO растёт. Митигация: мониторинг,
  приоритизация финансов.
- **Сценарий 10 (PSP)** — частично авто. Митигация: fallback PSP
  (фиксируется в ADR-005), degraded mode (наличные).
- **Сценарий 13 (orchestrator)** — ручной. Митигация: runbook,
  game day раз в квартал, orchestrator в зоне 2 (защищён от L2).

## Ссылки на предыдущие решения

- Kong, BFF, сервисы — **Task1 C2 Final**.
- Kafka RF=3, outbox, DLQ, retry — **Task2 ADR-002, ADR-005**.
- 4 региона, 2 зоны, RTO 45 мин — **ADR-001 Task3**.
- Репликация, RPO, placement Global — **ADR-002 Task3**.
- GLB, health checks, georouting — **ADR-003 Task3**.
- Комплаенс (SEA-PII, fallback PSP) — **ADR-005 Task3**.
- Security-overlay — **ADR-006 Task3**.