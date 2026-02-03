## Вопрос: HTTP Client
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
встроенный клиент для запросов к внешним API с ретраями, таймаутами, пулами и удобным API.

## Ответ
```php
class ExternalApiService {
    public function __construct(private HttpClientInterface $client) {}
    public function fetch(): array {
        $resp = $this->client->request('GET', 'https://api.example.com/data');
        return $resp->toArray(); // выбросит исключение на 4xx/5xx
    }
}
```

- Параллельные запросы: `HttpClient::create()->stream(...)`.  
- Тесты: `MockHttpClient` или `HttpClientInterface` подмена.  
- Настройки: base_uri, headers, timeout, retries через `framework.http_client`.

## Примеры

1. `request('GET', 'https://api.example.com')` + `toArray()`.
2. Таймаут и заголовки через `framework.http_client`.
3. `MockHttpClient` в тестах.

## Доп. теория

1. `toArray()` кидает исключение на 4xx/5xx.
2. Для параллельных запросов используйте `stream()`.
