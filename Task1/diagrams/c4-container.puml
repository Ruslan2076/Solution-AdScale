@startuml
!include <C4/C4_Container>

left to right direction

title C4 Container Diagram — AdScale (TO-BE, через 1 год)

' ====================== Акторы ======================
Person(advertiser, "Рекламодатель", "Управляет кампаниями, бюджетами и ставками через личный кабинет")
Person_Ext(endUser, "Пользователь сайта/приложения", "Просматривает рекламу, кликает по баннерам")

System_Ext(dspPartner, "DSP-партнёр", "Отправляет bid requests, получает bid responses по OpenRTB")
System_Ext(paymentGateway, "Платёжный шлюз", "Обработка пополнения рекламных бюджетов")

' ====================== API Gateway ======================
Container(apiGateway, "API Gateway", "тонкий слой маршрутизации", "Единая точка входа: auth, rate limiting, маршрутизация запросов к сервисам. Минимальный overhead для bid-пути")

' ====================== Bidding & Delivery (ключевой домен) ======================
System_Boundary(biddingCtx, "Bidding & Delivery Service") {
    Container(biddingService, "Bidding & Delivery Service", "Go / C++", "Таргетинг кандидатов, аукцион, формирование ответа и разметки баннера. Синхронный hot path ≤ 80 ms") <<Critical Path>>
    ContainerDb(biddingCache, "Bidding Cache", "Redis, cache-aside", "Кэш данных таргетинга и кампаний для чтения без похода в БД на каждый bid request")
    ContainerDb(biddingDb, "Bidding DB", "PostgreSQL + read-replica", "Кампании, ставки, данные таргетинга (read-optimized)")
}

' ====================== Event Ingestion (Stats) ======================
System_Boundary(statsCtx, "Event Ingestion Service") {
    Container(statsService, "Event Ingestion Service", "Python", "Приём и запись событий кликов и показов из Kafka")
    ContainerDb(statsDb, "Stats DB", "PostgreSQL", "События кликов и показов")
}

' ====================== Financial ======================
System_Boundary(financialCtx, "Financial Service") {
    Container(financialService, "Financial Service", "Go", "Списание средств, выставление счетов, управление балансами. Идемпотентный consumer") <<Strong Consistency>>
    ContainerDb(financialDb, "Financial DB", "PostgreSQL", "Балансы, счета, транзакции. Строгая ACID-согласованность")
}

' ====================== Analytics ======================
System_Boundary(analyticsCtx, "Analytics Service") {
    Container(analyticsService, "Analytics Service", "Python", "Аналитика и отчётность. Read-модель, построенная асинхронно из потока событий")
    ContainerDb(analyticsDb, "Analytics DB", "ClickHouse / PostgreSQL", "Агрегаты и отчёты для рекламодателей")
}

' ====================== Advertiser Dashboard ======================
System_Boundary(dashboardCtx, "Advertiser Dashboard Service") {
    Container(adCabinet, "Advertiser Dashboard Service", "Node.js + JavaScript", "Web UI и API: управление кампаниями, бюджетами, ставками")
    ContainerDb(campaignDb, "Campaign DB", "PostgreSQL", "Кампании, бюджеты, ставки")
}

' ====================== Брокер событий ======================
Container(kafka, "Kafka", "Message Broker / Event Streaming", "BidWon, ImpressionRecorded, ClickRecorded. Публикация/подписка, несколько независимых консьюмеров, аудиторский след") <<Event Broker>>

' ====================== Внешние связи ======================
Rel_U(dspPartner, apiGateway, "Bid requests / responses", "HTTPS / OpenRTB")
Rel_U(endUser, apiGateway, "Просмотр рекламы и клики", "HTTPS")
Rel(advertiser, apiGateway, "Управляет кампаниями и бюджетами", "HTTPS / REST")
Rel(adCabinet, paymentGateway, "Пополнение бюджета", "HTTPS / REST")

' ====================== API Gateway -> сервисы ======================
Rel(apiGateway, biddingService, "Маршрутизация bid requests", "gRPC, синхронно")
Rel(apiGateway, adCabinet, "Маршрутизация запросов ЛК", "REST, синхронно")

' ====================== Bidding: чтение ======================
Rel(biddingService, biddingCache, "Читает кампании и таргетинг", "cache-aside")
Rel(biddingCache, biddingDb, "Промах кэша: чтение из read-replica", "SQL")

' ====================== Асинхронные события ======================
Rel(biddingService, kafka, "Публикует BidWon, ImpressionRecorded, ClickRecorded", "async, publish")
Rel(kafka, statsService, "Потребляет события показов/кликов", "async, consume")
Rel(kafka, financialService, "Потребляет BidWon для списания", "async, consume, idempotent")
Rel(kafka, analyticsService, "Потребляет события для агрегации", "async, consume")

' ====================== Записи в собственные БД ======================
Rel(statsService, statsDb, "Записывает события", "SQL")
Rel(financialService, financialDb, "Транзакции и балансы", "SQL, ACID")
Rel(analyticsService, analyticsDb, "Строит агрегаты и отчёты", "SQL")
Rel(adCabinet, campaignDb, "CRUD кампаний и бюджетов", "SQL")

' ====================== Синхронные вызовы Dashboard ======================
Rel(adCabinet, financialService, "Запрашивает текущий баланс", "gRPC, синхронно")
Rel(adCabinet, analyticsService, "Запрашивает отчёты", "gRPC, синхронно")

' ====================== Layout hints ======================
Lay_D(apiGateway, biddingService)
Lay_R(biddingService, statsService)
Lay_R(statsService, financialService)
Lay_R(financialService, analyticsService)
Lay_D(adCabinet, campaignDb)
Lay_U(kafka, biddingService)

' ====================== Легенда с метриками ======================
legend right
  <#00AA00> **50 000 RPS (целевой)**
  <#00AA00> **hot path ≤ 80 ms** (Bidding & Delivery)
  <#00AA00> SLA 99,9%
  <#888888> Kafka: event streaming, множественные консьюмеры
  <#888888> Redis: cache-aside для Bidding
  <#888888> БД на сервис, отдельные схемы
  <#888888> Горизонтальное масштабирование Bidding независимо от остальных
endlegend
@enduml
