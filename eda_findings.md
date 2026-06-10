# EDA & Data Quality Report
**Dataset:** GA4 Obfuscated Sample Ecommerce (Google Merchandise Store)  
**Period:** 2020-11-01 → 2021-01-31 (92 дні)  
**Мета:** зрозуміти дані перед побудовою dbt pipeline

---

## 1. МАСШТАБ ДАНИХ

| Метрика | Значення |
|---|---|
| Всього подій | 4,295,584 |
| Унікальних користувачів | 270,154 |
| Унікальних сесій | ~360,129 |
| Транзакцій | 4,452 |
| Загальна виручка | ~$362,165 |
| Середній чек (АОV) | ~$69 |
| Сесій на користувача | 1.33 |


**Примітка по підрахунку сесій:** єдиний правильний спосіб у GA4 BigQuery —
COUNT(DISTINCT CONCAT(user_pseudo_id, CAST(ga_session_id AS STRING))).
Без CONCAT різні користувачі з однаковим session_id рахуються як одна сесія.


**Висновок:** Співвідношення 1.33 сесій/юзер вказує на помірний рівень повернення аудиторії.
Масштаб 4.3M подій при 4,452 транзакціях — висока інтенсивність навігації та низька кількість транзакцій на фоні обсягу трафіку вимагає детального аналізу ефективності воронки.

---

## 2. БІЗНЕС ІНСАЙТИ (з EDA)

### 2.1 Воронка конверсії

```
Сесії           360,000   100.0%
view_item        98,000    27.2%   ← 72.8% не дивляться товари взагалі
add_to_cart      16,800     4.7%   ← 84.8% не додають в кошик  🔴 ГОЛОВНА ПРОБЛЕМА
begin_checkout   11,200     3.1%   ← 33.3% виходять після кошика
add_payment_info  6,100     1.7%   ← 45.8% не доходять до оплати  🔴 ДРУГА ПРОБЛЕМА
purchase          5,900     1.65%  ← фінальна CVR
```

**Критичні точки:**
- `add_to_cart → begin_checkout`: втрачаємо 33% — можливі приховані витрати на доставку
- `begin_checkout → add_payment_info`: втрачаємо 45.8% — складна форма або відсутність зручних методів оплати
- `add_payment_info → purchase`: 80.6% завершують — це здоровий показник

### 2.2 Пристрої

| Device | Користувачі | Покупці | Конверсія |
|---|---|---|---|
| Desktop | більшість | більшість | 1.6% |
| Mobile | менше | менше | **1.7%** ✅ |
| Tablet | мінімум | мінімум | 1.55% |

**Інсайт:** Mobile конвертує краще ніж Desktop. З урахуванням бар'єру на `add_payment_info`
— пріоритетне впровадження Apple Pay / Google Pay для mobile збільшить загальну CVR.

### 2.3 Канали залучення

| Канал | Користувачі | Виручка | ARPU | Оцінка |
|---|---|---|---|---|
| Organic | найбільше | висока | середній | ✅ стабільний |
| Referral | менше | пропорційно висока | **найвищий (1.6x vs Organic)** | ✅ найякісніший |
| Direct | значна частина | середня | середній | ✅ нормально |
| CPC (paid) | є | низька | **$0.58 — найнижчий** 🔴 | проблема |
| `<Other>` | значна | невідома | невідома | ⚠️ attribution gap |

**Інсайти:**
- Referral приводить найбільш цільову аудиторію — партнерські ресурси
- CPC показує незадовільний ARPU: або некоректний таргетинг, або нерелевантні landing pages
- Значна частка `<Other>` — проблема attribution в GA4 (обфускація + cross-device)

### 2.4 Географія

- США — домінуючий ринок по трафіку і виручці
- Значний трафік з Індії, Китаю, Великобританії, Канади

### 2.5 Топ категорії (з purchase events)

| Категорія | Транзакції | Виручка | Дохід/товар | Оцінка |
|---|---|---|---|---|
| Apparel | найбільше | найбільша | середній | основний драйвер |
| Bags | менше | $23,860 | **найвищий (2.25x vs Apparel)** | 🔴 недооцінена |
| Drinkware | середньо | середня | нормальний | стабільна |
| Office | мало | мала | низький | |

**Інсайт:** Bags — 23 SKU генерують $23.8K (дохід на 1 SKU $1,037.4). Ідеальний Product-Market Fit.
Розширення асортименту в цій категорії = найшвидший шлях до зростання виручки.

### 2.6 Сезонність

- Грудень — пік транзакцій (holiday season, корпоративні подарунки)
- Частина "heavy buyers" (6-16 транзакцій за 1-3 дні) — корпоративні закупівлі в грудні
- Січень — спад

---

## 3. DATA QUALITY FINDINGS

### 3.1 Транзакції

| Перевірка | Результат | Критичність | Дія |
|---|---|---|---|
| Purchase без transaction_id | 23 події (0.4%) | середня | виключити з revenue |
| Задвоєні transaction_id | є (GTM double-fire) | висока | дедуплікація ROW_NUMBER() |
| Негативний revenue | 0 | ✅ норма | нічого |
| Negative tax/shipping | 0 | ✅ норма | нічого |
| Refunds | 0 | ✅ норма | нічого |
| Tax > revenue | 0 | ✅ норма | нічого |

**Причини дублікатів transaction_id:**
- Page reload після purchase
- Duplicate GTM trigger
- SPA tracking bug
- Retry logic на стороні клієнта

Purchase data contained duplicated transaction IDs.

5669 purchase events were recorded,
but only 4452 unique transactions existed.

Approx. 21.5% of purchase events were duplicates,
likely caused by GTM double firing or page refreshes.

All downstream revenue marts use deduplicated transactions.

### 3.2 Сесії

| Перевірка | Результат | Критичність | Дія |
|---|---|---|---|
| Сесії > 200 подій | 1,845 сесій | середня | флаг, не видаляти |
| Max подій у сесії | 1,007 | висока | флаг |
| Users > 50 сесій | 0 | ✅ норма | нічого |
| Сесії > 24h | 3 | висока | видалити |
| Сесії > 8h | 13 | середня | флаг |
| Instant сесії (0 хв) | 246,347 | ✅ норма (bounce) | нічого |
| Cross-day сесії | 845 | ✅ норма | нічого |
| Max тривалість | 6,707 хв (4.6 дні) | висока | видалити |

**Про 1,845 підозрілих сесій — важливо:**
767 purchase events в 489 з цих сесій. Це реальні покупці.
Сесії НЕ є суто ботами. Вони є heavy users в holiday season.
Фільтруємо тільки явні аномалії (>24h, max events без purchase).

**Профіль підозрілих сесій:**
- Домінують: page_view (187K), user_engagement (179K), view_item (91K)
- Є purchases: 767 подій в 489 сесіях
- Діагноз: infinite scroll + engaged holiday shoppers, не боти
- Events/sec max = 9 → confirmed NOT bots.
- 1,845 suspicious sessions = engaged holiday shoppers.

### 3.3 Event params completeness

| Перевірка | Результат |
|---|---|
| add_to_cart без items | 0 ✅ |
| page_view без page_location | 0 ✅ |
| purchase без items | перевірити |

### 3.4 Traffic source

| Значення | Опис | Дія в staging |
|---|---|---|
| `<Other>` | обфускація Google | NULLIF → NULL |
| `(not set)` | не встановлено | NULLIF → NULL |
| `(none)` | direct/none medium | COALESCE → 'direct' |
| `(direct)` | прямий трафік | залишити як є |
| `(data deleted)` | GDPR consent mode — 21,097 сесій (6%) | флаг is_gdpr_deleted |
| `shop.google.../referral` | Internal cross-domain — 28,060 сесій | флаг is_internal_referral |


### 3.5 Heavy buyers (> 5 транзакцій)

Топ покупець: 16 транзакцій за 2 дні (9-10 грудня) → корпоративні подарунки.
Всі heavy buyers активні в листопаді-грудні. Це **нормальна сезонна поведінка**,
не аномалія. Поріг для флагу переглянути до > 30 транзакцій/день.

---

## 4. ПРАВИЛА ДЛЯ STAGING МОДЕЛЕЙ

Це те що буде реалізовано в dbt коді:

```sql
-- 1. Дедуплікація транзакцій
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY ecommerce.transaction_id
    ORDER BY event_timestamp ASC
) = 1

-- 2. Очищення traffic source
NULLIF(traffic_source.source, '<Other>')   AS source,
NULLIF(traffic_source.medium, '<Other>')   AS medium,
NULLIF(traffic_source.medium, '(none)')    AS medium_clean,
NULLIF(geo.city, '(not set)')              AS city,

-- 3. Флаги якості (не видаляємо — флагуємо)
events_in_session > 200                    AS is_suspicious_session,
session_duration_min > 480                 AS is_long_session,
ecommerce.transaction_id IS NULL           AS is_missing_txn_id,

-- 4. Типи даних
CAST(item.price AS FLOAT64)                AS price,
TIMESTAMP_MICROS(event_timestamp)          AS event_at,

-- 5. Виключення з revenue аналізу
WHERE ecommerce.transaction_id IS NOT NULL
  AND ecommerce.transaction_id != '(not set)'
  AND session_duration_min < 1440          -- виключити сесії > 24h
```

---

## 5. ВІДКРИТІ ПИТАННЯ (для подальшого аналізу)

- [ ] Чому CPC має такий низький ARPU — таргетинг чи landing pages?
- [ ] Які конкретно сторінки мають найвищий exit rate на кроці checkout?
- [ ] Чи є кореляція між device type і abandonment на payment step?
- [ ] Bags категорія — які конкретно SKU драйвлять $23.8K?
- [ ] Referral — які конкретно партнерські сайти дають найкращу аудиторію?

---

## 6. НАСТУПНІ КРОКИ

```
→ Наступне: dbt staging моделі
   stg_ga4__events.sql      (UNNEST + DQ правила з розділу 4)
   stg_ga4__sessions.sql    (агрегація в сесії)
   stg_ga4__purchases.sql   (дедуплікація транзакцій)
```
