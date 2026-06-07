# clickpulse — сервис приёма и хранения метрик и батчевой записи в ClickHouse

![clickpulselogo](https://raw.githubusercontent.com/timur-developer/clickpulse/refs/heads/main/clickpulse_logo.png)

![Go](https://img.shields.io/badge/go-1.22%2B-00ADD8?logo=go&logoColor=white)
![ClickHouse](https://img.shields.io/badge/ClickHouse-FFCC01?logo=clickhouse&logoColor=black)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)
![License MIT](https://img.shields.io/badge/license-MIT-blue.svg)

Read this in other languages: [English](README.md)

`clickpulse` — небольшой Go-сервис для сбора аналитических событий по HTTP, буферизации их в памяти и пакетной записи в ClickHouse.

Проект сделан как практический backend/observability-сервис: HTTP ingestion, валидация входных событий, батчинг по размеру и времени, запись в ClickHouse, метрики для Prometheus, дашборд Grafana, локальный запуск через Docker Compose и базовые Kubernetes-манифесты.

Используйте `clickpulse`, если нужно:

- принимать события через простой HTTP API
- снижать нагрузку на ClickHouse за счёт пакетной записи вместо row-by-row inserts
- отправлять события в ClickHouse по размеру батча или по интервалу времени
- экспортировать технические метрики для Prometheus и Grafana
- быстро поднимать весь стек локально через Docker Compose

## Содержание

- [Как устроено](#как-устроено)
- [Быстрый старт](#быстрый-старт)
- [API](#api)
  - [POST /events](#post-events)
  - [GET /healthz](#get-healthz)
  - [GET /metrics](#get-metrics)
- [Конфигурация](#конфигурация)
- [Observability](#observability)
- [Docker Compose](#docker-compose)
- [Kubernetes](#kubernetes)
- [Разработка](#разработка)
- [Лицензия](#лицензия)

## Как устроено

```mermaid
flowchart LR
    Client[HTTP client] -->|POST /events| API[Go HTTP service]
    API --> Validator[JSON validation]
    Validator --> Batcher[In-memory batcher]
    Batcher -->|flush by size| CH[(ClickHouse)]
    Batcher -->|flush by interval| CH
    Prometheus[Prometheus] -->|scrape /metrics| API
    Grafana[Grafana] --> Prometheus
```

Сервис принимает события через `POST /events`, валидирует JSON payload и кладёт принятые события во внутренний in-memory batcher.

Batcher отправляет события в ClickHouse, когда выполняется одно из условий:

- размер батча достигает `BATCH_SIZE`
- с прошлого flush прошло `FLUSH_INTERVAL`

Так API остаётся простым, а ClickHouse не получает отдельный insert на каждое событие.

## Быстрый старт

Склонируйте репозиторий и запустите локальный стек:

```bash
git clone https://github.com/timur-developer/clickpulse.git
cd clickpulse
docker compose up --build -d
```

После запуска сервисы будут доступны по адресам:

| Сервис | URL |
| --- | --- |
| clickpulse | `http://localhost:8080` |
| ClickHouse HTTP | `http://localhost:8123` |
| Prometheus | `http://localhost:9090` |
| Grafana | `http://localhost:3000` |

Отправьте тестовое событие:

```bash
curl -X POST http://localhost:8080/events \
  -H "Content-Type: application/json" \
  -d '{
    "event_type": "page_view",
    "source": "landing",
    "user_id": "u123",
    "value": 1,
    "created_at": "2026-03-27T12:00:00Z"
  }'
```

Проверьте health check:

```bash
curl http://localhost:8080/healthz
```

Проверьте Prometheus metrics:

```bash
curl http://localhost:8080/metrics
```

## API

### `POST /events`

Принимает одно аналитическое событие в JSON-формате.

Пример запроса:

```json
{
  "event_type": "page_view",
  "source": "landing",
  "user_id": "u123",
  "value": 1,
  "created_at": "2026-03-27T12:00:00Z"
}
```

Поля:

| Поле | Тип | Обязательное | Описание |
| --- | --- | --- | --- |
| `event_type` | string | да  | Название события, например `page_view`, `signup`, `click` |
| `source` | string | да  | Источник события, например `landing`, `api`, `mobile` |
| `user_id` | string | нет | Идентификатор пользователя или клиента |
| `value` | number | нет | Числовое значение события |
| `created_at` | string | да  | Время события в формате RFC3339 |

Успешный ответ:

```json
{"status":"accepted"}
```

Некорректные payload возвращают `400 Bad Request` с текстом ошибки.

### `GET /healthz`

Endpoint для health check.

```bash
curl http://localhost:8080/healthz
```

Пример ответа:

```json
{"status":"ok"}
```

### `GET /metrics`

Endpoint для Prometheus scraping.

```bash
curl http://localhost:8080/metrics
```

Он отдаёт технические метрики по HTTP-трафику, принятым событиям, batch flush и ошибкам вставки в ClickHouse.

## Конфигурация

Сервис настраивается через переменные окружения.

| Переменная | Описание | Пример |
| --- | --- | --- |
| `HTTP_PORT` | Порт HTTP-сервера | `8080` |
| `CLICKHOUSE_DSN` | Строка подключения к ClickHouse | `http://localhost:8123?user=app&password=app` |
| `BATCH_SIZE` | Количество событий, после которого происходит flush | `100` |
| `FLUSH_INTERVAL` | Интервал времени, после которого происходит flush | `5s` |
| `LOG_LEVEL` | Уровень логирования приложения | `info` |

Пример локальной конфигурации:

```bash
export HTTP_PORT=8080
export CLICKHOUSE_DSN="http://localhost:8123?user=app&password=app"
export BATCH_SIZE=100
export FLUSH_INTERVAL=5s
export LOG_LEVEL=info
```

## Observability

`clickpulse` экспортирует метрики в Prometheus-формате и содержит Grafana-настройки для локальной разработки.

Полезные сигналы для наблюдения:

- request rate для `POST /events`
- HTTP latency
- количество принятых событий
- текущий размер батча
- количество batch flush
- ошибки вставки в ClickHouse

Это помогает быстро отвечать на вопросы:

- Получает ли сервис события?
- Не стали ли запросы медленнее?
- Регулярно ли отправляются батчи?
- Есть ли ошибки при записи в ClickHouse?
- Как изменение `BATCH_SIZE` или `FLUSH_INTERVAL` влияет на throughput?

## Docker Compose

Docker Compose setup предназначен для локального тестирования и демо.

Типичный workflow:

```bash
docker compose up --build -d
docker compose ps
docker compose logs -f clickpulse
```

Остановить стек:

```bash
docker compose down
```

Остановить стек и удалить volumes:

```bash
docker compose down -v
```

## Kubernetes

Базовые Kubernetes-манифесты находятся в директории `k8s/`.

Применить их:

```bash
kubectl apply -f k8s/
```

Манифесты намеренно минимальные и рассчитаны как стартовая точка. Они ожидают, что ClickHouse доступен через `CLICKHOUSE_DSN`.

## Разработка

Запустить тесты:

```bash
go test ./...
```

Запустить локально без Docker Compose:

```bash
export HTTP_PORT=8080
export CLICKHOUSE_DSN="http://localhost:8123?user=app&password=app"
export BATCH_SIZE=100
export FLUSH_INTERVAL=5s

go run ./cmd/app
```

Собрать сервис:

```bash
go build ./...
```

## Лицензия

MIT. См. [LICENSE](LICENSE).
