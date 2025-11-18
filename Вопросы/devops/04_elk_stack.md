# ELK Stack
- Elasticsearch — хранение и поиск; Logstash — сбор/парсинг/отправка; Kibana — визуализация и поиск.
- Используется для централизованного сбора логов, анализа, SIEM, DevOps/SRE мониторинга.
- Часто дополняют Beats (агенты), Fluentd/Graylog/OpenSearch — альтернативы.

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
