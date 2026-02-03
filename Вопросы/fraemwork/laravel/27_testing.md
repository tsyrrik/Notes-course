## Вопрос: Тестирование
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Тесты проверяют, что фичи и логика работают как нужно (feature/unit/HTTP).

## Ответ

Тестирование в Laravel построено на PHPUnit (или Pest в Laravel 11 по умолчанию) и предоставляет мощные инструменты для написания unit-, feature- и integration-тестов. Laravel разделяет тесты на две директории: `tests/Unit` (чистые юнит-тесты без загрузки фреймворка) и `tests/Feature` (интеграционные тесты с полным окружением Laravel, HTTP-запросами, базой данных).

Feature-тесты -- основной тип тестов в Laravel-приложениях. Они отправляют HTTP-запросы к вашему приложению и проверяют ответы, состояние БД, отправку email и другие побочные эффекты. Unit-тесты проверяют отдельные классы и методы в изоляции.

### Создание тестов

```bash
php artisan make:test UserRegistrationTest         # Feature (по умолчанию)
php artisan make:test Services/PriceCalculatorTest --unit   # Unit
```

### Feature-тест (HTTP)

```php
namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class UserRegistrationTest extends TestCase
{
    use RefreshDatabase;  // Пересоздаёт БД перед каждым тестом

    public function test_user_can_register(): void
    {
        $response = $this->postJson('/api/register', [
            'name'                  => 'John Doe',
            'email'                 => 'john@example.com',
            'password'              => 'password123',
            'password_confirmation' => 'password123',
        ]);

        $response
            ->assertStatus(201)
            ->assertJsonStructure(['data' => ['id', 'name', 'email']])
            ->assertJsonPath('data.email', 'john@example.com')
            ->assertJsonMissing(['password']);

        $this->assertDatabaseHas('users', [
            'email' => 'john@example.com',
        ]);
    }

    public function test_registration_requires_valid_email(): void
    {
        $response = $this->postJson('/api/register', [
            'name'     => 'John',
            'email'    => 'not-an-email',
            'password' => 'password123',
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['email']);
    }

    public function test_authenticated_user_can_view_profile(): void
    {
        $user = User::factory()->create();

        // Авторизация под пользователем
        $response = $this->actingAs($user, 'sanctum')
            ->getJson('/api/user');

        $response->assertOk()
            ->assertJsonPath('data.id', $user->id);
    }
}
```

### Unit-тест

```php
namespace Tests\Unit;

use App\Services\PriceCalculator;
use PHPUnit\Framework\TestCase;  // Без Laravel!

class PriceCalculatorTest extends TestCase
{
    public function test_discount_applied_correctly(): void
    {
        $calculator = new PriceCalculator();

        $this->assertEquals(90.0, $calculator->applyDiscount(100, 10));
        $this->assertEquals(100.0, $calculator->applyDiscount(100, 0));
    }

    public function test_negative_discount_throws_exception(): void
    {
        $this->expectException(\InvalidArgumentException::class);

        $calculator = new PriceCalculator();
        $calculator->applyDiscount(100, -5);
    }
}
```

### Mocking (подмена зависимостей)

```php
use App\Services\PaymentGateway;
use Mockery;

public function test_order_is_charged(): void
{
    // Создаём mock
    $gateway = Mockery::mock(PaymentGateway::class);
    $gateway->shouldReceive('charge')
        ->once()
        ->with(5000, Mockery::type('string'))
        ->andReturn(true);

    // Привязываем mock к контейнеру
    $this->app->instance(PaymentGateway::class, $gateway);

    $user = User::factory()->create();
    $this->actingAs($user)
        ->postJson('/api/orders', ['amount' => 5000])
        ->assertOk();
}
```

### Fake (подмена подсистем Laravel)

```php
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Notification;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Queue;

public function test_welcome_email_sent_on_registration(): void
{
    Mail::fake();  // Перехватываем отправку почты

    $this->postJson('/api/register', [
        'name' => 'John', 'email' => 'john@test.com',
        'password' => 'pass1234', 'password_confirmation' => 'pass1234',
    ]);

    // Проверяем, что письмо было "отправлено"
    Mail::assertSent(WelcomeMail::class, function ($mail) {
        return $mail->hasTo('john@test.com');
    });
}

public function test_event_dispatched(): void
{
    Event::fake([UserRegistered::class]);

    // ... действие ...

    Event::assertDispatched(UserRegistered::class, function ($event) {
        return $event->user->email === 'john@test.com';
    });
}

public function test_job_pushed_to_queue(): void
{
    Queue::fake();

    // ... действие ...

    Queue::assertPushed(SendEmailJob::class);
    Queue::assertPushedOn('emails', SendEmailJob::class);
}
```

### Тестирование базы данных

```php
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Foundation\Testing\DatabaseTransactions;

// RefreshDatabase -- мигрирует и откатывает БД (медленнее, надёжнее)
// DatabaseTransactions -- оборачивает тест в транзакцию (быстрее)

public function test_soft_delete(): void
{
    $user = User::factory()->create();

    $this->actingAs($user)->deleteJson("/api/users/{$user->id}");

    $this->assertSoftDeleted('users', ['id' => $user->id]);
    $this->assertDatabaseCount('users', 1); // запись ещё в БД
}
```

### Pest (альтернативный синтаксис, дефолт в Laravel 11)

```php
// tests/Feature/UserTest.php (Pest-стиль)
use App\Models\User;

it('can create a user', function () {
    $response = $this->postJson('/api/register', [
        'name' => 'John', 'email' => 'john@test.com',
        'password' => 'pass1234', 'password_confirmation' => 'pass1234',
    ]);

    $response->assertCreated();
    expect(User::count())->toBe(1);
    expect(User::first()->email)->toBe('john@test.com');
});

it('requires authentication for profile', function () {
    $this->getJson('/api/user')->assertUnauthorized();
});
```

### Запуск тестов

```bash
# Все тесты
php artisan test

# Конкретный файл
php artisan test --filter=UserRegistrationTest

# Конкретный метод
php artisan test --filter=test_user_can_register

# С покрытием
php artisan test --coverage --min=80

# Параллельный запуск (ускоряет)
php artisan test --parallel
```

### Практические советы

Используйте `RefreshDatabase` для feature-тестов и чистый `PHPUnit\Framework\TestCase` для юнит-тестов (без загрузки Laravel -- быстрее). Всегда тестируйте не только happy path, но и ошибки валидации, 403, 404. Factories создают реалистичные тестовые данные -- используйте states для разных сценариев. `actingAs($user)` позволяет тестировать от имени любого пользователя. Fake-фасады (`Mail::fake()`, `Queue::fake()`) -- мощный инструмент для проверки побочных эффектов без реального выполнения.

## Примеры

1. `php artisan test`.
2. `$this->getJson('/api/users')->assertOk()`.
3. `RefreshDatabase` для изоляции.

## Доп. теория

1. Тестовая БД должна быть изолирована.
2. Factories ускоряют подготовку данных.
