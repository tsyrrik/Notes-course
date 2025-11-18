# Dependency Injection vs Service Locator
- DI: зависимости приходят извне (конструктор/сеттер). Класс не создает `new` внутри → проще тестировать и подменять.
- Service Locator: объект сам берет сервис из контейнера (`App::get()`), нарушает IoC и скрывает зависимости.
- Итог: DI предпочтителен; Service Locator — антипаттерн.
```php
// DI
class UserService {
    public function __construct(private Mailer $mailer) {}
    public function register() { $this->mailer->send('Welcome!'); }
}
```
