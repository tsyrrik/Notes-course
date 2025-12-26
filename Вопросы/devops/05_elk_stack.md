## Вопрос: ELK Stack

## Простой ответ
- Система «принял логи → разобрал → сложил → искал/рисовал».
- Elasticsearch хранит/ищет, Logstash/Beats собирают, Kibana ищет и показывает.

```bash
# Filebeat -> Logstash
filebeat.inputs:
  - type: filestream
    paths: [/var/log/app.log]

output.logstash:
  hosts: ["localhost:5044"]
```

```bash
# Пример поиска в Kibana/ES DSL
GET /logs/_search
{
  "query": { "match": { "level": "error" } }
}
```

## Ответ
- Elasticsearch — хранение и поиск; Logstash — сбор/парсинг/отправка; Kibana — визуализация и поиск.
- Используется для централизованного сбора логов, анализа, SIEM, DevOps/SRE мониторинга.
- Часто дополняют Beats (агенты), Fluentd/Graylog/OpenSearch — альтернативы.
