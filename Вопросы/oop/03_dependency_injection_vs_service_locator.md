## Вопрос: Dependency Injection vs Service Locator

## Простой ответ
- DI явным образом передаёт зависимости в конструктор/метод, их видно и легко подменить.
- Service Locator прячет зависимости за глобальным контейнером, усложняет тесты и нарушает IoC.
```php
// DI
class UserService {
    public function __construct(private Mailer $mailer) {}
    public function register() { $this->mailer->send('Welcome!'); }
}
```

## Ответ
- DI: зависимости приходят извне (конструктор/сеттер). Класс не создает `new` внутри → проще тестировать и подменять.
- Service Locator: объект сам берет сервис из контейнера (`App::get()`), нарушает IoC и скрывает зависимости.
- Итог: DI предпочтителен; Service Locator — антипаттерн.
