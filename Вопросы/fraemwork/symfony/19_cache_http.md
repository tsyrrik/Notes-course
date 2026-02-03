## Вопрос: Кэширование и HTTP cache
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
компонент Cache (PSR-6/16) + HTTP cache заголовки для ответа; можно использовать встроенный reverse-proxy HttpCache или Varnish.

## Ответ
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

## Примеры

1. `CacheInterface::get()` с `expiresAfter(3600)`.
2. HTTP‑кэш: `setMaxAge(3600)` и `setPublic()`.
3. Tag‑инвалидация через `TagAwareCacheInterface`.

## Доп. теория

1. PSR‑6/16 позволяет менять бекенд без изменения кода.
2. HTTP cache работает только при корректных заголовках.
