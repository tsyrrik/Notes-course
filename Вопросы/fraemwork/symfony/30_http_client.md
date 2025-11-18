# HTTP Client

Простыми словами: встроенный клиент для запросов к внешним API с ретраями, таймаутами, пулами и удобным API.

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
