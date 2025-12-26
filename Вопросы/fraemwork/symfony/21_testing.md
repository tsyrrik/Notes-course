## Вопрос: Тестирование
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

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
