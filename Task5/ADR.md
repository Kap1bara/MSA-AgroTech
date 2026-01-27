### <a name="_b7urdng99y53"></a>**Название задачи:** PigtureMVP - SaaS интеграции (мультитенантная архитектура)
### <a name="_hjk0fkfyohdk"></a>**Автор:** Мурин Алексей Александрович
### <a name="_uanumrh8zrui"></a>**Дата:**  


## Ссылка на базовый вариант
Основано на выбранной архитектуре "Edge + Central (batch sync)" из Task4/DesignA.

---

# 1. Контекст

После успешного внедрения MVP на собственных фермах компания выводит решение как коммерческую SaaS-платформу для других агрохолдингов.

Требования SaaS:
- Полная изоляция данных между клиентами (tenants).
- Гибкая система подписок с тарифными планами.
- Публичный API для интеграции с системами клиентов.
- Self-service панель для управления аккаунтом.
- Горизонтальное масштабирование по мере роста tenants.
- Сбор метрик использования для аналитики и биллинга.
- Интеграция с российскими платежными системами.
- Документация и инструменты самостоятельного онбординга (dev portal).

---

# 2. Принцип разделения платформы

Платформа разделяется на две плоскости:

## 2.1 Control Plane
Отвечает за:
- управление tenant (onboarding, lifecycle, storage policy)
- identity and access (OIDC, RBAC, tenant scope)
- подписки, тарифы, квоты, feature flags
- биллинг, платежи, платежные webhooks
- usage metering (агрегация для биллинга)
- developer portal и управление ключами интеграции

## 2.2 Data Plane
Отвечает за:
- прием batch-синхронизации от edge-агентов
- нормализацию и дедупликацию событий
- доменную логику (Health, Feed)
- хранение доменных данных tenant
- экспорт данных и событий во внешние системы клиента

Разделение снижает связанность: изменения биллинга и подписок не должны влиять на pipeline обработки событий и доменную модель.

---

# 3. Мультитенантная изоляция данных: варианты и критерии

## 3.1 Критерии сравнения
1. Степень изоляции и снижение риска утечки данных.
2. Стоимость эксплуатации (ops).
3. Сложность provisioning (создание tenant).
4. Сложность миграций и обновлений схем.
5. Масштабирование по числу tenants и объему данных.
6. Возможность предложить enterprise-опции (повышенная изоляция).
7. Влияние шумных соседей.

---

## 3.2 A: Schema-per-tenant

### Суть
Один кластер PostgreSQL для tenant-данных. Для каждого tenant создается отдельная схема.

### Плюсы
- Быстрый онбординг (создание схемы).
- Низкая стоимость и простая плотность размещения.
- Хорошо подходит для большого числа клиентов.

### Минусы и риски
- Миграции сложнее: нужно применять к набору схем.
- Требуются жесткие механизмы tenant routing и проверки контекста.
- Шумные соседи в одном кластере.

### C2: Variant A (Schema-per-tenant) с Control/Data Plane и отдельной зоной хранилищ

```puml
\@startuml C2_SaaS_SchemaPerTenant_Fixed
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title C2 - SaaS Variant A (Schema-per-tenant) - Control/Data Plane

System_Ext(edge_agents, "Edge Agents", "Client farms (Video/Equipment/Sync)")
System_Ext(payment_providers, "RU Payment Providers", "YooKassa, Sber, Tinkoff, CloudPayments")
System_Ext(corporate_kafka, "Customer Corporate Kafka", "Optional client-side bus for integrations")

System_Boundary(saas, "Livestock SaaS Platform") {

  System_Boundary(control_plane, "Control Plane") {
    Container(api_gateway, "Public API Gateway", "Kong", "Auth, tenant routing, rate limits")
    Container(iam, "Identity Service", "OIDC/Keycloak", "OIDC, RBAC, tenant scope")

    Container(tenant_service, "Tenant Service", "Java/Kotlin", "Tenant registry, routing metadata")
    Container(subscription_service, "Subscription Service", "Java/Kotlin", "Plans, quotas, feature flags")
    Container(billing_service, "Billing Service", "Java/Kotlin", "Invoices, payments, dunning")
    Container(payment_adapter, "Payment Adapter", "Java/Kotlin", "Provider API + webhooks, idempotency")
    Container(usage_metering, "Usage Metering", "Go", "Usage aggregation for billing")
    Container(dev_portal, "Developer Portal", "Docusaurus/Backstage", "Docs, OpenAPI, onboarding")

    ContainerDb(control_db, "Control DB", "PostgreSQL", "Tenants, users, plans, billing")
  }

  System_Boundary(data_plane, "Data Plane") {
    Container(sync_gateway, "Sync Gateway", "Go/REST", "Batch intake from edge")
    Container(event_processor, "Event Processor", "Go", "Normalize, dedup, route (tenant scoped)")
    Container(health_service, "Health Service", "Java/Kotlin", "Health domain")
    Container(feed_service, "Feed Service", "Java/Kotlin", "Feed domain")

    Container(health_integration, "Health Integration Adapter", "Java/Kotlin", "Exports/webhooks to customer systems")
    Container(feed_integration, "Feed Integration Adapter", "Java/Kotlin", "Exports/webhooks to customer systems")

    ContainerDb(tenant_cluster, "Tenant Data Cluster", "PostgreSQL", "Shared cluster (schema-per-tenant)")
    ContainerDb(schema_t1, "Tenant Schema", "schema: tenant_123", "Isolated tenant data")
    ContainerDb(schema_t2, "Tenant Schema", "schema: tenant_456", "Isolated tenant data")
  }
}

Rel(edge_agents, sync_gateway, "Batch sync", "HTTPS/REST")
Rel(sync_gateway, event_processor, "Forward validated batch")
Rel(event_processor, health_service, "Health events (tenant scoped)")
Rel(event_processor, feed_service, "Feed events (tenant scoped)")

Rel(api_gateway, iam, "Auth", "OIDC/JWT")
Rel(api_gateway, tenant_service, "Resolve tenant, routing", "HTTPS/REST")
Rel(api_gateway, subscription_service, "Check quotas/features", "HTTPS/REST")
Rel(api_gateway, billing_service, "Billing APIs (self-service)", "HTTPS/REST")
Rel(api_gateway, health_service, "Tenant APIs", "HTTPS/REST")
Rel(api_gateway, feed_service, "Tenant APIs", "HTTPS/REST")

Rel(dev_portal, api_gateway, "OpenAPI, onboarding flows", "HTTPS")

Rel(tenant_service, control_db, "CRUD", "JDBC")
Rel(subscription_service, control_db, "CRUD", "JDBC")
Rel(billing_service, control_db, "CRUD", "JDBC")
Rel(iam, control_db, "Users/tenants mapping", "JDBC")

Rel(billing_service, payment_adapter, "Create/confirm payments", "HTTPS/REST")
Rel(payment_adapter, payment_providers, "Provider API + webhooks", "HTTPS")
Rel(usage_metering, billing_service, "Usage aggregates", "HTTPS/REST")
Rel(subscription_service, usage_metering, "Quota model", "HTTPS/REST")

Rel(health_service, tenant_cluster, "CRUD (tenant schema)", "JDBC")
Rel(feed_service, tenant_cluster, "CRUD (tenant schema)", "JDBC")

Rel(tenant_cluster, schema_t1, "Logical isolation")
Rel(tenant_cluster, schema_t2, "Logical isolation")

Rel(health_service, usage_metering, "Emit usage events", "HTTPS/REST")
Rel(feed_service, usage_metering, "Emit usage events", "HTTPS/REST")
Rel(sync_gateway, usage_metering, "Emit sync usage", "HTTPS/REST")

Rel(health_integration, corporate_kafka, "Publish tenant events", "Kafka")
Rel(feed_integration, corporate_kafka, "Publish tenant events", "Kafka")
Rel(health_service, health_integration, "Domain export", "HTTPS/REST")
Rel(feed_service, feed_integration, "Domain export", "HTTPS/REST")

@enduml

```

## 3.3 Вариант B: Database-per-tenant


### Суть
Для каждого tenant создается отдельная БД (database).

### Плюсы
- Быстрый онбординг (создание схемы).
- Низкая стоимость и простая плотность размещения.
- Хорошо подходит для большого числа клиентов.

### Минусы и риски
- Миграции сложнее: нужно применять к набору схем.
- Требуются жесткие механизмы tenant routing и проверки контекста.
- Шумные соседи в одном кластере.

### C2: B (Database-per-tenant)

```puml
@startuml C2_SaaS_VariantB_DatabasePerTenant
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title C2 - SaaS Variant B (Database-per-tenant) - Control/Data Plane

System_Ext(edge_agents, "Edge Agents", "Client farms (batch sync)")
System_Ext(payment_providers, "RU Payment Providers", "YooKassa, Sber, Tinkoff, CloudPayments")
System_Ext(corporate_kafka, "Customer Corporate Kafka", "Optional client bus for integrations")

System_Boundary(saas, "Livestock SaaS Platform") {

  System_Boundary(control_plane, "Control Plane") {
    Container(api_gateway, "Public API Gateway", "Kong", "Auth, tenant routing, rate limits")
    Container(iam, "Identity Service", "OIDC/Keycloak", "OIDC, RBAC, tenant scope")

    Container(tenant_service, "Tenant Service", "Java/Kotlin", "Tenant registry + DB routing")
    Container(subscription_service, "Subscription Service", "Java/Kotlin", "Plans, quotas")
    Container(billing_service, "Billing Service", "Java/Kotlin", "Invoices, payments")
    Container(payment_adapter, "Payment Adapter", "Java/Kotlin", "Provider API + webhooks")
    Container(usage_metering, "Usage Metering", "Go", "Usage aggregation")
    Container(dev_portal, "Developer Portal", "Docusaurus/Backstage", "Docs, OpenAPI, onboarding")
  }

  System_Boundary(data_plane, "Data Plane") {
    Container(sync_gateway, "Sync Gateway", "Go/REST", "Batch intake from edge")
    Container(event_processor, "Event Processor", "Go", "Normalize, dedup, route")

    Container(health_service, "Health Service", "Java/Kotlin", "Health domain")
    Container(feed_service, "Feed Service", "Java/Kotlin", "Feed domain")

    Container(health_integration, "Health Integration Adapter", "Java/Kotlin", "Exports to client systems")
    Container(feed_integration, "Feed Integration Adapter", "Java/Kotlin", "Exports to client systems")
  }

  System_Boundary(platform_stores, "Platform Data Stores") {
    ContainerDb(control_db, "Control DB", "PostgreSQL", "Tenants, users, plans, billing")
    ContainerDb(tenant_db_123, "Tenant DB", "PostgreSQL", "DB: tenant_123")
    ContainerDb(tenant_db_456, "Tenant DB", "PostgreSQL", "DB: tenant_456")
  }
}

Rel(edge_agents, sync_gateway, "Batch sync", "HTTPS/REST")
Rel(sync_gateway, event_processor, "Forward validated batch")
Rel(event_processor, health_service, "Health events (tenant scoped)")
Rel(event_processor, feed_service, "Feed events (tenant scoped)")

Rel(api_gateway, iam, "Auth", "OIDC/JWT")
Rel(api_gateway, tenant_service, "Resolve tenant -> DB routing", "HTTPS/REST")
Rel(api_gateway, subscription_service, "Check quotas/features", "HTTPS/REST")
Rel(api_gateway, billing_service, "Billing APIs (self-service)", "HTTPS/REST")
Rel(api_gateway, health_service, "Tenant APIs", "HTTPS/REST")
Rel(api_gateway, feed_service, "Tenant APIs", "HTTPS/REST")
Rel(dev_portal, api_gateway, "OpenAPI, onboarding", "HTTPS")

Rel(tenant_service, control_db, "CRUD", "JDBC")
Rel(subscription_service, control_db, "CRUD", "JDBC")
Rel(billing_service, control_db, "CRUD", "JDBC")
Rel(iam, control_db, "Users/tenants mapping", "JDBC")

Rel(billing_service, payment_adapter, "Create/confirm payments", "HTTPS/REST")
Rel(payment_adapter, payment_providers, "Provider API + webhooks", "HTTPS")
Rel(usage_metering, billing_service, "Usage aggregates", "HTTPS/REST")

Rel(health_service, tenant_db_123, "CRUD (tenant routing)", "JDBC")
Rel(feed_service, tenant_db_123, "CRUD (tenant routing)", "JDBC")
Rel(health_service, tenant_db_456, "CRUD (tenant routing)", "JDBC")
Rel(feed_service, tenant_db_456, "CRUD (tenant routing)", "JDBC")

Rel(sync_gateway, usage_metering, "Emit sync usage", "HTTPS/REST")
Rel(health_service, usage_metering, "Emit usage", "HTTPS/REST")
Rel(feed_service, usage_metering, "Emit usage", "HTTPS/REST")

Rel(health_service, health_integration, "Domain export", "HTTPS/REST")
Rel(feed_service, feed_integration, "Domain export", "HTTPS/REST")
Rel(health_integration, corporate_kafka, "Publish tenant events", "Kafka")
Rel(feed_integration, corporate_kafka, "Publish tenant events", "Kafka")

@enduml

```

# 3. Рекомендация

Выбор зависит от тарифных планов

По умолчанию используется A (Schema-per-tenant) для скорости онбординга и снижения стоимости.

Для больших тарифов предлагается B (Database-per-tenant) как опция повышенной изоляции.

Такой подход позволяет монетизировать изоляцию и закрывать разные сегменты рынка без избыточных затрат на базовый слой.

# 5. Задача 2: биллинг и монетизация

## 5.1 Сервисы биллинга

Для поддержки подписочной модели и монетизации добавляются сервисы Control Plane:

- **Subscription Service**
  - хранит тарифные планы, лимиты (quota) и feature flags
  - управляет жизненным циклом подписки (trial, active, suspended, cancelled)
  - является источником прав на функциональность (entitlements)

- **Billing Service**
  - формирует счета (invoices) и хранит их статусы
  - управляет статусами оплат и просрочек (dunning)
  - связывает подписку, начисления и фактические оплаты

- **Payment Adapter**
  - изолирует интеграцию с провайдерами оплат
  - принимает webhooks, валидирует подписи
  - обеспечивает idempotency и ретраи

- **Usage Metering**
  - собирает и агрегирует метрики использования (usage)
  - предоставляет агрегаты в Billing Service для расчета стоимости (при необходимости usage-based pricing)
  - поддерживает модель квот совместно с Subscription Service


## 5.2 Интеграция с российскими платежными системами

Требования к реализации интеграции:

- **Webhook-driven state machine**
  - финальный статус оплаты определяется событием от провайдера (webhook)
  - Billing Service хранит жизненный цикл платежа

- **Безопасность**
  - проверка подписи и секретов вебхуков
  - контроль IP / дополнительных атрибутов провайдера (по возможности)

- **Idempotency**
  - входящие webhooks обрабатываются идемпотентно по `provider_event_id`
  - исходящие операции по созданию платежа идемпотентны по `payment_id`

- **Надежность**
  - повторная обработка webhooks (at-least-once)
  - retry/backoff на вызовах провайдеров

Провайдеры, которые планируются к поддержке через адаптеры:
- YooKassa
- SberPay / Sber (эквайринг)
- Tinkoff (эквайринг)
- CloudPayments


---

# 6. Задача 3: интеграции с клиентами

## 6.1 Public API для клиентов SaaS

Публичный API предоставляется через **Public API Gateway (Kong)** и является tenant-scoped.

Ключевые возможности:
- **OIDC/OAuth2** (через IAM)
- **API keys** для server-to-server интеграций
- **Rate limiting** и квоты по тарифу
- **Tenant routing**
  - tenant определяется по токену/ключу
  - storage policy и маршрутизация запрашиваются у Tenant Service

## 6.2 Scope доступного функционала (MVP SaaS)

В MVP SaaS предоставляется:

- **Account / Tenant**
  - получение информации о tenant, квотах и текущем тарифе
  - управление API keys (self-service)

- **Farms / Edge**
  - регистрация edge-агентов
  - получение статусов синхронизации и состояния ферм

- **Monitoring**
  - чтение событий и агрегатов по Health и Feed
  - базовые отчеты и выгрузки

- **Billing (read-only на первых этапах)**
  - текущий тариф, счета, статус оплат


## 6.3 Developer Portal и онбординг

Для самостоятельного онбординга клиентов вводится **Developer Portal**:

- OpenAPI спецификация и reference
- примеры интеграции (SDK snippets)
- Postman коллекции
- руководство по регистрации tenant и edge-агентов
- документация по webhooks (если они используются)
- best practices по интеграции и ограничениям rate limiting


---

# 7. Финальная архитектура To-Be

В финальном варианте платформа построена как multi-tenant SaaS с явным разделением:

- **Control Plane**: tenants, IAM, subscriptions, billing, metering, dev portal
- **Data Plane**: sync intake, event processing pipeline, доменные сервисы Health/Feed, экспорт данных
- **Platform Data Stores**: физические хранилища control и tenant данных (для ясности границ)


```puml
@startuml C1_SaaS_ToBe
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

title C1 - Livestock SaaS Platform (To-Be)

Person(client_admin, "Client Admin", "Self-service, billing, keys")
Person(client_operator, "Client Operator", "Monitoring")

System(saas, "Livestock SaaS Platform", "Multi-tenant cloud platform")

System_Ext(edge_agents, "Edge Agents", "Client farms (batch sync)")
System_Ext(payment_providers, "RU Payment Providers", "YooKassa, Sber, Tinkoff, CloudPayments")
System_Ext(corporate_kafka, "Customer Corporate Kafka", "Optional client bus for integrations")

Rel(edge_agents, saas, "Batch sync")
Rel(client_admin, saas, "Self-service")
Rel(client_operator, saas, "Monitoring")
Rel(saas, payment_providers, "Payments")
Rel(saas, corporate_kafka, "Integration exports (optional)")

@enduml

```
```puml
@startuml C2_Example_AgroTechX_Tenant
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title C2 - Example: AgroTech X as SaaS Tenant

System_Ext(agrotech_farms, "AgroTech X Farms", "Multiple farms with edge agents")
System_Ext(agrotech_kafka, "AgroTech X Kafka", "Corporate bus (external)")
System_Ext(agrotech_erp, "AgroTech X ERP (1C:Agro)", "Client ERP")

System_Boundary(saas, "Livestock SaaS Platform") {
  System_Boundary(data_plane, "Data Plane") {
    Container(sync_gateway, "Sync Gateway", "Go/REST")
    Container(event_processor, "Event Processor", "Go")
    Container(health_service, "Health Service", "Java/Kotlin")
    Container(feed_service, "Feed Service", "Java/Kotlin")
    Container(health_integration, "Health Integration Adapter", "Java/Kotlin")
    Container(feed_integration, "Feed Integration Adapter", "Java/Kotlin")
  }

  System_Boundary(control_plane, "Control Plane") {
    Container(api_gateway, "Public API Gateway", "Kong")
  }
}

Rel(agrotech_farms, sync_gateway, "Batch sync", "HTTPS/REST")
Rel(sync_gateway, event_processor, "Forward batch")
Rel(event_processor, health_service, "Health events")
Rel(event_processor, feed_service, "Feed events")

Rel(health_service, health_integration, "Export")
Rel(feed_service, feed_integration, "Export")

Rel(health_integration, agrotech_kafka, "Publish events/metrics", "Kafka")
Rel(feed_integration, agrotech_kafka, "Publish events/metrics", "Kafka")
Rel(health_integration, agrotech_erp, "REST export", "HTTPS/REST")
Rel(feed_integration, agrotech_erp, "REST export", "HTTPS/REST")

@enduml

```


---

# 8. Итог

- SaaS платформа разделена на **Control Plane** и **Data Plane** для независимой эволюции биллинга и домена.
- Рассмотрены и зафиксированы 2 варианта изоляции данных:
  - **Schema-per-tenant** как default (быстрый онбординг, низкая стоимость)
  - **Database-per-tenant** как enterprise tier (усиленная изоляция)
- Биллинг реализован через отдельные сервисы тарификации и оплат:
  - Subscription Service, Billing Service, Payment Adapter, Usage Metering
- Интеграции с клиентами предоставляются через публичный API с tenant scope и dev portal.
- Корпоративная Kafka клиента является **внешней системой** и подключается через integration adapters в Data Plane.
