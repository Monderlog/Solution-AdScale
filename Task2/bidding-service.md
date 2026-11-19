# Bidding Service: спецификация

## Границы

В сервисе: приём bid-request по OpenRTB, отсечение по квоте и бюджету, подбор кандидатов, взвешивание ставок, выбор победителя, сборка bid-response, публикация события аукциона.

Вне сервиса: управление кампаниями (Campaign Service), формирование баннера (Delivery Service), учёт показов и кликов (Stats Ingest), списание бюджета (Finance), отчётность (Analytics).

Граница проведена просто. Всё, что нужно для ответа на bid-request в рамках бюджета сервиса (50 мс из 80 мс пути), живёт в сервисе. Остальное уходит асинхронно через Kafka.

## API

Внешний, для DSP через Gateway:

- `POST /openrtb/{dspId}/bid` по OpenRTB 2.5, JSON. Вход: bid-request (imp, device, user, bcat). Выход: bid-response (seatbid с ценой и adid) или `no-bid` (204). Заголовок `X-Request-Id` для трассировки, дедлайн в `tmax`.

Внутренний:

- `gRPC CampaignService/GetActiveCampaigns(since)` для nearline-выгрузки активных кампаний в кеш сервиса, раз в 30 с, а не на каждый запрос
- `gRPC DeliveryService/PrepareCreative(creativeId)` лениво, результат кешируется in-memory

## Зависимости

| Зависимость | Как | Если недоступна |
|---|---|---|
| Redis (кеш кампаний, бюджетов, таргетинга) | чтение <1 мс | работать last-known-кешем in-memory, деградация по floor-ставке |
| Campaign Service | nearline-выгрузка в кеш | аукцион продолжается на устаревшем кеше с пометкой stale |
| Kafka (rtb.bid-events) | асинхронная публикация | локальный буфер, ретрансляция при восстановлении |
| Delivery Service | gRPC + кеш креативов | отдавать adid без разметки, баннер дорисует ad server |

В отличие от AS-IS, синхронных вызовов к PostgreSQL в пути запроса нет.

## Модель данных

Сервис stateless, состояния почти нет:

- In-memory/Redis: `campaign_id → {budget_remaining, bid_floor, targeting, status}`, обновляется событиями `campaign.changed`; `creative_id → шаблон`
- Kafka `rtb.bid-events`: `{event_id, request_id, campaign_id, dsp_id, price, outcome(win/no-bid/loss), ts}`, источник для биллинга и аналитики
- Собственной БД нет, `event_id` генерируется до публикации и служит ключом идемпотентности win-событий, `request_id` остаётся для трассировки

## Масштабирование

Горизонтальное по CPU. 18 000 RPS держат 8-10 подов по 4 vCPU, этого хватает и с прицелом на 50 000 RPS. Локального состояния нет, поды одноразовые, при старте прогревают кеш.
