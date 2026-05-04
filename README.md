# Metrics Batch Collector

`Metrics Batch Collector` - это Go-сервис для приема аналитических событий по HTTP, буферизации, пакетной записи в ClickHouse и экспорта технических метрик для Prometheus и Grafana.

## Стек

- Go
- ClickHouse
- Prometheus
- Grafana
- Docker Compose
- Kubernetes manifests

## Что делает сервис

Приложение принимает события через `POST /events`, валидирует входной JSON, помещает события в batcher и записывает их в ClickHouse, когда выполняется одно из условий:

- батч достигает размера `BATCH_SIZE`
- проходит интервал `FLUSH_INTERVAL`

Дополнительно сервис отдает:

- `GET /healthz` для health check
- `GET /metrics` для Prometheus scraping

## Архитектура

```text
Client / Postman / script
        |
        v
   Go HTTP service
   - POST /events
   - GET /healthz
   - GET /metrics
        |
        v
   in-memory batcher
   - flush by size
   - flush by interval
        |
        v
    ClickHouse

Prometheus ---> scrapes /metrics
Grafana -----> dashboards from Prometheus
```

## Конфигурация

Сервис настраивается через переменные окружения.

| Variable | Required | Description |
| --- | --- | --- |
| `HTTP_PORT` | yes | HTTP server port |
| `CLICKHOUSE_DSN` | yes | ClickHouse connection string |
| `BATCH_SIZE` | yes | Maximum number of events in a batch |
| `FLUSH_INTERVAL` | yes | Max time between flushes |
| `LOG_LEVEL` | no | Log level, defaults to `info` |

Пример значений есть в `.env.example`.

## Локальный запуск через Docker Compose

Из корня проекта выполни:

```bash
docker compose up --build -d
docker compose ps
```

После запуска будут доступны:

- app: `http://localhost:8080`
- ClickHouse HTTP: `http://localhost:8123`
- Prometheus: `http://localhost:9090`
- Grafana: `http://localhost:3000`

Остановить стек:

```bash
docker compose down
```

Остановить и удалить volumes:

```bash
docker compose down -v
```

## API

### `POST /events`

Принимает одно событие в формате JSON.

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

Успешный ответ:

```json
{
  "status": "accepted"
}
```

Примеры ошибок валидации:

```json
{
  "error": "invalid request body"
}
```

```json
{
  "error": "missing required field: event_type"
}
```

### `GET /healthz`

Возвращает:

```json
{
  "status": "ok"
}
```

### `GET /metrics`

Возвращает метрики сервиса в формате Prometheus.

## Проверка сервиса

### Через Postman или curl

Запрос:

```bash
curl -i \
  -X POST http://localhost:8080/events \
  -H "Content-Type: application/json" \
  -d '{
    "event_type": "page_view",
    "source": "landing",
    "user_id": "u123",
    "value": 1,
    "created_at": "2026-03-27T12:00:00Z"
  }'
```

Проверка health endpoint:

```bash
curl http://localhost:8080/healthz
```

Проверка метрик:

```bash
curl http://localhost:8080/metrics
```

### Вспомогательные скрипты

В проекте есть два вспомогательных скрипта:

- `scripts/curl_examples.sh` отправляет один пример события
- `scripts/generate_events.sh` отправляет серию событий

Пример запуска:

```bash
sh scripts/curl_examples.sh
COUNT=200 sh scripts/generate_events.sh
```

## Проверка ClickHouse

Сервис пишет в ClickHouse батчами, поэтому одиночное событие может появиться в таблице только после срабатывания flush по таймеру.

Проверить общее количество записей:

```bash
docker compose exec clickhouse clickhouse-client --user app --password app --query "SELECT count() FROM default.events"
```

Посмотреть последние записи:

```bash
docker compose exec clickhouse clickhouse-client --user app --password app --query "SELECT event_type, source, user_id, value, created_at FROM default.events ORDER BY created_at DESC LIMIT 10 FORMAT PrettyCompact"
```

Примеры аналитических запросов:

```bash
docker compose exec clickhouse clickhouse-client --user app --password app --query "SELECT event_type, count() AS total FROM default.events GROUP BY event_type ORDER BY total DESC FORMAT PrettyCompact"
```

```bash
docker compose exec clickhouse clickhouse-client --user app --password app --query "SELECT source, count() AS total FROM default.events GROUP BY source ORDER BY total DESC FORMAT PrettyCompact"
```

```bash
docker compose exec clickhouse clickhouse-client --user app --password app --query "SELECT toStartOfMinute(created_at) AS minute, count() AS total FROM default.events GROUP BY minute ORDER BY minute DESC FORMAT PrettyCompact"
```

## Prometheus

Prometheus настраивается через `prometheus.yml` и забирает метрики с `app:8080/metrics`.

Полезные страницы:

- `http://localhost:9090/targets`
- `http://localhost:9090/graph`

Полезные запросы:

- `http_requests_total`
- `rate(http_requests_total[1m])`
- `events_received_total`
- `batch_flush_total`
- `rate(batch_flush_total[1m])`
- `batch_size`
- `clickhouse_insert_errors_total`
- `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))`

## Grafana

Grafana запускается с уже настроенным provisioning.

- URL: `http://localhost:3000`
- Login: `admin`
- Password: `admin`

Файлы provisioning:

- `grafana/provisioning/datasources/datasource.yml`
- `grafana/provisioning/dashboards/dashboard.yml`
- `grafana/provisioning/dashboards/app-dashboard.json`

В dashboard выведены:

- HTTP RPS
- HTTP latency p95
- total accepted events
- batch flush rate
- batch size
- ClickHouse insert errors

## Kubernetes manifests

В репозитории есть базовые манифесты в `k8s/`:

- `k8s/configmap.yaml`
- `k8s/deployment.yaml`
- `k8s/service.yaml`

Применение:

```bash
kubectl apply -f k8s/
```

Примечания:

- манифесты показывают базовую упаковку приложения
- deployment ожидает образ `metrics-batch-collector:latest`
- приложение ожидает доступный ClickHouse по адресу из `CLICKHOUSE_DSN`
- ClickHouse, Prometheus и Grafana этими манифестами не разворачиваются

## Структура репозитория

```text
metrics-batch-collector/
|-- cmd/app/main.go
|-- internal/
|   |-- batcher/
|   |-- config/
|   |-- event/
|   |-- http/
|   |-- metrics/
|   `-- storage/clickhouse/
|-- migrations/001_init.sql
|-- grafana/provisioning/
|-- scripts/
|-- k8s/
|-- docker-compose.yml
|-- Dockerfile
|-- prometheus.yml
`-- README.md
```
