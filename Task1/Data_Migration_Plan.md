# План миграции данных при переходе на микросервисную архитектуру

## Общие принципы
- Применяется **Strangler Fig Pattern** через API Gateway и feature flags
- Основная стратегия — **Dual Writes + CDC** (Debezium) для всех критичных доменов
- Cutover и откат — через feature flags в API Gateway (Kong)
- Все новые сервисы пишут в собственные схемы/БД (Database per Service — поэтапно)
- Используются идемпотентные операции и reconciliation jobs

## Этапы выноса и стратегии миграции

### 1. Notification Service
- **Таблицы**: `notifications`, `notification_templates`, `delivery_logs`
- **Стратегия**: Dual Writes + CDC (самая безопасная)
- **Этапы**:
  1. Создаём новую схему `notification` в отдельной БД
  2. Деплоим сервис -> он пишет и в старую таблицу, и в новую (dual write)
  3. Запускаем Debezium-коннектор -> догоняет исторические данные и ловит изменения
  4. 2 недели параллельной работы + сравнение (shadow traffic)
  5. Cutover: в API Gateway включаем флаг `use_new_notification=true`
  6. Откат: выключаем флаг -> трафик мгновенно возвращается на монолит
- **Риски и минимизация**: дубли писем -> идемпотентность по `notification_id`

### 2. Analytics Service 
- **Таблицы**: `driver_events`, `user_events`, `funnel_events`,
- **Стратегия**: Big Bang / One-time backfill
- **Этапы**:
  1. Новый сервис начинает писать все события только в ClickHouse
  2. За одну ночь делаем полный бэкфилл из Kafka-логов (Yandex Data Transfer / Kafka Connect -> ClickHouse)
  3. Cutover: включаем флаг `use_new_analytics=true` -> монолит перестаёт писать в старые таблицы
- **Риски**: нулевые (данные только на запись, потерять нельзя)

### 3. Fraud Service   
- **Таблицы**: `fraud_signals`, `scammers`, `risk_scores`, `suspicious_logins`, 
- **Стратегия**: Dual Writes + CDC
- **Этапы**:
  1. Создаём новую схему `fraud` в отдельной БД
  2. Деплоим сервис -> синхронизация небольшими порциями
  3. В течении 3 недель будет полный переезд
  4. Cutover: в API Gateway включаем флаг `use_new_fraud=true`
  5. Откат: выключаем флаг -> трафик мгновенно возвращается на монолит
- **Риски и минимизация**: дубли сигналов -> идемпотентность по `signal_id`

### 4. Geography Service 

- **Таблицы**: `zones`, `zone_type`, `address_cache`, `zone_stats`, 
- **Стратегия**: Dual Writes + CDC (самая безопасная)
- **Этапы**:
  1. Создаём новую схему `geography`
  2. Деплоим сервис -> данные переносяться из станой в новую БД, если есть новые записи они пишутся и в старую, и в новую (dual write)
  3. 2 недели параллельной работы + сравнение (shadow traffic)
  4. Cutover: в API Gateway включаем флаг `use_new_geography=true`
  5. Откат: выключаем флаг -> трафик мгновенно возвращается на монолит
- **Риски и минимизация**: дубли зон -> идемпотентность по `zone_id`

### 5. Pricing Service 

- **Таблицы**: `tariff_plans`, `tariff_modifiers`, `price_rules`, `dynamic_pricing_cache`, 
- **Стратегия**: Dual Writes + CDC (тарифы меняются редко)
- **Этапы**:
  1. Создаём новую схему `pricing` в отдельной БД
  2. Деплоим сервис -> останаваливаем систему, мигрируем данные все
  3. 1 день на миграцию + сравнение (shadow traffic)
  4. Cutover: в API Gateway включаем флаг `use_new_pricing=true`
  5. Откат: выключаем флаг -> трафик мгновенно возвращается на монолит
- **Риски и минимизация**: расхождение цен -> shadow traffic + сравнение ответов 2 недели
  
### 6. Driver Service 

- **Таблицы**: `drivers`, `driver_documents`, `driver_ratings`
- **Стратегия**: Dual Writes + CDC (самая безопасная)
- **Этапы**: стандартные (схема `driver`, dual write, `use_new_driver=true`, Debezium, 3–4 недели)
- **Риски и минимизация**: необходимы дополнительные рисурсы на развертывания новой версии паралельно

### 7. Payments Service 

- **Таблицы**: `payments`, `payouts`, `funnel_events`, `trip_events`, 
- **Стратегия**: Dual Writes + CDC + Canary Release
- **Этапы**:
  1. Создаём новую схему `payment` в отдельной БД
  2. Деплоим сервис -> он фильтрует отправляет часть пользователей на новый сервис для проверки работы сервиса, а также постепенного перехода
  3. 1 неделю тестов на 5% пользователей, потом на 2 неделе 25%, на 4 неделе переходем полностью
  4. Cutover: в API Gateway включаем флаг `use_new_payments=true`
  5. Откат: выключаем флаг -> трафик мгновенно возвращается на монолит
- **Риски и минимизация**: необходимы дополнительные рисурсы на развертывания новой версии паралельно, есть риск замедления релиза

### 8. Booking Service 

- **Таблицы**: `trips`, `trip_status`, `trip_routes`,
- **Стратегия**: Strangler Fig + Canary + постепенный backfill
- **Этапы**:
  1. Начинаем с 1 % новых поездок (по городу)
  2. Dual write + Debezium 3–6 месяцев
  3. Постепенно повышаем процент: 1 % -> 5 % -> 25 % -> 50 % -> 100 %
  4. Backfill истории по регионам (по 1–2 крупных города за раз)
  5. Финальный cutover — отключаем запись в старые таблицы
- **Риски и минимизация**: ризк минимальный

# План миграции данных при переходе на микросервисную архитектуру

## Общие принципы
- Применяется **Strangler Fig Pattern** через API Gateway и feature flags
- Основная стратегия — **Dual Writes + CDC** (Debezium) для всех критичных доменов
- Cutover и откат — через feature flags в API Gateway (Kong/Traefik)
- Все новые сервисы пишут в собственные схемы/БД (Database per Service — поэтапно)
- Используются идемпотентные операции и reconciliation jobs

## Этапы выноса и стратегии миграции

### 1. Notification Service
- **Таблицы**: `notifications`, `notification_templates`, `delivery_logs`, `user_notification_settings`
- **Объём**: ~15 млн строк, 80 ГБ
- **Стратегия**: Dual Writes + CDC
- **Этапы**:
  1. Создаём схему `notification` в отдельной PostgreSQL
  2. Деплой сервиса -> включаем dual write
  3. Запускаем Debezium -> догоняет историю (~2 часа)
  4. 2 недели shadow traffic + сравнение
  5. Cutover: `use_new_notification=true`
  6. Откат: выключение флага -> мгновенно
- **Риски**: дубли уведомлений -> идемпотентность по `notification_id`
- **Инструменты**: Debezium, Liquibase, feature flags

### 2. Analytics Service
- **Таблицы/потоки**: все сырые события -> ClickHouse (`trip_events`, `driver_events`, `user_events`, `clickstream`)
- **Объём**: > 50 млрд событий
- **Стратегия**: Big Bang + One-time backfill
- **Этапы**:
  1. Новый сервис пишет события только в ClickHouse
  2. За одну ночь — полный бэкфилл из Kafka-логов через Yandex Data Transfer
  3. Cutover: `use_new_analytics=true` -> монолит перестаёт писать в старые таблицы
- **Риски**: нулевые (данные только на запись)
- **Инструменты**: Yandex Data Transfer, Kafka Connect -> ClickHouse

### 3. Fraud Service
- **Таблицы**: `fraud_signals`, `risk_scores`, `suspicious_logins`, `device_fingerprints`, `fraud_incidents`
- **Стратегия**: Dual Writes + CDC
- **Этапы**: стандартные (схема `fraud`, dual write, Debezium, 3 недели, cutover)
- **Риски**: дубли сигналов -> идемпотентность по `signal_id`
- **Инструменты**: Debezium, shadow traffic

### 4. Geography Service
- **Таблицы**: `zones`, `zone_polygons`, `surge_multipliers`, `restricted_areas`, `address_cache`
- **Стратегия**: Dual Writes + CDC
- **Этапы**:
  1. Схема `geography`
  2. Dual write + Debezium
  3. 2 недели параллельно
  4. Cutover: `use_new_geography=true`
- **Риски**: дубли зон -> идемпотентность по `zone_id`
- **Инструменты**: Debezium, Redis Geo для горячих данных

### 5. Pricing Service
- **Таблицы**: `tariff_plans`, `tariff_modifiers`, `price_rules`, `surge_multipliers_history`
- **Стратегия**: Dual Writes + CDC (тарифы меняются редко -> идеальный кандидат)
- **Этапы**:
  1. Создаём новую схему `pricing` в отдельной БД
  2. Деплоим сервис -> включаем dual write (пишем и в старую, и в новую БД)
  3. Запускаем Debezium -> догоняет историю (небольшой объём)
  4. 2 недели параллельной работы + shadow traffic и сравнение цен
  5. Cutover: в API Gateway включаем флаг `use_new_pricing=true`
  6. Откат: выключаем флаг -> трафик мгновенно на монолит
- **Риски и минимизация**: расхождение цен -> shadow traffic + сравнение ответов 2 недели
- **Инструменты**: Debezium, Redis (кэш цен)

### 6. Driver Service
- **Таблицы**: `drivers`, `driver_documents`, `driver_licenses`, `driver_status_history`, `driver_ratings`, `driver_balance`
- **Объём**: ~20–50 млн строк
- **Стратегия**: Dual Writes + CDC
- **Этапы**: стандартные (схема `driver`, dual write, Debezium, 3–4 недели)
- **Риски**: рассинхрон статуса водителя -> reconciliation job каждые 15 мин
- **Инструменты**: Debezium, Redis (координаты и статус)

### 7. Payments Service
- **Таблицы**: `payments`, `payouts`, `refunds`, `driver_earnings`, `payment_methods`, `transactions_log`
- **Стратегия**: Dual Writes + CDC + Canary Release
- **Этапы**:
  1. Схема `payments` + строгий аудит
  2. Начинаем с 5 % транзакций -> 25 % -> 100 % (4 недели)
  3. Обязательный reconciliation всех сумм
  4. Cutover: `use_new_payments=true`
- **Риски**: финансовые расхождения -> double-entry bookkeeping + ежедневный аудит
- **Инструменты**: Debezium, Istio (трафик-сплит), reconciliation jobs

### 8. Booking Service (ядро системы)
- **Таблицы**: `trips`, `trip_status_history`, `trip_routes`, `trip_cancellations`, `active_trips`, `trip_payments`
- **Объём**: > 2 млрд строк, > 15 ТБ, рост 5–15 млн/день
- **Стратегия**: Strangler Fig + Canary по процентам/регионам + постепенный backfill
- **Этапы**:
  1. Начинаем с 1 % новых поездок (по гео-хэшу или тестовому городу)
  2. Dual write + Debezium 4–6 месяцев
  3. Постепенное увеличение: 1 % -> 5 % -> 25 % -> 50 % -> 100 %
  4. Backfill истории по регионам (по 1–2 крупных города за раз)
  5. Финальный cutover — отключение записи в старые таблицы
- **Откат**: в любой момент — выключение флага -> 100 % трафика на монолит
- **Риски**: рассинхрон статусов -> ежедневный reconciliation job + алерты при расхождении > 0.01 %
- **Инструменты**: Debezium, процентный роутинг в API Gateway, Yandex Data Transfer

## Общие инструменты и практики
- **Debezium + Kafka Connect** — CDC из PostgreSQL
- **Liquibase** — управление схемами
- **API Gateway** — feature flags и роутинг
- **Shadow traffic + reconciliation jobs** — проверка корректности
- **Yandex Data Transfer** — бэкфиллы больших объёмов
- **Redis** — кэширование горячих данных (координаты, surge, цены)