## Вопрос: DI контейнер и автоконфигурация
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
все сервисы регистрируются в контейнере; автоконфигурация/автовайринг подставляет зависимости по типам автоматически.

## Ответ
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
