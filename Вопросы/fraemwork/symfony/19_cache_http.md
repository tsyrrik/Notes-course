# Кэширование и HTTP cache

Простыми словами: компонент Cache (PSR-6/16) + HTTP cache заголовки для ответа; можно использовать встроенный reverse-proxy HttpCache или Varnish.

```php
// Cache компонент
$value = $cache->get('users', function(ItemInterface $item) {
    $item->expiresAfter(3600);
    return $this->userRepo->findActive();
});
```
HTTP cache в контроллере:
```php
$response = new Response($content);
$response->setPublic();
$response->setMaxAge(3600);
return $response;
```
