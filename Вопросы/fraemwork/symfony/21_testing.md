# Тестирование

Простыми словами: PHPUnit для unit/feature; WebTestCase для запросов; Panther для браузерных тестов.

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
