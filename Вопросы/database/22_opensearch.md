## Вопрос: OpenSearch

## Простой ответ
- Это поисковый движок + аналитика логов, свободная альтернатива Elasticsearch.
- Подойдёт для полнотекстового поиска, дашбордов и триаж логов.

## Ответ

### Что такое OpenSearch

OpenSearch -- это распределённый поисковый и аналитический движок, созданный как форк Elasticsearch 7.10 и Kibana 7.10 компанией Amazon (AWS) совместно с open-source сообществом. Причиной форка стала смена лицензии Elasticsearch с Apache 2.0 на SSPL/Elastic License, что ограничивало использование в облачных сервисах. OpenSearch распространяется под лицензией Apache 2.0, что гарантирует свободу использования, модификации и распространения без ограничений.

### Архитектура

OpenSearch построен на основе Apache Lucene и использует кластерную архитектуру с несколькими типами узлов:

| Тип узла | Роль |
|----------|------|
| **Cluster Manager** (ранее Master) | Управление состоянием кластера, создание/удаление индексов, распределение шардов |
| **Data node** | Хранение данных и выполнение поисковых/аналитических операций |
| **Ingest node** | Предварительная обработка документов перед индексацией (pipelines) |
| **Coordinating node** | Маршрутизация запросов, агрегация результатов от data nodes |
| **ML node** | Выполнение задач машинного обучения (anomaly detection, k-NN search) |

Данные организованы в **индексы**, каждый из которых делится на **шарды** (primary + replica). Шарды распределяются по data nodes, обеспечивая горизонтальное масштабирование и отказоустойчивость.

### Основные возможности

**Полнотекстовый поиск** -- ядро OpenSearch. Движок поддерживает анализаторы текста (tokenizers, filters, char_filters), fuzzy-поиск, фразовый поиск, highlighting, suggestions и scoring на основе BM25. Можно настраивать собственные анализаторы для разных языков, включая русский (с morphological analysis через плагины).

**Агрегации** позволяют выполнять аналитику в реальном времени: bucket aggregations (terms, histogram, date_histogram, range), metric aggregations (avg, sum, cardinality, percentiles), pipeline aggregations. Это делает OpenSearch полноценным аналитическим движком для логов и метрик.

**Безопасность** встроена из коробки через Security plugin: аутентификация (LDAP, SAML, OpenID Connect), авторизация на уровне индексов/полей/документов, шифрование данных at rest и in transit, audit logging.

### Типичные сценарии использования

1. **Централизованный сбор логов** -- классический стек OpenSearch + Data Prepper (или Logstash/Fluentd/Fluent Bit) + OpenSearch Dashboards. Логи приложений, инфраструктуры и сети стекаются в OpenSearch, где по ним строятся дашборды и алерты.
2. **Полнотекстовый поиск по продуктам/контенту** -- поиск в интернет-магазине, базе знаний, CMS с поддержкой автодополнения, фасетов, релевантности.
3. **SIEM и security analytics** -- анализ событий безопасности, корреляция, обнаружение аномалий.
4. **Observability** -- трассировка (OpenTelemetry), метрики, логи в едином решении.

### Пример: создание индекса и поиск

```json
// Создание индекса с настройками анализатора
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "russian_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "russian_stop", "russian_stemmer"]
        }
      },
      "filter": {
        "russian_stop": { "type": "stop", "stopwords": "_russian_" },
        "russian_stemmer": { "type": "stemmer", "language": "russian" }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "russian_analyzer" },
      "price": { "type": "float" },
      "category": { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}

// Поиск с фильтрацией и агрегацией
POST /products/_search
{
  "query": {
    "bool": {
      "must": { "match": { "title": "ноутбук" } },
      "filter": { "range": { "price": { "gte": 30000, "lte": 100000 } } }
    }
  },
  "aggs": {
    "by_category": { "terms": { "field": "category" } },
    "avg_price": { "avg": { "field": "price" } }
  }
}
```

### OpenSearch vs Elasticsearch

| Критерий | OpenSearch | Elasticsearch |
|----------|-----------|---------------|
| Лицензия | Apache 2.0 | SSPL / Elastic License |
| Безопасность | Встроена (Security plugin) | Платная (X-Pack) или Basic |
| Alerting | Встроен | Платный (Watcher) |
| Облако | Amazon OpenSearch Service | Elastic Cloud |
| Совместимость | API совместим с ES 7.10, дальше расхождения | Собственное развитие API |
| ML | Anomaly detection, k-NN | ML features (платные) |

### Практические рекомендации

- **Sizing**: планируйте 1 шард на 10-50 ГБ данных. Слишком много мелких шардов создаёт overhead на cluster manager, слишком большие шарды замедляют recovery.
- **Index Lifecycle Management (ISM)**: настраивайте политики ротации индексов (hot -> warm -> cold -> delete) для эффективного управления хранилищем логов.
- **Не используйте OpenSearch как primary database** -- это поисковый движок, а не СУБД. Данные должны быть в основной БД (PostgreSQL, MySQL и т.д.), а OpenSearch выступает как вторичное хранилище для поиска.
- **Bulk API**: для массовой загрузки данных используйте `_bulk` endpoint вместо поштучной индексации -- это на порядок быстрее.
- **Мониторинг**: следите за метриками `cluster health`, `pending tasks`, `JVM heap usage`, `indexing rate`, `search latency`.

## Примеры
```http
GET /_cluster/health
```

## Доп. теория
- OpenSearch — проект под Apache 2.0, ориентированный на поисковые и аналитические задачи, а не на транзакционную целостность.
- Для продакшн‑поиска используйте асинхронную индексацию и переиндексацию через алиасы.
