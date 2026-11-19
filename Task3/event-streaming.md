# Потоковая обработка событий (Kafka)

## Топики

| Топик | Ключ | Партиции | Кто пишет | Кто читает |
|---|---|---|---|---|
| `rtb.bid-events` | campaign_id | 24 | Bidding | Billing, Analytics |
| `rtb.impressions` | campaign_id | 48 | Stats Ingest | Billing, Analytics |
| `rtb.clicks` | campaign_id | 24 | Stats Ingest | Billing, Analytics |
| `campaign.changed` | campaign_id | 6 | Campaign | Bidding (кеш) |
| `budget.updated` | advertiser_id | 6 | Finance | Bidding (кеш) |

Ключ `campaign_id`. Так сохраняется порядок событий внутри кампании, а потребители масштабируются числом партиций. По источникам: `rtb.bid-events` пишет сам Bidding, а показы и клики приходят как маячки с тегов ad server в Stats Ingest, он их только складывает в топики.

## Схема событий: Avro

Avro + Confluent Schema Registry. Сообщения в 3-5 раз компактнее, при миллиардах событий это ощутимо; схема проверяется на broker при приёме; эволюция по правилам совместимости, добавляем поля с дефолтом и не ломаем потребителей. JSON оставляем только на границе OpenRTB, где протокол диктует партнёр.

Пример схемы `rtb.impression`:

```json
{"type":"record","name":"Impression","fields":[
  {"name":"event_id","type":"string"},
  {"name":"ts","type":"long","logicalType":"timestamp-millis"},
  {"name":"campaign_id","type":"long"},
  {"name":"advertiser_id","type":"long"},
  {"name":"dsp_id","type":"int"},
  {"name":"price","type":"long","doc":"цена в копейках"},
  {"name":"geo","type":"string"}]}
```

## Группы потребителей

- `billing-group` списывает по impression/click, идемпотентность по `event_id` (см. reliability.md)
- `analytics-group` загружает в ClickHouse батчами по 5000 событий или раз в 2 секунды
- `bidding-cache-group` обновляет кеш кампаний и бюджетов

Каждая группа читает своим offset, падение одной не мешает остальным. Лаг группы биллинга стоит на метрике с алертом: больше 60 с предупреждает, больше 10 мин эскалация.

## Политика хранения

Hot-хранение 7 дней, cleanup.policy=delete. Недели хватает на разбор инцидента и догонку упавшего потребителя. Долгое хранение не в Kafka: исторические события живут в ClickHouse годами, Kafka здесь только буфер. События `budget.updated` дополнительно чекпоинтятся в Finance.
