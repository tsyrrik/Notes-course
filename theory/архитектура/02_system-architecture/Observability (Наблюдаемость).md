
## Что это вообще такое

**Observability** — это способность понять **внутреннее состояние системы по её внешним сигналам**.  
Если проще: насколько легко ответить на вопрос _«почему всё тормозит?»_  
без того, чтобы лезть в код или SSH на прод.

Наблюдаемость строится на **трёх китах**:

|Компонент|Что даёт|Примеры|
|---|---|---|
|**Logs**|Текстовые события|Monolog, Loki, ELK, Graylog|
|**Metrics**|Численные метрики (время, счётчики)|Prometheus, Grafana, Datadog|
|**Traces**|“Следы” запросов между сервисами|OpenTelemetry, Jaeger, Zipkin|

---

## 1. Логирование

### Что логировать

- Старт и завершение ключевых процессов (job, request, cron).
    
- Исключения и ошибки (`error`, `critical`, `alert`).
    
- Бизнес-события (регистрация, покупка, webhook).
    
- Контекст (userId, requestId, traceId).
    
- Инфраструктурные данные (время, IP, сервис, окружение).
    

### Как логировать

Используй структурированные логи — **JSON**.  
Пусть парсится автоматически, а не regex’ом через боль.

Пример (Monolog):

```php
$log = new Logger('orders');
$log->pushHandler(new StreamHandler('php://stdout', Logger::INFO));
$log->info('Order placed', [
    'user_id' => 123,
    'order_id' => 42,
    'amount' => 1999,
]);
```

Вывод:

```json
{
  "level": "info",
  "message": "Order placed",
  "context": {"user_id": 123, "order_id": 42, "amount": 1999},
  "timestamp": "2025-10-31T09:00:21+03:00"
}
```

### Основные уровни логов

|Уровень|Когда|
|---|---|
|`debug`|вспомогательные данные для отладки|
|`info`|нормальное поведение (запуск, завершение)|
|`notice`|что-то интересное, но не ошибка|
|`warning`|подозрительно|
|`error`|что-то сломалось, но система живёт|
|`critical`|всё очень плохо|
|`alert`|нужен немедленный отклик|
|`emergency`|катастрофа (out of memory, disk full)|

### Куда писать

- `stdout` (через Docker — лучший вариант).
    
- `/var/log/app.log` (на bare metal).
    
- В централизованный сборщик (ELK, Loki, Datadog, Sentry).
    

---

## 2. Сбор и агрегация логов

### Основные схемы

#### Локально

`app.log` → `filebeat` → `Logstash` → `Elasticsearch` → `Kibana`

#### Современный вариант

`stdout` → `promtail` → `Loki` → `Grafana`

#### Альтернатива (облако)

Sentry, Datadog, Logtail, New Relic, Graylog — SaaS с алертами и дашбордами.

### Лучшие практики

- Не логируй пароли и токены.
    
- Не кидай всё в `error.log` — делай уровни.
    
- Добавляй `trace_id` или `request_id` в каждый лог (для склейки с трейсом).
    
- Логи должны быть **читаемы машиной**, не только глазами.
    

---

## 3. Метрики

### Виды метрик

|Тип|Что измеряет|Примеры|
|---|---|---|
|**Counter**|накапливающееся значение|количество запросов|
|**Gauge**|текущее значение|активные воркеры|
|**Histogram**|распределение|время ответа|
|**Summary**|агрегированные данные|перцентили (p95, p99)|

### Как собирать

#### Prometheus

Самый популярный open-source стандарт.

- Экспортер (PHP, Nginx, DB, Node, OS) отдаёт `/metrics` endpoint.
    
- Prometheus периодически **тянет** данные.
    
- Grafana визуализирует.
    

Пример PHP метрик (через `prometheus_client_php`):

```php
use Prometheus\CollectorRegistry;
use Prometheus\Storage\InMemory;
use Prometheus\RenderTextFormat;

$registry = new CollectorRegistry(new InMemory());

$requests = $registry->getOrRegisterCounter('app', 'requests_total', 'Total requests', ['method']);
$requests->inc(['GET']);

$renderer = new RenderTextFormat();
echo $renderer->render($registry->getMetricFamilySamples());
```

→ `/metrics` отдаёт:

```
# HELP app_requests_total Total requests
# TYPE app_requests_total counter
app_requests_total{method="GET"} 42
```

---

## 4. Трейсинг (распределённые трассы)

### Зачем

Когда один запрос проходит через 5 микросервисов и RabbitMQ,  
тебе нужно видеть всю цепочку: где тормоз, где ошибка.

### Как устроено

- Каждый запрос получает **Trace ID**.
    
- Все сервисы передают его дальше через HTTP-заголовки:
    
    ```
    traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-00
    ```
    
- Коллектор (Jaeger, Tempo, Zipkin, Datadog APM) визуализирует цепочку вызовов.
    

### Что использовать

|Инструмент|Назначение|
|---|---|
|**OpenTelemetry (OTel)**|стандарт API/SDK для всех языков|
|**Jaeger / Tempo / Zipkin**|коллекторы и UI|
|**Sentry APM / Datadog / New Relic**|SaaS с метриками и трейсом|

### Как в PHP

```bash
composer require open-telemetry/sdk
```

В коде:

```php
$tracer = (new TracerProvider())->getTracer('app');
$span = $tracer->spanBuilder('db_query')->startSpan();
// ... что-то делаем
$span->end();
```

---

## 5. Алертинг

### Основная идея

Алерты — не просто “у нас 500”.  
Они отвечают на вопрос: “Нужно ли человеку просыпаться?”

### Виды алертов

- **Threshold** — превышен порог (`cpu > 90%` 5 минут).
    
- **Anomaly** — метрика ведёт себя необычно.
    
- **Heartbeat** — сервис перестал присылать данные.
    
- **Composite** — комбинация нескольких условий (например, 500 > 1%, latency > 2s).
    

### Чем управлять

- **Alertmanager** (из Prometheus)
    
- **Grafana Alerts**
    
- **PagerDuty / OpsGenie / Slack Webhook**
    

Пример Alertmanager правила:

```yaml
groups:
- name: app
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
    for: 10m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"
      description: "More than 5% of requests are 5xx"
```

---

## 6. Дашборды

**Grafana** — де-факто стандарт визуализации.

Типичные панели:

- Request rate (`RPS`)
    
- Error rate (`5xx count`)
    
- Latency (p50 / p95 / p99)
    
- Memory, CPU
    
- Queue length / job duration
    
- Business metrics (новые пользователи, продажи, подписки)
    

### Рекомендации:

- Не лепи всё на один экран — делай дашборды по темам (Infra, API, Business).
    
- Добавляй _annotations_ (релизы, инциденты).
    
- Для трейсов — линк на Jaeger/Tempo.
    
- Для логов — линк на Loki/Sentry.
    

---

## 7. Полный стек наблюдаемости (классический open-source вариант)

```
┌───────────────┐
│    Приложение │
│     (PHP)     │
└───────┬───────┘
        │
  Logs  │→ Promtail → Loki → Grafana (просмотр логов)
  Metrics → Prometheus → Grafana (дашборды, алерты)
  Traces  → OpenTelemetry → Jaeger/Tempo (трейсинг)
```

---

## 8. Практические советы

- В каждом запросе генерируй **Request ID / Trace ID**.
    
- Передавай его в логи, HTTP-заголовки, метрики, трейсы.
    
- Логируй JSON — не пиши текст вроде “Error happened at line 34”.
    
- Не сливай “debug” на прод.
    
- Используй `structured logging` и единый формат по всем сервисам.
    
- Ставь **rate limit** на алерты — иначе заспамит.
    
- Мониторь не только ошибки, но и _молчание_ (если метрики исчезли).
    
- Обязательно алерт на “no data” и “не работает мониторинг”.
    

---

### Краткое резюме

|Компонент|Что|Инструменты|
|---|---|---|
|**Логи**|что произошло|Monolog → Loki / ELK / Sentry|
|**Метрики**|как себя ведёт|Prometheus → Grafana|
|**Трейсы**|как это связано|OpenTelemetry → Jaeger|
|**Алерты**|когда звать людей|Alertmanager / Grafana Alerts|
|**Дашборды**|видеть всё сразу|Grafana|
