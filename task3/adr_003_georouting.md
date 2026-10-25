# ADR-003: Геомаршрутизация пользователей к ближайшему региону

## Контекст

ADR-001 материализовал 4 региона (`region-ru`, `region-sea`, `region-latam`,
`region-mena`), ADR-002 зафиксировал репликацию данных внутри регуляторных
зон и placement Global Control Plane. Задание Task3 требует «механизм
геомаршрутизации пользователей к ближайшему региону».

Уже зафиксировано:
- Task1 C2 Final: клиенты → Kong → BFF → сервисы. **Однорегиональная**
  схема, без edge-слоя.
- Task2 C2 EDA: EDA-ландшафт, **один** Kafka-кластер (упрощение).
- ADR-001: 4 региона, регуляторные зоны, p95 ≤ 100 мс (assumption).
- ADR-002: Global Identity Directory (`user_id → home_region`, без PII) —
  для роуминга; cross-region PII не реплицируется, обмен через API.

**Задача:** спроектировать ingress-слой, который (а) направляет
пользователя в ближайший здоровый регион, (б) обеспечивает защиту
(WAF, DDoS, Firewall), (в) поддерживает роуминг (пассажир путешествует
между регионами), (г) не нарушает residency.

## Требования

1. Направлять пользователя в **ближайший здоровый** регион
   (p95 ≤ 100 мс для ingress).
2. Поддерживать **автоматический failover** между регионами
   (GLB health checks, RTO ≤ 45 мин, ADR-002).
3. Обеспечить **безопасность ingress**: WAF (OWASP Top-10), DDoS L3-L7,
   Network Firewall, mTLS.
4. Поддерживать **роуминг пассажиров** (пассажир из RU в SEA) через API
   к домашнему региону, без cross-region репликации PII (ADR-002).
5. **Водитель не роумится** — работает в своём регионе
   (assumption, зафиксировано в ADR-001).
6. Не дублировать Task1 C2 (Kong, BFF, сервисы) — только новый edge-слой.
7. Все новые компоненты — **красным** на диаграмме.

## Решение

### 1. Слои ingress (сверху вниз)

```
Клиент (Passenger / Driver / Corporate / Admin / Web)
│
▼
[1] GeoDNS (Cloudflare DNS)              ← региональная маршрутизация по IP
│
▼
[2] WAF (Cloudflare)                      ← OWASP Top-10, rate limiting, bot protection
│
▼
[3] DDoS Protection (L3-L7)               ← Cloudflare / Yandex AntiDDoS
│
▼
[4] Network Firewall (per-region)         ← IP allow/deny, geo-blocking (при необходимости)
│
▼
[5] Global Load Balancer (L7, health-checked) ← active-active, weight-based
│
▼
[6] Regional Ingress (Kong, per-region)   ← mTLS, JWT validation, канареечные релизы
│
▼
[7] BFF (Passenger / Driver / Corporate / Admin / Web) — из Task1
│
▼
[8] Микросервисы — из Task1/Task2
```

**Ключевые решения:**
- **GeoDNS + GLB** (не Anycast). Anycast даёт «ближайший по BGP», но
  не учитывает health региональных кластеров и бизнес-правила
  (например, «RU-пользователь всегда в RU, даже если SEA ближе по сети»).
  GeoDNS + GLB — управляемее.
- **WAF и DDoS — на edge (Cloudflare), Firewall — per-region**
  (Network Firewall в каждом регионе для региональных политик).
- **GLB — L7, active-active** для всех 4 регионов, weight-based.
  Health checks к каждому региональному ingress.
- **Regional Ingress = Kong** (уже существует из Task1 C2 Final, но
  теперь per-region).
- **Всё новое — красным** на `C2_Georouting.puml`.

### 2. Геомаршрутизация (алгоритм)

**Для обычного пользователя (пассажир, водитель, корп-менеджер, админ):**

1. Клиент делает DNS-запрос → GeoDNS возвращает IP ближайшего
   **здорового** региона.
2. Трафик идёт через WAF → DDoS → Firewall → GLB → Regional Ingress →
   Kong → BFF.
3. **Driver app:** GeoDNS всегда возвращает **домашний регион** водителя
   (по `driver_id` → home_region, кэш в приложении). Водитель
   не роумится.
4. **Passenger app:** GeoDNS возвращает **ближайший** регион по IP.
   Домашний регион определяется через Global Identity Directory
   (см. роуминг ниже).
5. **Corporate web / Admin web / Web frontend:** GeoDNS возвращает
   ближайший регион; корп-данные — через API к домашнему региону
   тенанта.

**Для роуминга пассажиров (пассажир из RU в SEA):**

1. Клиент в SEA → GeoDNS → SEA.
2. SEA BFF получает запрос → смотрит **Global Identity Directory**
   (`user_id → home_region = region-ru`).
3. Если операция **локальная** (создание поездки в SEA, поиск водителя
   в SEA) — выполняется в SEA. Локальные транзакции SEA-PII остаются
   в SEA (см. раздел 5 про PII поездки).
4. Если операция **требует данных домашнего региона** (профиль, история,
   платёжный метод, KYC) — SEA BFF делает **API-вызов** к region-ru.
   Данные не реплицируются (ADR-002), только читаются по запросу.
5. **Edge-кэш** (в SEA) только для **non-PII**:
   - `user_id → home_region` (из Global Identity Directory);
   - `payment_method_token` (opaque, PSP хранит карту у себя);
   - `hashed(email)` / `hashed(phone)` — только для дедупликации
     при логине, **не для отображения и не для звонков водителю**.
   
   **Email / phone в открытом виде не кэшируются.** TTL 5–15 мин.
   
   Для операций, требующих email/phone в открытом виде (водитель звонит
   пассажиру, отправка чека) — **API-вызов к домашнему региону
   в момент операции** (не кэш). Латентность приемлемая (секунды).
6. Если region-ru недоступен — **degraded mode** в SEA: локальные
   операции работают (создание поездки в SEA, поиск водителя),
   операции с домашними данными — 503 с retry. Алерт.

**Водитель не роумится:** работает только в своём регионе, при попытке
пересечь границу — приложение требует сменить регион вручную
(assumption, вне scope Task3).

### 3. Health checks и failover (механизм GLB)

- **GLB → Regional Ingress** health check каждые **10 сек**, таймаут
  3 сек, 3 фейла подряд = unhealthy.
- **Unhealthy регион** выводится из ротации GLB. Трафик → следующий
  ближайший здоровый (weight-based).
- **Полная потеря региона** (зона 1 + зона 2): GLB выводит регион
  из ротации → трафик в другие регионы. Для роуминга — degraded mode
  (см. выше). Failover-план — **ADR-004**.
- **Health check эндпоинт** — `/health` на Regional Ingress (Kong),
  проверяет: Kong up, Keycloak reachable, БД reachable, Kafka reachable.

### 4. Защита ingress (слой за слоем)

| Слой | Инструмент | Что защищает |
|---|---|---|
| **DNS** | Cloudflare DNS (GeoDNS) | От DNS-атак, обеспечивает геомаршрутизацию |
| **WAF** | Cloudflare WAF | OWASP Top-10 (SQLi, XSS, RCE), rate limiting (per-IP, per-user), bot protection, custom rules |
| **DDoS L3-L7** | Cloudflare / Yandex AntiDDoS | Volumetric, protocol, application-layer DDoS |
| **Network Firewall** | Cloud Firewall (per-region) | IP allow/deny, geo-blocking (например, «только SEA IP в region-sea» — опционально), egress filtering |
| **GLB** | L7 GLB | Health-checked, active-active, TLS termination |
| **Regional Ingress (Kong)** | Kong (Task1) | mTLS, JWT validation, rate limiting per-route, канареечные релизы |

**Всё, кроме Kong, — новое в Task3 (красным).** Kong уже существует
(Task1 C2), но теперь per-region — это уточнение, не новый компонент.

### 5. PII в региональных данных: что где лежит

**Ключевое уточнение:** не все региональные данные — non-PII. Разделяем:

| Хранилище | PII? | Что лежит | Регуляторика |
|---|---|---|---|
| **Global Identity Directory** | ❌ Нет | Только `user_id (UUID) → home_region (код)` | Non-PII (routing-данные) |
| **Regional ClickHouse** | ❌ Нет | Только агрегаты (хэшированные): `trips_per_hour`, `avg_price_by_zone`, `surge_avg` | Non-PII (агрегаты) |
| **Regional Edge Cache** | ❌ Нет | `home_region`, `payment_method_token`, `hashed(email/phone)` | Non-PII |
| **Regional Booking DB** | ✅ **Да** | Полные данные поездки: `passenger_id`, **имя**, **телефон**, pickup/dropoff координаты, сумма, водитель, время | **PII поездки**, собранный в регионе оказания услуги |
| **Regional Payments DB** | ✅ **Да** | Платёжные транзакции, `passenger_id`, суммы, PSP-токены | **PII поездки** |
| **Regional Driver Profile DB** | ✅ **Да** | KYC-документы, паспортные данные водителя | **PII водителя** (резидент региона) |
| **Regional Kafka** | ✅ Частично | События содержат `passenger_id`, координаты, суммы | PII транзитом, retention 14–30 дней |

**Cross-jurisdiction кейс (фиксируется в ADR-005):**
Если пассажир-резидент RU заказывает поездку в SEA:
- `passenger_id` — ссылка на RU-профиль (не PII);
- имя / телефон / координаты **этой поездки** — **SEA-PII**, собранные
  в SEA при оказании услуги, хранятся в SEA Booking DB, регулируются
  PDPA SG;
- это **не репликация** RU-PII в SEA, а **обработка в территории
  оказания услуги**. 152-ФЗ не нарушается (данные собраны за пределами
  РФ), PDPA SG соблюдается (согласие пассажира при заказе поездки).

**Cross-region PII-обмена по-прежнему нет:** email/phone/история
домашнего региона не реплицируются в SEA. Они читаются **по запросу**
через Roaming API в момент операции (не кэшируются в открытом виде).

### 6. Почему GeoDNS + GLB, а не Anycast

| Критерий | GeoDNS + GLB | Anycast |
|---|---|---|
| Health-aware | ✅ GLB видит здоровье регионов | ❌ BGP не знает про здоровье |
| Бизнес-правила | ✅ «RU всегда в RU» | ❌ Только «ближайший по сети» |
| Управляемость | ✅ Weight-based, canary | ❌ Сложно управлять |
| Latency | ✅ Хорошая (DNS-кэш) | ✅ Лучшая (BGP) |
| Стоимость | ✅ Дешевле | ❌ Дороже (свой AS, BGP) |
| Residency | ✅ Полный контроль | ❌ Сложно гарантировать |

**Выбор: GeoDNS + GLB.** Anycast отвергнут: не health-aware, сложно
соблюсти residency.

### 7. Диаграмма

`C2_Georouting.puml` — отдельный файл Task3. На ней:
- **Сверху** — клиенты (Passenger, Driver, Corporate, Admin, Web)
  из Task1 C2.
- **Новый edge-слой (красный):** GeoDNS → WAF → DDoS → Network
  Firewall → GLB.
- **4 региона**, каждый с Regional Ingress (Kong, синий — существует)
  → BFF (синий) → сервисы (синие).
- **Global Identity Directory** (из ADR-002) — для роуминга.
- **Health checks** от GLB к каждому Regional Ingress (красные стрелки).
- **Роуминг:** SEA BFF → API → region-ru BFF (стрелка «roaming API call»).
- **Edge Cache** (в каждом регионе, non-PII only) — красный.
- **Легенда:** красный — новое, синий — существующее из Task1/Task2.

### 8. Что НЕ входит в это решение

- Failover-план и таблица сценариев — **ADR-004**.
- Регуляторные детали — **ADR-005**.
- Security-overlay (детально WAF/Firewall) — **ADR-006**.
- Внутренняя маршрутизация Kong → BFF → сервисы — Task1 C2.
- Kafka-маршрутизация — Task2 ADR-002 (Kafka изолирована per-region,
  не участвует в ingress).
- CDN для статики — вне scope (предполагается Cloudflare CDN,
  детали не фиксируем).

## Альтернативы

1. **Anycast без GLB.** Отклонено: не health-aware, сложно соблюсти
   residency.
2. **Только GeoDNS без GLB.** Отклонено: нет health-aware failover,
   нет active-active.
3. **Единый глобальный API Gateway (без региональных).** Отклонено:
   единая точка отказа, latency, residency.
4. **Клиент сам выбирает регион (hardcode в приложении).** Отклонено:
   не масштабируется, не учитывает health, ломается при failover.
5. **Роуминг через репликацию PII.** Отклонено: нарушает residency
   (ADR-002).
6. **Роуминг через глобальную БД (без home_region).** Отклонено:
   cross-border PII, residency.
7. **Yandex Smart Web Security вместо Cloudflare WAF.** Отклонено:
   Cloudflare глобально (4 региона), Yandex — только RU. Единый WAF
   проще.
8. **Driver roaming (водитель работает в любом регионе).** Отклонено:
   усложняет граф данных (driver profile, KYC, payouts — cross-border),
   вне scope Task3.
9. **Кэшировать email/phone в открытом виде в региональном edge-кэше.**
   Отклонено: хранит PII домашнего региона в чужой регуляторной зоне,
   нарушает residency. Компромисс — хэши только для дедупликации,
   открытые значения — через API к домашнему региону в момент операции.

## Компромиссы

- **GeoDNS + GLB, не Anycast** — latency чуть выше идеальной (DNS-кэш),
  но health-aware и residency-friendly. Митигация: TTL DNS 60 сек,
  GLB L7.
- **Роуминг через API** — latency для операций с домашними данными
  (cross-ocean). Митигация: edge-кэш для non-PII, Roaming API только
  для операций, требующих открытых PII (звонок водителя, чек).
- **Email/phone в открытом виде — не в кэше** — каждая операция,
  требующая открытых данных, идёт в домашний регион. Митигация:
  для звонка водителя — латентность секунды, приемлемо;
  для отображения профиля — открытые данные не нужны (имя можно
  кэшировать как `hashed`, а отображать только после API-вызова).
- **Degraded mode при потере домашнего региона** — пассажир в SEA
  не может получить профиль из RU. Митигация: локальные операции
  работают; алерт; failover RU (ADR-004) восстанавливает.
- **Водитель не роумится** — assumption, ограничивает гибкость.
  Митигация: явно зафиксировано, вне scope Task3.
- **WAF на edge (Cloudflare)** — vendor lock-in. Митигация:
  стандартные правила OWASP, возможность миграции на Yandex WAF
  в RU-регионе (если потребуется).
- **Network Firewall per-region** — дополнительная ops-нагрузка.
  Митигация: IaC (Terraform), единые политики.
- **p95 ≤ 100 мс — assumption** — достижимо при GeoDNS + GLB +
  региональных кластерах. Митигация: нагрузочное тестирование,
  мониторинг.
- **Cross-jurisdiction PII поездки** — пассажир-резидент RU в SEA
  порождает SEA-PII. Это не репликация, а обработка в территории
  оказания услуги; фиксируется в ADR-005. Митигация: явное
  согласие пассажира при заказе поездки.

## Ссылки на предыдущие решения

- Kong, BFF, сервисы — **Task1 C2 Final**.
- 4 региона, регуляторные зоны — **ADR-001 Task3**.
- Global Identity Directory, cross-region API — **ADR-002 Task3**.
- Failover, таблица сценариев — **ADR-004 Task3**.
- Комплаенс (cross-jurisdiction PII поездки) — **ADR-005 Task3**.
- Security-overlay — **ADR-006 Task3**.