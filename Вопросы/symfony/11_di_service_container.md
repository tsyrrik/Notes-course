# DI контейнер и автоконфигурация

Простыми словами: все сервисы регистрируются в контейнере; автоконфигурация/автовайринг подставляет зависимости по типам автоматически.

```yaml
# config/services.yaml
services:
  _defaults:
    autowire: true
    autoconfigure: true
  App\:
    resource: '../src/*'
    exclude: '../src/{Entity,Tests,Kernel.php}'
```

Получение сервиса:
```php
public function __construct(private MailerInterface $mailer) {}
```
