## Вопрос: Тестирование
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
PHPUnit для unit/feature; WebTestCase для запросов; Panther для браузерных тестов.

## Ответ
```bash
php bin/console make:test UserControllerTest
```
```php
class UserControllerTest extends WebTestCase {
    public function testIndex(): void {
        $client = static::createClient();
        $client->request('GET', '/users');
        $this->assertResponseIsSuccessful();
        $this->assertSelectorTextContains('h1','Users');
    }
}
```

## Примеры

1. `WebTestCase` для проверки `GET /users`.
2. Unit‑тест сервиса с моками зависимостей.
3. Panther для E2E сценария логина.

## Доп. теория

1. Тестовая БД должна быть изолирована от прод‑данных.
2. Фикстуры ускоряют подготовку данных.
