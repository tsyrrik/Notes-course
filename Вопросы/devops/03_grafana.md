# Grafana
- Визуализация данных из разных источников (Prometheus, PostgreSQL и др.), дашборды, роли и права.
- Алёрты (Unified Alerting), плагины, переменные для фильтрации, интеграция с Loki/Tempo/Mimir.
- Использовать как единый observability-дашборд.

Пример панели:
- Data source: Prometheus
- Query: `rate(http_requests_total[5m])`
- Visualization: Time series, панели с переменной `$service` для выбора сервиса.
