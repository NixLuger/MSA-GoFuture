# ADR-006: Обоснование механизмов защиты

## Контекст

Задание Task3 требует «доработать диаграмму C2, отразив на ней изменения, связанные с внедрением инструментов защиты (WAF, Firewall и аналогичных), и добавить текстовое обоснование к схеме с объяснением выбранных механизмов».

Компоненты защиты уже визуально отражены:
- `C2_Replication.puml` — `Regional Edge & Security` в каждом регионе.
- `C2_Georouting.puml` — полная цепочка `GeoDNS → WAF → DDoS → Network Firewall → GLB → Regional Ingress (Kong)`.

**Что нужно:** текстовое обоснование выбранных механизмов — почему именно эти инструменты, а не альтернативы. Это и есть содержание ADR-006.

Уже зафиксировано:
- 4 региона, 4 регуляторных зоны (ADR-001).
- Global Control Plane в EU/Франкфурт (ADR-002).
- Роуминг через API, cross-region PII нет (ADR-003).
- Failover-сценарии, reconciliation (ADR-004).
- Комплаенс, breach notification, audit (ADR-005).

## Требования

1. Обосновать выбор каждого механизма защиты с альтернативами.
2. Не дублировать ADR-003 (геомаршрутизация) и ADR-004 (failover).
3. Показать, как механизмы работают в multi-region сценарии.
4. Зафиксировать границы ответственности (что где живёт).

## Решение

### 1. Слои защиты (defense in depth)

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 0: DDoS (L3-L7)         — Cloudflare / AntiDDoS      │
├─────────────────────────────────────────────────────────────┤
│ Layer 1: WAF                  — Cloudflare WAF             │
├─────────────────────────────────────────────────────────────┤
│ Layer 2: Network Firewall     — Cloud Firewall (per-region) │
├─────────────────────────────────────────────────────────────┤
│ Layer 3: GLB (TLS termination) — Cloudflare / L7           │
├─────────────────────────────────────────────────────────────┤
│ Layer 4: mTLS + JWT           — Kong (Regional Ingress)    │
├─────────────────────────────────────────────────────────────┤
│ Layer 5: Secrets              — Vault (Global, ADR-002)    │
├─────────────────────────────────────────────────────────────┤
│ Layer 6: Audit / SIEM         — OTel + Loki (Task2 ADR-006)│
└─────────────────────────────────────────────────────────────┘
```

Каждый слой независим: компрометация одного не компрометирует остальные.

### 2. WAF — Cloudflare WAF

**Что защищает:** OWASP Top-10 (SQLi, XSS, RCE, SSRF), rate limiting (per-IP, per-user), bot protection, custom rules (например, блокировка известных bad-ASN).

**Почему Cloudflare:**
- **Глобальное покрытие** (4 региона + EU для control plane) — единый дашборд, единые правила.
- **Managed ruleset** — регулярные обновления сигнатур без нашей ops-нагрузки.
- **Latency** — WAF на edge, не добавляет hop.
- **Интеграция с GLB и DDoS** в одном продукте — меньше vendor'ов.

**Альтернативы:**

| Альтернатива | Почему отклонено |
|---|---|
| **Yandex Smart Web Security** | Только RU-регион; для SEA/LatAm/MENA нужен отдельный WAF → 2+ vendor'а, раздробленные правила |
| **AWS WAF** | Vendor lock-in на AWS (у нас Yandex Cloud + другие), нет глобального edge как у Cloudflare |
| **Self-hosted ModSecurity** | Ops-нагрузка (обновление правил, тюнинг), нет глобального edge, latency выше |
| **Imperva / Akamai** | Дороже без явного выигрыша для нашего профиля |

**Кастомизация:** для RU-региона — дополнительный слой Yandex Smart Web Security как **второй WAF** (defense in depth) — опционально, если регулятор потребует локального WAF.

### 3. DDoS Protection — Cloudflare / Yandex AntiDDoS

**Что защищает:**
- **L3-L4 (volumetric, protocol):** SYN flood, UDP amplification, ICMP flood.
- **L7 (application-layer):** HTTP flood, slowloris, bot-based атаки.

**Почему Cloudflare (глобально) + Yandex AntiDDoS (RU):**
- **Cloudflare** — автоматическая L3-L7 mitigation, глобальный scrubbing capacity, не требует ручного вмешательства для типовых атак.
- **Yandex AntiDDoS** — для RU-региона, т.к. Cloudflare может быть недоступен в РФ (санкционные ограничения, требования локализации трафика).

**Альтернативы:**

| Альтернатива | Почему отклонено |
|---|---|
| **Только Cloudflare** | В RU-регионе — регуляторные и санкционные риски |
| **Только Yandex AntiDDoS** | Только RU; для остальных регионов нужен отдельный |
| **Self-hosted scrubbing** | Дорого (нужен свой scrubbing center), не масштабируется |
| **AWS Shield / GCP Cloud Armor** | Не работают с Yandex Cloud как primary |

**Граница:** Cloudflare — глобальный L3-L7 mitigation. Yandex AntiDDoS — L3-L4 для RU-трафика, если Cloudflare недоступен.

### 4. Network Firewall — per-region (Cloud Firewall)

**Что защищает:** IP allow/deny, geo-blocking (опционально, например, «только SEA IP в region-sea» — при регуляторном требовании), egress filtering (исходящие соединения только к whitelisted destinations), микросегментация между сервисами.

**Почему per-region, а не глобальный:**
- **Локальные политики** — RU-регион может требовать других правил (152-ФЗ, geo-blocking).
- **Blast radius** — компрометация firewall в одном регионе не даёт доступа ко всем.
- **Latency** — firewall ближе к трафику.
- **Регуляторика** — некоторые режимы требуют локального контроля периметра.

**Альтернативы:**

| Альтернатива | Почему отклонено |
|---|---|
| **Единый глобальный firewall** | Единая точка отказа, latency, регуляторные ограничения |
| **Только security groups (K8s NetworkPolicy)** | Не покрывает ingress из интернета, нет egress filtering |
| **Только WAF** | WAF — L7, не покрывает L3-L4 (кроме DDoS) |

**Реализация:** IaC (Terraform), единые базовые политики + per-region overrides.

### 5. GLB (TLS termination) — Cloudflare L7

**Что защищает:** TLS-терминация, health-checked routing, защита от TLS-атак (downgrade, weak ciphers).

**Почему GLB, а не Regional Ingress (Kong):**
- **Единая точка TLS-терминации** — сертификаты управляются централизованно (Cloudflare), не нужно раскатывать по 4 регионам.
- **Health checks** — GLB видит здоровье регионов, Kong — нет (Kong видит только свой регион).
- **Weight-based routing** — canary, blue-green на уровне GLB.

**Почему mTLS на Kong, а не на GLB:**
- **mTLS — между сервисами** (service-to-service), а не client-to-edge.
- **Kong** — сервис-меш ingress, знает identity сервиса (SPIFFE / cert).
- **GLB** — client-to-edge (TLS 1.3, сертификат клиента не нужен).

**Альтернативы:**

| Альтернатива | Почему отклонено |
|---|---|
| **mTLS на GLB** | Клиенты (мобильные приложения) не имеют сертификатов — только JWT |
| **TLS на Kong (без GLB)** | Нет health-checked routing между регионами, нет canary на уровне GLB |

### 6. mTLS + JWT — Kong (Regional Ingress)

**Что защищает:**
- **mTLS** — service-to-service (BFF → сервис, сервис → сервис) через SPIFFE.
- **JWT validation** — client-to-service (Kong валидирует JWT от Keycloak).
- **Rate limiting per-route** — защита от abuse на уровне API.
- **Canary releases** — feature flags + weight-based routing.

**Почему Kong:**
- Уже существует в Task1 C2 Final (переиспользуем).
- Поддерживает mTLS, JWT, rate limiting, canary — всё из коробки.
- Open-source, не vendor lock-in.

**Альтернативы:**

| Альтернатива | Почему отклонено |
|---|---|
| **NGINX Plus** | Платно, нет всех фич Kong |
| **Envoy** | Мощнее, но ops-нагрузка выше; Kong проще |
| **AWS API Gateway** | Vendor lock-in, нет в Yandex Cloud |

**Граница:** Kong — per-region (уже в C2_Replication и C2_Georouting). mTLS — только service-to-service. JWT — client-to-service.

### 7. Secrets — Vault (Global Control Plane)

**Что защищает:** JWT signing keys, encryption keys, DB credentials, PSP API keys.

**Почему Vault в Global Control Plane (EU/Франкфурт):**
- **Единый источник истины** — ключи синхронизируются во все регионы (ADR-002, корзина 1).
- **Non-PII** — secrets не содержат PII, residency не блокирует.
- **Rotation** — централизованная ротация ключей.
- **Read-копии в регионах** — pull-on-start + webhook, если EU недоступен — регионы работают на локальных копиях.

**Альтернативы:**

| Альтернатива | Почему отклонено |
|---|---|
| **Per-region Vault** | Нет единого источника, рассинхрон ключей → невозможно валидировать JWT cross-region |
| **K8s Secrets** | Не защищены (base64), нет ротации, нет audit |
| **AWS Secrets Manager / GCP Secret Manager** | Vendor lock-in |

**Граница:** Vault — master в EU, read-копии в регионах. Rotation — централизованно, rollout — через webhook.

### 8. Certificate management

**Что:** TLS-сертификаты для GLB (client-to-edge), mTLS-сертификаты для Kong (service-to-service).

**Как:**
- **GLB TLS:** Cloudflare Universal SSL + custom certificates (Let's Encrypt / wildcard).
- **mTLS:** SPIFFE / SPIRE — автоматическая выдача и ротация сертификатов для сервисов.
- **Certificate rotation:** автоматическая, без downtime (overlap period).

**Почему не вручную:** 4 региона × N сервисов = сотни сертификатов; ручное управление = ошибки, истечения, downtime.

**Альтернативы:**

| Альтернатива | Почему отклонено |
|---|---|
| **Ручные сертификаты** | Не масштабируется, риск истечения |
| **Только Let's Encrypt без автоматизации** | Rate limits, нет mTLS |
| **Vault PKI** | Возможно, но SPIRE проще для service-to-service |

### 9. Audit / SIEM — OTel + Loki (Task2 ADR-006)

**Что:** audit logs доступа к PII, cross-region API-вызовы, consent changes, breach-инциденты (ADR-005).

**Почему переиспользуем Task2 ADR-006:**
- **OTel Collector** уже собирает traces / metrics / logs.
- **Loki** — централизованные логи (per-region, residency).
- **Prometheus + Grafana + Alertmanager** — метрики и алерты.

**Что добавляем в Task3:**
- **Audit log schema** — стандартизированные события (`pii_access`, `cross_region_call`, `consent_change`).
- **Retention** — 5 лет для финансовых, 1 год для остальных (ADR-005).
- **Доступ к audit** — только DPO + security team, audit of audit.

**Альтернативы:**

| Альтернатива | Почему отклонено |
|---|---|
| **Splunk / Datadog** | Дорого, vendor lock-in |
| **Только Loki без schema** | Нет структурированного audit, сложно соответствовать регуляторике |
| **Отдельный SIEM (QRadar / ArcSight)** | Overkill для нашего масштаба, дорого |

### 10. Zero-trust principles

**Принципы:**
- **Never trust, always verify** — каждый запрос аутентифицируется и авторизуется (JWT + mTLS).
- **Least privilege** — сервисы имеют доступ только к своим ресурсам (Kafka ACL, DB per service).
- **Microsegmentation** — Network Firewall + K8s NetworkPolicy.
- **Assume breach** — audit logs, anomaly detection, breach notification (ADR-005).

**Реализация:**
- JWT везде (client-to-service).
- mTLS везде (service-to-service).
- Kafka ACL (Task2 ADR-002 — 1 producer = 1 domain).
- DB per service (Task1).
- Audit logs (раздел 9).

### 11. Границы ответственности (что где живёт)

| Механизм | Где | Кто управляет |
|---|---|---|
| GeoDNS | Global edge | Platform team |
| WAF | Global edge (Cloudflare) | Security team |
| DDoS | Global edge (Cloudflare) + RU (Yandex) | Security team |
| Network Firewall | Per-region | Platform team (IaC) |
| GLB | Global edge | Platform team |
| Kong (mTLS, JWT) | Per-region | Platform team |
| Vault (secrets) | Global Control Plane (EU) + read-копии | Security team |
| SPIRE (mTLS certs) | Per-region | Platform team |
| Audit / SIEM | Per-region (Loki) + агрегаты в EU | Security team |
| Breach notification | Per-region | DPO |

## Альтернативы (сводно)

1. **Единый глобальный WAF на своём железе.** Отклонено: ops-нагрузка, нет глобального edge, latency.
2. **Anycast вместо GeoDNS + GLB.** Отклонено в ADR-003 (не health-aware, residency).
3. **Только Cloudflare без Yandex AntiDDoS.** Отклонено: санкционные риски в RU.
4. **Только Yandex WAF / AntiDDoS.** Отклонено: только RU, не покрывает SEA/LatAm/MENA.
5. **mTLS на GLB.** Отклонено: клиенты не имеют сертификатов, только JWT.
6. **Per-region Vault без master.** Отклонено: рассинхрон ключей → невозможно валидировать JWT cross-region.
7. **Ручное управление сертификатами.** Отклонено: не масштабируется, риск истечения.
8. **Splunk / Datadog для SIEM.** Отклонено: дорого, vendor lock-in.
9. **Отдельный SIEM (QRadar / ArcSight).** Отклонено: overkill, дорого.

## Компромиссы

- **Cloudflare WAF** — vendor lock-in. Митигация: стандартные OWASP-правила, возможность миграции на Yandex WAF в RU.
- **Cloudflare DDoS + Yandex AntiDDoS** — 2 vendor'а. Митигация: Cloudflare — глобально, Yandex — только RU (регуляторное требование).
- **Network Firewall per-region** — ops-нагрузка ×4. Митигация: IaC (Terraform), единые базовые политики.
- **Vault в EU** — политическая нейтральность, но формально вне 4 регионов. Митигация: только non-PII secrets; read-копии в регионах.
- **mTLS на Kong, не на GLB** — доп. hop для service-to-service. Митигация: Kong близко к сервисам, latency минимальна.
- **Audit logs per-region** — дублирование инфраструктуры Loki. Митигация: shared стек, но данные per-region (residency).
- **Breach notification 24 ч** — жёсткий срок (ADR-005). Митигация: SIEM + алерты (Task2 ADR-006), runbook.
- **Zero-trust** — сложнее в реализации и эксплуатации. Митигация: стандартные паттерны (JWT, mTLS, ACL), автоматизация.

## Ссылки на предыдущие решения

- Kong, BFF, сервисы — **Task1 C2 Final**.
- 4 региона, регуляторные зоны — **ADR-001 Task3**.
- Global Control Plane (Vault), cross-region PII — **ADR-002 Task3**.
- Геомаршрутизация, WAF/DDoS/Firewall/GLB в C2_Georouting — **ADR-003 Task3**.
- Failover, reconciliation — **ADR-004 Task3**.
- Комплаенс, audit, breach notification — **ADR-005 Task3**.
- OTel / Loki / Prometheus / Alertmanager — **Task2 ADR-006**.
- Kafka ACL (1 producer = 1 domain) — **Task2 ADR-002**.
