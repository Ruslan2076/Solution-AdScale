@startuml
!include <C4/C4_Component>

left to right direction

title C4 Component Diagram — Bidding & Delivery Service (AdScale)

Container(apiGateway, "API Gateway", "тонкий слой маршрутизации")
ContainerDb(biddingCache, "Bidding Cache", "Redis")
ContainerDb(biddingDb, "Bidding DB", "PostgreSQL")
Container(kafka, "Kafka", "Message Broker")

System_Boundary(bidding, "Bidding & Delivery Service") {

    Component(requestHandler, "Request Handler", "gRPC endpoint", "Принимает запрос от API Gateway, валидирует структуру bid request, управляет общим таймаутом 80 ms")

    Component(targetingEngine, "Targeting Engine", "модуль", "Отбирает кандидатов на показ по данным таргетинга: гео, устройство, площадка")

    Component(budgetChecker, "Budget Checker", "модуль", "Проверяет remaining_budget по локальному кэшированному счётчику без синхронного вызова Financial Service")

    Component(auctionEngine, "Auction Engine", "модуль", "Взвешивает ставки кандидатов, применяет бизнес-правила, определяет победителя аукциона")

    Component(deliveryBuilder, "Delivery Builder", "модуль", "Формирует bid response и HTML/JS-разметку баннера победителя")

    Component(cacheClient, "Cache Client", "модуль", "Реализует cache-aside чтение кампаний и таргетинга, промах кэша ведёт к чтению из Bidding DB")

    Component(dbReader, "DB Reader", "модуль", "Читает из Bidding DB при промахе кэша")

    Component(eventPublisher, "Event Publisher", "модуль", "Публикует BidWon, ImpressionRecorded, ClickRecorded в Kafka асинхронно, вне критичного пути ответа")

    Component(campaignSyncConsumer, "Campaign Sync Consumer", "модуль", "Потребляет CampaignUpdated, BudgetUpdated, BalanceChanged из Kafka и обновляет Bidding DB и Budget Checker")
}

Rel(apiGateway, requestHandler, "gRPC: PlaceBid", "синхронно")

Rel(requestHandler, targetingEngine, "Передаёт запрос на отбор кандидатов")
Rel(targetingEngine, cacheClient, "Запрашивает данные таргетинга")
Rel(cacheClient, biddingCache, "Читает", "cache-aside")
Rel(cacheClient, dbReader, "Промах кэша")
Rel(dbReader, biddingDb, "SELECT", "SQL")

Rel(targetingEngine, budgetChecker, "Проверяет остаток бюджета кандидата")

Rel(targetingEngine, auctionEngine, "Передаёт отобранных кандидатов")
Rel(auctionEngine, deliveryBuilder, "Передаёт победителя аукциона")
Rel(deliveryBuilder, requestHandler, "Возвращает bid response")

Rel(requestHandler, eventPublisher, "Инициирует публикацию события\nпосле формирования ответа")
Rel(eventPublisher, kafka, "Publish", "async, не блокирует ответ")

Rel(kafka, campaignSyncConsumer, "Consume\nCampaignUpdated, BudgetUpdated, BalanceChanged", "async")
Rel(campaignSyncConsumer, biddingDb, "Обновляет read-модель")
Rel(campaignSyncConsumer, budgetChecker, "Обновляет remaining_budget")

legend right
  <#438DD5> Component — внутренний модуль сервиса
  <#999999> Внешний контейнер (за пределами сервиса)
  Все внешние зависимости асинхронны,
  кроме входящего вызова от API Gateway
endlegend

@enduml
