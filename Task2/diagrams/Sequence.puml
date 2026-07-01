@startuml
title Sequence — Обработка bid request от DSP-партнёра (AdScale)

actor "DSP-партнёр" as DSP
participant "API Gateway" as GW
participant "Bidding & Delivery\nService" as BID
participant "Bidding Cache\n(Redis)" as CACHE
database "Bidding DB\n(PostgreSQL)" as DB
queue "Kafka" as KAFKA
participant "Event Ingestion\nService" as STATS
participant "Financial\nService" as FIN
participant "Analytics\nService" as ANA

== Синхронный путь (критичный, budget ≤ 80 ms) ==

DSP -> GW: POST /openrtb2/bid\n{auction_id, impression, timeout}
activate GW

GW -> GW: Проверка API-ключа\nRate limiting

alt Circuit Breaker: Closed
    GW -> BID: gRPC: PlaceBid(request)
    activate BID

    BID -> CACHE: Читает кампании и таргетинг\n(cache-aside)
    activate CACHE

    alt Промах кэша
        CACHE -> DB: SELECT campaigns, bids\nWHERE targeting matches
        activate DB
        DB --> CACHE: campaign data
        deactivate DB
    end

    CACHE --> BID: campaign + targeting data
    deactivate CACHE

    BID -> BID: Проверка remaining_budget\n(локальный кэш, без синхронного\nвызова Financial Service)

    BID -> BID: Отбор кандидатов\nАукцион: взвешивание ставок\nОпределение победителя

    BID -> BID: Формирование bid response\nи разметки баннера

    BID --> GW: bid response\n{price, creative, latency_ms}
    deactivate BID

    GW --> DSP: 200 OK\n{bid response}

else Circuit Breaker: Open
    GW --> DSP: Пустой bid response\n(отказ от аукциона, fallback)
    note right of GW
      Bidding & Delivery не отвечает
      в срок или превышен порог ошибок.
      Gateway не тратит латентностный
      бюджет на заведомо неуспешный вызов.
    end note
end

deactivate GW

== Асинхронный путь (не блокирует ответ DSP-партнёру) ==

BID -> KAFKA: Publish BidWon,\nImpressionRecorded
note right of BID
  Публикация не входит
  в критичный путь ответа.
  Retry на уровне продюсера
  при сбое публикации.
end note

KAFKA -> STATS: Consume\n(async)
activate STATS
STATS -> STATS: Запись события\nв Stats DB
deactivate STATS

KAFKA -> FIN: Consume BidWon\n(async, idempotent)
activate FIN
FIN -> FIN: Проверка idempotency_key\n= bid_id + operation_type
FIN -> FIN: Списание средств\nс бюджета кампании
deactivate FIN

KAFKA -> ANA: Consume\n(async)
activate ANA
ANA -> ANA: Построение агрегатов\nдля отчётности
deactivate ANA

@enduml
