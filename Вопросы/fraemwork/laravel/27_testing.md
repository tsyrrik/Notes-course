## Вопрос: Тестирование
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Тесты проверяют, что фичи и логика работают как нужно (feature/unit/HTTP).

## Ответ
```bash
php artisan make:test UserTest          # Feature
php artisan make:test UserServiceTest --unit
```
```php
class UserTest extends TestCase {
    use RefreshDatabase;
    public function test_user_can_be_created() {
        $response = $this->post('/api/users', [
            'name'=>'John','email'=>'john@example.com','password'=>'password123'
        ]);
        $response->assertStatus(201)->assertJsonPath('data.email','john@example.com');
        $this->assertDatabaseHas('users',['email'=>'john@example.com']);
    }
}

class UserServiceTest extends TestCase {
    public function test_full_name() {
        $user = new User(['first_name'=>'John','last_name'=>'Doe']);
        $this->assertEquals('John Doe', $user->getFullNameAttribute());
    }
}
```
