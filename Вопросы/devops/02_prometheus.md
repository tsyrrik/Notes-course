## Вопрос: Prometheus

## Простой ответ
- Сам опрашивает приложения по `/metrics` и хранит числа по времени.
- Умеет алертить через Alertmanager и рисовать графы (обычно через Grafana).

```php
// Простой /metrics
echo "http_requests_total{path=\"/\"} 42\n";
```

```promql
rate(http_requests_total[5m]) by (path)
```

## Ответ
- Pull-модель сбора метрик (`/metrics`), хранение в time-series DB.
- PromQL для агрегаций, Alertmanager для уведомлений (Slack/Email/Telegram).
- Экспортеры: node_exporter, blackbox_exporter, php-fpm_exporter и др.
- Стандарт де‑факто для Kubernetes/облачных инсталляций.
