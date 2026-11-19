# Task 2. Проектирование RTB и интеграции с DSP

Во втором задании спроектирован Bidding Service, компонент, через который подключается новая DSP-площадка. Разобраны границы и API сервиса, протоколы с соседними системами, API Gateway для партнёров и паттерны надёжности.

- `bidding-service.md` спецификация сервиса ставок
- `interaction.md` протоколы: gRPC внутри, OpenRTB снаружи, Kafka для событий
- `api-gateway.md` API-слой для DSP
- `reliability.md` circuit breaker, ретраи, идемпотентность финансов, fallback-ставки
- `diagrams/bid-sequence.puml` путь bid-request от DSP до ответа
- `diagrams/bidding-component.puml` сервис и его зависимости

Бюджет времени на запрос по SLA партнёра (80 мс): Gateway 5 мс, сеть до Bidding 10 мс, сам Bidding 50 мс (кандидаты 20, аукцион 20, сборка ответа 10), резерв 15 мс.
