## Вопрос: Кэширование и HTTP cache
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

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
