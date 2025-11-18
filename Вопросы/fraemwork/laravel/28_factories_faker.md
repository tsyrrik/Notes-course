# Factory и Faker в тестах

Простыми словами: Factory+Faker генерирует тестовые данные и связанные модели для сценариев.
```bash
php artisan make:factory UserFactory --model=User
```
```php
class UserFactory extends Factory {
    protected $model = User::class;
    public function definition() {
        return [
            'name' => $this->faker->name(),
            'email' => $this->faker->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => bcrypt('password'),
            'remember_token' => Str::random(10),
        ];
    }
    public function admin() { return $this->state(fn() => ['role' => 'admin']); }
}

// Использование
$admin = User::factory()->admin()->create();
$user = User::factory()->create();
Post::factory(3)->for($user)->create();
```
